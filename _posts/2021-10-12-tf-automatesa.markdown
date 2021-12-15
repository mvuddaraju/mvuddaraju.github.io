---
layout: post
title:  "Automate storageaccount deployment using terraform"
description: "In this post, I walk through the automation of creating multiple storage accounts, file systems, and assigning ACLs to users/groups."
date:   2021-12-12 
categories: terraform, IaC, Azure, StorageAccounts, terraform modules, for_each loop and dynamic blocks
---

In this post, we will cover deploying multiple azure storage accounts with data objects and assigning ACLs using Terraform based on the following requirements.

1. Create multiple storage accounts with similar/custom properties.
2. Create gen2_filesystem as a storage data object ( data object created can be modified to suit your needs).
3. Assign ACLs at the data object level(gen2_filesystem) to different AD groups, users or technical accounts(Service Principal) : user:r-x, user:rwx, group:r-x, group:rwx
4. Use MSI(Managed Service Identity) for Terraform deployment(which eliminates the need for developers to manage credentials).
5. Promote the storage account creation to other environments (test, prod) by creating new variables with values relevant to the environment.

# Solution Overview

![Solution Overview](/images/solutionoverview.png)

This solution is implemented using modules, for_each loop, and dynamic block.  We will go through the key points in the coming sections.

# Sample git repository for this post

git repository with sample code is at https://github.com/mvuddaraju/azurestorageautomation-terraform. 
The file / folder structure is as follows:
![git file Structure](/images/gitfilestructure.png)

# terraform resources used

1. azurerm_resource_group                        : Creates resource group.
2. azurerm_storage_account                       : Creates storage account.
3. azurerm_role_assignment                       : Assign 'Key Vault Crypto User' role on keyvault to storageaccount.
4. azurerm_storage_account_customer_managed_key  : Enable encyption on storage account.
5. azurerm_management_lock                       : Create cannot delete lock on storage account.
4. azurerm_storage_data_lake_gen2_filesystem     : Create gen2_filesystem.

# Create multiple storageaccounts and gen2_filesystems

1. storage accounts are creating by looping through the variable "sa", defined as follows
{% highlight ruby %}
variable "sa" {
  description = "List of filesystems to create"
  type        = map
  default     = {
    sa1 = ["fs1","fs2"],
    sa2 = ["fs1","fs2"]
  }
}
#=> example
sa = {   
    "storage1" = ["st1fs1","st1fs2"]
    "storage2" = ["st2fs1"]
    "storage3" = ["st3fs1", "st4fs2", "st3fs3", "st3fs4"]
    "storage4" = [] 
}
{% endhighlight %}
All storage accounts and file systems that need to be created are defined as variables of type map, where the key is the name of the storage account and value is a list of filesystems to be created under each storage account.
main.tf calls the module storage using for_each loop and passes as input storage account name (KEY) and fs(VALUE).
Another for_each loop in the storage module will create filesystems under each storage account.

# Assign ACLs to gen2_filesystem

ACLs are set at the filesystem level, using the following variable definition
{% highlight ruby %}
variable "fsacl" {
  description = "List of acl to create for each fs"
  type        = map(object({
    grpread: list(string),
    grpwrite: list(string),
    usrread: list(string),
    usrwrite: list(string)
  }))
  default     = {
    "fs" = { 
       grpread = null
       grpwrite = null
       usrread = null
       usrwrite = null
    } 
  }
}

#=> example
fsacl = {
    "st1fs1" = {                         
                                   grpread = ["group1", "group2" ]
                                   grpwrite = ["group2"]
                                   usrread = ["user1"]
                                   usrwrite = []
                                  
                }
{% endhighlight %}
ACLs are divided into four categories: userread, userwrite, groupread, and groupwrite. To each category, a list of users/groups that require read/readwrite permissions is assigned. 
A dynamic block is defined for each of the above categories that will assign respective ACL t the user or group.

An example dynamic block is detailed below
{% highlight ruby %}
dynamic "ace" {
                     for_each =  setproduct((lookup(var.fsacl, each.value, null) == null ? [] : lookup(lookup(var.fsacl, each.value, null),"usrwrite",[]) ),local.type)                               
                     content {
                                type = "user"
                                scope = ace.value.1
                                id = lookup(var.objectid,ace.value.0,"")
                                permissions = "rwx"
                                                       
                            }
                }                     
           
}
{% endhighlight %}
Dynamic block assigning ACLs is part of azurerm_storage_data_lake_gen2_filesystem, runs for_each(gen2filesystem). we look up the filesystem in "fsacl" and retrieve the list of users to assign write permissions. 

A local variable is defined to handle ACL scopes: access, default.
{% highlight ruby %}
 locals {
  scopetype = [ "access" , "default" ]
}
{% endhighlight %}
We are using setproduct in the dynamic block to assign ACL to the user/group at both scope levels.

# Setting custom properties for the storage accounts

