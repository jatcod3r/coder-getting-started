# Coder Workspaces

Workspaces in Coder are ephemeral environments that developers work on. You can think of them as their own personal desktops hosted on the Internet. 

Similar to being on a personal laptop, developers will use an [IDE](https://en.wikipedia.org/wiki/Integrated_development_environment) to interact with the remote environment. This can be done with a wide-range of IDEs, most popular being [VS Code](https://code.visualstudio.com/), [Jetbrains](https://www.jetbrains.com/), [Jupyter](https://jupyter.org/), [code-server](https://github.com/coder/code-server),[Emacs](https://www.gnu.org/software/emacs/), and many more. 

To create these workspaces, they use a preconfigured `template` created by an administrator which defines a set of repeatable resources that can be spun up and torn down as needed. This includes the type of IDE you want the user to use, the workspace configurations, tools to install automatically, and anything else a developer may need.

# Terraform Review

Before getting started with creating templates, you'll need to know some Terraform first. Hashicorp provides an extensive [tutorials library](https://developer.hashicorp.com/terraform/tutorials) for deploying infrastructure with various providers ([AWS](https://developer.hashicorp.com/terraform/tutorials/aws-get-started), [Azure](https://developer.hashicorp.com/terraform/tutorials/azure-get-started), [GCP](https://developer.hashicorp.com/terraform/tutorials/gcp-get-started), etc.)

Here's some key components you should understand know though:

## TF CLI

The [`terraform`](https://developer.hashicorp.com/terraform/install) CLI is used to build, change, and destroy resources managed by Terraform. A common workflow is to:

1. Initialize your directory - `terraform init`
2. Plan your resources - `terraform plan`
3. Apply changes - `terraform apply`
4. Destroy when finished - `terraform destroy`

### Initialization

[`terraform init`](https://developer.hashicorp.com/terraform/cli/commands/init) installs [`providers`](###providers) and [`modules`](###modules) defined in your configuration template:

```terraform
terraform {
    required_providers {
        aws = {
            source = "hashicorp/aws"
        }
        ...
    }
}

module "vpc" {
    source = "terraform-aws-modules/vpc/aws"
    ...
}
```

These can involve cloud providers like ["hashicorp/aws"](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) to deploy AWS objects and run non-native functions, or modules like ["terraform-aws-modules/vpc/aws"](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) that bundles repeatable, ready-to-deploy infrastructure.

### Planning

[`terraform plan`](https://developer.hashicorp.com/terraform/cli/commands/validate) lets you see a preview of the infrastructure.

This helps avoid mistakes by ensuring that you're creating the resources you're intending to deploy.

### Applying

[`terraform apply`](https://developer.hashicorp.com/terraform/cli/commands/apply)


### Destroying

[`terraform destroy`](https://developer.hashicorp.com/terraform/cli/commands/destroy) is used to clean up. Once you're done with your project, you can run this command to quickly unprovision resources managed by 

## Template Components

To write Terraform code, you'll write in [`.tf`](https://developer.hashicorp.com/terraform/language/files) configuration files.

### Providers

### Resources

### Data Sources

### Variables

### Functions

### Modules





# Your First Template

Templates in Coder are [`.tf` scripts](###templates) defining your infrastructure as code. 

With Terraform, you can deploy a wide variety of resources. This includes the ability to create [AWS EC2 Instances](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance), [Azure Virtual Machines](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/virtual_machine), [GCP Compute Instances](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance), [Docker Containers](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs/resources/container), [K8s Pods](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/pod), and more.

As the platform administrator, you have the ability to manage these templates and dictate what kinds of resources they'll deploy.

To get started, we'll start with a [K8s Pod]((https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/pod)) as a workspace.

## Essential Resources

Coder uses Terraform to define and manage resources, as well as connect it back to the Coder control plane with it's own ["coder"](https://registry.terraform.io/providers/coder/coder/latest/docs) Terraform provider. This comes with repeatble resources which may not all be essential, but is reusable across deployments.

### `coder_agent`

The [`coder_agent`]([](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/agent)) defines the initialization script to connect the workspace to the Coder control plane.

```terraform
# The Coder Agent
resource "coder_agent" "dev" {

    # OS and Architecture to match the workspace's image.
    os             = "linux"
    arch           = "amd64"

}

# The agent that runs `init_script`, turning it into a workspace.
resource "kubernetes_pod" "dev" {
    spec {
        container {

            # The container's image which runs Ubuntu Jammy
            image = "ubuntu:jammy"

            # Runs the `sh` executable encapsulates startup script commands via `-c`
            # Allows the value in `args` to be ran as a single line.
            command = ["/bin/sh", "-c"]
            
            # Updates APT and installs `curl` before starting the Coder Agent's `init_script`
            args = ["apt update -y && apt install curl -y; ${coder_agent.dev.init_script}"]

            # Embeds the Coder Agent's authorization token as an environment variable.
            env {
                name  = "CODER_AGENT_TOKEN"
                value = coder_agent.dev.token
            }
            
        }
    }
}
```

To use this, it has to be tied to an agent that can run it's [`init_script`](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/agent#init_script-3). The agent also [requires](https://coder.com/docs/admin/templates/troubleshooting#agent-connection-issues) that it can reach the Coder control plane's access URL over HTTP/HTTPS and run `curl`, `wget`, or `busybox`.

### `coder_app`

To access the IDEs running on those workspaces, or to connect your local IDE to them, you use a [`coder_app`](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/app):

```terraform
# Defines the Coder Agent to run on the workspace
resource "coder_agent" "dev" {
    os             = "linux"
    arch           = "amd64"
}

# The `coder-server` IDE
resource "coder_app" "code-server" {
    agent_id     = coder_agent.dev.id
    slug         = "code-server"
    display_name = "VS Code"
    url          = "http://localhost:13337"
    share        = "owner"
    subdomain    = false
    open_in      = "slim-window"
    healthcheck {
        url       = "http://localhost:13337/healthz"
        interval  = 5
        threshold = 6
    }
}

# The `vim` IDE
resource "coder_app" "vim" {
    agent_id     = coder_agent.dev.id
    slug         = "vim"
    display_name = "Vim"
    command      = "vim"
}
```

This listens to any processes on the agent. This can be anything on [`localhost`](https://en.wikipedia.org/wiki/Localhost) or a continuous command such as [`vim`](https://en.wikipedia.org/wiki/Vim_(text_editor)):


### `coder_script`

[`coder_script`](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/script) execute's a shell script. The type of script you write will depends on the operating system (e.g. `powershell` for Windows, or `sh` for Linux/MacOS). If you have multiple `coder_script` blocks, then these will be ran in parallel.

```terraform
# Defines the Coder Agent to run on the workspace
resource "coder_agent" "dev" {
    os             = "linux"
    arch           = "amd64"
}

# Installs the `code-server` IDE and starts it
resource "coder_script" "code-server" {
    agent_id = coder_agent.dev.id
    display_name = "code-server"
    icon = "/icons/coder.svg"
    run_on_start = true
    start_blocks_login = true
    script = <<EOF
        curl -fsSL https://code-server.dev/install.sh | sh
        code-server --auth none --port 13337 > /tmp/code-server.log 2>&1 &
    EOF
}
```

### Auxillary Resources

To use information about the workspace in your template (e.g. workspace name, access URL and port, etc.), leverage [`coder_workspace`](https://registry.terraform.io/providers/coder/coder/latest/docs/data-sources/workspace).

```
# `coder_workspace` returns information about the workspace.
data "coder_workspace" "me" {}

# Defines the Coder Agent to run on the workspace
resource "coder_agent" "dev" {
    os             = "linux"
    arch           = "amd64"
}

# The K8s Pod that hosts the Coder Agent, turning it into a workspace
resource "kubernetes_pod" "dev" {
    count = data.coder_workspace.me.start_count

    metadata {
        # Name of the K8s Pod
        name = lower("${data.coder_workspace.me.name}")

        # The namespace to deploy the Pod to
        namespace = "coder-workspace"
    }

    spec {
        container {
            # Name of the container in the K8s Pod
            name = "workspace"

            # The container image
            image = "ubuntu:jammy"

            command = ["sh", "-c"]
            args = ["apt update -y && apt install curl -y; ${coder_agent.dev.init_script}"]
            env {
                name  = "CODER_AGENT_TOKEN"
                value = coder_agent.dev.token
            }
        }
    }
}
```

There's a chance that a workspace can

### [coder_workspace_owner](https://registry.terraform.io/providers/coder/coder/latest/docs/data-sources/workspace_owner)


### [coder_parameter](https://registry.terraform.io/providers/coder/coder/latest/docs/data-sources/parameter)

### [coder_metadata](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/metadata)

### [coder_env](https://registry.terraform.io/providers/coder/coder/latest/docs/resources/env)

## Common Workspaces

We're not limited to just K8-based workspaces. Coder can deploy a wide-variety of resources because of Terraform. So, whatever Terraform is able to create, Coder can too.

### K8s Pod

Combining our examples above, this is what K8s Pod-workspace would look like:

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
}

# The `coder-server` IDE
resource "coder_app" "code-server" {
  agent_id     = coder_agent.dev.id
  slug         = "code-server"
  display_name = "VS Code"
  icon         = "${data.coder_workspace.me.access_url}/icon/code.svg"
  url          = "http://localhost:13337"
  share        = "owner"
  subdomain    = false
  open_in      = "slim-window"
  healthcheck {
    url       = "http://localhost:13337/healthz"
    interval  = 5
    threshold = 6
  }
}

# The K8s Pod that hosts the Coder Agent, turning it into a workspace
resource "kubernetes_pod" "dev" {
    count = data.coder_workspace.me.start_count

    metadata {
        # Name of the K8s Pod
        name = lower("${data.coder_workspace_owner.me.name}-${data.coder_workspace.me.name}")

        # The namespace to deploy the Pod to
        namespace = "coder-workspace"
    }

    spec {
        container {
            # Name of the container in the K8s Pod
            name = "workspace"

            # The container image
            image = "ubuntu:jammy"

            command = ["sh", "-c"]
            args = ["apt update -y && apt install curl -y; ${coder_agent.dev.init_script}"]
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


### AWS EC2 Linux

Here's an example of an EC2 Instance-workspace running Linux

### AWS EC2 Windows

Here's an example of an EC2 Instance-workspace running Windows.

### AWS EC2 MacOS

Here's an example of an EC2 Instance-workspace running MacOS

### Azure VM Linux

Here's an example of an Azure Virtual Machine-workspace running Linux

### Azure VM Windows

Here's an example of an Azure Virtual Machine-workspace running Windows

### GCP VM Linux

Here's an example of an EC2 Instance-workspace running Linux

### GCP VM Windows

Here's an example of an EC2 Instance-workspace running Windows

# Coder & Terraform Review

If you're new to Terraform and Coder, it's highly recommended that you start experimenting with building templates yourself. 

This can involve creating standalone templates without Coder and doing manual deployments:

```terraform
terraform {
    required_providers {
        kubernetes = {
            source = "hashicorp/kubernetes"
        }
    }
}

resource "kubernetes_pod" "no-coder" {

    metadata {
        # Name of the K8s Pod
        name = "my-test-pod-no-coder"
        namespace = "default"
    }

    spec {
        container {
            name = "container1"
            image = "ubuntu:jammy"
            command = ["sh", "-c"]
            args = ["apt update -y && apt install curl -y; sleep 1000000"]
        }
    }
}
```

Or creating a very light-weight Coder workspace that's only accessible over a web terminal:

```terraform
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

resource "coder_agent" "dev" {
    os             = "linux"
    arch           = "amd64"
}

resource "kubernetes_pod" "with-coder" {

    metadata {
        name = "my-test-pod-with-coder"
        namespace = "default"
    }

    spec {
        container {
            name = "workspace"
            image = "ubuntu:jammy"
            command = ["sh", "-c"]
            args = ["apt update -y && apt install curl -y; ${coder_agent.dev.init_script}"]
            env {
                name  = "CODER_AGENT_TOKEN"
                value = coder_agent.dev.token
            }
        }
    }
}
```