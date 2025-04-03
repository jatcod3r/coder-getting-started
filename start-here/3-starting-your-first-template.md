# Your First Template

Templates in Coder are [`.tf` scripts](https://developer.hashicorp.com/terraform/intro) that defines your infrastructure as code. 

With Terraform, you can deploy a wide variety of resources. This includes the ability to create [AWS EC2 Instances](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance), [Azure Virtual Machines](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/virtual_machine), [GCP Compute Instances](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance), [Docker Containers](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs/resources/container), [K8s Pods](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/pod), and more.

As the platform administrator, you have the ability to manage these templates and dictate what kinds of resources they'll deploy.

To get started with this, we start with something light-weight, like creating a [K8s Pod]((https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/pod)) as a workspace.

## Essential Terraform Resources

Coder uses Terraform to define and manage resources, as well as connect it back to the Coder control plane by using their own ["coder"](https://registry.terraform.io/providers/coder/coder/latest/docs) Terraform provider.

The following is used to connect a workspace to the control plane:

- [coder_agent](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/agent)
- [coder_agent_instance](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/agent_instance)

To access the IDEs running on those workspaces, or to connect your local IDE to them, you use:

- [coder_app](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/app)

Here's an example that composes these together for a K8s pod would look like:

```terraform
# Required providers
terraform {
    required_providers {
        kubernetes = {
            source = "hashicorp/kubernetes"
        } 
        coder = {
            source = "coder/coder"
        }
    }
}

# Data sources that returns information about the workspace and it's owner
data "coder_workspace" "me" {}
data "coder_workspace_owner" "me" {}

# Defines the Coder Agent to run on the workspace
resource "coder_agent" "dev" {
    os             = "linux"
    arch           = "amd64"
    dir            = "/workspace"
}

# The `coder-server` IDE packaged as a module.
module "code-server" {
  count    = data.coder_workspace.me.start_count
  source   = "registry.coder.com/modules/code-server/coder"
  version  = "1.0.30"
  agent_id = coder_agent.dev.id
}

# The K8s Pod that hosts the Coder Agent, turning it into a workspace
resource "kubernetes_pod" "dev" {
    count = data.coder_workspace.me.start_count

    metadata {
        # Name of the K8s Pod
        name = lower("${data.coder_workspace_owner.me.name}-${data.coder_workspace.me.name}")

        # The namespace to deploy the Pod to
        namespace = "coder"
    }

    spec {
        container {
            # Name of the container in the K8s Pod
            name = "workspace"

            # The container image
            image = "debian:bookworm-slim"
            
            command = ["sh", "-c", coder_agent.dev.init_script]
            env {
                name  = "CODER_AGENT_TOKEN"
                value = coder_agent.dev.token
            }
            resources {
                requests = {
                    cpu = "250m"
                    memory = "500Mi"
                }
                limits = {
                    cpu = "1"
                    memory = "1G"
                }
            }
        }
    }
}
```
