# TiTO Ansible

This repository contains two Ansible playbooks which can be used to install the [Time Tracking Overview (TiTO) application](https://github.com/vmeoc/Tito).

This configuration builds on from a previous usecase of using Ansible with no central server. It assumes the setting up of Cloud Zone, Templates, Flavor\ Image Mapping and Network Profile is complete as described in [titoAnsibleHeadless repository](https://github.com/darrylcauldwell/titoAnsibleHeadless).

## Ansible Server Blueprint

This very simple blueprint simply deploys the VM template which contains Ansible with a a static IP address. There is no specific need for static IP here,  but I am using a static IP address as I am using a static DNS record.

As well as configuring network this generates a SSH authorization key which we can add to each Ansible client to create a simple security context server->client.

## Ansible Client Blueprint

This is a simple blueprint simply deploys the VM template which does not contain Ansible. This uses an input parameter of SSH authorization key and uses cloud-init to create a user called Ansible which can be remotely called from Ansible server.

