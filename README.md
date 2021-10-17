# Managing infrastructure as code with Terraform, Cloud Build, and GitOps

This is the repo for the [Managing infrastructure as code with Terraform, Cloud Build, and GitOps](https://cloud.google.com/solutions/managing-infrastructure-as-code) tutorial. This tutorial explains how to manage infrastructure as code with Terraform and Cloud Build using the popular GitOps methodology.

## Overview

In this tutorial, you use a single Git repository to define your cloud infrastructure. You orchestrate this infrastructure by having different branches corresponding to different environments:

- The `dev` branch contains the latest changes that are applied to the development environment.
- The `prod` branch contains the latest changes that are applied to the production environment.
- With this infrastructure, you can always reference the repository to know what configuration is expected in each environment and to propose new changes by first merging them into the `dev` environment.
- You then promote the changes by merging the `dev` branch into the subsequent `prod` branch.

The code in this repository is structured as follows:

- The `environments/` folder contains subfolders that represent environments, such as dev and prod, which provide logical separation between workloads at different stages of maturity, development and production, respectively. Although it's a good practice to have these environments as similar as possible, each subfolder has its own Terraform configuration to ensure they can have unique settings as necessary.

- The `modules/` folder contains inline Terraform modules. These modules represent logical groupings of related resources and are used to share code across different environments.

- The `cloudbuild.yaml` file is a build configuration file that contains instructions for Cloud Build, such as how to perform tasks based on a set of steps. This file specifies a conditional execution depending on the branch Cloud Build is fetching the code from, for example:

- For `dev` and `prod` branches, the following steps are executed:

  1. `terraform init`
  2. `terraform plan`
  3. `terraform apply`

- For any other branch, the following steps are executed:

  1. `terraform init` for all `environments` subfolders
  2. `terraform plan` for all `environments` subfolders

- The reason `terraform init` and `terraform plan` run for all `environments` subfolders is to make sure that the changes being proposed hold for every single environment.

- This way, before merging the pull request, you can review the plans to make sure access is not being granted to an unauthorized entity, for example.


## Before you begin

```bash
gcloud config get-value project
gcloud config set project PROJECT_ID
gcloud services enable cloudbuild.googleapis.com compute.googleapis.com
git config --global user.email "your-email-address"
git config --global user.name "your-name"
```

## Configuring Terraform to store state in a Cloud Storage bucket

- This section configures a remote state that points to a Cloud Storage bucket.
- Remote state is a feature of backends and is configured in the `backend.tf` files.

`environments/dev/backend.tf` :

```json
  terraform {
    backend "gcs" {
      bucket = "PROJECT_ID-tfstate"
      prefix = "env/dev"
    }
  }
  ```

1. Create the storage bucket

```bash
    PROJECT_ID=$(gcloud config get-value project)
    gsutil mb gs://${PROJECT_ID}-tfstate
```

2. Enable [Object Versioning](https://cloud.google.com/storage/docs/object-versioning) to keep the history of your deployments:

```bash
    gsutil versioning set on gs://${PROJECT_ID}-tfstate
```

3. Replace the `PROJECT_ID` placeholder with the project ID in both the `terraform.tfvars` and `backend.tf` files:

```bash
  sed -i s/PROJECT_ID/$PROJECT_ID/g environments/*/terraform.tfvars
  sed -i s/PROJECT_ID/$PROJECT_ID/g environments/*/backend.tf
```

4. Check whether all files were updated:

```bash
  git status
```

5. Commit and Push changes

```bash
  git add --all
  git commit -m "Update project IDs and buckets"
  git push origin dev
```

## Granting permissions to your Cloud Build service account

To allow [Cloud Build service account](https://cloud.google.com/build/docs/securing-builds/set-service-account-permissions) to run Terraform scripts with the goal of managing Google Cloud resources, you need to grant it appropriate access to your project. For simplicity, [project editor](https://cloud.google.com/iam/docs/understanding-roles#basic) access is granted in this tutorial. But when the project editor role has a wide-range permission, in production environments you must follow your company's IT security best practices, usually providing [least-privileged access](https://cloud.google.com/docs/enterprise/best-practices-for-enterprise-organizations#control-access).

1. In Cloud Shell, retrieve the email for your project's Cloud Build service account:

```bash
  CLOUDBUILD_SA="$(gcloud projects describe $PROJECT_ID \
      --format 'value(projectNumber)')@cloudbuild.gserviceaccount.com"
```

2. Grant the required access to your Cloud Build service account:

```bash
  gcloud projects add-iam-policy-binding $PROJECT_ID \
      --member serviceAccount:$CLOUDBUILD_SA --role roles/editor
```

## Directly connecting Cloud Build to your GitHub repository

## Changing your environment configuration in a new feature branch

## Enforcing Cloud Build execution success before merging branches

## Promoting changes to the development environment

## Promoting changes to the production environment

## What's next

Consider using Cloud Foundation Toolkit templates to quickly build a repeatable enterprise-ready foundation in Google Cloud.
Watch Repeatable GCP Environments at Scale With Cloud Build Infra-As-Code Pipelines from Next' 19 about the GitOps workflow described in this tutorial.
Check out the GitOps-style continuous delivery with Cloud Build tutorial.
Take a look at more advanced Cloud Build features: Configuring the order of build steps, Building, testing, and deploying artifacts, and Creating custom build steps.
Explore reference architectures, diagrams, tutorials, and best practices about Google Cloud. Take a look at our Cloud Architecture Center.
Read our resources about DevOps.
Learn more about the DevOps capabilities related to this tutorial:
Version control
Continuous integration
Continuous delivery
Continuous testing
Take the DevOps quick check to understand where you stand in comparison with the rest of the industry.

---

## Configuring your `dev` environment

```bash
cd environments/dev
terraform init
terraform plan
terraform apply
terraform destroy
```

## Promoting your environment to `production`

```bash
cd environments/prod
terraform init
terraform plan
terraform apply
terraform destroy
```
