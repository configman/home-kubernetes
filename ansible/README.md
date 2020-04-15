# Missing bits



* Create ssh keys for root user on each master node of the cluster
  *  ssh-keygen -f <secure-file-location>/master1
  * edit the public key file to change the user ID to be 'root@hostname-of-master'
* If not already done create your own ssh key files

* setup an ansible vault for secrets (remove existing group_vars/all.yml if it exists)
  * https://docs.ansible.com/ansible/latest/user_guide/vault.html
```
ansible-vault create --vault-id k8s@prompt group_vars/all.yml
```

  * Add your user's public ssh key to file group_vars/all.yml with key 
```
{{ super_user }}_public_key: "<public key value>"

```
  * add private key for root user on master node with key 
```
root_{{ inventory_hostname }}_private_key: "<private key value>"
```
    * The key value must be on a single line with '\n' defining the required line breaks in the key file.  If this is not done the created id_rsa file will  not be valid
  * add public key for root user on master node(s):
```
root_{{ inventory_hostname }}_public_key: "<public key value>"
```

(note that the above setup assumes that your username is provided by the environment variable $USER and that the public key file contains a user identity value that looks like '$USER@machine-name')

* Configure the hosts.yml file with your list of machines to be used in the cluster

## Running 

This setup runs on a clean copy of ubuntu 18.04 pi image.  The only require action prior to executing ansible playbooks is to log in on each machine to change the default password for the ubuntu user from 'ubuntu' to 'raspberry1' (no quotes).

1. Execute the os-update.yml playbook.  This runs apt-get update and apt-get upgrade on the servers to ensure we have the latest OS packages installed before starting the k8s install

```
asible-playbook --vault-id k8s@prompt -i hosts.yml os-update.yml
```

2. Execute the bootstrap.yml.  This configures cgroups as required by kubernetes and creates an admin user with the name provided on the commandline as 'super_user'

```
ansible-playbook --vault-id=k8s@prompt -i hosts.yml -e "super_user=<userId>" bootstrap.yml
```

3. Run kubernetes master setup

```
ansible-playbook --vault-id k8s@prompt -i hosts.yml -e "super_user=<userId>" kube-masters.yml
```

4. Run node setup

```
ansible-playbook --vault-id k8s@prompt -i hosts.yml -e "super-user=<userId>" kube-nodes.yml


## Issue running playbooks

Running the kube-masters.yml playbook tends to fail first time through.  This seems to be due to timeout when running the kubeadm init process.  Separating the image pull from the could help, but there is also a timeout when starting the pods for the control plane.  I haven't yet found anytihng that refers to how to control the init timeout, but running kubeadm reset followed by a rerun of the playbook usually works.

When installing K8S the process seems to ssh to the host that is being installed from that host, so needed to add the host's key to the known_hosts file before installing

## Runnig CNI

Tried Calico - it failed on startup while attempting to pull the image for one of the components.  Switched to Flannel and worked perfectly.  Other blogs also mention this issue.

## Something to consider

* In my setup I have a router that already has the mac addresses of the Pis assigned to a fixed IP, so I don't need to discover the IP address of the Pis and I do not have to configure fixed IP address in the system config.  You may want to add setup of the fixed IP to an ansible playbook or at least configure the fixed IP address prior to running this setup process.

# Additional requirements

Add /etc/docker/daemon.json file with 

{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}

then add folder

/etc/systemd/system/docker.service.d

then 

systemctl daemon-reload
systemctl docker restart

do this before installing K8S

see - https://kubernetes.io/docs/setup/production-environment/container-runtimes/

and note the warning that is dispalyed when installing without doing this


## Adding external drive for storage

format using parted 
select /dev/sda
mklabel sga
mkpart ext4

on exit from parted mkfs.ext4 on the partitions

Add to Ansible:

mount the external partitions
install nfs-kernel-server

add exports to /etc/exports

/folder/to/export <IPs allowed to Access>(rw,sync,[no\_]root\_squash,subtree\_check)

/mnt/pi1 \*(rw,sync,no\_root\_squash,subtree\_check)

root\_squash prevents root access with write privileges, only allows read by root user.  no\_root\_squash can be dangerous because it allows privileged access that could destroy data on a mounted partition so is usually avoid.  since K8S runs mostly as a root user I decided to allow root write privileges

## Setting up imac with ubuntu and k8s

* Added feature to allow the hosts on which the playbooks are applied to be overridden:
```
ansible ansible-playbook -e "masters=hostname list" kube-masters.yml
```

now able to use this on both pi master and imac master

* Improved kubernetes dependencies setup, included disabling swap, which is necessary to be able to use k8s

* Added a distribution playbook to run a full upgrade and dist-upgrade on a machine - this is necessary before installing k8s
  * \_\_BUG\_\_ containerd install initially fails with error that apt cannot find it but after manually running apt-get update then apt-get install containerd (which works) it then does work properly the playbook works correctly.


  # Restart mac mini automatically after powerfail

Mini uses nvidia chipset.  some resources report action for when intel chipset, which doesn't work on nvidia.

  http://dioxaz.free.fr/?p=320

  The above works, in short

  setpci -s 0:03.0 0x7b.b=20

  lspci -s 0:03.0 # to check before running, look at byte 7b, expected to be 60 (hex)

  after execution check state of byte by ...

  setpci -vD -s 0:03.0 0x7b.b

  # Mini - remove clout-init

  Using the ISO that solved the 32 bit boot problem brings cloud-init with it
.  Thisfailed to startup up networking properly after a whie. Removing it...

https://makandracards.com/operations/42688-how-to-remove-cloud-init-from-ubuntu

```shell
echo 'datasource_list: [ None ]' | sudo -s tee /etc/cloud/cloud.cfg.d/90_dpkg.cfg
sudo apt-get purge cloud-init
sudo rm -rf /etc/cloud/; sudo rm -rf /var/lib/cloud/
reboot
```

Now set netplan config...

Remove /etc/netplan/50-cloud-init.yaml

create /etc/netplan/01-netcfg.yaml with ...

```shell
network:
    version: 2
    renderer: networkd
    ethernets:
        enp4s0f0:
            dhcp4: true
            addresses: []
            optional: true
```

__NOTE__ This is getting a fixed IP from DHCP assigned to interface MAC address by router in my setup.  Fixed IP address is recommended if this is not your setup.

# Booting a mac from SD card

Hold down option key whilst machine is booting to boot from SD car or other external drive

# blanking screen on imac

edit /etc/default/grub, updating these lines to be as shown

```shell
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash consoleblank=10"
GRUB_CMDLINE_LINUX="consoleblank=10"
```

original lines 

```shell
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""
```
# Update 20200224 - rebuilding loses cmdline.txt for cgroups

After running a full apt-get upgrade on a new install of ubuntu on Pi the cmdline.txt file disappears and the changes are lost.  The /boot/firmware/README file details that there is a hierarchical collection of configuration files rather than a single one.  The changes need to be made to the /boot/firmware/usercfg.txt file instead of cmdline.txt

Actually the new system defaults to loading the nobtcmd.txt file, which contains the original boot cmd params.  We need to add to this one the cgroup command params that were originally added to cmdline.txt.  nobtcmd.txt is for pi without bluetooth; if we are using bluetooth then would have to change to the btcmd.txt file instead


