---
title: "Low-cost terraformed minecraft server on GCP"
date: 2019-12-15T12:50:00+00:00
draft: false
toc: true
tags:
  - terraform
  - minecraft
  - github-actions
  - gcp
---

Out of pure nostalgia, I decided to setup an overengineered and cost-efficient minecraft server on GCP with the help of [@Naramsim](https://github.com/Naramsim). The IaC code here can be found in [this repo](https://github.com/ichbinfrog/minecraft-public) and this 'blog' will detail some of the technical reasonings behind some opinionated choices.

## Setting up CICD pipelines

In terraform CICD pipelines, a google service account is often granted high privileges (with a lot lot of Admin roles) to terraform the infrastructure on the behest of the owner. Thus, external CICD systems (and to a certain extent even GCP offerings) have to get access to this service account one way or another:

- In the traditional way, a service account key could be exported and then stored in the external system's secret manager (e.g. [Github secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets), [CircleCI Contexts](https://circleci.com/docs/2.0/contexts/), ...). Management of these exported keys is however quite complexe and prone to errors; requiring leak detection, a rotation mecanism and a myriad of tools that are outside the scope of this mini project.

- One can also create [Self hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners) that would reside in a GCE instance group and could thus use IAM permissions out of the box. The cost factor inhibits this choice.

- Finally, [Workload Identity Federation](https://cloud.google.com/iam/docs/configuring-workload-identity-federation) can be setup between Github and GCP to allow [keyless authentication](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions). This effectively allows external resources (Github Actions in this case) to create and use short lived tokens to impersonate the service account. Since impersonation can be easily audited and the 'privilege escalation' required is restricted in duration, this option is chosen to alleviate maintenance and security burdens of the all-mighty terraform service account.

### Overview

{{< image src="/img/minecraft_pipeline.png"  position="center" >}}

### Bootstrapping 

As per usual, setting up a terraform pipeline requires some manual steps also done with terraform. The `bootstrap/` module should therefore be run manually, in order to setup prerequisites:

- The GCP project that hosts all resources
- The terraform service account, omnipotent within that project
- The [GCS state bucket](https://www.terraform.io/language/settings/backends/gcs) which is the source a chicken and egg problem, where we want terraform to store the state in the bucket but the bucket itself must be created by terraform. This is solved by running the `terraform apply` locally and then migrating the state to the newly created bucket with `terraform init -migrate-state`.
- Workload Identity Federation that would allow Github Actions to impersonate said GCP service account.  

*Takeaways*: A good pointer to whether or not a resource should be in the bootstrap module is if it's mandatory for the CICD pipelines. 

### Github Actions Workflow

The workflow is standard for terraform repos, where the terraform state is in sync with the HEAD of the master branch. All PRs to master will only show the diff between the desired infra and the current one.

```yaml
on:
  push:
    branches:
    - master
  pull_request:
    branches:
      - master
```

Then using the [google-auth](https://github.com/google-github-actions/auth) action, the Github Actions runner impersonates the terraform service account with [OICD tokens](https://github.blog/changelog/2021-10-27-github-actions-secure-cloud-deployments-with-openid-connect/). 
```yaml
jobs:
  terraform:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      id-token: write
      issues: write
      # for writing the plan output in the PR comments
      # or better yet, infra-cost reports
      pull-requests: write 
       
    steps:
    - uses: actions/checkout@v2
    - id: auth
      name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v0
      with:
        workload_identity_provider: projects/{{ project-numeric-id }}/locations/global/workloadIdentityPools/{{ pool-name }}/providers/{{ provider-name }}
        service_account: github-terraform-pipeline@{{ project-name }}.iam.gserviceaccount.com
        create_credentials_file: true
```

Finally, with the [hashicorp/setup-terraform](https://github.com/hashicorp/setup-terraform) action, we do the usual `init > plan > apply` workflow with apply only running in master.

```yaml
    - uses: hashicorp/setup-terraform@v1
    - id: init
      name: Terraform init
      run: |
        terraform init
    - id: plan
      name: Terraform plan
      run: |
        terraform plan -no-color
        
    - id: apply
      name: Terraform apply
      if: ${{ github.ref == 'refs/heads/master' }}
      run: |
        terraform apply -auto-approve
```

*Takeaways*: It's quite easy to overengineer the pipelines so be mindful of only adding the features that are trully required (e.g. having the terraform plan output in the PR is nice and all, but the logs also show the plan quite clearly)

---

## Core architecture

Once the pipelines are setup, iterating on the 'core' of the project will be much faster. Want a new feature? Open a PR, verify the plan, merge it and it's deployed.

### Overview

{{< image src="/img/minecraft_core.png"  position="center" >}}

A couple resource groups are created by the repo:

- A **preemptible** GCE VM with Container Optimized OS on which the [itzg/docker-minecraft-server](https://github.com/itzg/docker-minecraft-server) image is run. Preemptible VMs are at the heart of the cost optimization, coming with a 60-91% discount compared to standard ones but the instance will be live for 24h at max. One may also be asking why run it inside a docker container and the answer is lazyness and also the fact that the only persistence required by the workload is the server data (e.g. maps, users, ...) which is garanteed by the persistent disk.

- A static IP adress from which the users can connect to the server. This is the main cost contributor to the project.

- A persistent disk in which the minecraft data will be stored (with daily snapshot)

- A compute network and a single subnet in a specific region to which firewall rules are added which only allow the minecraft client ports (from any range) and SSH connections from the IAP source ranges

- A bugdet monitor with the appropriate notification channel to avoid losing track of the server costs.

- IAM permissions that allows a subset of users to start and stop the VMs. This is a workaround for preemptible VMs as game servers, if some friends want to start playing and the server is shutdown, they can start it themselves with the garantee that even if they forget to shutdown after their session, the cost incurred would be < 24h of compute engine time.

### Potential improvements

- A [Log based metric](https://cloud.google.com/logging/docs/logs-based-metrics) that parses the minecraft server logs to find the amount of active users should be created which will alert admins when there's no one online for a 30min interval prompting them to turn of the instance.

- A cloud function for turning the instance on and off can be developped and triggered when the above alert is triggered to automated the turning off of VMs.

- A Cloud DNS entry can be added to redirect to the external IP, avoiding the hassle of remembering the IP at the cost of a domain name.

- There's an issue with the size of the VMs, wherein, for too small configurations, the cloud-init mount for the persistent disk is not fast enough, leading to the docker container with the server starting before the data disk is mounted. A restart of the VM usually fixes this issue.