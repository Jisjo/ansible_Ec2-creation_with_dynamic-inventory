# Ansible Ec2 Creation with Dynamic Inventory

Here is a project for creating a ec2 instance with ansible using dynamic inventory. In our case static invetory will not work because the public/private IP of an EC2 instance will only know during the provisioning time. Using the dynamic invetory we are created 2nd play on the same playbook. It will deploy a simple HTML website.

## Overview

In this demo, we will be provisioning the following :

- Provision an SSH keypair, security groups and EC2 instance through Ansible.
- Retrieve the IP Address of instance during Ec2 creating and crearting dynamic inventory.
- Configuring the web server through Ansible using dynamic inventory.
