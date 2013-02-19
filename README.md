ansible-vagrant
===============

A vagrant module for ansible, that lets you control vagrant VMs from an ansible playbook.

## Overview

Main page on GitHub:

* https://github.com/robparrott/ansible-vagrant

Ansible-vagrant is a module for [ansible](http://ansible.cc) that allows you to create VMs on your local system using [Vagrant](http://vagrantup.com/). 
This lets you write ansible [playbooks](http://ansible.github.com/playbooks.html) that dynamically create local guests and configure them via the ansible playbook. 
This makes testing and development of orchestrated, distributed applications via ansible easy, by letting you run guests on your local system.

Ansible-vagrant should not be confused with [vagrant-ansible](https://github.com/dsander/vagrant-ansible) which allows you to run ansible playbooks on vagrant launched hosts

## Dependencies & Installation

The vagrant module for ansible requires 

 * a working [Vagrant](http://vagrantup.com/) install, which will itself 
   require [VirtualBox](https://www.virtualbox.org/wiki/Downloads).
 * that the [vagrant-python](https://github.com/todddeluca/python-vagrant) is installed and in your python path

To install, couple the file "vagrant" to your ansible isntallation, under the "./library" directory.

To run the tests from the repository, cd into "./test" and run

    ansible-playbook -v -i hosts vagrant-test.yaml
    
## Getting started with ansible-vagrant

### Command Line Use
(Note that the following example assumes you know something about ansible and its syntax and conventions)

The following is a simple example, and assumes a linux or unix shell prompt.

First create a simple host inventory file for ansible that only includes your local machine:

    echo "127.0.0.1" > hosts
    
The target machine image is called a "box" under vagrant, and can be managed ahead of time using a vagrant command:

    vagrant box add lucid64 http://files.vagrantup.com/lucid64.box 

or instead the path can be specified when you start the VM itself. We'll start up a standard VM with ansible-vagrant as follows: 

    ansible all -i hosts -c local -m vagrant -a "cmd=up vm_name=myvm box_name=lucid64 box_path=http://files.vagrantup.com/lucid64.box"
    
This command will download and register the image as "lucid64," then start up the image and attribute 
the name "myvm" to it. If you had left off the "vm_name" argument, the module will set the vm_name to "ansible" for you.

Note: names in vagrant have to be compatible with the ruby syntax used in Vagrantfiles, 
so they can't use "-" or "_", and must start with a lowercase letter. *In the future, you can leave 
off the "box_path" argument, as vagrant registers and caches the image locally.

Once up, the image can be logged into via the vagrant command:

    vagrant ssh myvm

or more importantly, registered in the ansible inventory and managed via a playbook.

You can get the status of the machine via the command

    ansible all -i hosts -c local -m vagrant -a "cmd=status vm_name=myvm"

and it's configuration (in particular ssh config values) via the config subcommand

    ansible all -i hosts -c local -m vagrant -a "cmd=config vm_name=myvm"

Then you can halt the machine when you're done:

    ansible all -i hosts -c local -m vagrant -a "cmd=halt vm_name=myvm"  

and remove it's record:

    ansible all -i hosts -c local -m vagrant -a "cmd=destroy vm_name=myvm"  

This last command will halt and then remove the record if the machine had been running.

Lastly, we can shutdown and clear out all records with the "clear" subcommand:

    ansible all -i hosts -c local -m vagrant -a "cmd=clear"

### Playbooks
         
Playbooks allow you to start up instances in vagrant dynamically, 
add them to the inventory and then run "plays" on those newly created instances 
from a single command. 

Note: when playing around with builds using vagrant, it's useful to clear out your Vagrantfile 
and VMs before running a playbook. The following is the equivalent of a "make clean" :

    ansible all -i hosts -c local -m vagrant -a "cmd=clear"
    
(it's kinda fun to watch the VirtualBox manager window, and see the instances shutdown and disappear like good little vms :-> )
    
Here's an example playbook:

    ---
	#
	# This section fires up a guest dynamically using vagrant,
	#  registers it in the inventory under 
	#  the group "vagrant_hosts"
	# then logs in and pokes around.
	#
	- hosts:
	  - localhost
	  connection: local
	  gather_facts: False
	
	  vars:
	    box_name: lucid32
	    vm_name: frank
	
	  tasks:
	  - name: Fire up a fresh new vagrant instance to log into
	    local_action: vagrant
	        command=up
	        box_name=${box_name}
	        vm_name=${vm_name}
	    register: vagrant
	    
	  - name: Remind us about the vagrnat private key ...
	    action: debug 
	            msg="Be sure to add the ssh private key for vagrant, here ... '${vagrant.instances[0].key}'."
	  
	  - name: Capture that host's contact info into the inventory
	    action: add_host hostname='${vagrant.instances[0].public_ip}:${vagrant.instances[0].port}' groupname=vagrant_hosts
	    
	#
	# Run on the vagrant_hosts group, checking that we have basic ssh access...
	#    
	- hosts:
	  - vagrant_hosts
	  user: vagrant
	  
	  gather_facts: False
	             
	  tasks:
	  
	  - name: Let's see if we can login
	    action: command uname -a
	    
	  - name: Let's see all the ansible vars about vagrant hosts...
	    action: setup
	  
	  - name: Generate a ./blah_ansible.vars to check for hostvars
	    action: template src=test-vagrant-hostinfo.j2 dest=/tmp/localhost_ansible.vars
    
      
## Methods and Data Structures      

Here, let's talk about the data returned by the various commands. 

These data structures take into account that there may be multiple named sets of isntances, and multiple instances within each "up invocation"

### UP

The up subcommand

  ansible -m vagrant -a "command=up box_name=lucid32 vm_name=fred count=2
  
asks vagrant to start up two identical instances of the "lucid32" box, and name then "fred." When successful produces output of this type:

  {
    "changed": true, 
    "instances": [
      {
        "id": "fred0", 
        "internal_ip": "192.168.179.253", 
        "key": "/Users/username/.vagrant.d/insecure_private_key", 
        "name": "fred", 
        "port": "2200", 
        "public_dns_name": "127.0.0.1", 
        "public_ip": "127.0.0.1", 
        "status": "running", 
        "username": "vagrant", 
        "vagrant_name": "fred0"
      }, 
      {
        "id": "fred1", 
        "internal_ip": "192.168.179.252", 
        "key": "/Users/username/.vagrant.d/insecure_private_key", 
        "name": "fred", 
        "port": "2201", 
        "public_dns_name": 
        "127.0.0.1", 
        "public_ip": "127.0.0.1", 
        "status": "running", 
        "username": "vagrant", 
        "vagrant_name": "fred1"
      }
    ]
  }

### Status

Subcommand invocation

  ansible -m vagrant -a "command=status vm_name=fred"

reports back with a dictionary of vm_names (only one in the case) with an array of status strings:

  {
    "changed": false, 
    "status": {
      "fred": [
         "running", 
         "running"
       ]
     }
   }

if we don't specify a vm_name, we get the status of all instances:

  {
    "changed": false, 
    "status": {
      "ansible": ["running"], 
      "fred": ["running", "running"]
    }
  }

### Config

Subcommand invocation for config data is similar ot status, but with much values returned:

  ansible -m vagrant -a "command=config vm_name=fred"

reports back with a dictionary of vm_names (only one in the case) with an array of dicts about the SSH configuration of the hosts:

   {
     "changed": false, 
     "config": {
       "fred": [
         {
           "Host": "fred0", 
           "HostName": "127.0.0.1", 
           "IdentitiesOnly": "yes", 
           "IdentityFile": "/Users/username/.vagrant.d/insecure_private_key", 
           "PasswordAuthentication": "no", 
           "Port": "2200", 
           "StrictHostKeyChecking": "no", 
           "User": "vagrant", 
           "UserKnownHostsFile": "/dev/null"
         }, 
         {
           "Host": "fred1", 
           "HostName": "127.0.0.1", 
           "IdentitiesOnly": "yes", 
           "IdentityFile": "/Users/username/.vagrant.d/insecure_private_key", 
           "PasswordAuthentication": "no", 
           "Port": "2201", 
           "StrictHostKeyChecking": "no", 
           "User": "vagrant", 
           "UserKnownHostsFile": "/dev/null"
         }
       ]
     }
   }


### Halt

When halting an instance, ort set of isntances, as in 

    ansible -m vagrant -a "command=halt vm_name=fred"
    
report back the resulting status

  {
    "changed": true, 
    "status": {
      "fred": [
         "poweroff", 
         "poweroff"
       ]
     }
   }
### Destroy

### Clear

## LICENSE:
Not yet ...







.