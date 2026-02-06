---
name: deploy-rapid
description: Deploy a rapid service to the staging environment.
---

# Deploy Rapid

## Instructions

1. Deploy the given rapid service to the DataDog staging environment. Run the command `rapid release --env staging` in the background. Make any adjustments necerssary to correctly execute the user's query.
2. Once the release process begins, there will be several gitlab jobs and pipelines executed. The URLs to these resources will be provided in the output of the command run above. You should use the github cli `glab` to check on the progress of the deployment. If there are any errors, read the logs thoroughly and provide the reason the job failed along with remediation steps.
3. Once the job finishes, a mosaic link will be provided by rapid. This mosaic provides the status of the Kubernetes deployment of the rapid service. If you do not know the context of the Kubernetes cluster or namespace, do not assume them. You should ask the user what these values are. Once you have the information, use `kubectl` to check on the status of the deployment. If there are any errors, provide reasoning and remediation steps.
