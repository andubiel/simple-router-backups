# simple-backup repo
This repo includes files for the "Router Backups to Everywhere Demo". The main idea of this repository is to demonstrate using different methods to backup the Cisco always-on Devnet Sandbox routers to various destinations. For demostration purposes we also use different credential types.

## Warning
Please note the Cisco sandbox might refuse connections from time to time due to connection treshholds on the vty lines or issues with the router. As a precaution we must create an account on the second always-on router to increase the chances of connecting to an always-on router that responds.

### Example of a possible issue
~~~
ssh admin@sandbox-iosxe-latest-1.cisco.com
user = admin
password = C1sco12345

Connection reset by peer
Connection reset by 131.226.217.143 port 22
~~~

Ansible Controller: 
~~~
fatal: [sandbox-iosxe-latest-1.cisco.com]: FAILED! => {"changed": false, "msg": "ssh connection failed: ssh connect failed: Socket error: Connection reset by peer"}
~~~
### Resolution: You must configure the second router by running this playbook prior to the demo
second_router.yml
~~~
---
- name: Second Router
  hosts: sandbox-iosxe-recomm-1.cisco.com
  gather_facts: false
  vars:
    ansible_user: developer
    ansible_password: 'lastorangerestoreball8876'
  tasks:
    - name: Create User Admin
      cisco.ios.ios_user:
        name: admin
        configured_password: 'C1sco12345'
        state: present
~~~

## Output Ansible-navigator
~~~
ansible-navigator run second_router.yml -m stdout
---------------------------------------------------------------------------------
Execution environment image and pull policy overview
---------------------------------------------------------------------------------
Execution environment image name:     registry.gitlab.com/tdubiel1/network-ee:latest
Execution environment image tag:      latest
Execution environment pull arguments: None
Execution environment pull policy:    tag
Execution environment pull needed:    True
---------------------------------------------------------------------------------
Updating the execution environment
---------------------------------------------------------------------------------
Running the command: podman pull registry.gitlab.com/tdubiel1/network-ee:latest
Trying to pull registry.gitlab.com/tdubiel1/network-ee:latest...
Getting image source signatures
PLAY [Second Router] ***********************************************************

TASK [Create User Admin] *******************************************************
changed: [sandbox-iosxe-recomm-1.cisco.com]

PLAY RECAP *********************************************************************
sandbox-iosxe-recomm-1.cisco.com : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
~~~~
# Demo
## This Demo includes the following scenarios
* Backing up the Router Configs to a Local Directory
* Backing up the Router Configs to a directory on the AAP Controller
* Backing up the Router Configs to a Git Repo using a new Branch

## Getting started

This demo will require the following environment:


  * A local Ansible Install (ansible-playbook)
  * An Ansible-Navigator (network-ee execution environment image)
  * Ansible Automation Platform 2.3 (network-ee execution environment image)
  * Dependency to this repo for demo "" https://gitlab.com/tdubiel1/simple-backup-git.git
  * Create new or Fork these two repos to save the router backups
    * router_backups: https://gitlab.com/tdubiel1/router_backups.git
    * router_backups_merge: https://gitlab.com/tdubiel1/routers_backups_merge.git
  * A remote Rhel server (an external server separate from the AAP server)

## Backing up Router Config to a Local Directory
This demo uses the old_school.yml playbook to backup the Cisco devnet router config to a folder named old_school_backups/ in your local ansible project.

### Dependencies:
* (you must create this file since it's excluded in .gitignore)
    * vault_password_file= <your vault password file>
* ansible-vault create group_vars/ios/vault.yml
        * ansible_user: admin
        * ansible_password: C1sco12345
* hosts.yml
* group_vars/ios/vars.yml

### old_school.yml
~~~
---
- name: Simple Backup
  hosts:
    - ios
  gather_facts: false
  tasks:
    - name: Backup cisco ios configuration
      cisco.ios.ios_config:
        backup: true
        backup_options:
          dir_path: old_school_backups/
~~~
### Output ansible core engine
~~~
$ ansible-playbook old_school.yml 

PLAY [Simple Backup] ******************************************************************************************

TASK [Backup cisco ios configuration] *************************************************************************
changed: [sandbox-iosxe-latest-1.cisco.com]
changed: [sandbox-iosxe-recomm-1.cisco.com]

PLAY RECAP ****************************************************************************************************
sandbox-iosxe-latest-1.cisco.com : ok=1    changed=1   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
sandbox-iosxe-recomm-1.cisco.com : ok=1    changed=1   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
~~~

## Backing up the Router Config to a directory on the AAP Controller
This demo uses the backup.yml playbook to backup the Cisco devnet router config to a folder named backups/ on the Ansible Automation Controller home directoy of the user account to access AAP contoller from the network-ee execution environment. Please note the AAP controller and Ansible-Navigator runs playbooks from a Ansible Core Engine installed on the execution environment container. This requires a credential for accessing the AAP server file system. Basically you are coping the files from the container to the AAP server file system.

## Ansible-Navigator 
### Dependencies:
* ansible.cfg (you must create this file since it's excluded in .gitignore)
    * vault_password_file= <your vault password file>
* ansible-vault create group_vars/ios/vault.yml
        * ansible_user: admin
        * ansible_password: C1sco12345
* ansible-navigator.yml ( mounted volumes to access home directory (vault password file) from where ansible-navigator is installed)        
* hosts.yml
* group_vars/ios/vars.yml

### backup.yml
~~~
---
- name: Simple Backup
  hosts:
    - ios
    - local_servers
  gather_facts: false
  tasks:
    - name: Backup cisco ios configuration
      cisco.ios.ios_config:
        backup: true
        backup_options:
          dir_path: /tmp/backups
      when: inventory_hostname in groups['ios']

    - name: Copy to AAP Server
      ansible.builtin.copy:
        src: "/tmp/backups/"
        dest: "~/backups/"
      when: inventory_hostname in groups['local_servers']
~~~

### Output ansible-navigator 
~~~
$ ansible-navigator run backup.yml -m stdout
---------------------------------------------------------------------------------
Execution environment image and pull policy overview
---------------------------------------------------------------------------------
Execution environment image name:     registry.gitlab.com/tdubiel1/network-ee:latest
Execution environment image tag:      latest
Execution environment pull arguments: None
Execution environment pull policy:    tag
Execution environment pull needed:    True
---------------------------------------------------------------------------------
Updating the execution environment
---------------------------------------------------------------------------------
PLAY [Simple Backup] ***********************************************************

TASK [Backup cisco ios configuration] ******************************************
changed: [sandbox-iosxe-latest-1.cisco.com]

TASK [Copy to AAP Server] ******************************************************
changed: [local_server]

PLAY RECAP *********************************************************************
local_server               : ok=1    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
sandbox-iosxe-latest-1.cisco.com : ok=1    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
~~~
### Verify AAP server backups
[tdubiel@rhel84 ~]$ cd backups/
[tdubiel@rhel84 backups]$ ls
sandbox-iosxe-latest-1.cisco.com_config.2023-05-10@14:16:54

## AAP Controller Job-template "Cisco-Simple-Backup-AAP" 
### Dependencies:
* credentials (we are not using vault )
    * custom credential (devnet)
    * machine credential for AAP remote user  (username/password)
* project
* inventory
* inventory source is the project
* hosts.yml
* group_vars/ios/vars.yml

#### Custom Credential (devnet)
Input Configuration
~~~
fields:
  - id: username
    type: string
    label: Username
  - id: password
    type: string
    label: Password
    secret: true
~~~
Injector Configuration
~~~
extra_vars:
  username: '{{ username }}'
  password: ' {{ password }}'
~~~

### Output Ansible Controller
~~~
PLAY [Simple Backup] ***********************************************************
TASK [Backup cisco ios configuration] ******************************************
skipping: [local_server]
changed: [sandbox-iosxe-latest-1.cisco.com]
changed: [sandbox-iosxe-recomm-1.cisco.com]
TASK [Copy to AAP Server] ******************************************************
skipping: [sandbox-iosxe-latest-1.cisco.com]
skipping: [sandbox-iosxe-recomm-1.cisco.com]
changed: [local_server]
PLAY RECAP *********************************************************************
local_server               : ok=1    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
sandbox-iosxe-latest-1.cisco.com : ok=1    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    
ignored=0   
sandbox-iosxe-recomm-1.cisco.com : ok=1    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    
ignored=0   
~~~
### Verify AAP Server backups
~~~
tdubiel@rhel84 backups]$ ls
~~~ 
## Backing up the Router Configs to a Gitlab Repo using a new Branch
This demo uses the ansible.scm collection that was included in the network-ee execution environment.
Available from - registry.gitlab.com/tdubiel1/network-ee:latest
This collection is used to clone down the repo to the execution environmement and ultimately stage, commit, and push the backup configs to a new stage on the remote repo based on the play_name and timestamp. Subsequently the branch contains the backups from the playbook run and can be merged into the main branch of the repo. The repo is named  
'router_backups_merge'

## Ansible-Navigator 
### Dependencies:
* ansible.cfg (you must create this file since it's excluded in .gitignore)
    * vault_password_file= <your vault password file>
* ansible-vault create group_vars/ios/vault.yml
        * ansible_user: admin
        * ansible_password: C1sco12345
* ansible-vault create vars/vault.yml
        * token: <your gitlab or github token>
        * user:  <your gitlab or github user>
        * email: <your gitlab or github email>
* ansible-navigator.yml ( mounted volumes to access home directory (vault password file) from where ansible-navigator is installed)        
* hosts.yml
* group_vars/ios/vars.yml

## backup_git_scm.yml 
~~~
---
- name: Simple Git Backup to Gitlab Repo using Merge
  hosts: localhost, ios
  gather_facts: false
  vars_files:
    - vars/git.yml
    #- vars/vault.yml #uncomment to run locally
  tasks:
   
    - name: Retrieve a repository from a distant location and make it available locally
      ansible.scm.git_retrieve:
        origin:
          url: "{{ repo_merge }}"
        parent_directory: /tmp/
      register: repository
      when: inventory_hostname == 'localhost'

    - name: Backup cisco ios configuration
      cisco.ios.ios_config:
        backup: true
        backup_options:
          dir_path: /tmp/routers_backups_merge
      when: inventory_hostname in groups['ios']   

    - name: This command list the files on the EE
      shell:
        cmd: ls 
        chdir: /tmp/routers_backups_merge
      when: inventory_hostname == 'localhost'
      register: files 

    - name: Print files on the screen
      ansible.builtin.debug:
          msg: "{{ files.stdout_lines }}"
      when: inventory_hostname == 'localhost'

    - name: Publish the changes
      ansible.scm.git_publish:
        path: "{{ repository['path'] }}"
        token: "{{ token }}"
        user:
          name: "{{ username }}"
          email: "{{ email }}"
      when: inventory_hostname == 'localhost'
~~~



