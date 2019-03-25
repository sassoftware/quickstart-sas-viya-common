# SAS Cloud Rapid Deployment common templates

These templates assume a topology with an "ansible controller" VM and one or more SAS/Viya VMs.

The "ansible controller" serves as jump host into the environment, and may be the only machine with a public IP address.

The intent of these templates is to simplify the creation of software install templates for any cloud platform, 
any topology and for any product combination.

This project is meant to be included into the sas install directory on your ansible controller (e.g. `git clone https://github.com/sassoftware/quickstart-sas-viya-common.git /sas/install/common`)

Some parts of this project may be modified to fit your exact needs. Those parts (scripts,  roles, or playbooks) you would copy into the corresponding `scripts` or `playbooks` directory in your project then modify.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**   

- [Overview: How to create your own Cloud Rapid Deployment template](#overview-how-to-create-your-own-cloud-rapid-deployment-template)
- [How the playbooks and roles and scripts work](#how-the-playbooks-and-roles-and-scripts-work)
- [Installation users](#installation-users)
- [Steps](#steps)
  - [VM post initialization -executed on the individual hosts](#vm-post-initialization--executed-on-the-individual-hosts)
  - [Input parameter validation](#input-parameter-validation)
  - [Additional preparatory steps - driven by ansible from the ansible controller](#additional-preparatory-steps---driven-by-ansible-from-the-ansible-controller)
    - [IAAS specific setup steps:](#iaas-specific-setup-steps)
  - [Set up Users  (OpenLDAP install)](#set-up-users--openldap-install)
  - [Prepare Deployment](#prepare-deployment)
  - [Run VIRK](#run-virk)
  - [Install Viya](#install-viya)
  - [Post Deployment Steps](#post-deployment-steps)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->



## Overview: How to create your own Cloud Rapid Deployment template


- create the infrastructure (networks, VMs, firewalls) using the cloud provider's templating language (e.g. AWS Cloudformation on AWS, Azure Resource Manager, terraform ... )
- copy the contents of this project's `ansible/playbooks` into your project's `ansible/playbooks` 
- copy and modify the static topology definition files(`ansible/playbooks/inventory.ini|group_vars`) to match your topology
- copy and modify any other parts of the scripts and playbooks that have cloud-specific elements and/or need to be adjusted for your environment
- add any scripts or playbooks that are unique to your environment


## How the playbooks and roles and scripts work

Most of the steps for the deployment are being implemented in ansible. You find all that code in the `playbooks` and `roles` directories.

Before we can start using ansible, we need to run some preparatory statements on the VMs. Execute the bootstrapping code `scripts/ansiblecontroller_prereqs.sh` during initialization of the ansible controller and `scripts/sasnodes_prereqs.sh` during initialization of all SAS VMs.

When you create your infrastructure, all the cloud providers' templating languages provide some way to execute bootstrapping code on the VMs 
(usually some form of [cloud-init](https://cloud-init.io/), often implemented as `UserData` VM attribute).
That is where you run the prereq scripts, and, on the ansible controller, kick off the download of all other scripts and ansible playbooks and execute those scripts and playbooks.

First, you execute the bootstrapping scripts on each VM, which set up the necessary pieces so that the VMs can communicate with each other.
After that, everything will be driven by a number of playbooks executed from the ansible controller.

You invoke a series of ansible playbooks, which in turn execute a number of roles. Using playbooks and roles allows us to clearly structure the process. 
Each logical step corresponds to a role, and the roles are grouped into separate playbook invocations which correspond to the main deployment steps (environment preparation, install, post-install steps etc.) 

Controlling everything through ansible playbook roles allows us to 
- specify the topology in only one place: the `inventory.ini` file
- define all global variables in only one place: the `group_vars/all.yml` file. 
- make any changes or overrides to individual steps by modifying the existing roles
- adding steps as needed by simply adding additional roles

You can change the implementation of each role, or add additional roles.

To change the implementation of a role, copy it into the corresponding location in the `ansible/roles` directory in your project and modify it there.
The overwritten role will automatically be picked up (controlled by an ansible search path option).

To add a role, write its implemention and put it into the `ansible/roles` directory in your project. Then edit one of the playbooks in your `ansible/playbooks` directory and add that role.
Or add a playbook and make sure to invoke it from your ansible controller `cloud-init` script. 

After running the prereq scripts, copy the contents of your project and the contents of this "common" project into the following file structure on the ansible controller 

```
/sas/install
|-- common
|   |-- ansible
|       `-- playbooks
|       `-- roles
|   `-- scripts
|-- ansible
|   `-- playbooks
|   `-- roles
`-- scripts
...
```

(use the [scripts/download_file_tree.sh](scripts/download_file_tree.sh) as an example ) 

The playbooks are run as the [installation user](#installation-users) from `/sas/install/ansible/playbooks`. 

The roles search path is set in  `/sas/install/ansible/playbooks/ansible.cfg` as `roles_path = /sas/install/ansible/roles:/sas/install/common/ansible/roles`.
That means any roles in `/sas/install/ansible/roles` will be used first. All the remaining roles for which  you did not provide an IAAS specific implementation 
will be found in  `/sas/install/common/ansible/roles`.
  

NB: All roles executed via the playbooks are tagged with their name. 
E.g. if you want to run `prepare_nodes.yml` and exclude `set_host_routing`, invoke it with

```
ansible-playbook prepare_nodes.yml --skip-tags set_host_routing
```




## Installation users

All scripts and templates are written to be executed by an "installation user" with sudo privileges.
It needs to be the same user on all VMs.

 AWS      | Azure  | Google 
:-------- |:-------|:-------
 `ec2-user` | `vmuser` | `sasinstall`       



## Steps


### VM post initialization -executed on the individual hosts


 1. __Ansible controller VM setup (optional)__

    Additional bootstrapping on the ansible controller. 
    Should be called inline in VM bootstrap ("user-data" section or equivalent):
    - yum installs
    - security config
    - ...

 1. __Ansible controller VM prereqs__

    Preparing ansible:
    - install java, ansible, git
    - export nfs share
    - create ansible ssh key

    ```
    scripts/ansiblecontroller_prereqs.sh
     --->/tmp/prereqs.sh &> /tmp/prereqs.log
    ```

 1. __SAS VMs setup (optional)__

    CLOUD SPECIFIC implementation
    
    Additional bootstrapping on the sas nodes. 
    
    Should be called inline in VM bootstrap ("user-data" section or equivalent). 
    - yum installs
    - security config
    - ...

 1. __SAS VMs prereqs__

    Preparing ansible:
    - mount nfs share
    - set ansible ssh key
    - post readiness flag

    ```
    scripts/sasnodes_prereqs.sh
     --->/tmp/prereqs.sh &> /tmp/prereqs.log
    ```

 1. __Ansible controller: download project files__

    Download all the additional scripts and playbooks need for the deployment.
    
    This script loops over the contents of the file `file_tree.txt` which was created using the script `common/scripts/create_file_tree.sh`. 
    It downloads the files from the IAAS specific storage location and puts the into `/sas/install` on the ansible controller.
      
    The script has implementation for AWS, Azure, and GCP and requires the following environment variables to be set:
    
    ```
       IAAS=[aws|azure|gcp]
       FILE_ROOT=<IAAS specific location>
    ```
    
    Example invocation (for AWS):
    
    ```
       !Sub
       - su -l ec2-user -c 'IAAS=aws FILE_ROOT=${S3_FILE_ROOT} /tmp/download_file_tree.sh &>/tmp/download_file_tree.log'
       - S3_FILE_ROOT: !Sub "${QSS3BucketName}/${QSS3KeyPrefix}"
    ```

### Input parameter validation

As much as possible, input parameter validation should happen in the IAAS template. 
But the templates have different capabilities, and not everything can be checked at that level.

For example, the template checking mechanism might be able to determine that the name of the SOE file is a valid file name, 
but it will likely not be able to check that it actually contains a SAS license. 

It is good practice to add additional checking as early as possible. We want to avoid setting up and configuring infrastructure  
only to fail 20 minutes in because the specified Mirror location does not actually contain a valid mirror etc.

- verify ssl 
- verify hosted zone
- verify mirror
- verify elb has been created
- verify bucket not public

    
### Additional preparatory steps - driven by ansible from the ansible controller  

The playbook `playbooks\prepare_nodes.yml` does additional steps needed before installing SAS, including

- host routing
- volume attachments
- setting up directories and users 

Logs are routed to `/var/log/sas/install/prepare_nodes.yml`
 
Example invocation:

``` 
export ANSIBLE_LOG_PATH=/var/log/sas/install/prepare_nodes.log
export ANSIBLE_CONFIG=/sas/install/ansible/playbooks/ansible.cfg
ansible-playbook -v /sas/install/ansible/playbooks/prepare_nodes.yml \
               -e "USERLIB_DISK=/dev/xvdl" \
               -e "SAS_INSTALL_DISK=/dev/xvdg" \
               -e "CASCACHE_DISK="
```


 1. __Wait for all SAS VMs to be ready__

    Waits for all hosts to post their readiness flag in the `/sas/install/nfs/readiness_flags` directory.

    The default timeout is set to 20 minutes. It should be similar to or smaller than the timeout set on the resource creation for
    the SAS VMs in your IAAS. 
    (If you let the IAAS template fail on a resource creation the error message is likely going to be more helpful and immediate than having to scour the ansible logs). 
    For example AWS CloudFormation has this EC2 attribute:
    
    ```
        CreationPolicy:
          ResourceSignal:
            Timeout: 'PT20M'
    ```


    ```    
    Role: prepare_nodes/wait_for_viya_vms
    ```

 1. __Set up hosts routing__

    Add routing information into the `/etc/hosts` file on all machines and set hostnames.
    This is not needed if other host routing mechanisms are in place (e.g. Azure provides a built-in dns server that allows to set hostnames for the VMs).
    
    (Reminder: to skip this role, add `--skip-tags set_host_routing` to the playbook invocation)

    ```
    Role: prepare_nodes/set_host_routing
    ```

 1. __Create SASWORK dir__

    Creates the `/sastmp/saswork` directory on all machines in the `[ProgrammingServicesServers]` host group.  
    
    ```
    Inputs: 
       group_vars: 
         SASWORK_DIR: "/sastmp/saswork"     
    Role: prepare_nodes/create_saswork_dir
    ```
    
 1. __Mount disks__
    
    Mounts disks on the SAS VMs for the SAS Installation directory and user library
    
    ```
    Host Groups:
       NeedMountUserlibDrive
       NeedMountSASInstallDrive
    Inputs: 
       group_vars: 
         SAS_INSTALL_DIR: "/opt/sas"
         USERLIB_DIR: "/opt/sas/viya/config/data/cas"       
       extra_vars: SAS_INSTALL_DISK, USERLIB_DISK
    Role: prepare_nodes/create_saswork_dir
    ```
        
 1. __Mount disks for CAS Cache__

    CLOUD SPECIFIC implementation
    
    Mounts disks on the SAS VMs for the SAS Installation directory and user library
    
    ```
    Host Groups:
       CASControllerServer
    Inputs: 
       group_vars: 
         CASCACHE_DIR: "/sastmp/cascache"
       extra_vars: CASCACHE_DISK
    Role: prepare_nodes/mount_disk
          If no disk is passed in it assumes ephemeral disks or no mount and executes
          prepare_nodes/mount_cascache
    ```

 1. __Download the VIRK predeployment playbook__

    Make the [VIRK pre-install playbook](https://github.com/sassoftware/virk/tree/viya-3.4/playbooks/pre-install-playbook) available
    
    ```
    Host Groups:
       AnsibleController
    Inputs: 
       group_vars: 
         VIRK_COMMIT_ID: 
         VIRK_URL:
         VIRK_DIR:
       extra_vars: VIYA_VERSION
    Role: prepare_nodes/get_virk
    ```
    

#### IAAS specific setup steps:

AWS:
- cloudwatch log
- messages


    
### Set up Users  (OpenLDAP install)

The playbook `ansible/playbooks/openldapsetup.yml` sets up an OpenLDAP server that can be used as initial identity provider for SAS Viya.
Out of the box, these two groups and users are being created:

```
   Group: sasadmin
    User: sasadmin
    
   Group: sasusers
    User: sasuser
```

You can edit the `user_list` variable in `group_vars/openldapall.yml` to create additional users in the `sasusers` group.

```    
    Host Groups:
       OpenLdapServer: single machine that runs the OpenLDAP server
       OpenLdapClients: all machines to configure with the openldap server  
    Inputs: 
       - group_vars/openldapall.yml
       - OLCROOTPW: base64 encoded password for adminuser
       - OLCUSERPW: base64 encoded password for sasuser  
     
```

Example invocation (from aws cfn-init):

``` 
  command: !Sub
    - |
      if [ -n "${ADMINPASS}" ]  && [ -n "${USERPASS}" ]; then
        su -l ec2-user -c '
          export ANSIBLE_LOG_PATH=/var/log/sas/install/openldap.log
          export ANSIBLE_CONFIG=/sas/install/ansible/playbooks/ansible.cfg
          ansible-playbook -v /sas/install/ansible/playbooks/openldapsetup.yml \
            -e "OLCROOTPW='${ADMINPASS}'" \
            -e "OLCUSERPW='${USERPASS}'"
        '
      fi
    - USERPASS: !Base64
        "Ref": SASUserPass
      ADMINPASS: !Base64
        "Ref": SASAdminPass
```    

### Prepare Deployment 

The playbook `ansible/playbooks/prepare_deployment.yml` does additional steps needed before installing SAS, including
- download sas-orchestration
- set up access to deployment mirror (optional)
- build playbook from SOE file
- modify inventory.ini and vars.yml


Logs are routed to `/var/log/sas/install/prepare_deployment.yml`

```
    Inputs: 
       - DEPLOYMENT_MIRROR: location of pre-existing deployment mirror
       - DEPLOYMENT_DATA_LOCATION: location of SOE file
       - ADMINPASS: base64 encoded password for adminuser
       - VIYA_VERSION: 3.3 or 3.4
```

Example invocation:

``` 
   command: !Sub
      - |
      su -l ec2-user -c '
         export ANSIBLE_LOG_PATH=/var/log/sas/install/prepare_deployment.log
         export ANSIBLE_CONFIG=/sas/install/common/ansible/playbooks/ansible.cfg
         ansible-playbook -v /sas/install/common/ansible/playbooks/prepare_deployment.yml \
            -e "DEPLOYMENT_MIRROR=${DeploymentMirror}" \
            -e "DEPLOYMENT_DATA_LOCATION=${DeploymentDataLocation}" \
            -e "ADMINPASS=${ADMINPASS}" \
            -e "VIYA_VERSION=${VIYA_VERSION}"
         '
      - VIYA_VERSION: !GetAtt LicenseInfo.ViyaVersion
        ADMINPASS: !Base64
           "Ref": SASAdminPass
```

 1. __Download sas-orchestration cli__

    Download the sas viya orchestration cli
    
    ```
    Host Groups:
       AnsibleController
    Inputs: 
       group_vars: 
         SAS_ORCHESTRATION_CLI_URL: 
         UTILITIES_DIR:
    Role: prepare_deployment/download_orchestration_cli
    
    ```

 1. __Provide Access to Mirror Repository__

    CLOUD SPECIFIC implementation
    
    Provides access to SAS repository mirror files
    The repository mirror needs to have been created earlier.
    
    For AWS, we do the following 
    - create mirror directory
    - downloads mirror files from s3
    - set up web server  
    
    ```
    Host Groups:
       MirrorServer
    Inputs: 
       group_vars: 
          MIRROR_DIR: 
       extra_vars:
          DEPLOYMENT_MIRROR
    
    Role: prepare_deployment/deployment_mirror
    
    ```

 1. __Download SOE file__

    CLOUD SPECIFIC implementation
    
    in AWS the SOE file, DEPLOYMENT_DATA_LOCATION, is stored in S3
    
    ```
    Host Groups:
       AnsibleController
    Inputs: 
       group_vars: 
          TEMPORARY_SOE_FILE    
       extra_vars:
          DEPLOYMENT_DATA_LOCATION
    
    Role: prepare_deployment/download_soe_file
    
    ```

 1. __Create viya playbook__
    
    runs the `sas-orchestration` cli against the soe file 
    
    ```
    Host Groups:
       AnsibleController
    Inputs: 
       group_vars: 
          MIRROR_URL:
          MIRROR_OPT:
          INSTALL_DIR:
          UTILTIES_DIR: 
          TEMPORARY_SOE_FILE:
       extra_vars:
          DEPLOYMENT_MIRROR (optional)
    
    Role: prepare_deployment/create_viya_playbook
    
    ```

 1. __Modify sas_viya_playbook/vars.yml__
    
    Makes any topology related changes in `vars.yml`.
    
    ```
    Host Groups:
       AnsibleController
    Inputs: 
       group_vars: 
          VIYA_PLAYBOOK_DIR:
          SAS_INSTALL_DIR:
          CASCACHE_DIR:
          SASWORK_DIR:
       other:
          PostgresPrimaryServer host group
    
    Role: prepare_deployment/update_vars
    
    ```

 1. __Modify sitedefault.yml__

    Copies `sitedefault.yml` from the openldap playbook (if that was run) or else creates it.
    Makes any topology related changes in `sitedefault.yml`.
    
    ```
    Host Groups:
       AnsibleController
    Inputs: 
       group_vars: 
          VIYA_PLAYBOOK_DIR:
          SAS_INSTALL_DIR:
          BACKUP_DIR
       extra_vars:
          ADMINPASS
    
    Role: prepare_deployment/update_sitedefault
    
    ```

 1. __Modify inventory__

    Copies `/sas/install/common/playbooks/inventory.ini` to the beginning of `/sas/install/sas_viya_playbook/inventory.ini` 
    and distributes the servers across the viya host groups. 

    ```
    Host Groups:
       AnsibleController
    Inputs: 
       group_vars: 
          VIYA_PLAYBOOK_DIR:
          SAS_INSTALL_DIR:
       extra_vars:
          ADMINPASS
    
    Role: prepare_deployment/update_inventory
    
    ```


### Run VIRK 
    
The [VIRK pre-install playbook](https://github.com/sassoftware/virk/tree/viya-3.4/playbooks/pre-install-playbook) covers most of the Viya Deployment Guide prereqs in one fell swoop.

It is being installed with the `prepare_nodes/get_virk` role in the `prepare_nodes.yml` playbook, into the location `VIRK_DIR: /sas/install/ansible/virk'

Example invocation (run as install user):

```
export ANSIBLE_LOG_PATH=/var/log/sas/install/virk.log
export ANSIBLE_INVENTORY=/sas/install/ansible/sas_viya_playbook/inventory.ini
ansible-playbook -v /sas/install/ansible/virk/playbooks/pre-install-playbook/viya_pre_install_playbook.yml \
 -e "use_pause=false" \
 --skip-tags skipmemfail,skipcoresfail,skipstoragefail,skipnicssfail,bandwidth
```

NOTE: We are using `inventory.ini` from the SAS Viya playbook. One reason for this is: VIRK uses the `[sas-all]` host group,
which only available after setting up the SAS Viya playbook and merging is with the project `inventory.ini`. 



### Install Viya 

Invoke the SAS Viya playbook.

Logs are routed to `/var/log/sas/install/prepare_deployment.yml`
 
Example invocation:

``` 
export ANSIBLE_LOG_PATH=/var/log/sas/install/viya_deployment.log
pushd /sas/install/ansible/sas_viya_playbook
  ansible-playbook -v site.yml
```

### Post Deployment Steps 

Some steps can only be run after the deployment.

Example Invocation (as always, run as install user):

```
export ANSIBLE_LOG_PATH=/var/log/sas/install/post_deployment.log
export ANSIBLE_CONFIG=/sas/install/ansible/playbooks/ansible.cfg
ansible-playbook -v /sas/install/ansible/playbooks/post_deployment.yml
```

 1. __Create Shared Backup directory__

    The Viya Backup/Restore manager requires a shared common location called "sharedVault". In short, the backup works in 2 steps:
    1 - it creates a local backup on a disk location local to each VM
    2 - it copies and consolidates the local backups into a shared location.
    http://go.documentation.sas.com/?cdcId=calcdc&cdcVersion=3.3&docsetId=calbr&docsetTarget=p0mo412bkanvwwn1off0b1gd10tz.htm&lo
    
    The vault location needs to be set to the "sas" user. The "sas" user is created by the VIRK playbook, so this step needs to run at some time after VIRK.
    
    We have two roles for this:
    - `create_shared_backup_dir` creates a directory on the `[SharedVaultServer]` and exports it as nfs export
    - `mount_shared_backup_dir` mounts the directory on all `[sas-servers]`
    
    
    NB: the location of the backup directory is set in `sitedefault.yml` as
    
    ```
    sas.deploymentbackup:
        sharedVault:
    ```
    
    See the `prepare_deployment/update_sitedefault` role for details.
    
    ```
    Host Group:
       SharedVaultServer
    Inputs:
       group_vars:
         BACKUP_NFS_DIR:
         BACKUP_DIR:
    
    Role: post_deployment/create_shared_backup_dir
    ```

 1. __Mount Shared Backup directory__

    For details, see the previous section "Create Shared Backup Directory"
    
    ```
    Host Group:
       sas-servers
    Inputs:
       group_vars:
         BACKUP_NFS_DIR:
         BACKUP_DIR:
    
    Role: post_deployment/mount_shared_backup_dir
    ```
    
 1. __Install MySQL clilent__
    
     Installs the mysql client 5.6 on workspace server and cas controller
     
     ```
     Host Group: 
        NeedDatabaseAccessClients
     Inputs: 
        -none-
        
     Role: post_deployment/install_mysql_client
        ```