---
title: Create a Kubernetes cluster with Azure Kubernetes Service (AKS) using Terraform
description: Learn how to create a Kubernetes Cluster with Azure Kubernetes Service and Terraform.
keywords: azure devops terraform aks kubernetes
ms.topic: how-to
ms.date: 03/09/2020
ms.custom: devx-track-terraform, devx-track-azurecli
---

# Create a Kubernetes cluster with Azure Kubernetes Service using Terraform

[Azure Kubernetes Service (AKS)](/azure/aks/) manages your hosted Kubernetes environment. AKS allows you to deploy and manage containerized applications without container orchestration expertise. AKS also enables you to do many common maintenance operations without taking your app offline. These operations include provisioning, upgrading, and scaling resources on demand.

In this article, you learn how to do the following tasks:

> [!div class="checklist"]
> * Use HCL (HashiCorp Language) to define a Kubernetes cluster
> * Use Terraform and AKS to create a Kubernetes cluster
> * Use the kubectl tool to test the availability of a Kubernetes cluster

## Prerequisites

[!INCLUDE [open-source-devops-prereqs-azure-subscription.md](../includes/open-source-devops-prereqs-azure-subscription.md)]

- **Configure Terraform**: Follow the directions in the article, [Terraform and configure access to Azure](get-started-cloud-shell.md)

- **Azure service principal**: Follow the directions in the **Create the service principal** section in the article, [Create an Azure service principal with Azure CLI](/cli/azure/create-an-azure-service-principal-azure-cli). Take note of the values for the `appId`, `displayName`, `password`, and `tenant`.

## Create the directory structure

The first step is to create the directory that holds your Terraform configuration files for the exercise.

1. Browse to the [Azure portal](https://portal.azure.com).

1. Open [Azure Cloud Shell](/azure/cloud-shell/overview). If you didn't select an environment previously, select **Bash** as your environment.

    ![Cloud Shell prompt](./media/create-k8s-cluster-with-tf-and-aks/azure-portal-cloud-shell-button-min.png)

1. Change directories to the `clouddrive` directory.

    ```bash
    cd clouddrive
    ```

1. Create a directory named `terraform-aks-k8s`.

    ```bash
    mkdir terraform-aks-k8s
    ```

1. Change directories to the new directory:

    ```bash
    cd terraform-aks-k8s
    ```

## Declare the Azure provider

Create the Terraform configuration file that declares the Azure provider.

1. In Cloud Shell, create a file named `main.tf`.

    ```bash
    code main.tf
    ```

1. Paste the following code into the editor:

    ```hcl
    provider "azurerm" {
        # The "feature" block is required for AzureRM provider 2.x. 
        # If you are using version 1.x, the "features" block is not allowed.
        version = "~>2.0"
        features {}
    }

    terraform {
        backend "azurerm" {}
    }
    ```

1. Save the file (**&lt;Ctrl>S**) and exit the editor (**&lt;Ctrl>Q**).

## Define a Kubernetes cluster

Create the Terraform configuration file that declares the resources for the Kubernetes cluster.

1. In Cloud Shell, create a file named `k8s.tf`.

    ```bash
    code k8s.tf
    ```

1. Paste the following code into the editor:

    ```hcl
    resource "azurerm_resource_group" "k8s" {
        name     = var.resource_group_name
        location = var.location
    }
    
    resource "random_id" "log_analytics_workspace_name_suffix" {
        byte_length = 8
    }

    resource "azurerm_log_analytics_workspace" "test" {
        # The WorkSpace name has to be unique across the whole of azure, not just the current subscription/tenant.
        name                = "${var.log_analytics_workspace_name}-${random_id.log_analytics_workspace_name_suffix.dec}"
        location            = var.log_analytics_workspace_location
        resource_group_name = azurerm_resource_group.k8s.name
        sku                 = var.log_analytics_workspace_sku
    }

    resource "azurerm_log_analytics_solution" "test" {
        solution_name         = "ContainerInsights"
        location              = azurerm_log_analytics_workspace.test.location
        resource_group_name   = azurerm_resource_group.k8s.name
        workspace_resource_id = azurerm_log_analytics_workspace.test.id
        workspace_name        = azurerm_log_analytics_workspace.test.name

        plan {
            publisher = "Microsoft"
            product   = "OMSGallery/ContainerInsights"
        }
    }

    resource "azurerm_kubernetes_cluster" "k8s" {
        name                = var.cluster_name
        location            = azurerm_resource_group.k8s.location
        resource_group_name = azurerm_resource_group.k8s.name
        dns_prefix          = var.dns_prefix

        linux_profile {
            admin_username = "ubuntu"

            ssh_key {
                key_data = file(var.ssh_public_key)
            }
        }

        default_node_pool {
            name            = "agentpool"
            node_count      = var.agent_count
            vm_size         = "Standard_D2_v2"
        }

        service_principal {
            client_id     = var.client_id
            client_secret = var.client_secret
        }

        addon_profile {
            oms_agent {
            enabled                    = true
            log_analytics_workspace_id = azurerm_log_analytics_workspace.test.id
            }
        }
        
        network_profile {
            load_balancer_sku = "Standard"
            network_plugin = "kubenet"
        }

        tags = {
            Environment = "Development"
        }
    }
    ```

    The preceding code sets the name of the cluster, location, and the resource group name. The prefix for the fully qualified domain name (FQDN) is also set. The FQDN is used to access the cluster.

    The `linux_profile` record allows you to configure the settings that enable signing into the worker nodes using SSH.

    With AKS, you pay only for the worker nodes. The `default_node_pool` record configures the details for these worker nodes. The `default_node_pool record` includes the number of worker nodes to create and the type of worker nodes. If you need to scale up or scale down the cluster in the future, you modify the `count` value in this record.

1. Save the file (**&lt;Ctrl>S**) and exit the editor (**&lt;Ctrl>Q**).

## Declare the variables

1. In Cloud Shell, create a file named `variables.tf`.

    ```bash
    code variables.tf
    ```

1. Paste the following code into the editor:

    ```hcl
    variable "client_id" {}
    variable "client_secret" {}

    variable "agent_count" {
        default = 3
    }

    variable "ssh_public_key" {
        default = "~/.ssh/id_rsa.pub"
    }

    variable "dns_prefix" {
        default = "k8stest"
    }

    variable cluster_name {
        default = "k8stest"
    }

    variable resource_group_name {
        default = "azure-k8stest"
    }

    variable location {
        default = "Central US"
    }

    variable log_analytics_workspace_name {
        default = "testLogAnalyticsWorkspaceName"
    }

    # refer https://azure.microsoft.com/global-infrastructure/services/?products=monitor for log analytics available regions
    variable log_analytics_workspace_location {
        default = "eastus"
    }

   # refer https://azure.microsoft.com/pricing/details/monitor/ for log analytics pricing 
   variable log_analytics_workspace_sku {
        default = "PerGB2018"
   }
    ```

1. Save the file (**&lt;Ctrl>S**) and exit the editor (**&lt;Ctrl>Q**).

## Create a Terraform output file

[Terraform outputs](https://www.terraform.io/docs/configuration/outputs.html) allow you to define values that will be highlighted to the user when Terraform applies a plan, and can be queried using the `terraform output` command. In this section, you create an output file that allows access to the cluster with [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/).

1. In Cloud Shell, create a file named `output.tf`.

    ```bash
    code output.tf
    ```

1. Paste the following code into the editor:

    ```hcl
    output "client_key" {
        value = azurerm_kubernetes_cluster.k8s.kube_config.0.client_key
    }

    output "client_certificate" {
        value = azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate
    }

    output "cluster_ca_certificate" {
        value = azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate
    }

    output "cluster_username" {
        value = azurerm_kubernetes_cluster.k8s.kube_config.0.username
    }

    output "cluster_password" {
        value = azurerm_kubernetes_cluster.k8s.kube_config.0.password
    }

    output "kube_config" {
        value = azurerm_kubernetes_cluster.k8s.kube_config_raw
    }

    output "host" {
        value = azurerm_kubernetes_cluster.k8s.kube_config.0.host
    }
    ```

1. Save the file (**&lt;Ctrl>S**) and exit the editor (**&lt;Ctrl>Q**).

## Set up Azure storage to store Terraform state

Terraform tracks state locally via the `terraform.tfstate` file. This pattern works well in a single-person environment. In a multi-person environment, [Azure storage](/azure/storage/) is used to track state.

In this section, you see how to do the following tasks:
- Retrieve storage account information (account name and account key)
- Create a storage container into which Terraform state information will be stored.

1. In the Azure portal, select **All services** in the left menu.

1. Select **Storage accounts**.

1. On the **Storage accounts** tab, select the name of the storage account into which Terraform is to store state. For example, you can use the storage account created when you opened Cloud Shell the first time.  The storage account name created by Cloud Shell typically starts with `cs` followed by a random string of numbers and letters. Take note of the storage account you select. This value is needed later.

1. On the storage account tab, select **Access keys**.

    ![Storage account menu](./media/create-k8s-cluster-with-tf-and-aks/storage-account.png)

1. Make note of the **key1** **key** value. (Selecting the icon to the right of the key copies the value to the clipboard.)

    ![Storage account access keys](./media/create-k8s-cluster-with-tf-and-aks/storage-account-access-key.png)

1. In Cloud Shell, create a container in your Azure storage account. Replace the placeholders with appropriate values for your environment.

    ```azurecli
    az storage container create -n tfstate --account-name <YourAzureStorageAccountName> --account-key <YourAzureStorageAccountKey>
    ```

## Create the Kubernetes cluster

In this section, you see how to use the `terraform init` command to create the resources defined in the configuration files you created in the previous sections.

1. In Cloud Shell, initialize Terraform. Replace the placeholders with appropriate values for your environment.

    ```bash
    terraform init -backend-config="storage_account_name=<YourAzureStorageAccountName>" -backend-config="container_name=tfstate" -backend-config="access_key=<YourStorageAccountAccessKey>" -backend-config="key=codelab.microsoft.tfstate" 
    ```
    
    The `terraform init` command displays the success of initializing the backend and provider plug-in:

    ![Example of "terraform init" results](./media/create-k8s-cluster-with-tf-and-aks/terraform-init-complete.png)

1. Export your service principal credentials. Replace the placeholders with appropriate values from your service principal.

    ```bash
    export TF_VAR_client_id=<service-principal-appid>
    export TF_VAR_client_secret=<service-principal-password>
    ```

1. Run the `terraform plan` command to create the Terraform plan that defines the infrastructure elements. 

    ```bash
    terraform plan -out out.plan
    ```

    The `terraform plan` command displays the resources that will be created when you run the `terraform apply` command:

    ![Example of "terraform plan" results](./media/create-k8s-cluster-with-tf-and-aks/terraform-plan-complete.png)

1. Run the `terraform apply` command to apply the plan to create the Kubernetes cluster. The process to create a Kubernetes cluster can take several minutes, resulting in the Cloud Shell session timing out. If the Cloud Shell session times out, you can follow the steps in the section "Recover from a Cloud Shell timeout" to enable you to complete the process.

    ```bash
    terraform apply out.plan
    ```

    The `terraform apply` command displays the results of creating the resources defined in your configuration files:

    ![Example of "terraform apply" results](./media/create-k8s-cluster-with-tf-and-aks/terraform-apply-complete.png)

1. In the Azure portal, select **All resources** in the left menu to see the resources created for your new Kubernetes cluster.

    ![All resources in the Azure portal](./media/create-k8s-cluster-with-tf-and-aks/k8s-resources-created.png)

## Recover from a Cloud Shell timeout

If the Cloud Shell session times out, you can do the following steps to recover:

1. Start a Cloud Shell session.

1. Change to the directory containing your Terraform configuration files.

    ```bash
    cd /clouddrive/terraform-aks-k8s
    ```

1. Run the following command:

    ```bash
    export KUBECONFIG=./azurek8s
    ```
    
## Test the Kubernetes cluster

The Kubernetes tools can be used to verify the newly created cluster.

1. Get the Kubernetes configuration from the Terraform state and store it in a file that kubectl can read.

    ```bash
    echo "$(terraform output kube_config)" > ./azurek8s
    ```

1. Set an environment variable so that kubectl picks up the correct config.

    ```bash
    export KUBECONFIG=./azurek8s
    ```

1. Verify the health of the cluster.

    ```bash
    kubectl get nodes
    ```

    You should see the details of your worker nodes, and they should all have a status **Ready**, as shown in the following image:

    ![The kubectl tool allows you to verify the health of your Kubernetes cluster](./media/create-k8s-cluster-with-tf-and-aks/kubectl-get-nodes.png)

## Monitor health and logs

When the AKS cluster was created, monitoring was enabled to capture health metrics for both the cluster nodes and pods. These health metrics are available in the Azure portal. For more information on container health monitoring,
see [Monitor Azure Kubernetes Service health](/azure/azure-monitor/insights/container-insights-overview).

[!INCLUDE [terraform-troubleshooting.md](includes/terraform-troubleshooting.md)]

## Next steps

> [!div class="nextstepaction"] 
> [Learn more about using Terraform in Azure](/azure/terraform)
