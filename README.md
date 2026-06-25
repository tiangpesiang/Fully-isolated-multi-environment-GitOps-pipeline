
# Fully-isolated-multi-environment-GitOps-pipeline
Fully isolated multi-environment GitOps pipeline. GitHub Actions to manage separate remote states for staging and production environments. Configure the setup to automatically apply policy validations, generate plans, and prevent any manual changes from mutating the environment


# What I Am Building
GitHub Repository
├── environments/
│   ├── staging/
│   └── production/
│
Every git push triggers:
  staging:    auto-plan → auto-apply
  production: auto-plan → manual approval → apply

┌─────────────────────────────────────────────────────┐
│                  COMPLETE FLOW                       │
│                                                      │
│  Developer                                           │
│     │ git push feature branch                        │
│     ▼                                                │
│  Pull Request                                        │
│     │ triggers                                       │
│     ▼                                                │
│  GitHub Actions                                      │
│  ├── tfsec scan (security)                           │
│  ├── checkov scan (compliance)                       │
│  ├── terraform validate                              │
│  ├── terraform plan → posted as PR comment           │
│  └── infracost (cost delta)                          │
│     │ PR approved + merged to main                   │
│     ▼                                                │
│  GitHub Actions                                      │
│  ├── Deploy to STAGING (auto)                        │
│  │   ├── terraform plan                              │
│  │   └── terraform apply                             │
│  └── Deploy to PRODUCTION (manual gate)              │
│      ├── Requires approval from senior engineer      │
│      ├── terraform plan                              │
│      └── terraform apply                             │
│                                                      │
│  Remote State:                                       │
│  ├── Azure Blob: tfstate-staging                     │
│  └── Azure Blob: tfstate-production                  │
│                                                      │
│  Drift Detection:                                    │
│  └── Scheduled job every 6 hours                     │
│      compares live Azure vs Terraform state          │
│      alerts if drift detected                        │
└─────────────────────────────────────────────────────┘

# Step 1: Create Remote State Storage
# Login to Azure
az login --use-device-code

# Set your subscription
az account set --subscription "<your-subscription-id>"

# Create dedicated resource group for Terraform state
az group create \
  --name RG-TerraformState \
  --location southeastasia

# Create storage account for state files
az storage account create \
  --name tfstate$(openssl rand -hex 4) \
  --resource-group RG-TerraformState \
  --location southeastasia \
  --sku Standard_LRS \
  --encryption-services blob \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

# Save the storage account name
STORAGE_ACCOUNT=$(az storage account list \
  --resource-group RG-TerraformState \
  --query "[0].name" -o tsv)

echo "Storage Account: $STORAGE_ACCOUNT"

# Create separate containers for each environment
az storage container create \
  --name tfstate-staging \
  --account-name $STORAGE_ACCOUNT

az storage container create \
  --name tfstate-production \
  --account-name $STORAGE_ACCOUNT

# Enable versioning (recover from accidental state corruption)
az storage account blob-service-properties update \
  --account-name $STORAGE_ACCOUNT \
  --resource-group RG-TerraformState \
  --enable-versioning true

echo "Remote state storage ready"

# Step 2: Create Service Principal For GitHub Actions
# Create Service Principal with Contributor role
SP_OUTPUT=$(az ad sp create-for-rbac \
  --name "sp-github-actions-terraform" \
  --role Contributor \
  --scopes /subscriptions/<your-subscription-id> \
  --sdk-auth)

echo $SP_OUTPUT
# SAVE THIS OUTPUT — you need it for GitHub Secrets

# Extract individual values
CLIENT_ID=$(echo $SP_OUTPUT | jq -r .clientId)
CLIENT_SECRET=$(echo $SP_OUTPUT | jq -r .clientSecret)
TENANT_ID=$(echo $SP_OUTPUT | jq -r .tenantId)
SUBSCRIPTION_ID=$(echo $SP_OUTPUT | jq -r .subscriptionId)

echo "Client ID: $CLIENT_ID"
echo "Tenant ID: $TENANT_ID"

# Also grant Storage Blob Data Contributor
# so pipeline can read/write Terraform state
az role assignment create \
  --assignee $CLIENT_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<your-subscription-id>/resourceGroups/RG-TerraformState

 # Step 3: Lock Down State Storage (Prevent Manual Changes)
# Enable resource lock on state storage
# Prevents accidental deletion
az lock create \
  --name "lock-tfstate-nodelete" \
  --resource-group RG-TerraformState \
  --lock-type CanNotDelete \
  --notes "Terraform state — do not delete"

# Enable soft delete on blob storage
az storage account blob-service-properties update \
  --account-name $STORAGE_ACCOUNT \
  --resource-group RG-TerraformState \
  --enable-delete-retention true \
  --delete-retention-days 30

# Step 4: Create The Repository Structure
mkdir ~/gitops-pipeline
cd ~/gitops-pipeline
git init

# Create full directory structure
mkdir -p .github/workflows
mkdir -p environments/staging
mkdir -p environments/production
mkdir -p modules/networking
mkdir -p modules/compute
mkdir -p modules/security
mkdir -p policies/tfsec
mkdir -p policies/checkov
mkdir -p scripts

# Create all files
touch .github/workflows/pr-validation.yml
touch .github/workflows/deploy-staging.yml
touch .github/workflows/deploy-production.yml
touch .github/workflows/drift-detection.yml
touch environments/staging/main.tf
touch environments/staging/variables.tf
touch environments/staging/outputs.tf
touch environments/staging/backend.tf
touch environments/staging/terraform.tfvars
touch environments/production/main.tf
touch environments/production/variables.tf
touch environments/production/outputs.tf
touch environments/production/backend.tf
touch environments/production/terraform.tfvars
touch .gitignore
touch .tfsec.yml
touch .checkov.yml

echo "Structure created"
tree .


#Step 5: Create .gitignore
nano .gitignore

# Terraform
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.backup
*.tfplan
crash.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Sensitive
*.pem
*.key
.env
secrets.tfvars
*secrets*

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/

# Step 6: Staging Backend Configuration
nano environments/staging/backend.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }

  # REMOTE STATE — staging environment
  # Completely isolated from production state
  backend "azurerm" {
    resource_group_name  = "RG-TerraformState"
    storage_account_name = "tfstate<your-suffix>"
    container_name       = "tfstate-staging"
    key                  = "staging/terraform.tfstate"
    use_oidc             = true   # Workload Identity Federation
  }
}

provider "azurerm" {
  features {}
  use_oidc = true
}

# Step 7: Production Backend Configuration
nano environments/production/backend.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }

  # REMOTE STATE — production environment
  # Completely isolated from staging state
  backend "azurerm" {
    resource_group_name  = "RG-TerraformState"
    storage_account_name = "tfstate<your-suffix>"
    container_name       = "tfstate-production"
    key                  = "production/terraform.tfstate"
    use_oidc             = true
  }
}

provider "azurerm" {
  features {}
  use_oidc = true
}

# Step 8: Staging Variables
nano environments/staging/variables.tf

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "staging"
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "southeastasia"
}

variable "resource_group_name" {
  description = "Resource group name"
  type        = string
  default     = "RG-WebServer-Staging"
}

variable "vm_size" {
  description = "VM size"
  type        = string
  default     = "Standard_B2s"
}

variable "vm_count" {
  description = "Number of VMs"
  type        = number
  default     = 1   # Staging = 1 VM (cost saving)
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default = {
    Environment = "staging"
    ManagedBy   = "Terraform"
    GitOps      = "true"
    Owner       = "PeSiang"
  }
}

# Step 9: Production Variables
nano environments/production/variables.tf

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "southeastasia"
}

variable "resource_group_name" {
  description = "Resource group name"
  type        = string
  default     = "RG-WebServer-Production"
}

variable "vm_size" {
  description = "VM size"
  type        = string
  default     = "Standard_B2s"
}

variable "vm_count" {
  description = "Number of VMs"
  type        = number
  default     = 2   # Production = 2 VMs (HA)
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default = {
    Environment = "production"
    ManagedBy   = "Terraform"
    GitOps      = "true"
    Owner       = "PeSiang"
    CostCenter  = "PROD-001"
  }
}

# Step 10: Main Infrastructure (Shared Pattern)
nano environments/staging/main.tf

# ============================================
# RESOURCE GROUP
# ============================================
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
  tags     = var.tags
}

# ============================================
# VIRTUAL NETWORK
# ============================================
resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.environment}"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  tags                = var.tags
}

resource "azurerm_subnet" "main" {
  name                 = "subnet-${var.environment}"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

# ============================================
# NETWORK SECURITY GROUP
# ============================================
resource "azurerm_network_security_group" "main" {
  name                = "nsg-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "Allow-HTTP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow-SSH"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = var.tags
}

resource "azurerm_subnet_network_security_group_association" "main" {
  subnet_id                 = azurerm_subnet.main.id
  network_security_group_id = azurerm_network_security_group.main.id
}

# ============================================
# PUBLIC IP
# ============================================
resource "azurerm_public_ip" "main" {
  name                = "pip-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
  tags                = var.tags
}

# ============================================
# NETWORK INTERFACE
# ============================================
resource "azurerm_network_interface" "main" {
  name                = "nic-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.main.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.main.id
  }

  tags = var.tags
}

# ============================================
# LINUX VM WITH NGINX
# ============================================
resource "azurerm_linux_virtual_machine" "main" {
  name                = "vm-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = var.vm_size
  admin_username      = "azureuser"

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  network_interface_ids = [
    azurerm_network_interface.main.id
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  custom_data = base64encode(templatefile(
    "${path.module}/../../scripts/install-nginx.sh",
    { environment = var.environment }
  ))

  tags = var.tags
}

# Copy to production (same structure, different tfvars):
cp environments/staging/main.tf environments/production/main.tf


# Step 11: tfsec Configuration
nano .tfsec.yml

# tfsec custom checks configuration
minimum_severity: HIGH

exclude:
  # Exclude checks not relevant to learning environment
  - AVD-AZU-0037   # Storage account HTTPS (we handle this)

custom_checks:
  - code: CUS001
    description: "All resources must have environment tag"
    impact: "Resources without tags cannot be tracked"
    resolution: "Add environment tag to all resources"
    checks:
      - code: >
          deny[msg] {
            resource := input.document[_].resource[_][_]
            not resource.tags.Environment
            msg := "Resource missing Environment tag"
          }

# Step 12: Checkov Configuration
nano .checkov.yml

# Checkov policy configuration
framework:
  - terraform

check:
  # Azure Security checks — all mandatory
  - CKV_AZURE_1    # Ensure Azure Instance does not use basic auth
  - CKV_AZURE_2    # Ensure Azure managed disk is encrypted
  - CKV_AZURE_3    # Ensure storage account is using HTTPS
  - CKV_AZURE_35   # Ensure storage account has deny default action
  - CKV_AZURE_36   # Ensure storage account uses TLS 1.2

skip_check:
  # Skip checks not applicable to learning setup
  - CKV_AZURE_50   # Skip VM extension publisher check

soft_fail: false    # HARD FAIL — pipeline stops on violation

# Step 13: PR Validation Workflow
nano .github/workflows/pr-validation.yml

# ============================================
# PR VALIDATION PIPELINE
# Runs on every Pull Request
# Must pass before PR can be merged
# ============================================
name: PR Validation

on:
  pull_request:
    branches:
      - main
    paths:
      - 'environments/**'
      - 'modules/**'
      - '**.tf'

permissions:
  contents: read
  pull-requests: write
  id-token: write   # Required for OIDC/Workload Identity

env:
  ARM_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
  ARM_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_USE_OIDC:        true

jobs:

  # ==========================================
  # JOB 1: SECURITY SCAN WITH TFSEC
  # ==========================================
  security-scan:
    name: Security Scan (tfsec)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          soft_fail: false
          github_token: ${{ secrets.GITHUB_TOKEN }}
          working_directory: environments/staging

  # ==========================================
  # JOB 2: COMPLIANCE SCAN WITH CHECKOV
  # ==========================================
  compliance-scan:
    name: Compliance Scan (Checkov)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: environments/
          framework: terraform
          soft_fail: false
          output_format: github_failed_only

  # ==========================================
  # JOB 3: TERRAFORM VALIDATE
  # ==========================================
  terraform-validate:
    name: Terraform Validate
    runs-on: ubuntu-latest
    needs: [security-scan, compliance-scan]
    strategy:
      matrix:
        environment: [staging, production]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~1.5"

      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id:       ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init

      - name: Terraform Validate
        working-directory: environments/${{ matrix.environment }}
        run: terraform validate

      - name: Terraform Format Check
        working-directory: environments/${{ matrix.environment }}
        run: terraform fmt -check -recursive

  # ==========================================
  # JOB 4: TERRAFORM PLAN + POST AS PR COMMENT
  # ==========================================
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    needs: [terraform-validate]
    strategy:
      matrix:
        environment: [staging, production]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~1.5"

      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id:       ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init

      - name: Terraform Plan
        id: plan
        working-directory: environments/${{ matrix.environment }}
        run: |
          terraform plan \
            -no-color \
            -out=tfplan-${{ matrix.environment }} \
            2>&1 | tee plan-output.txt

      - name: Post Plan To PR Comment
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync(
              'environments/${{ matrix.environment }}/plan-output.txt',
              'utf8'
            );
            const truncated = plan.length > 60000
              ? plan.substring(0, 60000) + '\n... [truncated]'
              : plan;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan — \`${{ matrix.environment }}\`
              \`\`\`hcl
              ${truncated}
              \`\`\`
              `
            });

  # ==========================================
  # JOB 5: COST ESTIMATION WITH INFRACOST
  # ==========================================
  cost-estimation:
    name: Cost Estimation
    runs-on: ubuntu-latest
    needs: [terraform-validate]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Infracost
        uses: infracost/actions/setup@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Generate Infracost diff
        run: |
          infracost diff \
            --path environments/staging \
            --format json \
            --out-file /tmp/infracost.json

      - name: Post cost to PR
        uses: infracost/actions/comment@v1
        with:
          path: /tmp/infracost.json
          behavior: update

# Step 14: Deploy Staging Workflow
nano .github/workflows/deploy-staging.yml

# ============================================
# STAGING DEPLOYMENT
# Auto-deploys when PR merges to main
# No manual approval required
# ============================================
name: Deploy Staging

on:
  push:
    branches:
      - main
    paths:
      - 'environments/staging/**'
      - 'modules/**'

permissions:
  contents: read
  id-token: write

env:
  ARM_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
  ARM_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_USE_OIDC:        true

jobs:
  deploy-staging:
    name: Deploy To Staging
    runs-on: ubuntu-latest
    environment: staging   # Links to GitHub Environment

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~1.5"

      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id:       ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        working-directory: environments/staging
        run: terraform init

      - name: Terraform Plan
        working-directory: environments/staging
        run: terraform plan -out=tfplan-staging

      - name: Terraform Apply
        working-directory: environments/staging
        run: terraform apply -auto-approve tfplan-staging

      - name: Verify Deployment
        working-directory: environments/staging
        run: |
          STAGING_IP=$(terraform output -raw public_ip_address)
          echo "Staging IP: $STAGING_IP"

          # Wait for Nginx to start
          sleep 120

          # Health check
          HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            http://$STAGING_IP)

          if [ $HTTP_STATUS -eq 200 ]; then
            echo "✅ Staging deployment verified - HTTP $HTTP_STATUS"
          else
            echo "❌ Health check failed - HTTP $HTTP_STATUS"
            exit 1
          fi

      - name: Notify Teams On Success
        if: success()
        run: |
          curl -X POST "${{ secrets.TEAMS_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d '{
              "text": "✅ Staging deployment successful",
              "themeColor": "00FF00"
            }'

      - name: Notify Teams On Failure
        if: failure()
        run: |
          curl -X POST "${{ secrets.TEAMS_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d '{
              "text": "❌ Staging deployment FAILED",
              "themeColor": "FF0000"
            }'

# Step 15: Deploy Production Workflow
nano .github/workflows/deploy-production.yml

# ============================================
# PRODUCTION DEPLOYMENT
# Requires manual approval from senior engineer
# Only runs after staging succeeds
# ============================================
name: Deploy Production

on:
  workflow_run:
    workflows: ["Deploy Staging"]
    types:
      - completed
    branches:
      - main

permissions:
  contents: read
  id-token: write

env:
  ARM_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
  ARM_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_USE_OIDC:        true

jobs:
  # ==========================================
  # GATE: Only proceed if staging succeeded
  # ==========================================
  check-staging:
    name: Verify Staging Succeeded
    runs-on: ubuntu-latest
    if: >
      github.event.workflow_run.conclusion == 'success'
    steps:
      - name: Staging check passed
        run: echo "Staging succeeded — proceeding to production"

  # ==========================================
  # PRODUCTION PLAN (no apply yet)
  # ==========================================
  production-plan:
    name: Production Plan
    runs-on: ubuntu-latest
    needs: check-staging
    environment: production-plan   # No approval needed for plan

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~1.5"

      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id:       ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        working-directory: environments/production
        run: terraform init

      - name: Terraform Plan
        working-directory: environments/production
        run: |
          terraform plan \
            -no-color \
            -out=tfplan-production \
            2>&1 | tee plan-output.txt
          cat plan-output.txt

      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-production
          path: environments/production/tfplan-production
          retention-days: 1

  # ==========================================
  # MANUAL APPROVAL GATE
  # Senior engineer must approve in GitHub UI
  # ==========================================
  production-approval:
    name: "⏸ Awaiting Production Approval"
    runs-on: ubuntu-latest
    needs: production-plan
    environment: production   # ← This environment has required reviewers
    # Configure in GitHub:
    # Settings → Environments → production
    # → Required reviewers: add senior engineers
    # → Wait timer: 0 minutes (or set delay)

    steps:
      - name: Approval received
        run: echo "Production deployment approved — proceeding"

  # ==========================================
  # PRODUCTION APPLY
  # Only runs after human approval
  # ==========================================
  production-deploy:
    name: Deploy To Production
    runs-on: ubuntu-latest
    needs: production-approval

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~1.5"

      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id:       ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Download Plan Artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan-production
          path: environments/production/

      - name: Terraform Init
        working-directory: environments/production
        run: terraform init

      - name: Terraform Apply
        working-directory: environments/production
        run: terraform apply -auto-approve tfplan-production

      - name: Production Health Check
        working-directory: environments/production
        run: |
          PROD_IP=$(terraform output -raw public_ip_address)
          echo "Production IP: $PROD_IP"
          sleep 120
          HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            http://$PROD_IP)
          if [ $HTTP_STATUS -eq 200 ]; then
            echo "✅ Production verified - HTTP $HTTP_STATUS"
          else
            echo "❌ Production health check FAILED"
            exit 1
          fi

      - name: Notify Production Success
        if: always()
        run: |
          STATUS="${{ job.status }}"
          COLOR=$([ "$STATUS" = "success" ] && echo "00FF00" || echo "FF0000")
          EMOJI=$([ "$STATUS" = "success" ] && echo "✅" || echo "❌")
          curl -X POST "${{ secrets.TEAMS_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d "{
              \"text\": \"$EMOJI Production deployment $STATUS\",
              \"themeColor\": \"$COLOR\"
            }"
            
# Step 16: Drift Detection Workflow
nano .github/workflows/drift-detection.yml

# ============================================
# DRIFT DETECTION
# Runs every 6 hours
# Detects if someone manually changed Azure
# outside of Terraform (portal, CLI, etc.)
# ============================================
name: Drift Detection

on:
  schedule:
    - cron: '0 */6 * * *'   # Every 6 hours
  workflow_dispatch:          # Manual trigger too

permissions:
  contents: read
  issues: write
  id-token: write

env:
  ARM_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
  ARM_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_USE_OIDC:        true

jobs:
  detect-drift:
    name: Detect Infrastructure Drift
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [staging, production]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~1.5"

      - name: Azure Login (OIDC)
        uses: azure/login@v1
        with:
          client-id:       ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init

      - name: Check For Drift
        id: drift
        working-directory: environments/${{ matrix.environment }}
        run: |
          # Run plan — if output contains changes, drift detected
          PLAN_OUTPUT=$(terraform plan \
            -detailed-exitcode \
            -no-color 2>&1)

          EXIT_CODE=$?

          echo "plan_output<<EOF" >> $GITHUB_OUTPUT
          echo "$PLAN_OUTPUT" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

          if [ $EXIT_CODE -eq 0 ]; then
            echo "drift_detected=false" >> $GITHUB_OUTPUT
            echo "✅ No drift detected in ${{ matrix.environment }}"
          elif [ $EXIT_CODE -eq 2 ]; then
            echo "drift_detected=true" >> $GITHUB_OUTPUT
            echo "⚠️ DRIFT DETECTED in ${{ matrix.environment }}"
          else
            echo "drift_detected=error" >> $GITHUB_OUTPUT
            echo "❌ Error during drift check"
            exit 1
          fi

      - name: Create GitHub Issue On Drift
        if: steps.drift.outputs.drift_detected == 'true'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const plan = `${{ steps.drift.outputs.plan_output }}`;
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `⚠️ Infrastructure Drift Detected — ${{ matrix.environment }}`,
              body: `## Drift Detected in \`${{ matrix.environment }}\`

              Someone made manual changes to Azure outside of Terraform.

              **Action Required:**
              1. Review the changes below
              2. Either update Terraform to match (if intentional)
              3. Or run \`terraform apply\` to revert (if accidental)

              **Detected Changes:**
              \`\`\`
              ${plan.substring(0, 60000)}
              \`\`\`

              **Time:** ${new Date().toISOString()}
              `,
              labels: ['drift', 'infrastructure', '${{ matrix.environment }}']
            });

      - name: Alert Teams On Drift
        if: steps.drift.outputs.drift_detected == 'true'
        run: |
          curl -X POST "${{ secrets.TEAMS_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d '{
              "text": "⚠️ DRIFT DETECTED in ${{ matrix.environment }} — check GitHub Issues",
              "themeColor": "FFA500"
            }'

  # Step 17: Configure GitHub Secrets
  In GitHub Repository:
Settings → Secrets and variables → Actions → New secret

Add these secrets:

AZURE_CLIENT_ID       = <service principal client ID>
AZURE_TENANT_ID       = <your tenant ID>
AZURE_SUBSCRIPTION_ID = <your subscription ID>
INFRACOST_API_KEY     = <from infracost.io free account>
TEAMS_WEBHOOK_URL     = <from Teams channel connector>

# Step 18: Configure GitHub Environments
Settings → Environments → New environment

1. Create: staging
   → No protection rules (auto-deploy)
   → Add secret: (none needed, uses repo secrets)

2. Create: production
   → Required reviewers: add yourself
   → Wait timer: 0 minutes
   → Deployment branches: main only

3. Create: production-plan
   → No protection rules (plan only, safe)


# Step 19: Configure Branch Protection
Settings → Branches → Add rule

Branch name pattern: main

Rules:
  ✓ Require pull request before merging
  ✓ Require approvals: 1
  ✓ Require status checks to pass:
      - Security Scan (tfsec)
      - Compliance Scan (Checkov)
      - Terraform Validate (staging)
      - Terraform Validate (production)
      - Terraform Plan (staging)
      - Terraform Plan (production)
  ✓ Require branches to be up to date
  ✓ Do not allow bypassing above settings
  ✓ Restrict who can push to matching branches

  # Step 20: Push And Test
cd ~/gitops-pipeline

# Add all files
git add .

# Initial commit
git commit -m "feat: initial GitOps pipeline setup

- Multi-environment Terraform (staging + production)
- Isolated remote state per environment
- PR validation with tfsec + checkov + cost estimation
- Auto-deploy to staging on merge
- Manual approval gate for production
- Drift detection every 6 hours"

# Create repo on GitHub first, then:
git remote add origin https://github.com/<you>/gitops-pipeline.git
git branch -M main
git push -u origin main


# The Complete Flow In Practice
DAY-TO-DAY DEVELOPER WORKFLOW:

1. git checkout -b feature/add-redis
2. Make Terraform changes
3. git push origin feature/add-redis
4. Open Pull Request → GitHub Actions fires:
   ├── tfsec:    ✅ passed
   ├── checkov:  ✅ passed
   ├── validate: ✅ passed
   ├── plan:     Posted as PR comment (you review it)
   └── cost:     +$45/month posted as PR comment
5. Senior engineer reviews + approves PR
6. Merge to main →
   └── Staging deploys automatically (2 min)
       └── Health check passes ✅
           └── Production plan generated
               └── Senior engineer approves in GitHub UI
                   └── Production deploys (2 min)
                       └── Health check passes ✅
                           └── Teams notification sent

EVERY 6 HOURS:
  Drift detection runs →
  If someone used portal to change VM size →
  GitHub Issue created automatically →
  Teams alert sent →
  Engineer investigates →
  Either fix Terraform or revert Azure

# Summary
✓ Environment isolation: staging and production
  never share state
✓ Policy as code: security enforced at commit time
✓ Zero manual changes: portal = read only
✓ Drift detection: catches violations automatically
✓ Cost governance: every PR shows cost impact
✓ Immutable infrastructure: destroy and recreate
  rather than patch in place
✓ OIDC authentication: zero static secrets
  in the pipeline





