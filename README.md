# Ansible Ec2 Creation with Dynamic Inventory

Here is a project for creating an ec2 instance with ansible using dynamic inventory. In our case, static inventory will not work because the public/private IP of an EC2 instance will only know during the provisioning time. Using the dynamic inventory we are created 2nd play on the same playbook. It will deploy a simple HTML website.

## Overview

In this demo, we will be provisioning the following :

- Provision an SSH keypair, security groups, and EC2 instance through Ansible.
- Retrieve the IP Address of instance during Ec2 creating and creating dynamic inventory.
- Configuring the webserver through Ansible using dynamic inventory.

## Prerequisite

- Access key and secret key for AWS IAM user with administrator access.
- Install Ansible on your machine (ansible master).
- Install python module boto, boto3 and botocore on ansible master server.
> Thses module help ansible to do  Python API for AWS infrastructure services to perfom its task.
```
pip2 install boto boto3 botocore
```

---

## Variables Used

01-01-access-key.vars
```
    access_key: "xxxxxxxxxxxxxxxxxxxxx"
    secret_key: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

01-02-ec2.vars 
```
    keypair_name: "ansible-remote-key"
    region: "us-east-2"
    sg1: "ansible-remote"                                 # security group name for ssh access
    sg2: "ansible-webserver"                              # security group name for webserver access
    ami_id: ami-002068ed284fb165b
```

01-03-webserver.vars
```
    httpd_port: 80                                        # apche port number for new virtual host entry
    httpd_user: "apache"                                  # apache user for changing file permission of document root and website files.
    httpd_group: "apache"                                 # apache group for changing file permission of document root and website files.
    domain_name: "www.abc.com"                            # servername entry for virtual host entry.
```

---

## Provisioning Steps:

> Creating SSH Keypair to access newly creating EC2 instance and it will save our local.

```
    - name: "AWS - Creating Ssh KeyPair"
      ec2_key:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ keypair_name }}"
        state: present
      register: keypair_content
    
    - name: "AWS - Saving Private Key Content"
      when: keypair_content.changed == true
      copy:
        content: "{{ keypair_content.key.private_key }}"
        dest: "{{ keypair_name }}.pem"
        mode: 0400
```
> Security group for remote ssh(allow 22 port) access and website (allow 80, 443 port)

```
    - name: "AWS - Creating Security Group {{ sg1 }}"
      ec2_group:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ sg1 }}"
        description: "Allows Only 22 Conntenction"
        rules:
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip:
              - 0.0.0.0/0
            cidr_ipv6:
              - ::/0
        tags:
            Name: "{{ sg1 }}"
      register: sg1_status
                  
                  
    - name: "AWS - Creating Security Group {{ sg2 }}"
      ec2_group:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ sg2 }}"
        description: "Allows Only 80,443 Conntenction"
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip:
              - 0.0.0.0/0
            cidr_ipv6:
              - ::/0
                  
          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip:
              - 0.0.0.0/0
            cidr_ipv6:
              - ::/0
        tags:
          Name: "{{ sg2 }}"
      register: sg2_status

    - debug:
        msg: "{{ sg1_status.group_id }} - {{ sg2_status.group_id }}"
```
> Spinning up new EC2 instance
```

    - name: "AWS - Creating Ec2 Instance"
      ec2:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        key_name: "{{ keypair_name }}"
        instance_type: "t2.micro"
        image: "{{ ami_id }}"
        wait: yes
        group_id:
          - "{{ sg1_status.group_id }}"
          - "{{ sg2_status.group_id }}"
        instance_tags:
            Name: "webserver"
        count_tag:
          Name: "webserver"
        wait_timeout: 500
        exact_count: 1
      register: ec2_status
```
>  Creating dynamic inventory from the created EC2 instance.
```

    - name: "Creating Dynamic Inventory"
      add_host:
        name: "{{ item.public_ip }}"
        groups: "webserver"
        ansible_host: "{{ item.public_ip }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ keypair_name }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ ec2_status.tagged_instances }}"

```
> Here provisioning the website files
```
- name: "Hosting HTML Website"
  become: true
  hosts: webserver
  vars:
    httpd_port: 80
    httpd_user: "apache"
    httpd_group: "apache"
    domain_name: "www.abc.com"
  tasks:

    - name: "Installing Apache"
      yum:
        name:
          - httpd
        state: present
          
    - name: "Creating Virtualhost"
      template:
        src: "virtualhost.conf.j2"
        dest: "/etc/httpd/conf.d/{{ domain_name }}.conf"
        owner: "root"
        group: "root"
      register: vhost_status
      
    - name: "Creating Documentroot"
      file:
        path: "/var/www/html/{{ domain_name }}"
        state: directory
        owner: "{{  httpd_user }}"
        group: "{{ httpd_group }}"
          
    - name: "downoad websiute files"
      unarchive:
        src: https://www.tooplate.com/zip-templates/2121_wave_cafe.zip
        dest: /tmp/
        remote_src: true

    - name: "copy file to doc root"
      when: ansible_os_family == "RedHat"
      copy:
        src: /tmp/2121_wave_cafe/
        dest: /var/www/html/{{ domain_name }}/
        owner: "{{ httpd_user }}"
        group: "{{ httpd_group }}"
        remote_src: true
      register: copy_status
    - name: "Restarting/enabling apache"
      when: copy_status.changed == true or vhost_status.changed == true
      service:
        name: httpd
        state: restarted
        enabled: true
```

## Usage

Clone the repository using the below command
```
git clone https://github.com/Jisjo/ansible_Ec2-creation_with_dynamic-inventory.git
```
    Then, make the required changes in above mentioned files for variables and Finally, you can run the ansible playbook using the command
```
ansible-playbook 01-ec2_setup_main.yml 
```

## Result

Now our web server has been configured and we can visit our website using the public IP or domain name.

![image](https://github.com/Jisjo/ansible_Ec2-creation_with_dynamic-inventory/blob/main/Screenshot-01-ec2-ansible.png)
