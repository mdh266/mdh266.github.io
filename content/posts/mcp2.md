+++
authors = ["Mike Harmon"]
title = "Agents, Tools, MCPs and All That: Part 2"
date = "2026-11-01"
tags = [
    "",
    "LangChain",
    "FastMCP",
    "GitHub Actions",
    "Terraform",
    "AI",
    "Agents"
]
series = ["Agents"]
aliases = ["migrate-from-jekyl"]
+++

## 1. Introduction
-------------------

In [my last post](https://michael-harmon.com/posts/mcp1/) I went over how to create agents with [LangChain](https://www.langchain.com/) and give them access to tools through a local MCP server using [FastMCP](https://gofastmcp.com/getting-started/welcome). In this post I want to go over how to deploy this MCP server through a CI/CD pipeline using [Terraform](https://www.hashicorp.com/en/products/terraform), [GitHub Actions](https://github.com/features/actions), and [Google Cloud Run](https://cloud.google.com/run?hl=en). I'll show how to use the remote MCP server with a [LangChain](https://www.langchain.com/) agent like I did previously. Setting up a remote MCP server is important since it allows people (or agents) anywhere in the world to use your tools.

Let's get into it!

## 2. CI/CD With Terraform & GitHub Actions
-------------------

In a [prior post](https://michael-harmon.com/posts/chatbot2/) I covered how to use [GitHub Actions](https://github.com/features/actions) to create a [CI/CD](https://en.wikipedia.org/wiki/CI/CD) pipeline to deploy a containerized application to [Google Cloud Run](https://cloud.google.com/run?hl=en). Google Cloud Run is serverless, which is a great option for our MCP server because the server itself does not have any state. Instead, each of the tools calls APIs or databases ([MongoDB](https://www.mongodb.com/) and [Redis](https://redis.io/)) and just requires access to them by passing my credentials into the [Docker container](https://www.docker.com). I deployed my [MCP server](https://github.com/mdh266/langchain-fastmcp/blob/main/mymcp/server.py) using a GitHub Actions YAML file called [build-image.yml](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/build-image.yml). This job builds a Docker image, pushes it to Docker Hub, and then deploys it to Google Cloud Run. All my credentials (API keys, database usernames/passwords) are stored in [GitHub secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets) and are [pushed to the container](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/build-image.yml#L54) during deployment to Cloud Run. 

**It is important to note that we do not pass the secrets when [building the image](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/build-image.yml#L34), as that would make our credentials accessible to anyone who pulls the image!** I covered a lot of the details of this CI/CD pipeline in a prior post, so I won't repeat them here. The one new thing I will mention is that I added [this line](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/build-image.yml#L18) to the job to allow me to have a "Run workflow" button on the GitHub Actions page so that I can trigger the build without pushing code changes.

Now let's discuss the new [GitHub Actions workflow](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/terraform.yaml) that runs [Terraform](https://www.hashicorp.com/en/products/terraform). Terraform is a declarative langauge and tool that is used to manage [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code). Declarative means you declare what you need and Terraform figures out how to create it. 

The Terraform for this projcet code is located in the [terraform](https://github.com/mdh266/langchain-fastmcp/tree/main/terraform) directory. It contains a file called [main.tf](https://github.com/mdh266/langchain-fastmcp/blob/main/terraform/main.tf), which declares the cloud provider and resources we use. 

The first part of this code is:



```terraform
terraform {
  required_version = ">=1.15.3"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~>5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

This block of code tells Terraform we will be using the Google Cloud provider with the specified `project` and `region`. The `var.` prefix (such as `var.project_id`) signifies that these are input variable values. These variables are defined in [variables.tf](https://github.com/mdh266/langchain-fastmcp/blob/main/terraform/variables.tf). The format for variables is:


```terraform
variable "<variable-name>" {
  description = ...
  type        = ...
  sensitive   = true or false
}
```

Non-sensitive variables have their values set in the [terraform.tfvars](https://github.com/mdh266/langchain-fastmcp/blob/main/terraform/terraform.tfvars) file, which is committed to the repo. Sensitive variables have their values supplied as secrets during CI/CD; they should not be stored in the code.

Next, in the `main.tf` file, we define the [resources in Terraform](https://developer.hashicorp.com/terraform/plugin/framework/resources). Each resource has the basic form:

```terraform
resource "<cloud-provider-service-name>" "<resource-name>" {
  name = ...
  region = ...
  
  ....
}
```

The Terraform code to create a Google Cloud Run service is:

```terraform
resource "google_cloud_run_service" "mcp_service" {
  name     = var.service_name
  location = var.region

  template {
    spec {
      containers {
        image = var.container_image

        resources {
          limits = {
            cpu    = "1"
            memory = "256Mi"
          }
        }
      }
    }
  }
}
```

Note that this pulls the container image that we deployed in the [build-image.yml](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/build-image.yml) GitHub Actions job through the variable [var.container_image](https://github.com/mdh266/langchain-fastmcp/blob/main/terraform/terraform.tfvars#L3). The resource limits the container to 1 CPU and a maximum of 256 MB of RAM. This is all that's needed to create the Cloud Run service! However, we have not given anyone access to it yet. To make it publicly accessible, we must create an IAM policy binding:

```terraform
resource "google_cloud_run_service_iam_member" "public_invoker" {
  location = google_cloud_run_service.mcp_service.location
  project  = var.project_id
  service  = google_cloud_run_service.mcp_service.name

  role   = "roles/run.invoker"
  member = "allUsers"
}
```

Notice that the location references [google_cloud_run_service.mcp_service.location](https://github.com/mdh266/langchain-fastmcp/blob/main/terraform/terraform.tfvars#L1) so that they are co-located, and `service` references [google_cloud_run_service.mcp_service.name](https://github.com/mdh266/langchain-fastmcp/blob/main/terraform/terraform.tfvars#L2). Lastly, setting `role = "roles/run.invoker"` and `member = "allUsers"` grants all users permission to invoke the service.

Now, you may be asking: how does GitHub Actions have permissions to create these resources? If you look at the workflow YAML files, there is a step that [authenticates with GCP](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/terraform.yaml#L31). This uses a JSON key associated with a [service account](https://docs.cloud.google.com/iam/docs/service-account-overview), which is a special non-human identity used by applications, workloads, or automation tools to authenticate and gain authorized access to Google Cloud resources. I went over how to do this manually in the [Web UI in a prior post](https://michael-harmon.com/posts/chatbot2/#deploying-on-google-cloud-run). The one thing I will say is that the Service account needs the following privelages,

<img src="https://github.com/mdh266/langchain-fastmcp/blob/main/images/service_account.png?raw=1">


Note this needs to be created before either of the GitHub actions are run!

The last file we need to discuss is [outputs.tf](https://github.com/mdh266/langchain-fastmcp/blob/main/terraform/outputs.tf):

```terraform
output "service_url" {
  description = "Public URL of the Cloud Run service"
  value       = google_cloud_run_service.mcp_service.status[0].url
}
```

This defines the output value of the Terraform build, which returns the URL of the Google Cloud Run MCP endpoint.

Now we `cd` into the `terraform` directory and run the [init` command](https://developer.hashicorp.com/terraform/cli/commands/init):

    terraform init

<img src="https://github.com/mdh266/langchain-fastmcp/blob/main/images/terraform1.png?raw=1">

This will create a `.terraform` directory along with a `.terraform.lock.hcl` file.

Next we run the [plan command](https://developer.hashicorp.com/terraform/cli/commands/plan):

    terraform plan -var-file=terraform.tfvars

This creates an execution plan for the infrastructure using the non-sensitive variable values from [terraform.tfvars](https://github.com/mdh266/langchain-fastmcp/blob/main/terraform/terraform.tfvars). The command above prints the plan to the screen, but to save it to a file called `tfplan`, we use:

    terraform plan -var-file=terraform.tfvars -out=tfplan

Terraform will prompt you for the sensitive variable values and create the infrastructure plan as shown below:

<img src="https://github.com/mdh266/langchain-fastmcp/blob/main/images/terraform2.png?raw=1">

To build the actual infrastructure from the plan `tfplan`, we run the [apply command](https://developer.hashicorp.com/terraform/cli/commands/appl):

    terraform apply -auto-approve tfplan

After the infrastructure is built, Terraform will output the URL for the endpoint! The endpoint will look like `https://agentsmcp-service-{mcp_num}.us-central1.run.app`. The `agentsmcp-service` and `us-central1` values come from `service_name` and `region` in [terraform.tfvars](https://github.com/mdh266/langchain-fastmcp/blob/main/terraform/terraform.tfvars), respectively.

To provision this infrastructure as part of a CI/CD pipeline, I created the [terraform.yaml](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/terraform.yaml) GitHub Actions workflow. A couple of things I'll call out: first, [starting on line 6](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/terraform.yaml#L6), we set the build to run whenever the workflow YAML file or any file in the `terraform` folder changes. Second, `project_id` is set as [an environment variable](https://github.com/mdh266/langchain-fastmcp/blob/main/.github/workflows/terraform.yaml#L18) that begins with `TF_VAR_` (`TF_VAR_project_id`) to signify that it's a Terraform variable. The rest of the YAML file is standard, and the steps in the job follow the sequence of commands we just discussed.

## 3. Using A Remote MCP Server With An Agent
-------------------

Now that we have a remote MCP server, we can connect an agent to it just the same way we did in the last post. To give a LangChain agent access to the tools supplied by an MCP server, we need to use a specialized MCP client for LangChain called [LangChain MCP Adapters](https://reference.langchain.com/python/langchain-mcp-adapters). This can be imported and configured with the URL of the MCP server and the transport type set to `"http"`:


```python
from dotenv import load_dotenv, find_dotenv
import os
from langchain_mcp_adapters.client import MultiServerMCPClient  
load_dotenv(find_dotenv())

mcp_num = os.environ.get("MCP_NUM")
client = MultiServerMCPClient({"config": 
                        {"url": f"https://agentsmcp-service-{mcp_num}.us-central1.run.app/mcp", 
                         "transport": "http"}
})
```

Now we can get the tools:


```python
tool_list = await client.get_tools()
```

Notice these are [LangChain tools](https://docs.langchain.com/oss/python/langchain/tools)! The adapter converted the [FastMCP](https://gofastmcp.com/getting-started/welcome) tools to LangChain ones! Now let's create an agent and give it access to the tools from the MCP server:


```python
from langchain_groq import ChatGroq
from langchain.agents import create_agent

model = ChatGroq(model="openai/gpt-oss-120b")

agent = create_agent(model=model, tools=tool_list)

result = await agent.ainvoke({
    'messages': [{'role': 'user', 
                 'content': 'What police precinct is 306 West 54th Street, Manhattan, NY in? Get me their info.'
}]})
```


```python
print(result.get("messages")[-1].content)
```

    **NYC Police Precinct:** **Precinct 18**  
    
    **Details**
    
    | Item | Information |
    |------|--------------|
    | **Borough** | Manhattan |
    | **Address** | 306 West 54th Street, New York, NY 10019 |
    | **Phone** | 212‑767‑8400 |
    | **Coverage Area** | The 18th Precinct serves the Midtown West area of Manhattan, including the neighborhoods around Hell’s Kitchen, the Theater District, and the Upper West Side near the address you provided. |
    
    If you need to contact the precinct, you can call the non‑emergency number above (or dial **911** for emergencies).


It worked!

## 4. Using A Remote MCP Server With Google Antigravity
-------------------

Because MCP is an open standard, any MCP-compatible client or AI coding assistant can connect to our remote server without needing custom integration code. To connect [Google Antigravity](https://antigravity.google/) to our Cloud Run MCP server, we simply add the server URL to `mcp_config.json` (either globally in `~/.gemini/config/mcp_config.json` or inside the project's `.agents/mcp_config.json` directory):

```json
{
  "mcpServers": {
    "remote-server-example": {
      "url": "https://agentsmcp-service-{mcp_num}.us-central1.run.app/mcp"
    }
  }
}
```

Once configured, Antigravity automatically discovers the tools exposed by the server (`get_weather`, `convert_address_to_point`, `get_police_precinct`, `find_closest_restroom`, and `get_precinct_info`) along with their input schemas.

We can then ask Antigravity questions directly in chat:
- **"What's the info on police precinct 17?"** — Antigravity calls `get_precinct_info(precinct_number=17)` on the remote Cloud Run server and returns the Manhattan 17th Precinct's address (`167 East 51st Street`) and phone number (`212-826-3211`).
- **"What's the closest bathroom to 270 Park Ave, New York?"** — Antigravity chains `convert_address_to_point` (`40.7559364, -73.9757813`) and `find_closest_restroom` to return the nearest public restroom (`575 Fifth Avenue POPS`).

## 5. Conclusions
--------------------

In this post I covered how to deploy a remote MCP server to Google Cloud Run using Terraform and GitHub Actions, and how to connect both a LangChain agent and Google Antigravity to the remote server. Using Terraform allows us to create infrastructure automatically using code instead of manually configuring it through the Google Cloud web user interface (UI) as I have shown in the past. This makes building and deploying infrastructure much more repeatable.

<!--  -->
