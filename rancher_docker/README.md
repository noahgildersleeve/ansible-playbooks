# Rancher Docker

This is a ansible playbook that deploys out all of the pre-requisites for Docker and then pulls and deploys Rancher as a single node Docker instance based on an image tag from Docker Hub.

This is part of the more general ansible_playbooks repo available [here](../README.md)

## Tested Configurations

I tested this as deployed VMs on [Harvester](https://github.com/harvester/harvester) using live cloud images with the following guest operating systems:

- openSUSE 15.6
- SLES 15.5
- Ubuntu Focal (20.04)
- Ubuntu Jammy (24.04)

There are some specific configuration options that are worth pointing out. Also you need to have your SSH key added to the machines in order for Ansible to connect. You can do this either manually or with `cloud-init`.

### openSUSE

For openSUSE to work with these playbooks you need to install `python311`. This will allow you to specify `python3.11` as the Python interpreter. The default version of Python won't work with ansible.

You can do that by either running `sudo zypper install python311`, or by including it in the cloud-init if you are deploying it via harvester or any platform supporting `cloud-init`. An example of a packages section of a `cloud-init` block in User Data is as follows:

```yaml
packages:
  - qemu-guest-agent
  - python311
  - pciutils
```

### SLES

I tested this on SLES 15.5, but it should work on other versions although you'll need to add the associated version specific modules outlined later. You will need an active SLES license, and some other registrations depending on the playbook you're using. This is a copy of the cloud-init script I was using. You can see by comments which ones are needed for each playbook.

`python311` is installed in the `runcmd` section because the packages section didn't install them properly. You can also manually activate the associated modules via the command line. I also created a default user. The ansible playbook is pointing at the homedirectory for the `sles` user, but you can change it in the config.

```yaml
#cloud-config
password: password
chpasswd: { expire: False }
sshpwauth: True
users:
  - default
  - name: sles
    sudo: 'ALL=(ALL) NOPASSWD:ALL'
    lock_passwd: false
    shell: /bin/bash
    plain_text_passwd: password
package_update: true
ssh_authorized_keys:
  - ssh-key <insert ssh key here>
packages:
  - qemu-guest-agent
  - python311
  - pciutils
runcmd:
  # This is needed to activate SLES
  - sudo SUSEConnect -r REGCODE -e emailAddress@email.com
  # This is for the rancher_docker deploy
  - sudo SUSEConnect --product sle-module-containers/15.5/x86_64
  - SUSEConnect --product sle-module-public-cloud/15.5/x86_64
  # This is needed for the GPU deployment
  - sudo SUSEConnect -p sle-module-desktop-applications/15.5/x86_64
  - sudo SUSEConnect --product sle-we/15.5/x86_64 -r REGCODE
  - sudo SUSEConnect -p PackageHub/15.5/x86_64
  # This is for running ansible, qemu-guest-agent, and lspci
  - sudo zypper -n install python311 pciutils qemu-guest-agent
  - - systemctl
    - enable
    - --now
    - qemu-guest-agent.service

```

### Ubuntu Focal 20.04

There is no specific requirements for Ubuntu Focal. There is just some conditional logic for the Python Docker module.

### Ubuntu Jammy 24.04

There is no specific requirements for Ubuntu Jammy. There is just some conditional logic for the Python Docker module.

## Ansible Requirements

There are a few ansible collections required to use the ansible playbooks

### Rancher_docker

Run the following commands to install the associated collections

- `ansible-galaxy collection install community.docker community.general`

## Configuration

Configuration for this playbook is done mostly in the [hosts](./hosts) file. You can set the hosts there and specify which distro you are running on them. You can also specify the `default_container_image` for all of the hosts in `vars:all` or you can specify a specific one per host and override the default.

The other configuration is done with the [default.yml](./vars/default.yml) file. You won't generally have to change this, although you can set the default Rancher password (`CATTLE_BOOTSTRAP_PASSWORD`) there after deploy. It is set by default to `password1234`