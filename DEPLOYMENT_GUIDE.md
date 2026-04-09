# EpicBook Azure DevOps Deployment Guide
## Step-by-Step for Complete Beginners

---

## What You Are Building

You will deploy the EpicBook app on Azure using two pipelines:

1. **Infra Pipeline** (in `infra-epicbook` repo): Uses Terraform to create all Azure resources.
2. **App Pipeline** (in `theepicbook` repo): Uses Ansible to configure VMs and deploy the app.

The final result: EpicBook running on Azure, accessible via a public IP in your browser.

---

## Prerequisites Checklist

Before you start, make sure you have:

- [ ] An active Azure subscription
- [ ] An Azure DevOps organization and project
- [ ] An App Registration in Azure AD (gives you Client ID, Client Secret, Tenant ID)
- [ ] Your Subscription ID
- [ ] Two repos ready: `infra-epicbook` and `theepicbook` (GitHub or Azure Repos)
- [ ] Your SSH key pair at `~/.ssh/id_ed25519` (private) and `~/.ssh/id_ed25519.pub` (public)

---

## PHASE 1: Azure Setup (Do This First)

### Step 1.1 — Create a Storage Account for Terraform State

Terraform needs a place to store its state file. You create this once manually.

1. Go to the Azure Portal: https://portal.azure.com
2. Search for **Resource Groups** and click **Create**
   - Name: `epicbook-tfstate-rg`
   - Region: West Europe (or your preferred region)
   - Click **Review + Create**, then **Create**

3. Search for **Storage Accounts** and click **Create**
   - Resource group: `epicbook-tfstate-rg`
   - Storage account name: `epicbooktfstate` (must be globally unique, all lowercase, no dashes)
   - Region: Same as above
   - Performance: Standard
   - Redundancy: LRS
   - Click **Review + Create**, then **Create**

4. Once created, go into the storage account. On the left menu click **Containers**
5. Click **+ Container**
   - Name: `tfstate`
   - Access level: Private
   - Click **Create**

You are done with this step. Terraform will use this to store its state.

---

### Step 1.2 — Get Your Azure Credentials

You need four values. Here is where to find them:

**Tenant ID and Subscription ID:**
1. Go to Azure Portal
2. Search for **Subscriptions**
3. Click your subscription
4. Copy the **Subscription ID** (save it)
5. Click **Overview** on the left. Copy the **Directory (tenant) ID** (save it)

**Client ID and Client Secret (App Registration):**
1. Search for **Azure Active Directory** (or **Microsoft Entra ID**)
2. Click **App registrations** on the left
3. Click **+ New registration**
   - Name: `epicbook-terraform-sp`
   - Supported account types: Accounts in this organizational directory only
   - Click **Register**
4. On the app page, copy the **Application (client) ID** (save it)
5. On the left menu click **Certificates & secrets**
6. Click **+ New client secret**
   - Description: `epicbook-secret`
   - Expires: 24 months
   - Click **Add**
7. Copy the **Value** immediately (it disappears after you leave the page). Save it as your Client Secret.

**Give the App Registration permission to manage Azure resources:**
1. Search for **Subscriptions**
2. Click your subscription
3. Click **Access control (IAM)** on the left
4. Click **+ Add** then **Add role assignment**
5. Role: **Contributor**
6. Click **Next**, then click **+ Select members**
7. Search for `epicbook-terraform-sp` and select it
8. Click **Review + assign**

---

## PHASE 2: Azure DevOps Setup

### Step 2.1 — Create a Service Connection

This lets Azure DevOps (and Terraform) talk to your Azure subscription.

1. Go to your Azure DevOps project
2. Click **Project Settings** (bottom left corner)
3. Click **Service connections**
4. Click **+ New service connection**
5. Select **Azure Resource Manager**
6. Select **Service principal (manual)**
7. Fill in:
   - Environment: Azure Cloud
   - Scope level: Subscription
   - Subscription ID: (paste yours)
   - Subscription Name: (your subscription name)
   - Service Principal Id: (this is your Client ID)
   - Service Principal key: (this is your Client Secret)
   - Tenant ID: (paste yours)
8. Click **Verify** to confirm it works
9. Name it exactly: `epicbook-azure-connection`
10. Check **Grant access permission to all pipelines**
11. Click **Save**

---

### Step 2.2 — Upload Your SSH Private Key as a Secure File

1. In Azure DevOps, go to **Pipelines** > **Library**
2. Click **Secure files** tab
3. Click **+ Secure file**
4. Upload the file at `~/.ssh/id_ed25519` (your private key)
5. Once uploaded, click on the file name
6. Check **Authorize for use in all pipelines**
7. Click **Save**

The file must be named exactly `id_ed25519` in Azure DevOps Secure Files.

---

### Step 2.3 — Create Variable Groups

**Variable Group 1: `epicbook-infra-vars`** (for the Infra pipeline)

1. Go to **Pipelines** > **Library**
2. Click **+ Variable group**
3. Name it: `epicbook-infra-vars`
4. Add these variables (click **+ Add** for each):

   | Variable Name       | Value                        | Secret? |
   |---------------------|------------------------------|---------|
   | ARM_SUBSCRIPTION_ID | your subscription ID         | No      |
   | ARM_CLIENT_ID       | your client ID               | No      |
   | ARM_CLIENT_SECRET   | your client secret           | YES     |
   | ARM_TENANT_ID       | your tenant ID               | No      |
   | SSH_PUBLIC_KEY      | content of id_ed25519.pub    | No      |
   | MYSQL_ADMIN_PASSWORD| a strong password you choose | YES     |

   To get SSH_PUBLIC_KEY: open a terminal and run `cat ~/.ssh/id_ed25519.pub`. Copy the entire output.

5. Click **Save**

**Variable Group 2: `epicbook-app-vars`** (for the App pipeline)

1. Click **+ Variable group** again
2. Name it: `epicbook-app-vars`
3. Add these variables:

   | Variable Name       | Value                              | Secret? |
   |---------------------|------------------------------------|---------|
   | APP_PUBLIC_IP       | (fill this AFTER infra pipeline runs) | No  |
   | BACKEND_PRIVATE_IP  | (fill this AFTER infra pipeline runs) | No  |
   | MYSQL_FQDN          | (fill this AFTER infra pipeline runs) | No  |
   | MYSQL_ADMIN_PASSWORD| same password as above             | YES     |

   Leave the IP and FQDN values blank for now. You will fill them after running the infra pipeline.

4. Click **Save**

---

## PHASE 3: Push Your Code to Repos

### Step 3.1 — Push Infra Repo

On your local machine:

```bash
cd infra-epicbook

# Copy the example vars file and fill in your real values
cp terraform.tfvars.example terraform.tfvars
nano terraform.tfvars   # fill in your real values

# Initialize git and push
git init
git add .
git commit -m "Initial infra setup"
git remote add origin https://github.com/YOUR_USERNAME/infra-epicbook.git
git push -u origin main
```

**Important:** `terraform.tfvars` is in `.gitignore` so it will NOT be pushed. Good. That file has secrets.

### Step 3.2 — Push App Repo

```bash
cd theepicbook

git init
git add .
git commit -m "Initial app and ansible setup"
git remote add origin https://github.com/YOUR_USERNAME/theepicbook.git
git push -u origin main
```

---

## PHASE 4: Create and Run the Infra Pipeline

### Step 4.1 — Create the Infra Pipeline

1. In Azure DevOps, go to **Pipelines** > **Pipelines**
2. Click **+ New pipeline**
3. Select where your code is (GitHub or Azure Repos)
4. Select the `infra-epicbook` repository
5. On the configure step, select **Existing Azure Pipelines YAML file**
6. Branch: `main`. Path: `/azure-pipelines.yml`
7. Click **Continue**
8. Click **Run** (this starts the pipeline)

### Step 4.2 — Watch the Pipeline Run

The pipeline runs in three stages:
- **Validate**: Checks your Terraform syntax is correct
- **Plan**: Shows what Azure resources will be created (no changes yet)
- **Apply**: Actually creates the resources in Azure

The Apply stage requires an environment approval the first time. When it pauses and says **Waiting for approval**, click the notification and click **Approve**.

The full pipeline takes 5 to 10 minutes.

### Step 4.3 — Get the Terraform Outputs

When the pipeline finishes successfully:

1. Click on the completed pipeline run
2. Click on the **Apply** stage
3. Look for the step called **Extract and Publish Terraform Outputs**
4. In the logs you will see three lines like:
   ```
   Frontend IP: 20.X.X.X
   Backend IP: 10.0.2.X
   MySQL FQDN: epicbook-mysql-srv.mysql.database.azure.com
   ```
5. Copy all three values. You need them for the next phase.

You can also go to **Artifacts** on the pipeline run page and download `terraform_outputs.env` which has all three values.

---

## PHASE 5: Update App Variable Group and Run App Pipeline

### Step 5.1 — Fill in the IP Values

1. Go to **Pipelines** > **Library**
2. Click on `epicbook-app-vars`
3. Update these three variables with the values from the Terraform outputs:
   - `APP_PUBLIC_IP`: the Frontend IP
   - `BACKEND_PRIVATE_IP`: the Backend IP
   - `MYSQL_FQDN`: the MySQL FQDN
4. Click **Save**

### Step 5.2 — Create the App Pipeline

1. Go to **Pipelines** > **Pipelines**
2. Click **+ New pipeline**
3. Select your `theepicbook` repository
4. Select **Existing Azure Pipelines YAML file**
5. Branch: `main`. Path: `/azure-pipelines.yml`
6. Click **Continue**
7. Click **Run**

The App pipeline will:
1. Install Ansible on the build agent
2. Download your SSH key from Secure Files
3. Inject the IPs and MySQL FQDN into the Ansible inventory and variables
4. SSH into both VMs and configure them
5. Clone and deploy the EpicBook app
6. Start the app with PM2
7. Configure Nginx on the frontend
8. Run a health check

This takes 10 to 20 minutes depending on your VM size.

---

## PHASE 6: Verify It Works

1. After the App pipeline succeeds, open your browser
2. Go to: `http://YOUR_FRONTEND_IP` (the APP_PUBLIC_IP value)
3. You should see the EpicBook application

To also verify the backend and database:
- Go to `http://YOUR_FRONTEND_IP/api/` and check for a JSON response
- If the books are loading in the UI, the database connection is working

---

## Troubleshooting Common Problems

### Problem: Terraform Apply fails with "InvalidTemplateDeployment"
- Your VM size (`Standard_B2s`) may not be available in `westeurope`.
- Fix: In `variables.tf` change the default to `Standard_B1s` or try a different region.

### Problem: Ansible cannot connect to the VM ("SSH timeout")
- The VMs may still be booting after Terraform finishes.
- Fix: In the App pipeline, add a manual sleep step of 60 seconds before the ping step.

### Problem: Ansible ping works but playbook fails at npm install
- The VM ran out of disk space or memory.
- Fix: Increase VM size to `Standard_B2ms` in `variables.tf` and rerun the infra pipeline.

### Problem: Nginx returns 502 Bad Gateway
- The backend app is not running yet or crashed.
- Fix: SSH into the backend VM and run `pm2 logs epicbook-backend` to see the error.
- Common cause: Wrong database credentials or MySQL not yet accepting connections.

### Problem: App pipeline says "MYSQL_FQDN_PLACEHOLDER not replaced"
- You forgot to update the variable group `epicbook-app-vars`.
- Fix: Go back to the Library, update the three IP/FQDN variables, and rerun the pipeline.

### Problem: MySQL connection refused
- Azure MySQL Flexible Server takes 3 to 5 minutes to be fully ready after Terraform apply.
- Fix: Rerun just the App pipeline after waiting a few minutes.

### Problem: "Permission denied (publickey)" during Ansible
- The SSH key in Secure Files does not match the public key used during VM provisioning.
- Fix: Make sure `SSH_PUBLIC_KEY` in the infra variable group matches your `id_ed25519.pub`.

---

## Clean Up (When You Are Done)

To delete everything and stop paying for Azure resources:

```bash
# In the infra-epicbook directory
terraform destroy -auto-approve
```

Or in Azure DevOps, add a destroy stage to the infra pipeline and run it manually.

---

## Quick Reference: All Variable Names

**Variable Group: epicbook-infra-vars**
- `ARM_SUBSCRIPTION_ID`
- `ARM_CLIENT_ID`
- `ARM_CLIENT_SECRET` (secret)
- `ARM_TENANT_ID`
- `SSH_PUBLIC_KEY`
- `MYSQL_ADMIN_PASSWORD` (secret)

**Variable Group: epicbook-app-vars**
- `APP_PUBLIC_IP`
- `BACKEND_PRIVATE_IP`
- `MYSQL_FQDN`
- `MYSQL_ADMIN_PASSWORD` (secret)

**Secure File:** `id_ed25519`

**Service Connection:** `epicbook-azure-connection`
