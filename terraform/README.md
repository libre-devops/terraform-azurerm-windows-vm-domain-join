```hcl
module "rg" {
  source = "registry.terraform.io/libre-devops/rg/azurerm"

  rg_name  = "rg-${var.short}-${var.loc}-${terraform.workspace}-build" // rg-ldo-euw-dev-build
  location = local.location                                            // compares var.loc with the var.regions var to match a long-hand name, in this case, "euw", so "westeurope"
  tags     = local.tags

  #  lock_level = "CanNotDelete" // Do not set this value to skip lock
}

module "network" {
  source = "registry.terraform.io/libre-devops/network/azurerm"

  rg_name  = module.rg.rg_name // rg-ldo-euw-dev-build
  location = module.rg.rg_location
  tags     = local.tags

  vnet_name     = "vnet-${var.short}-${var.loc}-${terraform.workspace}-01" // vnet-ldo-euw-dev-01
  vnet_location = module.network.vnet_location

  address_space   = ["10.0.0.0/16"]
  subnet_prefixes = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  subnet_names    = ["sn1-${module.network.vnet_name}", "sn2-${module.network.vnet_name}", "sn3-${module.network.vnet_name}"] //sn1-vnet-ldo-euw-dev-01
  subnet_service_endpoints = {
    "sn1-${module.network.vnet_name}" = ["Microsoft.Storage"] // Adds extra subnet endpoints to sn1-vnet-ldo-euw-dev-01
    "sn2-${module.network.vnet_name}" = ["Microsoft.Storage", "Microsoft.Sql"], // Adds extra subnet endpoints to sn2-vnet-ldo-euw-dev-01
    "sn3-${module.network.vnet_name}" = ["Microsoft.AzureActiveDirectory"] // Adds extra subnet endpoints to sn3-vnet-ldo-euw-dev-01
  }
}

// Default behaviour uses "registry.terraform.io/libre-devops/windows-os-plan-calculator/azurerm"
module "win_vm_simple" {
  source = "registry.terraform.io/libre-devops/windows-vm/azurerm"

  rg_name  = module.rg.rg_name
  location = module.rg.rg_location
  tags     = module.rg.rg_tags

  vm_amount          = 1
  vm_hostname        = "win${var.short}${var.loc}${terraform.workspace}" // winldoeuwdev01 & winldoeuwdev02 & winldoeuwdev03
  vm_size            = "Standard_B2ms"
  use_simple_image   = true
  vm_os_simple       = "WindowsServer2019"
  vm_os_disk_size_gb = "127"

  asg_name = "asg-${element(regexall("[a-z]+", element(module.win_vm_simple.vm_name, 0)), 0)}-${var.short}-${var.loc}-${terraform.workspace}-01" //asg-vmldoeuwdev-ldo-euw-dev-01 - Regex strips all numbers from string

  admin_username = "LibreDevOpsAdmin"
  admin_password = data.azurerm_key_vault_secret.mgmt_local_admin_pwd.value // Created with the Libre DevOps Terraform Pre-Requisite script

  subnet_id            = element(values(module.network.subnets_ids), 0) // Places in sn1-vnet-ldo-euw-dev-01
  availability_zone    = "alternate"                                    // If more than 1 VM exists, places them in alterate zones, 1, 2, 3 then resetting.  If you want HA, use an availability set.
  storage_account_type = "Standard_LRS"
  identity_type        = "SystemAssigned"
}

module "domain_join" {
  source = "registry.terraform.io/libre-devops/windows-vm-domain-join/azurerm"

  attempt_restart       = "true"
  domain_admin_password = data.azurerm_key_vault_secret.mgmt_local_admin_pwd.value
  domain_admin_username = "LibreDevOpsAdmin"
  domain_name           = "libredevops.org"
  ou_path               = "OU=${title(terraform.workspace)},OU=Customers,OU=Computers,DC=libredevops,DC=org"
  vm_id                 = element(values(module.win_vm_simple.vm_ids), 0)
  tags                  = module.rg.rg_tags
}

module "domain_join_with_lookup" {
  source = "registry.terraform.io/libre-devops/windows-vm-domain-join/azurerm"

  attempt_restart       = "true"
  domain_admin_password = data.azurerm_key_vault_secret.mgmt_local_admin_pwd.value
  domain_admin_username = "LibreDevOpsAdmin"
  domain_name           = "libredevops.org"
  ou_path               = "OU=${title(terraform.workspace)},OU=Customers,OU=Computers,DC=libredevops,DC=org"
  tags                  = module.rg.rg_tags

  lookup_vm_id          = true
  vm_name               = "vmlbdouksprd01"
  rg_name               = "rg-lbdo-uks-prd-01"
}

```

## Requirements

No requirements.

## Providers

| Name | Version |
|------|---------|
| <a name="provider_azurerm"></a> [azurerm](#provider\_azurerm) | n/a |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [azurerm_application_security_group.example_asg](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/application_security_group) | resource |
| [azurerm_resource_group.example_rg](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group) | resource |
| [azurerm_key_vault.mgmt_kv](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/key_vault) | data source |
| [azurerm_key_vault_secret.mgmt_local_admin_pwd](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/key_vault_secret) | data source |
| [azurerm_resource_group.mgmt_rg](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/resource_group) | data source |
| [azurerm_ssh_public_key.mgmt_ssh_key](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/ssh_public_key) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_AZURE_BACKEND_SA_KEY"></a> [AZURE\_BACKEND\_SA\_KEY](#input\_AZURE\_BACKEND\_SA\_KEY) | This is passed as an environment variable, it is for the state backend storage | `string` | n/a | yes |
| <a name="input_AZURE_BACKEND_SA_NAME"></a> [AZURE\_BACKEND\_SA\_NAME](#input\_AZURE\_BACKEND\_SA\_NAME) | This is passed as an environment variable, it is for the state backend storage | `string` | n/a | yes |
| <a name="input_AZURE_CLIENT_ID"></a> [AZURE\_CLIENT\_ID](#input\_AZURE\_CLIENT\_ID) | This is passed as an environment variable, it is for the client ID of the service principle | `string` | n/a | yes |
| <a name="input_AZURE_CLIENT_SECRET"></a> [AZURE\_CLIENT\_SECRET](#input\_AZURE\_CLIENT\_SECRET) | This is passed as an environment variable, it is for the client secret of the service principle | `string` | n/a | yes |
| <a name="input_AZURE_SUBSCRIPTION_ID"></a> [AZURE\_SUBSCRIPTION\_ID](#input\_AZURE\_SUBSCRIPTION\_ID) | This is passed as an environment variable, it is for the target subscription | `string` | n/a | yes |
| <a name="input_AZURE_TENANT_ID"></a> [AZURE\_TENANT\_ID](#input\_AZURE\_TENANT\_ID) | n/a | `string` | `"This is passed as an environment variable, it is for the Azure tenant ID"` | no |
| <a name="input_env"></a> [env](#input\_env) | This is passed as an environment variable, it is for the shorthand environment tag for resource.  For example, production = prod | `string` | n/a | yes |
| <a name="input_loc"></a> [loc](#input\_loc) | The shorthand name of the Azure location, for example, for UK South, use uks.  For UK West, use ukw | `string` | n/a | yes |
| <a name="input_regions"></a> [regions](#input\_regions) | Long-hand names of regions in terraform | `map(string)` | <pre>{<br>  "eus": "East US",<br>  "euw": "West Europe",<br>  "uks": "UK South",<br>  "ukw": "UK West"<br>}</pre> | no |
| <a name="input_short"></a> [short](#input\_short) | This is passed as an environment variable, it is for a shorthand name for the environment, for example hello-world = hw | `string` | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_non_sensitive"></a> [non\_sensitive](#output\_non\_sensitive) | A non sensitive value |
| <a name="output_sensitive"></a> [sensitive](#output\_sensitive) | Sensitive |
