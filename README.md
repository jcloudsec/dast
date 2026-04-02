trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

variables:
  TF_WORKING_DIR: environments/staging

stages:

- stage: Plan
  displayName: Terraform Plan
  jobs:
    - job: TerraformPlan
      displayName: Run Terraform Plan
      steps:
        - checkout: self

        - task: TerraformInstaller@1
          inputs:
            terraformVersion: '1.6.0'

        - script: |
            cat <<EOF > ~/.edgerc
            [default]
            client_token=$(AKAMAI_CLIENT_TOKEN)
            client_secret=$(AKAMAI_CLIENT_SECRET)
            access_token=$(AKAMAI_ACCESS_TOKEN)
            host=$(AKAMAI_HOST)
            EOF
          displayName: Create Akamai Credentials

        - script: |
            terraform -chdir=$(TF_WORKING_DIR) init
          displayName: Terraform Init

        - script: |
            terraform -chdir=$(TF_WORKING_DIR) validate
          displayName: Terraform Validate

        - script: |
            terraform -chdir=$(TF_WORKING_DIR) plan \
              -var="edgerc_path=$HOME/.edgerc" \
              -var="edgerc_section=default" \
              -out=tfplan
          displayName: Terraform Plan

        - publish: $(TF_WORKING_DIR)/tfplan
          artifact: tfplan

- stage: Apply
  displayName: Terraform Apply
  dependsOn: Plan
  condition: succeeded()
  jobs:
    - job: TerraformApply
      displayName: Run Terraform Apply
      steps:
        - checkout: self

        - task: TerraformInstaller@1
          inputs:
            terraformVersion: '1.6.0'

        - download: current
          artifact: tfplan

        - script: |
            cat <<EOF > ~/.edgerc
            [default]
            client_token=$(AKAMAI_CLIENT_TOKEN)
            client_secret=$(AKAMAI_CLIENT_SECRET)
            access_token=$(AKAMAI_ACCESS_TOKEN)
            host=$(AKAMAI_HOST)
            EOF
          displayName: Create Akamai Credentials

        - script: |
            terraform -chdir=$(TF_WORKING_DIR) init
          displayName: Terraform Init

        - script: |
            terraform -chdir=$(TF_WORKING_DIR) apply -auto-approve $(Pipeline.Workspace)/tfplan/tfplan
          displayName: Terraform Apply
