# TiTO Ansible

This repository contains two Ansible playbooks which can be used to install the [Time Tracking Overview (TiTO) application](https://github.com/vmeoc/Tito).

This configuration builds on from a previous usecase of using Ansible with no central server. It assumes the setting up of Cloud Zone, Templates, Flavor\ Image Mapping and Network Profile is complete as described in [titoAnsibleHeadless repository](https://github.com/darrylcauldwell/titoAnsibleHeadless).

## Ansible Server Blueprint

This very simple blueprint simply deploys the VM template which contains Ansible with a a static IP address. There is no specific need for static IP here,  but I am using a static IP address as I am using a static DNS record.

As well as configuring network this generates a SSH authorization key which we can add to each Ansible client to create a simple security context server->client.

## Simplest Ansible Client Blueprint

This is a simple blueprint simply deploys the VM template. This uses an input parameter of SSH authorization key and uses cloud-init to create a user called 'ansible' which can be remotely called from Ansible server.

### Test Communications

If we deploy the Ansible Server then connect via SSH and type the public key string to screen.

```
cat ~/.ssh/id_rsa.pub
```

We can copy this from SSH session and paste as input to Ansible Client blueprint.

If all is good then when the Ansible Client blueprint is deployed you should be able to SSH to it from Ansible server without password and have sudo rights.

```
ssh ansible@<ansible-client-ip>
sudo yum update
```

## Cloud Assembly Ansible Integration

There is a native integration of Ansible with Cloud Assembly there is [full documentation for configuring the integration](https://docs.vmware.com/en/VMware-Cloud-Assembly/services/Using-and-Managing/GUID-9244FFDE-2039-48F6-9CB1-93508FCAFA75.html?hWord=N4IghgNiBc4HYGcCWAjCBTEBfIA).

The Cloud Assembly integration requires some client side configuration which the ansibleClient blueprint deploys and also some ansibleServer configuration.  I haven't yet added and tested this in the ansibleServer blueprint so for now we need to run some commands.

First we need to set a Vault password and change Ansible configuration file to use this.

```bash
cat >/etc/ansible/vault <<\EOF
---
vault_sudo_password: “VMware1!”
EOF

sed -i 's/#vault_password_file/vault_password_file/g' /etc/ansible/ansible.cfg
sed -i 's#/path/to/vault_password_file#/etc/ansible/vault#g' /etc/ansible/ansible.cfg
```

We also need to disable host key checking, this is also a change to Ansible configuration file.

```
sed -i 's/#host_key_checking/host_key_checking/g' /etc/ansible/ansible.cfg
```

Once Ansible server is configured we can add it as an Integration in Cloud Assembly (Infrastructure > Connections > Integrations).

## Add To Ansible Inventory Blueprint

So now Ansible server is available in Cloud Assembly we can this to a blueprint.  When we attach this to a VM we populate some basic details about our Ansible server such as which private key to use,  if using the example ansibleCas.yml in this repository ensure account matches the name you gave Ansible integration,  for me this was string 'DC-Ansible'.

When we deploy this blueprint we can check see the IP address added to Ansible server inventory file /etc/ansible/hosts.

In this blueprint I added to root,  but the Ansible object can be passed groups to be member.

## Run a Ansible Playbook

In the [titoAnsibleHeadless repository](https://github.com/darrylcauldwell/titoAnsibleHeadless) I created two playbooks.  One of which installs TiTO web server and the other a mysql server.

If we download these to Ansible server.

```
mkdir /etc/ansible/playbooks
wget https://raw.githubusercontent.com/darrylcauldwell/titoAnsible/master/titoPlaybook.yml -O /etc/ansible/playbooks/titoPlaybook.yml
```
