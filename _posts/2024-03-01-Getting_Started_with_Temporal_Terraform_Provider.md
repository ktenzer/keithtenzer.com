--- 
layout: single
title:  "Getting Started with Temporal Terraform Provider"
categories:
- Temporal
tags:
- Workflow
- Terraform
- Temporal
- Temporal Cloud
- Temporal Cloud Namespace
- Automation
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
Temporal Cloud just celebrated its 1000th customer and it only launched 18 months ago! This is incredible growth. As Temporal Cloud adoption grows, automation becomes more and more critical. As such, Temporal recently released a Terraform provider for Temporal Cloud which enables automation of Temporal Cloud resources. We can manage CRUD operations for Temporal Cloud namespaces, users or even access tokens. In this article we will look at how to provision and destroy a Temporal Cloud namespace using the Terraform provider.

Documentation Resources
- [API keys](https://docs.temporal.io/cloud/api-keys)
- [Cloud Ops API](https://docs.temporal.io/ops)
- [Terraform provider](https://registry.terraform.io/providers/temporalio/temporalcloud/latest/docs)

## Create Temporal Cloud API KEY
Using the Temporal Cloud UI, create an API key.

![Create API Key Part I](/assets/2024-03-01/api_key_1.png)
![Create API Key Part II](/assets/2024-03-01/api_key_2.png)

## Install Terraform
In order to use Terraform we need to install the cli.

```bash
$ brew install terraform
```

## Configure Terraform
Create a file called main.tf for the resource you would like to manage. In this case we will be creating a Temporal Cloud namespace called "terraform". First, lets set environment variables. It is strongly recommended to set the Temporal Cloud API key and endpoint using the environment and not store that in the main.tf.

```bash
$ export TEMPORAL_CLOUD_ENDPOINT=saas-api.tmprl.cloud:443
$ export TEMPORAL_CLOUD_API_KEY=<api key>
```

Create main.tf
```bash
$ mkdir terraform
$ cd terraform
$ vi main.tf
terraform {
  required_providers {
    temporalcloud = {
      source = "temporalio/temporalcloud"
    }
  }
}

provider "temporalcloud" {
  # Also can be set by environment variable `TEMPORAL_CLOUD_ALLOW_INSECURE`
  allow_insecure = false
}

resource "temporalcloud_namespace" "terraform" {
  name               = "terraform"
  regions            = ["aws-us-east-1"]
  accepted_client_ca = base64encode(file("/Users/ktenzer/certs/ca.pem"))
  retention_days     = 14
}
```

# Running Terraform
If not done so already it is important to initialize Terraform. This will download the Temporal Cloud Terraform provider.
```bash
$ terraform init
```

Next, we should validate our configuration and ensure everything looks right.
```bash
$ terraform plan
```

Finally, we can apply our configuration which will in this case create a Temporal Cloud namespace called "terraform".
```bash
$ terraform apply
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

temporalcloud_namespace.terraform: Creating...
temporalcloud_namespace.terraform: Still creating... [10s elapsed]
temporalcloud_namespace.terraform: Still creating... [20s elapsed]
temporalcloud_namespace.terraform: Still creating... [30s elapsed]
temporalcloud_namespace.terraform: Still creating... [40s elapsed]
temporalcloud_namespace.terraform: Still creating... [50s elapsed]
temporalcloud_namespace.terraform: Still creating... [1m0s elapsed]
temporalcloud_namespace.terraform: Still creating... [1m10s elapsed]
temporalcloud_namespace.terraform: Still creating... [1m20s elapsed]
temporalcloud_namespace.terraform: Still creating... [1m30s elapsed]
temporalcloud_namespace.terraform: Still creating... [1m40s elapsed]
temporalcloud_namespace.terraform: Still creating... [1m50s elapsed]
temporalcloud_namespace.terraform: Still creating... [2m0s elapsed]
temporalcloud_namespace.terraform: Still creating... [2m10s elapsed]
temporalcloud_namespace.terraform: Creation complete after 2m16s [id=terraform.sdvdw]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

While applying Terraform configuration, we should also see our namespace activating in Temporal Cloud UI.
![Activating](/assets/2024-03-01/activating.png)

When complete our Temporal Cloud namespace should show active in the Temporal Cloud UI and is ready for workflows.
![Active](/assets/2024-03-01/active.png)

To delete the namespace, we can simply run a destroy.
```bash
$ terraform destroy
```

## Summary
In this article we demonstrated how to setup Terraform and use the Temporal Cloud Terraform provider. We provisioned and destroyed a Temporal Cloud namespace using the provider. As adoption of Temporal Cloud grows, organizations will have many more namespaces, users and access keys or certificates. The Temporal Cloud Terraform provider is a simple, but excellent way to automate namespaces, users and even access CRUD operations for Temporal Cloud. 

(c) 2024 Keith Tenzer




