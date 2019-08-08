# TiTO Ansible

This repository contains an [Ansible playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html) that can be used to install the [Time Tracking Overview (TiTO) application](https://github.com/vmeoc/Tito). The TiTO application has a simple two tier architecture, the frontend uses Apache to host a PHP website and the backend is a MySQL database.

It also contains a VMware Cloud Automation Services blueprint which builds two VMs with Ansible installed. The blueprint then instructs each VM to pull and execute the correct playbooks to install the application to the correct tier.

## Ansible Control Node

To execute the Ansible playbook we need to configure a Ansible Control Node.

The full installtion steps for the various Linux distributions  can be found [here](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).

For the lab I tested in I used Ubuntu 16.04 so used following commands.

```bash
apt update
apt install software-properties-common
apt-add-repository --yes --update ppa:ansible/ansible
apt install ansible
```

Full details of the Ansible configuration required for Cloud Assembly integration can be found [here](https://docs.vmware.com/en/VMware-Cloud-Assembly/services/Using-and-Managing/GUID-9244FFDE-2039-48F6-9CB1-93508FCAFA75.html).

For Cloud Assembly integration we require Ansible 2.6 or higher, we can check version by running.

```bash
ansible --version
```

For Cloud Assembly integration we are required to create a vault password file with a sudo password.  To create this change my-password to what you need and run.

```bash
cat >/etc/ansible/vault <<\EOF
---
vault_sudo_password: “my-password”
EOF
```

Ansible runtime configuration is kept in a file. Part of the configuration is regarding the vault password file. The default installation has this commented out and with an invalid path.

As well as vault file path Cloud Assembly integration requires we need to disable host key checking.

We can un-comment the path line, update vault password path and update host key checking using sed and pattern matching.

```bash
sed -i 's/#vault_password_file/vault_password_file/g' /etc/ansible/ansible.cfg
sed -i 's#/path/to/vault_password_file#/etc/ansible/vault#g' /etc/ansible/ansible.cfg
sed -i 's/#host_key_checking/host_key_checking/g' /etc/ansible/ansible.cfg
```

In order for the Ansible control node to connect to managed clients we require an SSH authorization key pair. The Cloud Assembly integration with Ansible will require access to the private key file on the Ansible control node. The account which will be used for the Cloud Assembly integration with Ansible should be used to create the SSH authorization key pair.

To create the private public key pair and output the public key. Ensure your shell is running as the Cloud Assembly integration with Ansible user and run these two commands. 

```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub
```

The string output of id_rsa.pub is used later as an input parameter to the Cloud Assembly blueprint.

When ran interactively an Ansible playbook outputs to the console. As Cloud Assembly is running the Ansible playbooks by default we don't see the output. Ansible logging is disabled by default. For testing and learning in a lab environment it is very useful to enable Ansible logging. We can do this by uncommenting a line in the Ansible configuration file.

```
sed -i 's/#log_path/log_path/g' /etc/ansible/ansible.cfg
```

When we are running Cloud Assembly blueprints which call Ansible it is useful to have an SSH session to the Ansible server and follow the log file.

```bash
tail -f /var/log/ansible.log
```

## Cloud Assembly Ansible Integration

Once the pre-requsit Ansible control node configuration is complete this can be added as an integration with Cloud Assembly. Full details of how to add the integration can be found [here](https://docs.vmware.com/en/VMware-Cloud-Assembly/services/Using-and-Managing/GUID-9244FFDE-2039-48F6-9CB1-93508FCAFA75.html). 

## TiTO Ansible Playbook

The TiTO Ansible Server blueprint calls the Ansible control node to manage execution of the Ansible playbook. The TiTO Ansible playbook needs to be stored on the Ansible control node prior to deploying TiTO Cloud Assembly blueprint.

The Ansible playbooks can be located anywhere on Ansible control node. In the example TiTO Cloud Assembly blueprint the /etc/ansible/playbooks is used.  To create this folder and populate the Ansible playbook run the following.

```
mkdir /etc/ansible/playbooks
wget https://raw.githubusercontent.com/darrylcauldwell/titoAnsible/master/titoXosPlaybook.yml -O /etc/ansible/playbooks/titoXosPlaybook.yml
```

## TiTO Ansible Inventory File

The TiTO Ansible playbook depends on some configuration on Ansible control node.  It requires two host groups creating in the [Ansible inventory file](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html). The group names which need to be added to the inventory are are [titoXosDatabase] and [titoXosWebservers]. These group names are also specified in the Cloud Assembly blueprint in order that during provisioning VMs get placed in correct Ansible inventory group which playbook relies on.

```bash
root@localhost:/# vi /etc/ansible/hosts
[titoXosWebservers]

[titoXosDatabase]
```

## TiTO Ansible Server Cloud Assembly Blueprint

The TiTO Ansible Server blueprint depends on a deployment environment configured in Cloud Assembly with some explicit properties. The explicit properties can be changed in the blueprint to match environment.

1. A valid Cloud Zone tagged with target:vsphere 
2. A Ubuntu 16.04 VM template with cloud-init installed and working in Cloud Zone with Image Mapping named im-ubuntu16046
3. A Flavor Mapping named fl-small with 2x CPU and 4GB RAM
4. A valid Network Profile with internet connectivity in Cloud Zone tagged with environment:production
5. The Ansible integration is named int-ansible-public

The TiTO Cloud Assembly blueprint is a file stored in this repository named [tito-ansibleServer.yml](https://raw.githubusercontent.com/darrylcauldwell/titoAnsible/master/tito-ansibleServer.yml). Create a new Cloud Assembly blueprint and copy paste the contents of the file into the new blueprint.

Next take the SSH public key we created on Ansible control node earlier and update the default value for the input parameter ansible-key with the contents of Ansible control node public key.

You should now be able to deploy the blueprint. Once it completes successfully you should be able to point web browser to IP of web server and get TiTO.

* Troubleshooting Tip 1, Before deploying the blueprint you can see more of what is happening by following the Ansible control node log file and Ansible inventory file. When we deploy this blueprint we can check see the IP address added to Ansible server inventory file /etc/ansible/hosts. We can then see the Ansible playbook output in the log file /var/log/ansible.log as it executes.

* Troubleshooting Tip 2, the Ansible playbook is written to install application on either Ubuntu16.04 or CentOS7. It uses conditional logic to do this so you will see some tasks skipped,  this is normal.

* Troubleshooting Tip 3, it is important that there is good network connectivity between Ansible control node and provisioned guests.  In lab we experienced issue with poor connectivity which caused SFTP issues, moving Ansible control node to more stable network helped.

* Troubleshooting Tip 4, if new Linux user on Ansible control node is setup ensure account is owner of ~/.ansible this is where temporary files are created before being SFTP to client.  If issues occur and permissions change Ansible fact gathering can hang,  if this occurs clear all files and folders from ~/.ansible.

## Perform Day 2 Operation

After the initial deployment of VMs and application, day two operations will be required.  As the application VMs have their configuration state managed by the Ansible server this can be used to perform the day two operations.

To demonstrate this first download this very simple playbook to the Ansible server.

```
wget https://raw.githubusercontent.com/darrylcauldwell/titoAnsible/master/rename.yml -O /etc/ansible/playbooks/rename.yml
```

We can then execute this to change the hostname of the member of the inventory group titoWebserver.

```
ansible-playbook rename.yml
```

If we refresh the TiTO application page we can see the change as index.php now shows 'Runs on : ive-been-changed'.
