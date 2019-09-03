# ansible-nginx-loadbalancing

Playbook to deploy 3 ec2 instances and configuring nginx loadbalancing.
This will deploy 3 ec2 instances as one ec2 as frontend and 2 ec2 as backend.  Installs httpd in backend servers and installs ngninx in frontend ec2 and setup it as a loadbalancer for the backend servers. 

### Prerequisite:
1. Install Ansible on an ec2 Instance and setup it as Ansible-master
2. Python boto library
3. Create an IAM Role with Policy AmazonEC2FullAccess and attach it to the Ansible master instance.
---

##### nginxlb.yml
```sh
---
 - name: "Load Balancing Using Nginx"
   hosts: localhost

   vars:
     domain: nginxlb.com
     instance_type: t2.micro
     security_group: ansible-sg
     image: ami-0b898040803850657
     region: us-east-1
     keypair: jwl-pc
     exact_count: 1
     subnet: subnet-5074287e
     hostkey: /home/ec2-user/jwl-pc.pem

   tasks:
     - name: "Creating a security group"
       ec2_group:
         name: "{{ security_group }}"
         description: Ansible-webserver
         region: "{{ region }}"
         rules:
           - proto: tcp
             from_port: 22
             to_port: 22
             cidr_ip: 0.0.0.0/0
           - proto: tcp
             from_port: 80
             to_port: 80
             cidr_ip: 0.0.0.0/0
           - proto: tcp
             from_port: 443
             to_port: 443
             cidr_ip: 0.0.0.0/0
         rules_egress:
           - proto: all
             cidr_ip: 0.0.0.0/0

     - name: "Launching ec2 Instance for Frontend"
       ec2:
         instance_type: "{{instance_type}}"
         key_name: "{{keypair}}"
         image: "{{image}}"
         region: "{{region}}"
         group: "{{security_group}}"
         vpc_subnet_id: "{{subnet}}"
         wait: yes
         count_tag:
           Name: Frontend-Nginx
         instance_tags:
           Name: Frontend-Nginx
         exact_count: "{{exact_count}}"
       register: ec2_frontend_status

     - name: "Setting Frontend IP in Variable"
       set_fact:
         frontendip: "{{ ec2_frontend_status.tagged_instances.0.public_ip }}"

     - name: "Adding host to inventory"
       add_host:
        hostname: frontend
        ansible_host: "{{ frontendip }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ hostkey }}"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"

     - name: "Launching ec2 Instance for Backend Webserver1"
       ec2:
         instance_type: "{{instance_type}}"
         key_name: "{{keypair}}"
         image: "{{image}}"
         region: "{{region}}"
         group: "{{security_group}}"
         vpc_subnet_id: "{{subnet}}"
         wait: yes
         count_tag:
           Name: Backend1-Apache
         instance_tags:
           Name: Backend1-Apache
         exact_count: "{{exact_count}}"
       register: ec2_backend1_status

     - name: "Setting Backend1 IP in Variable"
       set_fact:
         backendip1: "{{ ec2_backend1_status.tagged_instances.0.public_ip }}"


     - name: "Adding backend1 to inventory"
       add_host:
        hostname: backend1
        groups: backend
        ansible_host: "{{ backendip1 }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ hostkey }}"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"


     - name: "Launching ec2 Instance for Backend Webserver2"
       ec2:
         instance_type: "{{instance_type}}"
         key_name: "{{keypair}}"
         image: "{{image}}"
         region: "{{region}}"
         group: "{{security_group}}"
         vpc_subnet_id: "{{subnet}}"
         wait: yes
         count_tag:
           Name: Backend2-Apache
         instance_tags:
           Name: Backend2-Apache
         exact_count: "{{exact_count}}"
       register: ec2_backend2_status

     - name: "Setting Backend2 IP in Variable"
       set_fact:
         backendip2: "{{ ec2_backend2_status.tagged_instances.0.public_ip }}"


     - name: "Adding backend2 to inventory"
       add_host:
        hostname: backend2
        groups: backend
        ansible_host: "{{ backendip2 }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ hostkey }}"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"


     - name: "Waiting for OpenSSH to open in all 3 servers"
       wait_for:
         port: 22
         host: "{{ item }}"
         timeout: 80
         state: started
         delay: 10
       with_items:
         - "{{ frontendip }}"
         - "{{ backendip1 }}"
         - "{{ backendip2 }}"

     - name: "Installing nginx on Frontend"
       delegate_to: frontend
       become: yes
       shell: amazon-linux-extras install nginx1.12

     - name: 'Creating virtual host nginx'
       delegate_to: frontend
       become: yes
       template:
         src: nginx.j2
         dest: /etc/nginx/conf.d/{{domain}}.conf

     - name: "Nginx - Restarting"
       delegate_to: frontend
       become: yes
       service:
         name: nginx
         state: restarted
         enabled: yes

     - name: "Apache - Installing"
       delegate_to: "{{ item }}"
       become: yes
       yum: name=httpd,php,php-mysql state=present
       with_items:
         - backend1
         - backend2

     - name: "Apache - Creating index.php"
       delegate_to: "{{ item }}"
       become: yes
       copy:
         content: "<h1><center><?php echo gethostname(); ?> </center></h1>"
         dest: /var/www/html/index.php
       with_items:
         - backend1
         - backend2

     - name: "Apache - Creating phpinfo.php"
       delegate_to: "{{ item }}"
       become: yes
       copy:
         content : "<?php phpinfo(); ?>"
         dest : /var/www/html/phpinfo.php
       with_items:
         - backend1
         - backend2

     - name: "Apache - Restarting/Enabling"
       delegate_to: "{{ item }}"
       become: yes
       service:
         name: httpd
         state: restarted
         enabled: yes
       with_items:
         - backend1
         - backend2

     - debug:
         msg: FrontendIP- "{{ frontendip }}" Backend IPs- "{{ backendip1 }}" ,  "{{ backendip2 }}"
         
```

#### Template: nginx.j2

```sh

    upstream backend {
        server {{ backendip1 }};
        server {{ backendip2 }};
    }

    server {
        listen 80;
        server_name {{domain}} www.{{domain}};
        location / {
            proxy_pass http://backend;
        }
    }
```

##### You can edit the template according to your need. Add ip_hash etc.
#
#
#### Executing the playbook - # ansible-playbook nginxlb.yml


