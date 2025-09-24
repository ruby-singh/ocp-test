# OpenShift PXE Boot Installation Guide - Agent-based Installer

## Prerequisites

- OpenShift Container Platform 4.17 access credentials
- Linux system with `dnf` package manager
- Network infrastructure configured for PXE booting
- HTTP/HTTPS server for hosting PXE assets
- Pull secret from Red Hat OpenShift console
- SSH public key for cluster access

---

## Step 1: Download Required Tools

1. **Log in to OpenShift web console**
   - Navigate to: https://console.redhat.com/openshift/create/datacenter
   
2. **Download the Agent-based Installer**
   - Click "Run Agent-based Installer locally"
   - Select your operating system and architecture
   - Click "Download Installer" to download and extract

3. **Download Command Line Tools**
   - Click "Download command-line tools"
   - Place the `openshift-install` binary in a directory on your PATH

4. **Save Pull Secret**
   - Click "Download pull secret" or "Copy pull secret"
   - Save for later use in configuration

---

## Step 2: Install Dependencies

Install the nmstate dependency:

```bash
sudo dnf install /usr/bin/nmstatectl -y
```

---

## Step 3: Create Installation Directory

Create a working directory for installation files:

```bash
mkdir ~/ocp-pxe-install
cd ~/ocp-pxe-install
```

---

## Step 4: Create install-config.yaml

Create the main installation configuration file:

```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: my-cluster
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 2
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.1.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  baremetal:
    apiVIPs:
    - 192.168.1.10
    ingressVIPs:
    - 192.168.1.11
pullSecret: '<YOUR_PULL_SECRET>'
sshKey: '<YOUR_SSH_PUBLIC_KEY>'
```

Replace:
- `<YOUR_PULL_SECRET>` with your actual pull secret
- `<YOUR_SSH_PUBLIC_KEY>` with your SSH public key
- Network CIDRs with your actual network configuration
- Domain and cluster name as needed

---

## Step 5: Create agent-config.yaml

Create the agent configuration file:

```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: my-cluster
rendezvousIP: 192.168.1.20
bootArtifactsBaseURL: http://192.168.1.100/pxe
hosts:
- hostname: master-0
  role: master
  interfaces:
  - name: eno1
    macAddress: 00:11:22:33:44:55
  rootDeviceHints:
    deviceName: /dev/sda
  networkConfig:
    interfaces:
    - name: eno1
      type: ethernet
      state: up
      ipv4:
        enabled: true
        address:
        - ip: 192.168.1.21
          prefix-length: 24
        dhcp: false
    dns-resolver:
      config:
        server:
        - 192.168.1.1
    routes:
      config:
      - destination: 0.0.0.0/0
        next-hop-address: 192.168.1.1
        next-hop-interface: eno1
- hostname: master-1
  role: master
  interfaces:
  - name: eno1
    macAddress: 00:11:22:33:44:56
  rootDeviceHints:
    deviceName: /dev/sda
  networkConfig:
    interfaces:
    - name: eno1
      type: ethernet
      state: up
      ipv4:
        enabled: true
        address:
        - ip: 192.168.1.22
          prefix-length: 24
        dhcp: false
    dns-resolver:
      config:
        server:
        - 192.168.1.1
    routes:
      config:
      - destination: 0.0.0.0/0
        next-hop-address: 192.168.1.1
        next-hop-interface: eno1
- hostname: master-2
  role: master
  interfaces:
  - name: eno1
    macAddress: 00:11:22:33:44:57
  rootDeviceHints:
    deviceName: /dev/sda
  networkConfig:
    interfaces:
    - name: eno1
      type: ethernet
      state: up
      ipv4:
        enabled: true
        address:
        - ip: 192.168.1.23
          prefix-length: 24
        dhcp: false
    dns-resolver:
      config:
        server:
        - 192.168.1.1
    routes:
      config:
      - destination: 0.0.0.0/0
        next-hop-address: 192.168.1.1
        next-hop-interface: eno1
- hostname: worker-0
  role: worker
  interfaces:
  - name: eno1
    macAddress: 00:11:22:33:44:58
  rootDeviceHints:
    deviceName: /dev/sda
  networkConfig:
    interfaces:
    - name: eno1
      type: ethernet
      state: up
      ipv4:
        enabled: true
        dhcp: true
- hostname: worker-1
  role: worker
  interfaces:
  - name: eno1
    macAddress: 00:11:22:33:44:59
  rootDeviceHints:
    deviceName: /dev/sda
  networkConfig:
    interfaces:
    - name: eno1
      type: ethernet
      state: up
      ipv4:
        enabled: true
        dhcp: true
```

Key fields to configure:
- `rendezvousIP`: IP address for bootstrap node
- `bootArtifactsBaseURL`: URL where PXE assets will be hosted
- `hosts`: Define each node with network configuration
- MAC addresses for each interface
- Static IP or DHCP configuration

---

## Step 6: Generate PXE Boot Files

Run the following command to create PXE assets:

```bash
openshift-install agent create pxe-files
```

This creates a `boot-artifacts` directory containing:
- `agent.x86_64-initrd.img` - Initial RAM disk image
- `agent.x86_64-vmlinuz` - Kernel image
- `agent.x86_64-rootfs.img` - Root filesystem image
- `agent.x86_64.ipxe` - iPXE script (optional)

---

## Step 7: Upload PXE Assets to Server

1. **Copy files to your HTTP/TFTP server:**

```bash
# Example for Apache web server
sudo cp boot-artifacts/* /var/www/html/pxe/

# Set proper permissions
sudo chmod -R 755 /var/www/html/pxe/
```

2. **Verify files are accessible:**

```bash
curl http://192.168.1.100/pxe/agent.x86_64-vmlinuz
```

---

## Step 8: Configure PXE Server

### For Traditional PXE (TFTP/DHCP)

1. **Configure DHCP server** to point to TFTP server:

```conf
# dhcpd.conf example
next-server 192.168.1.100;
filename "pxelinux.0";
```

2. **Create PXE boot menu** (`/tftpboot/pxelinux.cfg/default`):

```
DEFAULT openshift
LABEL openshift
    KERNEL http://192.168.1.100/pxe/agent.x86_64-vmlinuz
    APPEND initrd=http://192.168.1.100/pxe/agent.x86_64-initrd.img coreos.live.rootfs_url=http://192.168.1.100/pxe/agent.x86_64-rootfs.img
```

### For iPXE

Use the generated `agent.x86_64.ipxe` script:

```bash
#!ipxe
kernel http://192.168.1.100/pxe/agent.x86_64-vmlinuz
initrd http://192.168.1.100/pxe/agent.x86_64-initrd.img
imgargs agent.x86_64-vmlinuz initrd=agent.x86_64-initrd.img coreos.live.rootfs_url=http://192.168.1.100/pxe/agent.x86_64-rootfs.img
boot
```

---

## Step 9: Boot the Nodes

1. **Configure BIOS/UEFI** on each node:
   - Enable PXE boot
   - Set network boot as first priority
   
2. **Power on nodes in order:**
   - Start all master nodes first
   - Wait for control plane to initialize
   - Start worker nodes

3. **Monitor installation progress:**

```bash
# Watch bootstrap progress
openshift-install agent wait-for bootstrap-complete --dir ~/ocp-pxe-install --log-level=info

# After bootstrap completes
openshift-install agent wait-for install-complete --dir ~/ocp-pxe-install --log-level=info
```

---

## Step 10: Verify Installation

1. **Check cluster status:**

```bash
export KUBECONFIG=~/ocp-pxe-install/auth/kubeconfig
oc get nodes
oc get clusterversion
```

2. **Access console:**
   - URL will be displayed after installation completes
   - Login with kubeadmin credentials from `auth/kubeadmin-password`

---

## Additional Configurations

### Dual-Stack Networking

Add to `install-config.yaml`:

```yaml
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  - cidr: fd02::/48
    hostPrefix: 64
  machineNetwork:
  - cidr: 192.168.1.0/24
  - cidr: 2001:db8::/32
  serviceNetwork:
  - 172.30.0.0/16
  - fd03::/112
```

### Disconnected/Air-gapped Installation

Add to `install-config.yaml`:

```yaml
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  <MIRROR_REGISTRY_CERTIFICATE>
  -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - <MIRROR_REGISTRY>/<REPO_NAME>/release
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - <MIRROR_REGISTRY>/<REPO_NAME>/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

### Static IP Override for Special Network Devices

Add to kernel parameters:

```
ai.ip_cfg_override=1
```

---

## Troubleshooting

### Common Issues

1. **Nodes not booting from PXE:**
   - Verify DHCP configuration
   - Check TFTP service is running
   - Confirm network connectivity

2. **Installation hanging:**
   - Check rendezvous IP is reachable
   - Verify all required ports are open
   - Review agent logs on nodes

3. **Asset download failures:**
   - Confirm HTTP server is accessible
   - Check file permissions
   - Verify URLs in boot parameters

### Debug Commands

```bash
# Check agent status on a node
ssh core@<node-ip>
sudo journalctl -u agent.service -f

# View installation logs
openshift-install agent wait-for install-complete --dir ~/ocp-pxe-install --log-level=debug

# Check cluster operators
oc get co
```

---

## Summary

This guide provides a complete PXE boot installation process for OpenShift 4.17 using the Agent-based Installer:

1. Download tools and prepare environment
2. Create installation and agent configuration files
3. Generate PXE boot assets
4. Configure PXE/HTTP infrastructure
5. Boot nodes and monitor installation
6. Verify successful deployment

The Agent-based Installer with PXE provides a flexible, automated method for deploying OpenShift clusters in on-premise environments.