# Install Code Ready Containers (CRC)

In this lab we will see how to install CRC which is a development version of OpenShift. We will run it on an Azure VM.

> **Disclaimer:**
> This section of the course requires a VM in Azure that costs money. You are welcome to follow along but please note that we are not responsible for any charges you may incur.

## Parameters

Use the following parameters

- VM Size: Standard D8s v3
- OS: Ubuntu 20.04
- vCPUs: 8
- RAM: 32 GiB
- Disk: 30 GB (comes default can't change in the beginning)
- Security Ingress ports: 80, 443, 22, 8200, 5000, 8001, 8002 (You actually only need 80, 443, 22 as we'll use routes with OpenShift)
- username: ubuntu
- Generate a new ssh-key or use an existing one you have in Azure

After you've created the VM. Stop it and go to disks and then click on the OS disk name and then size + performance from the left pane. Then resize it to 100 GB and restart the VM.

## Download and Install CRC

ssh into the machine

```bash
ssh -i <ssh-key.pem> ubuntu@<x.x.x.x>
```

Install libvirt and NetworkManager packages for ubuntu
```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon libvirt-daemon-system network-manager
```

From your computer's browser go to https://console.redhat.com/openshift/create/local

Login or create a RedHat account

Choose `Linux` and right-click `Download OpenShift Local` and `copy link address`

At the time of this writing this is the link:
https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz

Also download the pull secret.

in your Ubuntu machine run this:

```bash
wget https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz
tar xvf crc-linux-amd64.tar.xz
sudo mv crc-linux-*-amd64/crc /usr/bin
crc setup
```

You can find the [Linux install guide](https://access.redhat.com/documentation/en-us/red_hat_codeready_containers/2.0/html/getting_started_guide/installation_gsg#linux)


## Change the Network Stack

You may run into an issue around networking as shown below:

```
INFO Checking if systemd-networkd is running
Network configuration with systemd-networkd is not supported. Perhaps you can try this new network mode: https://github.com/code-ready/crc/wiki/VPN-support--with-an--userland-network-stack
```

In this case, you can change the network stack as follows:

- Cleanup the previous installation of crc.
Run crc delete, crc cleanup and remove the folder $HOME/.crc
- Remove eventual *.crc.testing records in your hosts file /etc/hosts.
- Activate the user network mode.
```bash
crc config set network-mode user
```
- Prepare the host machine
```bash
crc cleanup
crc setup
```
- Increase the CPU and RAM from the defaults
```bash
crc config set cpus 8
crc config set memory 25600
```
- Start the virtual machine
Remember to copy and paste the pull secret when asked.
```bash
crc start
```

Make sure to save the password for the `kubeadmin` user. 

## Update Your Local Computer's Host File

To access CRC from your local computer's browser, we will need to update the host file on your local computer to point to the ip of the VM in Azure.

For Windows users:

Open Notepad and run it as an administrator:
then click open inside of Notepad and open this file:
`C:\Windows\System32\drivers\etc\hosts`

Your local host file would look something like this:

```
<azure_public_ip_address> kubernetes.docker.internal api.crc.testing canary-openshift-ingress-canary.apps-crc.testing console-openshift-console.apps-crc.testing default-route-openshift-image-registry.apps-crc.testing downloads-openshift-console.apps-crc.testing oauth-openshift.apps-crc.testing cluster-openshift-gitops.apps-crc.testing kam-openshift-gitops.apps-crc.testing 
<azure_public_ip_address> openshift-gitops-server-openshift-gitops.apps-crc.testing 
<azure_public_ip_address> argocd-server-argocd.apps-crc.testing
<azure_public_ip_address> schoolapp-schoolapp.apps-crc.testing 
<azure_public_ip_address> api-schoolapp.apps-crc.testing
<azure_public_ip_address> vault-vault.apps-crc.testing
```

For more details for windows you can [follow this guide.](https://helpdeskgeek.com/networking/edit-hosts-file/)

## Log into CRC

Make sure you're ssh'ed into the Azure VM and run this command to enable oc commands:
```bash
eval $(crc oc-env)
oc login -u kubeadmin https://api.crc.testing:6443
```

Open a browser window on your local computer and go to the following URL to access the UI
https://console-openshift-console.apps-crc.testing

login as the `kubeadmin` user and use the password that was shown in the output of the `crc start` command.

> This concludes this lab