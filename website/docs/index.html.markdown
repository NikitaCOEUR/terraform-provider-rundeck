---
layout: "rundeck"
page_title: "Provider: Rundeck"
sidebar_current: "docs-rundeck-index"
description: |-
  The Rundeck provider configures projects, jobs, ACLs and keys in Rundeck.
---

# Rundeck Provider

The Rundeck provider allows Terraform to create and configure Projects,
Jobs, ACLs and Keys in [Rundeck](http://www.rundeck.com/). Rundeck is a tool
for Runbook Automation and execution of arbitrary management tasks,
allowing operators to avoid logging in to individual machines directly.

The provider configuration block accepts the following arguments:

* ``url`` - (Required) The root URL of a Rundeck server. May alternatively be set via the
  ``RUNDECK_URL`` environment variable.

* ``api_version`` - (Optional) The API version of the server. Defaults to `14`, the
  minium supported version. May alternatively be set via the ``RUNDECK_API_VERSION``
  environment variable.

* ``auth_token`` - (Required) The API auth token to use when making requests. May alternatively
  be set via the ``RUNDECK_AUTH_TOKEN`` environment variable.

Use the navigation to the left to read about the available resources.

## Example Usage

A full Example Exercise is included on the [Rundeck Learning site](https://docs.rundeck.com/docs/learning/howto/use-terraform-provider.html).

For those familiar with Terraform and Rundeck use the contents below.

```hcl
terraform {
  required_providers {
    rundeck = {
      source  = "rundeck/rundeck"
      version = "0.4.2"
    }
  }
}

provider "rundeck" {
  url         = "http://rundeck.example.com:4440/"
  api_version = "38"
  auth_token  = "abcd1234"
}

resource "rundeck_project" "terraform" {
  name        = "terraform"
  description = "Sample Application Created by Terraform Plan"
  ssh_key_storage_path = "${rundeck_private_key.terraform.path}"
  resource_model_source {
    type = "file"
    config = {
      format = "resourcexml"
      # This path is interpreted on the Rundeck server.
      file = "/home/rundeck/resources.xml"
      writable = "true"
      generateFileAutomatically = "true"
    }
  }
  extra_config = {
    "project.label" = "Terraform Example"
  }
}

resource "rundeck_job" "bounceweb" {
  name              = "Bounce All Web Servers"
  project_name      = "${rundeck_project.terraform.name}"
  node_filter_query = "tags: web"
  description       = "Restart the service daemons on all the web servers"

  command {
    shell_command = "sudo service anvils restart"
  }
}

resource "rundeck_public_key" "terraform" {
  path         = "terraform/id_rsa.pub"
  key_material = "ssh-rsa yada-yada-yada"
}

resource "rundeck_private_key" "terraform" {
  path         = "terraform/id_rsa"
  key_material = "$${file(\"id_rsa.pub\")}"
}

data "local_file" "acl" {
  filename = "${path.cwd}/acl.yaml"
}

resource "rundeck_acl_policy" "example" {
  name = "ExampleAcl.aclpolicy"

  policy = "${data.local_file.acl.content}"
}
```

> Note: This example uses an ACL Policy file stored at the current working directory named `acl.yaml`.  Valid contents for that file are shown below.

```
by:
  group: terraform
description: Allow terraform Key Storage Access
for:
  storage:
  - allow:
    - read
context:
  application: rundeck
---
by:
  group: terraform
description: Allow Terraform Group [read] for all projects
for:
  project:
  - allow:
    - read
context:
  application: rundeck
---
by:
  group: terraform
description: Terraform Project Full Admin
for:
  project:
  - allow:
    - admin
    match:
      name: terraform
context:
  application: rundeck
```
