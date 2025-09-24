# PXE Boot Installation Guide for OpenShift 4.

# (CoreOS)

## Step 1: Prepare PXE Server

```
Install required services (on a Linux bastion/PXE server):
```
```
sudodnfinstall -y dhcp-servertftp-serversyslinux httpd
sudosystemctlenable --nowdhcpdtftphttpd
```
```
Configure DHCP /etc/dhcp/dhcpd.conf:
```
```
subnet 192.168.1.0 netmask 255.255.255.0 {
range 192.168.1.100 192.168.1.200;
option routers 192.168.1.1;
option domain-name-servers 8.8.8.8;
next-server 192.168.1.10;
filename "pxelinux.0";
}
```
```
Configure TFTP Boot Directory /var/lib/tftpboot/:
```
```
sudocp /usr/share/syslinux/pxelinux.0/var/lib/tftpboot/
sudomkdir-p /var/lib/tftpboot/pxelinux.cfg
```
## Step 2: Prepare RHCOS Images

```
Download the OpenShift 4.17 CoreOS images:
```
```
wgethttps://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.17/
latest/rhcos-4.17.0-x86_64-live.x86_64.iso
wgethttps://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.17/
latest/rhcos-4.17.0-x86_64-metal.x86_64.raw.gz
```
```
Extract kernel and initramfs:
```
### 1.

### 2.

### 3.

### 1.

### 2.


```
mkdir-p /var/lib/tftpboot/rhcos
bsdtar -C /var/lib/tftpboot/rhcos-xf rhcos-4.17.0-x86_64-live.x86_64.iso
```
```
Create PXE boot config /var/lib/tftpboot/pxelinux.cfg/default:
```
```
DEFAULT rhcos
LABEL rhcos
KERNEL rhcos/images/pxeboot/vmlinuz
APPEND initrd=rhcos/images/pxeboot/initrd.img coreos.inst.install_dev=sda
coreos.inst.image_url=http://192.168.1.10/rhcos/rhcos-4.17.0-x86_64-
metal.x86_64.raw.gz ip=dhcp
```
## Step 3: Boot Nodes via PXE

```
Set nodes (masters and workers) to boot from network.
Nodes will automatically fetch the kernel/initrd and start CoreOS installation.
Confirm the installation via console or serial.
```
## Step 4: Prepare OpenShift Installation

```
Create install-config.yaml:
```
```
apiVersion: v
baseDomain: example.com
metadata:
name: openshift
platform:
none: {}
compute:
```
- name: worker
    replicas: 2
controlPlane:
    name: master
    replicas: 3
networking:
    networkType: OVNKubernetes
pullSecret: '{"auths":...}'
sshKey: 'ssh-rsa AAAAB3...'

```
Create ignition files:
```
### 3.

### 1.

### 2.

### 3.

### 1.

### 2.


```
openshift-installcreate ignition-configs --dir=install_dir
```
## Step 5: Use oc to Verify and Join Nodes

```
Export KUBECONFIG:
```
```
export KUBECONFIG=install_dir/auth/kubeconfig
```
```
Verify nodes:
```
```
oc get nodes
```
```
Approve CSR for new nodes if required:
```
```
oc get csr
oc adm certificateapprove <csr_name>
```
## Step 6: Post-install Configuration

```
Label nodes as needed:
```
```
oc labelnode<node_name> node-role.kubernetes.io/worker=worker
```
```
Check cluster operators:
```
```
oc get co
```
```
Ensure all nodes are ready:
```
```
oc get nodes-o wide