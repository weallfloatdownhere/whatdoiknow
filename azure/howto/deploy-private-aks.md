# How To Deploy a Private AKS Cluster With Terraform that (optionally) uses Azure Key Vault.

# Getting started
Here are the resources that we’ll create in Azure:

* 2 virtual networks: vnet-hub and vnet-aks
* A bastion
* A Container Registry with private endpoint to ensure private traffic
* A Virtual Machine that we’ll use as an Azure DevOps agent (our cluster being private, we cannot use Azure DevOps hosted agents)
* A private DNS zone for the cluster: privatelink.canadacentral.azmk8s.io
* A private DNS zone for the container registry: privatelink.azurecr.io
* A private DNS zone for the Key Vault: privatelink.vaultcore.azure.net
* An Application Gateway with a public and a private IP address (some applications will be exposed on the internet, some only internally)
* A Key Vault with private endpoint to ensure private traffic
* Our AKS cluster with 3 worker nodes

</br>

![aks](https://images.squarespace-cdn.com/content/v1/5dd2ea709a1e462ada017e94/3ed7bcb9-b9a3-4732-bcb6-0bd27849129d/Azure+Kubernetes+Service.jpeg?format=2500w)

## Setup Terraform With Azure

1. To install Terraform, download the binary file and add it to a directory included in your system’s PATH.

2. Install Azure CLI.

3. Connect to Azure and choose the subscription where you want to deploy the solution.

```shell
$ az login
$ az account set --subscription {your subscription ID}
```

4. Create an Azure storage account with az cli. We’ll use this storage to save the Terraform state.

```shell
$resourceGroupName = 'rg-tfstate-cac'
$storageAccountName = 'satfstatecac'
$location = 'canadacentral'

$ az group create --location $location --name $resourceGroupName
$ az storage account create --name $storageAccountName --resource-group $resourceGroupName --location $location --sku Standard_LRS
$ az storage container create --name state --account-name $storageAccountName
$ az storage account blob-service-properties update --account-name $storageAccountName --enable-change-feed --enable-versioning true
```

5. Configure Terraform to save the state to the storage account.

Create a new folder private-aks-cluster. We’ll use this folder as our working directory.

Add the following code to a main.tf file. Replace {your subscription id} with the Azure subscription ID that you used at step 3.

```groovy
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = ">=3.25.0"
    }
  }

  backend "azurerm" {
    subscription_id      = "{your subscription id}"
    resource_group_name  = "rg-tfstate-cac"
    storage_account_name = "satfstatecac"
    container_name       = "state"
    key                  = "terraform.tfstate"
  }
}

provider "azurerm" {
  subscription_id = "{your subscription id}"
  features {}
}
```

Run the command terraform init. You should see a message saying Terraform has been successfully initialized!

Our environment is now ready and we can start creating resources in Azure with Terraform.

## Create The Resource Group
We’ll add all the resources in a resource group called rg-pvaks-cac.

In the main.tf file, add the following code.

```groovy
resource "azurerm_resource_group" "rg" {
  name     = "rg-pvaks-cac"
  location = "canadacentral"
}
```

## Create The Virtual Networks
Add a file vnets.tf and insert the following code to create the VNETs and peer them together.

```groovy
resource "azurerm_virtual_network" "vnet_hub" {
  name                = "vnet-hub"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_virtual_network" "vnet_aks" {
  name                = "vnet-aks"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.1.0.0/16"]
}

resource "azurerm_virtual_network_peering" "to_vnet_aks" {
  name                         = "peer-to-vnet-aks"
  resource_group_name          = azurerm_resource_group.rg.name
  virtual_network_name         = azurerm_virtual_network.vnet_hub.name
  remote_virtual_network_id    = azurerm_virtual_network.vnet_aks.id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = true
}

resource "azurerm_virtual_network_peering" "to_vnet_hub" {
  name                         = "peer-to-vnet-hub"
  resource_group_name          = azurerm_resource_group.rg.name
  virtual_network_name         = azurerm_virtual_network.vnet_aks.name
  remote_virtual_network_id    = azurerm_virtual_network.vnet_hub.id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
}
```

## Create The Subnets
Update the vnets.tf file to include the subnets. Note the parameter enforce_private_link_endpoint_network_policies required for the subnets where private link will be added.

```groovy
resource "azurerm_virtual_network" "vnet_hub" {
  name                = "vnet-hub"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_virtual_network" "vnet_aks" {
  name                = "vnet-aks"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.1.0.0/16"]
}

resource "azurerm_virtual_network_peering" "to_vnet_aks" {
  name                         = "peer-to-vnet-aks"
  resource_group_name          = azurerm_resource_group.rg.name
  virtual_network_name         = azurerm_virtual_network.vnet_hub.name
  remote_virtual_network_id    = azurerm_virtual_network.vnet_aks.id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = true
}

resource "azurerm_virtual_network_peering" "to_vnet_hub" {
  name                         = "peer-to-vnet-hub"
  resource_group_name          = azurerm_resource_group.rg.name
  virtual_network_name         = azurerm_virtual_network.vnet_aks.name
  remote_virtual_network_id    = azurerm_virtual_network.vnet_hub.id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
}

resource "azurerm_subnet" "bastion" {
  name                 = "AzureBastionSubnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet_hub.name
  address_prefixes     = ["10.0.0.0/27"]
}

resource "azurerm_subnet" "global" {
  name                                      = "snet-global"
  resource_group_name                       = azurerm_resource_group.rg.name
  virtual_network_name                      = azurerm_virtual_network.vnet_hub.name
  address_prefixes                          = ["10.0.1.0/24"]
  private_endpoint_network_policies_enabled = true
}

resource "azurerm_subnet" "agw" {
  name                 = "snet-agw"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet_aks.name
  address_prefixes     = ["10.1.0.0/24"]
}

resource "azurerm_subnet" "aks" {
  name                                      = "snet-aks"
  resource_group_name                       = azurerm_resource_group.rg.name
  virtual_network_name                      = azurerm_virtual_network.vnet_aks.name
  address_prefixes                          = ["10.1.1.0/24"]
  private_endpoint_network_policies_enabled = true
}

resource "azurerm_subnet" "utils" {
  name                                      = "snet-utils"
  resource_group_name                       = azurerm_resource_group.rg.name
  virtual_network_name                      = azurerm_virtual_network.vnet_aks.name
  address_prefixes                          = ["10.1.2.0/24"]
  private_endpoint_network_policies_enabled = true
}
```

## Create The Bastion
Add the bas.tf file with the following code.

```groovy
resource "azurerm_public_ip" "bas" {
  name                = "pip-bas-cac-001"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_bastion_host" "bas" {
  name                = "bas-pvaks-cac-001"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                 = "configuration"
    subnet_id            = azurerm_subnet.bastion.id
    public_ip_address_id = azurerm_public_ip.bas.id
  }
}
```

## Create The Virtual Machine
Add the file vm.tf with the following code.

```groovy
resource "azurerm_network_interface" "nic" {
  name                = "nic-vm-1"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.global.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_windows_virtual_machine" "vm1" {
  name                = "vm-1"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_D2s_v3"
  admin_username      = var.vm_username
  admin_password      = var.vm_password
  network_interface_ids = [
    azurerm_network_interface.nic.id,
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "MicrosoftWindowsDesktop"
    offer     = "Windows-10"
    sku       = "21h1-ent"
    version   = "latest"
  }
}

variable "vm_username" {
  type        = string
  description = "Username for vm-1"
}

variable "vm_password" {
  type        = string
  sensitive   = true
  description = "Password for vm-1"
}
```

## Create The Container Registry
In this section, we’ll:

* Create the registry

* Create the private DNS zone for the registry

* Link the private DNS zone with the vnets

* Add a private endpoint for the registry

To do so, add the acr.tf file with the following code.

```groovy
resource "azurerm_container_registry" "acr" {
  name                          = "acrpvakscac"
  resource_group_name           = azurerm_resource_group.rg.name
  location                      = azurerm_resource_group.rg.location
  sku                           = "Premium"
  admin_enabled                 = false
  public_network_access_enabled = false
}

resource "azurerm_private_dns_zone" "acr" {
  name                = "privatelink.azurecr.io"
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "acr1" {
  name                  = "pdznl-acr-cac-001"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.acr.name
  virtual_network_id    = azurerm_virtual_network.vnet_hub.id
}

resource "azurerm_private_dns_zone_virtual_network_link" "acr2" {
  name                  = "pdznl-acr-cac-002"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.acr.name
  virtual_network_id    = azurerm_virtual_network.vnet_aks.id
}

resource "azurerm_private_endpoint" "acr" {
  name                = "pe-acr-cac-001"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  subnet_id           = azurerm_subnet.global.id

  private_service_connection {
    name                           = "psc-acr-cac-001"
    private_connection_resource_id = azurerm_container_registry.acr.id
    subresource_names              = ["registry"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "pdzg-acr-cac-001"
    private_dns_zone_ids = [azurerm_private_dns_zone.acr.id]
  }
}
```

## Create The Application Gateway
The application gateway will be managed by AKS with the AGIC addon.

The application gateway will have a public IP address and a private IP address.

The private IP address will be 10.1.0.4. The public IP address will be determined by Microsoft.

Add the file agw.tf with the following code to create the application gateway.

```groovy
locals {
  backend_address_pool_name              = "bapn-pvaks"
  frontend_port_name                     = "fpn-pvaks"
  private_frontend_ip_configuration_name = "ficn-pvaks-private"
  public_frontend_ip_configuration_name  = "ficn-pvaks-public"
  http_setting_name                      = "hsn-pvaks"
  private_listener_name                  = "ln-pvaks-http-private"
  public_listener_name                   = "ln-pvaks-http-public"
  request_routing_rule_name              = "rrrn-pvaks"
  redirect_configuration_name            = "rrn-pvaks"
}

resource "azurerm_public_ip" "ip" {
  name                = "pip-pvaks"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_application_gateway" "agw" {
  name                = "agw-pvaks"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location

  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
    capacity = 1
  }

  gateway_ip_configuration {
    name      = "ipc-pvaks"
    subnet_id = azurerm_subnet.agw.id
  }

  frontend_port {
    name = local.frontend_port_name
    port = 80
  }

  frontend_ip_configuration {
    name                 = local.public_frontend_ip_configuration_name
    public_ip_address_id = azurerm_public_ip.ip.id
  }

  frontend_ip_configuration {
    name                          = local.private_frontend_ip_configuration_name
    private_ip_address            = "10.1.0.4"
    private_ip_address_allocation = "Static"
    subnet_id                     = azurerm_subnet.agw.id
  }

  backend_address_pool {
    name = local.backend_address_pool_name
  }

  backend_http_settings {
    name                  = local.http_setting_name
    cookie_based_affinity = "Disabled"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 60
  }

  http_listener {
    name                           = local.public_listener_name
    frontend_ip_configuration_name = local.public_frontend_ip_configuration_name
    frontend_port_name             = local.frontend_port_name
    protocol                       = "Http"
  }

  request_routing_rule {
    name                       = local.request_routing_rule_name
    rule_type                  = "Basic"
    http_listener_name         = local.public_listener_name
    backend_address_pool_name  = local.backend_address_pool_name
    backend_http_settings_name = local.http_setting_name
    priority                   = 1
  }

  #ignore changes since AGW is managed by AGIC
  lifecycle {
    ignore_changes = [
      tags,
      backend_address_pool,
      backend_http_settings,
      frontend_port,
      http_listener,
      probe,
      redirect_configuration,
      request_routing_rule,
      ssl_certificate
    ]
  }
}
```

## Create a Log Analytics Workspace
All the AKS logs can be sent to a log analytics workspace with the OMS agent addon.

Create the workpace by adding the file logs.tf with the following code.

```groovy
resource "azurerm_log_analytics_workspace" "log" {
  name                = "log-pvaks-cac-001"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}
```

Create The Cluster
First, we create the private DNS zone and link it with the vnets.

Azure will create some identities like the kubelet identity automatically but we need to create some manually:

* an identity for the AKS cluster (used by the control plane)

* an identity for the pod (used by the aad-pod-identity addon, it allows a kubernetes pod to access Azure Key Vault)

After creating thoses identities, we assign the proper role to each one:

* DNS contributor for the AKS identity (used when updating the private DNS zone)

* Network Contributor for the AKS identity (used when creating different network resources like Load Balancers)

* AcrPull for the kubelet identity (used to pull the container images from the registry)

* Contributor for the Application Gateway identity (this identity is created when installing the AGIC addon. It is used to configure the Application Gateway)

* Monitor Metric Publisher for the OMS agent identity (used to send the logs to the log analytics workspace)

Finally, we create the cluster with 3 worker nodes.

Add the aks.tf file with the following code.

```groovy
### DNS zone
resource "azurerm_private_dns_zone" "aks" {
  name                = "privatelink.canadacentral.azmk8s.io"
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "aks1" {
  name                  = "pdzvnl-aks-cac-001"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.aks.name
  virtual_network_id    = azurerm_virtual_network.vnet_hub.id
}

resource "azurerm_private_dns_zone_virtual_network_link" "aks2" {
  name                  = "pdzvnl-aks-cac-002"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.aks.name
  virtual_network_id    = azurerm_virtual_network.vnet_aks.id
}

### Identity
resource "azurerm_user_assigned_identity" "aks" {
  name                = "id-aks-cac-001"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
}

resource "azurerm_user_assigned_identity" "pod" {
  name                = "id-pod-cac-001"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
}

### Identity role assignment
resource "azurerm_role_assignment" "dns_contributor" {
  scope                = azurerm_private_dns_zone.aks.id
  role_definition_name = "Private DNS Zone Contributor"
  principal_id         = azurerm_user_assigned_identity.aks.principal_id
}

resource "azurerm_role_assignment" "network_contributor" {
  scope                = azurerm_virtual_network.vnet_aks.id
  role_definition_name = "Network Contributor"
  principal_id         = azurerm_user_assigned_identity.aks.principal_id
}

resource "azurerm_role_assignment" "acr" {
  scope                = azurerm_container_registry.acr.id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
}

resource "azurerm_role_assignment" "agw" {
  scope                = azurerm_resource_group.rg.id
  role_definition_name = "Contributor"
  principal_id         = azurerm_kubernetes_cluster.aks.ingress_application_gateway[0].ingress_application_gateway_identity[0].object_id
}

resource "azurerm_role_assignment" "monitoring" {
  scope                = azurerm_kubernetes_cluster.aks.id
  role_definition_name = "Monitoring Metrics Publisher"
  principal_id         = azurerm_kubernetes_cluster.aks.oms_agent[0].oms_agent_identity[0].object_id
}

### AKS cluster creation
resource "azurerm_kubernetes_cluster" "aks" {
  name                       = "aks-pvaks-cac-001"
  location                   = azurerm_resource_group.rg.location
  resource_group_name        = azurerm_resource_group.rg.name
  dns_prefix_private_cluster = "aks-pvaks-cac-001"
  private_cluster_enabled    = true
  private_dns_zone_id        = azurerm_private_dns_zone.aks.id

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.aks.id]
  }

  default_node_pool {
    name           = "default"
    node_count     = 3
    vm_size        = "Standard_D2_v2"
    vnet_subnet_id = azurerm_subnet.aks.id
  }

  network_profile {
    network_plugin     = "azure"
    dns_service_ip     = "10.1.3.4"
    docker_bridge_cidr = "172.16.0.1/16"
    service_cidr       = "10.1.3.0/24"
  }

  ingress_application_gateway {
    gateway_id = azurerm_application_gateway.agw.id
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.log.id
  }

  depends_on = [
    azurerm_role_assignment.network_contributor,
    azurerm_role_assignment.dns_contributor
  ]
}
```

## Create The Key Vault
Applications deployed to AKS can access the key vault with the pod identity in order to retrieve secrets.

To create the key vault, add the kv.tf file with the following code.

```groovy
data "azurerm_client_config" "current" {}

resource "azurerm_key_vault" "kv" {
  name                = "kv-pvaks-cac-001"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  tenant_id           = data.azurerm_client_config.current.tenant_id

  sku_name = "standard"

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = azurerm_user_assigned_identity.pod.principal_id

    secret_permissions = [
      "Get", "List",
    ]
  }
}

resource "azurerm_private_dns_zone" "kv" {
  name                = "privatelink.vaultcore.azure.net"
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "kv1" {
  name                  = "pdznl-vault-cac-001"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.kv.name
  virtual_network_id    = azurerm_virtual_network.vnet_aks.id
}

resource "azurerm_private_dns_zone_virtual_network_link" "kv2" {
  name                  = "pdznl-vault-cac-002"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.kv.name
  virtual_network_id    = azurerm_virtual_network.vnet_hub.id
}

resource "azurerm_private_endpoint" "kv" {
  name                = "pe-vault-cac-001"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  subnet_id           = azurerm_subnet.utils.id

  private_service_connection {
    name                           = "psc-vault-cac-001"
    private_connection_resource_id = azurerm_key_vault.kv.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "pdzg-vault-cac-001"
    private_dns_zone_ids = [azurerm_private_dns_zone.kv.id]
  }
}
```

## Conclusion
We have created the code for all the resources. You can now run the command terraform apply to create the resources.

You will be prompted to enter a username and password for vm-1. If you prefer, you can create a terraform.tfvars to initialize the variables.

The file looks like this.

```yaml
vm_username=some_username
vm_password=SomeRe@llySecureP@ssw0rdToNotUseIn2021
```

Terraform will also ask you to confirm the deployment by typing yes.

And… that’s it. You now have a complete solution to deploy a fully private AKS cluster with Terraform.

The code is available on github.

In a next article, we’ll see how to deploy a web application to this cluster with Azure DevOps.