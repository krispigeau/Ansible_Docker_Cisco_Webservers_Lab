# Overview

In this lab I demonstrate various ways of executing ansible playbooks. One way is by using running ansible locally and the other by running ansible through a docker container. There are four main networks used in this lab.

- A docker network (Cloud 1) is used as the "Managment Network". This network is used to issue configuration commands to the routers in the GNS3 network.
- The GNS3 network has routers configured to use OSPF though ansible playbooks.
- There is also a VirtualBox network (Cloud 2) which holds Ubuntu and CentOS servers. Ansible is used directly on the host machine to configure the webservers
- Another VirtualBox network (Cloud 3) which has an Ubuntu Desktop.

Once everything is configured the Ubuntu Desktop should be able to access the webservers running in VirtualBox through the GNS3 network.

This lab demonstrates the use of playbooks, roles, ansible.cfg files, inventory and host files, ssh configurations, building docker containers with a dockerfile, using host\_vars files, and using jinja2 templates.

# Topology

![](RackMultipart20231005-1-bwc04p_html_4a3e1678c0d288f2.png)

_https://github.com/krispigeau/Ansible_Docker_Cisco_Webservers_Lab/blob/main/topology.png_

# Lab Setup

**Step 1 - Clone GitHub repo and cd into the cloned repo**

git clone https://github.com/krispigeau/Ansible_Docker_Cisco_Webservers_Lab.git

cd sdn-lab-1

**Step 2 - Create Host-Only Networks in VirtualBox**

#Create two host-only in virtual box

Set Network Properties as:

Name: vboxnet0

IPv4 Address: 192.168.56.1

Ipv4 Network Mask: 255.255.255.0

DCHP: False

Name: vboxnet1

IPv4 Address: 192.168.57.1

Ipv4 Network Mask: 255.255.255.0

DCHP: True

**Step 3 - Create two virtual machines and attach to network vboxnet** 0

#One will be Ubuntu Server and the other CentOS server

- Ubuntu Server 1

Network Adapters:

Adapter 1: NAT

Adapter 2: Host-only adapter

Static IP Address: 192.168.56.101

- Centos Server 2

Network Adapters:

Adapter 1: NAT

Adapter 2: Host-only adapter

Static IP Address: 192.168.56.201

**Step 4 - Make sure both servers have a user with the same name as the user account on your workstation**

 In my case my workstation and both servers have a user named kris

**Step 5 - Create one VM and attach to network vboxnet1**

#This should be an Ubuntu Desktop

- Ubuntu Desktop

Network Adapters:

Adapter 1: Host-only adapter

Dynamic IP Address: assigned dynamically

**Step 6 - Add static routes so webservers can reach Ubuntu Desktop after setup is complete**

#Command for Ubuntu & CentOS Servers

sudo route add -net 192.168.0.0 netmask 255.255.0.0 gw 192.168.56.2

**Step 7 - Add static routes so Ubuntu Desktop can reach webservers**

#Command for Ubuntu Desktop VM

sudo route add -net 192.168.0.0 netmask 255.255.0.0 gw 192.168.57.1

**Step 8 - Generate ssh key for ansible running on local computer to access Servers in Virtual Box**

#name the key ansible

ssh-keygen -t ed25519 -C "ansible"

**Step 9 - Copy the keys the servers**

ssh-copy-id -i ~/.ssh/ansible.pub 192.168.56.101

ssh-copy-id -i ~/.ssh/ansible.pub 192.168.56.201

**Step 10 - make sure you can use the ansible ping module without a password**

ansible all -m ping

**Step 11 - Run ansible playbook to setup webservers**

#Note while ssh no longer needs a password we still need to enter the password to become sudo

ansible-playbook --ask-become-pass virtualbox.yml

**Step 12 - Create Docker Network**

sudo docker network create mgmt

**Step 13 - Create topology in GNS3**

#Connect everything as per the topology image, see topoloy.png

Components:

Cloud1: Attach to docker mgmt network

Cloud2: Attach to vboxnet0

Cloud2: Attach to vboxnet1

Switch: Connects management network

Router1: Cisco router

Router2: Cisco router

Router3: Cisco router

**Step 14 - Initialize Routers**

#Run the following commands on all three routers

#Create user for Ansible

username robot privilege 15 secret letmein123

#Enable SSH on all routers

ip domain-name kris.com

#Enter module = 1024, after the following command

crypto key generate rsa

ip ssh version 2

line vty 0 15

login local

transport input ssh

**Step 15 – Assign an IP address to each router so they all have an interface in the docker network**

R1: ip address 172.18.0.51

R2: ip address 172.18.0.52

R3: ip address 172.18.0.53

**Step 16 - change to the directory call docker\_mgmt**

cd docker\_mgmt

**Step 17 - Build Docker Container from Dockerfile in directory**

sudo docker build . -t cisco-controller

**Step 18 - create docker container with bind mount to transfer files to container**

sudo docker run -it --rm --network=mgmt -v $PWD/ansible:/etc/ansible cisco-controller

_ **@@@ Run the following steps inside the docker container @@@** _

**Step 19 - Inside docker container Gather facts from routers**

ansible-playbook facts.yml

**Step 20 - Add IP Addresses to Interfaces and configure OSPF**

ansible-playbook r1.yml

ansible-playbook r2.yml

ansible-playbook r3.yml

**Step 21 - Now the Ubuntu Desktop VM should be able to browse the webservers**

[http://192.168.56.101](http://192.168.56.101/)

[http://192.168.56.201](http://192.168.56.201/)

#What does the code do?

## Ansible running on workstation:

~/sdn-lab-1

In the ansible.cfg file I've configured some settings. I tell ansible where to find the inventory file. I also tell ansible where to find the private ssh key used to talk with webservers running in virtualbox.

In the inventory file I have the IP addresses for the webservers in a group called web\_servers.

The ansible playbook virtualbox.yml updates the repos for both webservers. It uses the ansible\_distribution variable and when statements to determine if the Operating System is either CentOS or Ubuntu. Based on the OS it will run the module which uses the package manager specific to the Operating System. The ansible playbook will then call the role web\_servers

I have a role directory called web\_servers. In this directory you can see the main.yml taskbook. Most of the tasks in this task book can be run on both Ubuntu or Centos. The way I make this happen is by using variables found in the host\_vars folder.

In the host\_vars folder I have variables set for each webserver since the apache package name and service name varies depending on which Linux distro you are using.

The main.yml file uses those host\_vars variables when installing the web server packages and starting/restarting the service. In CentOS you also need to open a firewall rule, which is handled by a task written for CentOS. Also, ansible will use a jinja2 template to configure the default site for each webserver.

There is a folder in the web\_servers role folder called templates. This folder is used to store the default\_site.j2 template file. The template file uses the variable linux\_distro. This variable is resolved when the template is copied to a webserver and replaced with the value stored n the linux\_distro variable.

## Ansible running in docker container:

~/sdn-lab-1/docker\_mgmt

This folder has a Dockerfile which is used to build a container image. The Dockerfile uses ubuntu:22.04 as its base. It then runs commands used to install ansible as well as some networking tools.

There is an issue when you try to ssh into the versions of cisco IOS I'm using in my GNS3 topology. The Key Algorithms and Ciphers used by the Cisco devices are older than the Algorithms and Ciphers used by the Ubuntu Docker container.

To get around this I've created a configuration file while enables legacy options for ssh. The configuration file is named cisco.conf. This ssh configuration file is copied to the container as it is being built. The enables the use of legacy ssh configurations.

I this lab we won't be using ssh keys for authentication with our routers. Instead, an ansible username and password is configured on each router. The username and password are written as variables in the hosts file.

Also, host\_key\_checking is set to false in the ansible.cfg file to get around using ssh keys.

To demonstrate using facts, I've written the fact.yml playbook. This playbook uses the cisco.ios.ios\_facts module to gather facts from the devices. The facts are displayed on screen with the debug task.

In the host files I've also set the IP addresses of each router as variables to use with each router's host name.

The hostnames are used in each playbook, r1.yml, r2.yml and r3.yml. Each of these playbooks are specific to the router they are configuring and use the hostnames configured.

Each playbook has tasks that will set IP addresses on the router using the ios\_config module. Once the IP addresses are assigned the ios\_config module is used again in tasks to add networks into OSPF.

There are also tasks to gather facts from devices and display the IP addresses configured as well as the ospf configuration.

