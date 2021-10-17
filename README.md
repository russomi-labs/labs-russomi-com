# Managing infrastructure as code with Terraform, Cloud Build, and GitOps

This is the repo for the [Managing infrastructure as code with Terraform, Cloud Build, and GitOps](https://cloud.google.com/solutions/managing-infrastructure-as-code) tutorial. This tutorial explains how to manage infrastructure as code with Terraform and Cloud Build using the popular GitOps methodology.

- [Overview](#overview)
- [Before you begin](#before-you-begin)
- [Configuring Terraform to store state in a Cloud Storage bucket](#configuring-terraform-to-store-state-in-a-cloud-storage-bucket)
- [Granting permissions to your Cloud Build service account](#granting-permissions-to-your-cloud-build-service-account)
- [Directly connecting Cloud Build to your GitHub repository](#directly-connecting-cloud-build-to-your-github-repository)
- [Changing your environment configuration in a new feature branch](#changing-your-environment-configuration-in-a-new-feature-branch)
- [Enforcing Cloud Build execution success before merging branches](#enforcing-cloud-build-execution-success-before-merging-branches)
- [Promoting changes to the development environment](#promoting-changes-to-the-development-environment)
- [Promoting changes to the production environment](#promoting-changes-to-the-production-environment)
- [What's next](#whats-next)

## Overview

In this tutorial, you use a single Git repository to define your cloud infrastructure. You orchestrate this infrastructure by having different branches corresponding to different environments:

![Architecture](https://cloud.google.com/architecture/images/managing-infrastructure-as-code-infrastructure.svg)

- The `dev` branch contains the latest changes that are applied to the development environment.
- The `prod` branch contains the latest changes that are applied to the production environment.
- With this infrastructure, you can always reference the repository to know what configuration is expected in each environment and to propose new changes by first merging them into the `dev` environment.
- You then promote the changes by merging the `dev` branch into the subsequent `prod` branch.

The code in this repository is structured as follows:

- The `environments/` folder contains subfolders that represent environments, such as dev and prod, which provide logical separation between workloads at different stages of maturity, development and production, respectively. Although it's a good practice to have these environments as similar as possible, each subfolder has its own Terraform configuration to ensure they can have unique settings as necessary.

- The `modules/` folder contains inline Terraform modules. These modules represent logical groupings of related resources and are used to share code across different environments.

- The `cloudbuild.yaml` file is a build configuration file that contains instructions for Cloud Build, such as how to perform tasks based on a set of steps. This file specifies a conditional execution depending on the branch Cloud Build is fetching the code from, for example:

- For `dev` and `prod` branches, the following steps are executed:

  01. `terraform init`
  02. `terraform plan`
  03. `terraform apply`

- For any other branch, the following steps are executed:

  01. `terraform init` for all `environments` subfolders
  02. `terraform plan` for all `environments` subfolders

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

```hcl
  terraform {
    backend "gcs" {
      bucket = "PROJECT_ID-tfstate"
      prefix = "env/dev"
    }
  }
  ```

01. Create the storage bucket

```bash
    PROJECT_ID=$(gcloud config get-value project)
    gsutil mb gs://${PROJECT_ID}-tfstate
```

02. Enable [Object Versioning](https://cloud.google.com/storage/docs/object-versioning) to keep the history of your deployments:

```bash
    gsutil versioning set on gs://${PROJECT_ID}-tfstate
```

03. Replace the `PROJECT_ID` placeholder with the project ID in both the `terraform.tfvars` and `backend.tf` files:

```bash
  sed -i s/PROJECT_ID/$PROJECT_ID/g environments/*/terraform.tfvars
  sed -i s/PROJECT_ID/$PROJECT_ID/g environments/*/backend.tf
```

04. Check whether all files were updated:

```bash
  git status
```

05. Commit and Push changes

```bash
  git add --all
  git commit -m "Update project IDs and buckets"
  git push origin dev
```

## Granting permissions to your Cloud Build service account

To allow [Cloud Build service account](https://cloud.google.com/build/docs/securing-builds/set-service-account-permissions) to run Terraform scripts with the goal of managing Google Cloud resources, you need to grant it appropriate access to your project. For simplicity, [project editor](https://cloud.google.com/iam/docs/understanding-roles#basic) access is granted in this tutorial. But when the project editor role has a wide-range permission, in production environments you must follow your company's IT security best practices, usually providing [least-privileged access](https://cloud.google.com/docs/enterprise/best-practices-for-enterprise-organizations#control-access).

01. In Cloud Shell, retrieve the email for your project's Cloud Build service account:

```bash
  CLOUDBUILD_SA="$(gcloud projects describe $PROJECT_ID \
      --format 'value(projectNumber)')@cloudbuild.gserviceaccount.com"
```

02. Grant the required access to your Cloud Build service account:

```bash
  gcloud projects add-iam-policy-binding $PROJECT_ID \
      --member serviceAccount:$CLOUDBUILD_SA --role roles/editor
```

## Directly connecting Cloud Build to your GitHub repository

This section shows you how to install the [Cloud Build GitHub app](https://github.com/marketplace/google-cloud-build). This installation allows you to connect your GitHub repository with your Google Cloud project so that Cloud Build can automatically apply your Terraform manifests each time you create a new branch or push code to GitHub.

The following steps provide instructions for installing the app only for the solutions-terraform-cloudbuild-gitops repository, but you can choose to install the app for more or all your repositories.

01. Go to the GitHub Marketplace page for the Cloud Build app:
    [Open the Cloud Build app page](https://github.com/marketplace/google-cloud-build)

       - If this is your first time configuring an app in GitHub: Click **Setup with Google Cloud Build** at the bottom of the page. Then click **Grant this app access to your GitHub account**.
       - If this is not the first time configuring an app in GitHub: Click **Configure access**. The **Applications** page of your personal account opens.

02. Click **Configure** in the Cloud Build row.

03. Select **Only select repositories**, then select `solutions-terraform-cloudbuild-gitops` to connect to the repository.

04. Click **Save** or **Install** — the button label changes depending on your workflow. You are redirected to Google Cloud to continue the installation.

05. Sign in with your Google Cloud account. If requested, authorize Cloud Build integration with GitHub.

06. On the **Cloud Build** page, select your project. A wizard appears.

07. In the **Select repository** section, select your GitHub account and the `solutions-terraform-cloudbuild-gitops` repository.

08. If you agree with the terms and conditions, select the checkbox, then click **Connect**.

09. In the **Create a trigger** section, click **Create a trigger** and add the trigger name.

10. In the **Event** section, select **Push to a branch**, enter `.*` in the **Base Branch** field, and then click **Create**.

The Cloud Build GitHub app is now configured, and your GitHub repository is linked to your Google Cloud project. From now on, changes to the GitHub repository trigger Cloud Build executions, which report the results back to GitHub by using [GitHub Checks](https://developer.github.com/v3/checks/).

## Changing your environment configuration in a new feature branch

By now, you have most of your environment configured. So it's time to make some code changes in your development environment.

01. On GitHub, navigate to the main page of your forked repository.

02. Make sure you are in the dev branch.

03. To open the file for editing, go to the modules/firewall/main.tf file and click the pencil icon.

04. On line 30, fix the "http-server2" typo in target_tags field.  The value must be "http-server".

05. Add a commit message at the bottom of the page, such as "Fixing http firewall target", and select Create a new branch for this commit and start a pull request.

06. Click Propose changes.

07. On the following page, click Create pull request to open a new pull request with your change.  Once your pull request is open, a Cloud Build job is automatically initiated.

08.  Click Show all checks and wait for the check to become green.

09. Click **Details** to see more information, including the output of the terraform plan at **View more details on Google Cloud Build** link.

Note that the Cloud Build job ran the pipeline defined in the `cloudbuild.yaml` file. As discussed previously, this pipeline has different behaviors depending on the branch being fetched. The build checks whether the `$BRANCH_NAME` variable matches any environment folder. If so, Cloud Build executes `terraform plan` for that environment. Otherwise, Cloud Build executes `terraform plan` for all environments to make sure that the proposed change holds for them all. If any of these plans fail to execute, the build fails as well.

 `cloudbuild.yaml`

```yaml
- id: 'tf plan'
  name: 'hashicorp/terraform:1.0.0'
  entrypoint: 'sh'
  args:
  - '-c'
  - |
      if [ -d "environments/$BRANCH_NAME/" ]; then
        cd environments/$BRANCH_NAME
        terraform plan
      else
        for dir in environments/*/
        do
          cd ${dir}
          env=${dir%*/}
          env=${env#*/}
          echo ""
          echo "*************** TERRAFOM PLAN ******************"
          echo "******* At environment: ${env} ********"
          echo "*************************************************"
          terraform plan || exit 1
          cd ../../
        done
      fi
```

Similarly, the `terraform apply` command runs for environment branches, but it is completely ignored in any other case. In this section, you have submitted a code change to a new branch, so no infrastructure deployments were applied to your Google Cloud project.

```yaml
- id: 'tf apply'
  name: 'hashicorp/terraform:1.0.0'
  entrypoint: 'sh'
  args:
  - '-c'
  - |
      if [ -d "environments/$BRANCH_NAME/" ]; then
        cd environments/$BRANCH_NAME
        terraform apply -auto-approve
      else
        echo "***************************** SKIPPING APPLYING *******************************"
        echo "Branch '$BRANCH_NAME' does not represent an oficial environment."
        echo "*******************************************************************************"
      fi
```

## Enforcing Cloud Build execution success before merging branches

To make sure merges can be applied only when respective Cloud Build executions are successful, proceed with the following steps:

01. On GitHub, navigate to the main page of your forked repository.
02. Under your repository name, click **Settings**.
03. In the left menu, click **Branches**.
04. Under **Branch protection rules**, click **Add rule**.
05. In **Branch name pattern**, type `dev` .
06. In the **Protect matching branches** section, select **Require status checks to pass before merging**.
07. Search for your Cloud Build trigger name created previously, and click **Save changes**.
08. Click **Create**.
09. Repeat steps 3–7, setting **Branch name pattern** to `prod` .

This configuration is important to [protect](https://help.github.com/en/articles/about-protected-branches) both the `dev` and `prod` branches. Meaning, commits must first be pushed to another branch, and only then they can be merged to the protected branch. In this tutorial, the protection requires that the Cloud Build execution be successful for the merge to be allowed.

## Promoting changes to the development environment

You have a pull request waiting to be merged. It's time to apply the state you want to your `dev` environment.

01. On GitHub, navigate to the main page of your forked repository.
02. Under your repository name, click Pull requests.
03. Click the pull request you just created.
04. Click Merge pull request, and then click Confirm merge.
05. Check that a new Cloud Build has been triggered:

    - [Go to the Cloud Build page](https://console.cloud.google.com/cloud-build/builds)

06. Open the build and check the logs.
    When the build finishes, you see something like this:

```bash
    Step #3 - "tf apply": external_ip = external-ip-value
    Step #3 - "tf apply": firewall_rule = dev-allow-http
    Step #3 - "tf apply": instance_name = dev-apache2-instance
    Step #3 - "tf apply": network = dev
    Step #3 - "tf apply": subnet = dev-subnet-01
```

Copy **external-ip-value** and open the address in a web browser.

This provisioning might take a few seconds to boot the VM and to propagate the firewall rule, but at the end, you see `Environment: dev` in the web browser.

## Promoting changes to the production environment

Now that you have your development environment fully tested, you can promote your infrastructure code to production.

01. On GitHub, navigate to the main page of your forked repository.
02. Under your repository name, click **Pull requests**.
03. For the **base repository**, select your just-forked repository.
04. For **base**, select `prod` from your own base repository. For **compare**, select `dev` .
05. For **title**, enter a title such as `Promoting networking changes` , and then click **Create pull request**.
06. Review the proposed changes, including the terraform plan details from Cloud Build, and then click **Merge pull request**.
07. Click **Confirm merge**.
08. In the Cloud Console, open the **Build History** page to see your changes being applied to the production environment:

    - [Go to the Cloud Build page](https://console.cloud.google.com/cloud-build/builds)

09.  Wait for the build to finish, and then check the logs.
    At the end of the logs, you see something like this:

```bash
  Step #3 - "tf apply": external_ip = external-ip-value
  Step #3 - "tf apply": firewall_rule = prod-allow-http
  Step #3 - "tf apply": instance_name = prod-apache2-instance
  Step #3 - "tf apply": network = prod
  Step #3 - "tf apply": subnet = prod-subnet-01
```

Copy **external-ip-value** and open the address in a web browser.

This provisioning might take a few seconds to boot the VM and to propagate the firewall rule, but in the end, you see **Environment: prod** in the web browser.

You have successfully configured a serverless infrastructure-as-code pipeline on Cloud Build. In the future, you might want to try the following:

- Add deployments for separate use cases.
- Create additional environments to reflect your needs.
- Use a project per environment instead of a VPC per environment.

## What's next

- Consider using [Cloud Foundation Toolkit templates](https://cloud.google.com/foundation-toolkit) to quickly build a repeatable enterprise-ready foundation in Google Cloud.
- Watch [Repeatable GCP Environments at Scale With Cloud Build Infra-As-Code Pipelines](https://www.youtube.com/watch?v=3vfXQxWJazM) from Next' 19 about the GitOps workflow described in this tutorial.
- Check out the [GitOps-style continuous delivery with Cloud Build](https://cloud.google.com/kubernetes-engine/docs/tutorials/gitops-cloud-build) tutorial.
- Take a look at more advanced Cloud Build features: [Configuring the order of build steps](https://cloud.google.com/build/docs/configuring-builds/configure-build-step-order), [Building, testing, and deploying artifacts](https://cloud.google.com/build/docs/configuring-builds/build-test-deploy-artifacts), and [Creating custom build steps](https://cloud.google.com/build/docs/create-custom-build-steps).
- Explore reference architectures, diagrams, tutorials, and best practices about Google Cloud. Take a look at our [Cloud Architecture Center](https://cloud.google.com/architecture).
- Read our resources about [DevOps](https://cloud.google.com/devops).
- Learn more about the DevOps capabilities related to this tutorial:
  - [Version control](https://cloud.google.com/solutions/devops/devops-tech-version-control)
  - [Continuous integration](https://cloud.google.com/solutions/devops/devops-tech-continuous-integration)
  - [Continuous delivery](https://cloud.google.com/solutions/devops/devops-tech-continuous-delivery)
  - [Continuous testing](https://cloud.google.com/solutions/devops/devops-tech-test-automation)
- Take the [DevOps quick check](https://www.devops-research.com/quickcheck.html) to understand where you stand in comparison with the rest of the industry.
