Skip to content
logo
n8n Docs
Search

Using n8n
Integrations
Hosting n8n
Code in n8n
Advanced AI
API
Embed
n8n home ↗
Forum ↗
Blog ↗
Hosting n8n
Community vs Enterprise
Installation
npm
Docker
Server setups
Digital Ocean
Heroku
Hetzner
Amazon Web Services
Azure
Google Cloud
Docker Compose
Updating
Configuration
Environment variables
Configuration methods
Configuration examples
Supported databases and settings
Task runners
User management
Logging and monitoring
Logging
Monitoring
Security audit
Scaling and performance
Overview
Performance and benchmarking
Configuring queue mode
Concurrency control
Execution data
Binary data
External storage for binary data
Memory-related errors
Securing n8n
Overview
Set up SSL
Set up SSO
Security audit
Disable the API
Opt out of data collection
Blocking nodes
Hardening task runners
Starter Kits
AI Starter Kit
Architecture
Overview
Database structure
Using the CLI
CLI commands
Table of contents
Prerequisites
Hosting options
Open the Azure Kubernetes Service
Create a cluster
Set Kubectl context
Clone configuration repository
Configure Postgres
Configure volume for persistent storage
Postgres environment variables
Configure n8n
Create a volume for file storage
Pod resources
Optional: Environment variables
Deployments
Services
Send to Kubernetes cluster
Set up DNS
Delete resources
Next steps
Hosting n8n
Installation
Server setups
Hosting n8n on Azure#
This hosting guide shows you how to self-host n8n on Azure. It uses n8n with Postgres as a database backend using Kubernetes to manage the necessary resources and reverse proxy.

Prerequisites#
You need The Azure command line tool

Self-hosting knowledge prerequisites

Self-hosting n8n requires technical knowledge, including:

Setting up and configuring servers and containers
Managing application resources and scaling
Securing servers and applications
Configuring n8n
n8n recommends self-hosting for expert users. Mistakes can lead to data loss, security issues, and downtime. If you aren't experienced at managing servers, n8n recommends n8n Cloud.

Latest and Next versions

n8n releases a new minor version most weeks. The latest version is for production use. next is the most recent release. You should treat next as a beta: it may be unstable. To report issues, use the forum.

Current latest: 1.91.3
Current next: 1.92.2

Hosting options#
Azure offers several ways suitable for hosting n8n, including Azure Container Instances (optimized for running containers), Linux Virtual Machines, and Azure Kubernetes Service (containers running with Kubernetes).

This guide uses the Azure Kubernetes Service (AKS) as the hosting option. Using Kubernetes requires some additional complexity and configuration, but is the best method for scaling n8n as demand changes.

The steps in this guide use a mix of the Azure UI and command line tool, but you can use either to accomplish most tasks.

Open the Azure Kubernetes Service#
From the Azure portal select Kubernetes services.

Create a cluster#
From the Kubernetes services page, select Create > Create a Kubernetes cluster.

You can select any of the configuration options that suit your needs, then select Create when done.

Set Kubectl context#
The remainder of the steps in this guide require you to set the Azure instance as the Kubectl context. You can find the connection details for a cluster instance by opening its details page and then the Connect button. The resulting code snippets shows the steps to paste and run into a terminal to change your local Kubernetes settings to use the new cluster.

Clone configuration repository#
Kubernetes and n8n require a series of configuration files. You can clone these from this repository. The following steps tell you which file configures what and what you need to change.

Clone the repository with the following command:


git clone https://github.com/n8n-io/n8n-kubernetes-hosting.git -b azure
And change directory to the root of the repository you cloned:


cd azure
Configure Postgres#
For larger scale n8n deployments, Postgres provides a more robust database backend than SQLite.

Configure volume for persistent storage#
To maintain data between pod restarts, the Postgres deployment needs a persistent volume. The default storage class is suitable for this purpose and is defined in the postgres-claim0-persistentvolumeclaim.yaml manifest.

Specialized storage classes

If you have specialised or higher requirements for storage classes, read more on the options Azure offers in the documentation.

Postgres environment variables#
Postgres needs some environment variables set to pass to the application running in the containers.

The example postgres-secret.yaml file contains placeholders you need to replace with your own values. Postgres will use these details when creating the database..

The postgres-deployment.yaml manifest then uses the values from this manifest file to send to the application pods.

Configure n8n#
Create a volume for file storage#
While not essential for running n8n, using persistent volumes is required for:

Using nodes that interact with files, such as the binary data node.
If you want to persist manual n8n encryption keys between restarts. This saves a file containing the key into file storage during startup.
The n8n-claim0-persistentvolumeclaim.yaml manifest creates this, and the n8n Deployment mounts that claim in the volumes section of the n8n-deployment.yaml manifest.


…
volumes:
  - name: n8n-claim0
    persistentVolumeClaim:
      claimName: n8n-claim0
…
Pod resources#
Kubernetes lets you optionally specify the minimum resources application containers need and the limits they can run to. The example YAML files cloned above contain the following in the resources section of the n8n-deployment.yaml file:


…
resources:
  requests:
    memory: "250Mi"
  limits:
    memory: "500Mi"
…    
This defines a minimum of 250mb per container, a maximum of 500mb, and lets Kubernetes handle CPU. You can change these values to match your own needs. As a guide, here are the resources values for the n8n cloud offerings:

Start: 320mb RAM, 10 millicore CPU burstable
Pro (10k executions): 640mb RAM, 20 millicore CPU burstable
Pro (50k executions): 1280mb RAM, 80 millicore CPU burstable
Optional: Environment variables#
You can configure n8n settings and behaviors using environment variables.

Create an n8n-secret.yaml file. Refer to Environment variables for n8n environment variables details.

Deployments#
The two deployment manifests (n8n-deployment.yaml and postgres-deployment.yaml) define the n8n and Postgres applications to Kubernetes.

The manifests define the following:

Send the environment variables defined to each application pod
Define the container image to use
Set resource consumption limits with the resources object
The volumes defined earlier and volumeMounts to define the path in the container to mount volumes.
Scaling and restart policies. The example manifests define one instance of each pod. You should change this to meet your needs.
Services#
The two service manifests (postgres-service.yaml and n8n-service.yaml) expose the services to the outside world using the Kubernetes load balancer using ports 5432 and 5678 respectively.

Send to Kubernetes cluster#
Send all the manifests to the cluster with the following command:


kubectl apply -f .
Namespace error

You may see an error message about not finding an "n8n" namespace as that resources isn't ready yet. You can run the same command again, or apply the namespace manifest first with the following command:


kubectl apply -f namespace.yaml
Set up DNS#
n8n typically operates on a subdomain. Create a DNS record with your provider for the subdomain and point it to the IP address of the n8n service. Find the IP address of the n8n service from the Services & ingresses menu item of the cluster you want to use under the External IP column. You need to add the n8n port, "5678" to the URL.

Static IP addresses with AKS

Read this tutorial for more details on how to use a static IP address with AKS.

Delete resources#
Remove the resources created by the manifests with the following command:


kubectl delete -f .
Next steps#
Learn more about configuring and scaling n8n.
Or explore using n8n: try the Quickstarts.
Was this page helpful?


 Back to top
Previous
Amazon Web Services
Next
Google Cloud
Popular integrations
Google Sheets
Telegram
MySQL
Slack
Discord
Postgres

Trending combinations
HubSpot and Salesforce
Twilio and WhatsApp
GitHub and Jira
Asana and Slack
Asana and Salesforce
Jira and Slack

Top integration categories
Development
Communication
Langchain
AI
Data & Storage
Marketing

Trending templates
Creating an API endpoint
AI agent chat
Scrape and summarize webpages with AI
Very quick quickstart
Pulling data from services that n8n doesn’t have a pre-built integration for
Joining different datasets

Top guides
Telegram bots
Open-source chatbot
Open-source LLM
Open-source low-code platforms
Zapier alternatives
Make vs Zapier

Pricing ↗
Workflow templates ↗
Feature highlights ↗
AI highlights ↗
Change cookie settings
Made with Material for MkDocs Insiders
