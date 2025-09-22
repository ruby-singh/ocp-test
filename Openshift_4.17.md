# OpenShift 4.17 Installation - Simple Command Guide

## STEP 1: Prepare Your Bastion Computer
**What this does:** Gets your management computer ready with all the tools needed

Run these commands on your bastion server:
```bash
# Update your system
sudo dnf update -y

# Install required tools
sudo dnf install -y git wget curl vim bind-utils telnet httpd-tools jq podman buildah skopeo firewalld dnsmasq httpd openssl

# Start firewall
sudo systemctl enable --now firewalld
```

## STEP 2: Download OpenShift Files (Internet Required)
**What this does:** Downloads OpenShift software from Red Hat (do this on a computer with internet)

Run these commands on a connected computer:
```bash
# Create download folder
mkdir -p ~/openshift-downloads
cd ~/openshift-downloads

# Download OpenShift tools
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.17.0/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.17.0/openshift-install-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.17.0/oc-mirror.tar.gz

# Download operating system
<table><tr><td bgcolor="yellow"><b>
Use OS not RHEL – Once we boot the live ISO it needs IP configuration.
</b></td></tr></table>

wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.17/4.17.0/rhcos-4.17.0-x86_64-live.x86_64.iso

# Package everything
tar -czf openshift-tools.tar.gz *.tar.gz *.iso

# Transfer openshift-tools.tar.gz to your bastion computer
```

## STEP 3: Install OpenShift Tools on Bastion
...

## STEP 4: Create Your Local App Store (Container Registry)

<table><tr><td bgcolor="yellow"><b>
STEP 4: Create Your Local App Store (Container Registry) – Ask the team for the root certificate for the container registry and the exact path where they clone this 4.17 release image.
</b></td></tr></table>

**What this does:** Creates a local place to store OpenShift software since your cluster can't reach the internet

Run these commands on bastion:
```bash
...
```

## STEP 5: Create Phone Book (DNS Setup) - This is for DNS Team

<table><tr><td bgcolor="yellow"><b>
⚠️ Note: Get Load Balancer setup for the nodes with the Team. Once the load balancer IP is available, check with the DNS Team to map it with the addresses.  
Ref: [Using integrated load balancing with on-premises OpenShift 4 IPI](https://www.redhat.com/en/blog/using-integrated-load-balancing-with-on-premises-openshift-4-ipi)
</b></td></tr></table>

**What this does:** Sets up name resolution so computers can find each other by name

Run these commands on bastion (replace IP addresses with your actual ones):

```bash
# Configure DNS server
sudo tee /etc/dnsmasq.conf << 'EOF'
interface=*
bind-interfaces
domain=example.com
expand-hosts
no-resolv
no-poll
# Your server addresses (CHANGE THESE TO MATCH YOUR NETWORK)

<table><tr><td bgcolor="yellow">
address=registry.example.com/192.168.1.100  
address=api.eiab.us.eclub.com/192.168.1.101  
address=api-int.eiab.us.eclub.com/192.168.1.101  
address=*.apps.eiab.us.eclub.com/192.168.1.102  
address=master1.us.eclub.com/192.168.1.102  
address=master2.us.eclub.com/192.168.1.102  
address=master3.us.eclub.com/192.168.1.102  
address=worker1.us.eclub.com/192.168.1.102  
address=worker2.us.eclub.com/192.168.1.102  
address=worker3.us.eclub.com/192.168.1.102  
address=worker4.us.eclub.com/192.168.1.102  
address=worker5.us.eclub.com/192.168.1.102  
address=infra1.us.eclub.com/192.168.1.102  
</td></tr></table>

cache-size=1000
log-queries
log-facility=/var/log/dnsmasq.log
EOF
```

## STEP 6: Fill Your App Store (Copy OpenShift Software)

<table><tr><td bgcolor="yellow"><b>
STEP 6: Fill Your App Store (Copy OpenShift Software) – Ignore if registry setup already done and we have clone for ocp 4.17
</b></td></tr></table>

**What this does:** Downloads all OpenShift software and puts it in your local registry
...

## STEP 8: Write Your Cluster Plan

<table><tr><td bgcolor="yellow"><b>
STEP 8: Write Your Cluster Plan – Check with Team for registry URL and the path under imageContentSources
</b></td></tr></table>

...

## STEP 15: Set Up Internal Storage -- Not Needed 
...

---

**Important files to keep safe:**
- `~/ocp-install/auth/kubeconfig` - Access to your cluster  
- `~/ocp-install/auth/kubeadmin-password` - Admin password  
- `~/.ssh/openshift-key` - SSH access to servers  
- `/opt/registry/certs/registry.crt` - Registry certificate  
