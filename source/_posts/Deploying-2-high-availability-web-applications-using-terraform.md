---
title: Deploying 2 high availability web applications using terraform
date: 2023-02-13 18:39:03
tags:
categories: [Blogs]
---

# Deploying 2 high availability web applications using terraform

Overview :

Hello, in this article we are going to create step by step two virtual machines to run our api and then create a loadbalancer and link it to them .

This is the architecture that we are going to build :

![](https://cdn-images-1.medium.com/max/2000/1*eZ68clEp2x7E36iEZLDxyQ.png)

First of all, let's talk about the idea behind Infrastructure as code and Terraform.

### Infrastructure as code :

Infrastructure as Code (IaC) is the managing and provisioning of infrastructure through code instead of through manual processes.

With IaC, configuration files are created that contain your infrastructure specifications, which makes it easier to edit and distribute configurations. It also ensures that you provision the same environment every time.

### Terraform :

![](https://cdn-images-1.medium.com/max/2000/1*i2XsJPq5gQ9N_kgDmQzksw.png)

HashiCorp Terraform is an infrastructure as code tool that lets you define both cloud and on-prem resources in human-readable configuration files that you can version, reuse, and share.

Now, let's explain our script main.tf which will hold the resources for our infrastructure :

```bash
 terraform {
   required_providers {
     azurerm = {
       source  = "hashicorp/azurerm"
       version = "~> 3.0.0"
     }
   }
   required_version = ">= 0.14.9"
 }

 provider "azurerm" {
   # Configuration options
   features {}
   subscription_id = var.subscription_id
 }
```

Here we are going to use the ┬½ azurerm ┬╗ provider and we set it to use our azure subsrcition id.

Then we start creating our resource group, then create our Virtual network which will act as private cloud for our machines :

```bash
 resource "azurerm_resource_group" "notes-api-azure-resource-group" {
   name     = "my-notes-api-rg"
   location = "West Europe"
 }

 resource "azurerm_virtual_network" "notes-api-virtual-network" {
   name                = "my-virtual-network"
   location            = azurerm_resource_group.notes-api-azure-resource-group.location
   address_space       = ["10.0.0.0/16"]
   resource_group_name = azurerm_resource_group.notes-api-azure-resource-group.name
 }
```

We need to create our virtual machines now :

```bash
resource "azurerm_subnet" "notes-api-subnet" {
  name                 = "my-subnet"
  resource_group_name  = azurerm_resource_group.notes-api-azure-resource-group.name
  virtual_network_name = azurerm_virtual_network.notes-api-virtual-network.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_interface" "notes-api-network-interface" {
  count               = 2
  name                = "acctni${count.index}"
  resource_group_name = azurerm_resource_group.notes-api-azure-resource-group.name
  location            = azurerm_resource_group.notes-api-azure-resource-group.location
  ip_configuration {
    name                          = "testConfiguration"
    subnet_id                     = azurerm_subnet.notes-api-subnet.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_availability_set" "notes-api-aset" {
  name                         = "my-availability-set"
  location                     = azurerm_resource_group.notes-api-azure-resource-group.location
  resource_group_name          = azurerm_resource_group.notes-api-azure-resource-group.name
  platform_fault_domain_count  = 2
  platform_update_domain_count = 2
  managed                      = true
}

resource "azurerm_linux_virtual_machine" "notes-api-vm" {
  count                 = 2
  name                  = "acctvm${count.index}"
  location              = azurerm_resource_group.notes-api-azure-resource-group.location
  availability_set_id   = azurerm_availability_set.notes-api-aset.id
  resource_group_name   = azurerm_resource_group.notes-api-azure-resource-group.name
  network_interface_ids = [element(azurerm_network_interface.notes-api-network-interface.*.id, count.index)]
  size                  = "Standard_B1s"

  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }
  admin_ssh_key {
    username   = "azureuser"
    public_key = file(var.public_key_file)
  }

  os_disk {
    name                 = "os_disk_${count.index}"
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  computer_name                   = "myvm${count.index}"
  admin_username                  = "azureuser"
  disable_password_authentication = true

  provisioner "remote-exec" {
    inline = [
      "export PORT=${var.PORT}",
      "export DB_URI=${var.DB_URI}",
      "sudo apt update",
      "curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.2/install.sh | bash",
      "chmod +x ~/.nvm/nvm.sh",
      "source ~/.bashrc",
      "nvm install 16",
      "node -v",
      "sudo apt install npm -y",
      "sudo apt install git -y",
      "mkdir /app",
      "cd app ",
      "git clone ${var.repo_url}",
      "cd notes-api",
      "npm i ",
      "npm run start:dev",
    ]
    connection {
      type        = "ssh"
      user        = "azureuser"
      port        = 22
      host        = azurerm_public_ip.notes-api-public-ip.ip_address
      private_key = file(var.private_key_file)
    }

  }

  tags = {
    environment = "production :D"
  }
}
```

We created the subnet for the virtual machines inside the virtual network, we setup the availability set, the 2 network interfaces for our machines and finally the machines which are Ubuntu Servers.

> Note: We used a provisioner to inject the api inside the machines and make it run .

Then we create a public ip and assign to a loadbalancer :

```bash
#public ip for azure loadbalancer
resource "azurerm_public_ip" "notes-api-public-ip" {
name = "my-public-ip"
resource_group_name = azurerm_resource_group.notes-api-azure-resource-group.name
location = azurerm_resource_group.notes-api-azure-resource-group.location
allocation_method = "Static"
}

#the loadbalancer itself
resource "azurerm_lb" "notes-api-loadbalancer" {
    name = "my-azure-loadbalancer"
    resource_group_name = azurerm_resource_group.notes-api-azure-resource-group.name
    location = azurerm_resource_group.notes-api-azure-resource-group.location
    frontend_ip_configuration {
    name = "publicIPAddress"
    public_ip_address_id = azurerm_public_ip.notes-api-public-ip.id
    }
}
```

Now after we configured the frontend for the loadbalancer, we need to set up the backend pool address :

```bash
resource "azurerm_lb_backend_address_pool" "notes-api-lb-backend-address-pool" {
    loadbalancer_id = azurerm_lb.notes-api-loadbalancer.id
    name = "BackEndAddressPool"
}
```

# for 1st vm

```bash
resource "azurerm_lb_backend_address_pool_address" "notes-api-lb-backend-address-pool-address" {
    name = "BackEndAddressPoolAddress"
    backend_address_pool_id = azurerm_lb_backend_address_pool.notes-api-lb-backend-address-pool.id
    virtual_network_id = azurerm_virtual_network.notes-api-virtual-network.id
    ip_address = "10.0.1.4"
}
```

# for 2nd vm

```bash
resource "azurerm_lb_backend_address_pool_address" "notes-api-lb-backend-address-pool-address" {
    name = "BackEndAddressPoolAddress"
    backend_address_pool_id = azurerm_lb_backend_address_pool.notes-api-lb-backend-address-pool.id
    virtual_network_id = azurerm_virtual_network.notes-api-virtual-network.id
    ip_address = "10.0.1.5"
}
```

After that, we add some NAT inboud rules so we could ssh to our virtual machines by passing through the loadbalancer (i'm still a newbie so i added them through the azure portal) :

![](https://cdn-images-1.medium.com/max/2000/1*vKpdwyIGJpd6FJse2wFBzA.png)

![](https://cdn-images-1.medium.com/max/2000/1*Ef95Os6OxSmUfNr_4v0hFw.png)

### Testing out things :

> ssh azureuser@40.68.58.158 -i ssh-key.pem

![](https://cdn-images-1.medium.com/max/2000/1*A8lm7bsUc3rPIkbps5DMSA.png)

> ssh -p 2222 azureuser@40.68.58.158 -i ssh-key.pem

![](https://cdn-images-1.medium.com/max/2000/1*gjlE_dmUT_bVv4dZ5o0dCQ.png)

And everything seems to be working perfectly .

Fo the last step, we setup the loadbalancig rule as below :

![](https://cdn-images-1.medium.com/max/2000/1*nbo1E3lRJBBsh-Hx22UI6w.png)

Let's test the api now :

![](https://cdn-images-1.medium.com/max/2000/1*WCN-p-e4trzfTRXaPopuiQ.png)

And voila everything seems working perfectly !

To conclude, we learnt in this article about Infrastructure as code, its most popular open-source tool Terraform and the way to deploy two high availability virtual machines with a public ip loadbalancer .

Was this helpful ? Confusing ? If you have any questions, feel free to comment below ! Make sure to follow on Linkedin : [https://www.linkedin.com/in/malekzaag/](https://www.linkedin.com/in/malekzaag/) and github : [https://github.com/Malek-Zaag](https://github.com/Malek-Zaag) if you're interested in similar content and want to keep learning alongside me!
