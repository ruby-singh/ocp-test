# OpenShift 4.17 Installation using PXE Boot Method

This guide provides **step-by-step instructions** to install OpenShift 4.17 via **PXE (network boot)** using pre-generated Ignition files and mirrored images on **RHEL 9**.

---

## 🧩 Prerequisites

Before starting, ensure:

- You have already generated the Ignition files (`*.ign`) — `bootstrap.ign`, `master*.ign`, `worker*.ign`.
- You have mirrored the OpenShift release images to your local registry.
- You have a **PXE server** with:
  - DHCP service configured to hand out IPs and boot instructions
  - TFTP service serving PXE boot files
  - HTTP server serving ignition and boot images
- DNS and Load Balancer entries for API and Ingress are already configured.
- All nodes (bootstrap, masters, workers) are set to boot from the network (PXE).

---

## ⚙️ Step 1 — Install PXE and Web Server Packages

```bash
sudo dnf install -y httpd tftp-server syslinux
sudo systemctl enable --now httpd
sudo systemctl enable --now tftp.socket
```

---

## 📁 Step 2 — Prepare HTTP Server Directory

```bash
sudo mkdir -p /var/www/html/boot
sudo chown -R root:root /var/www/html/boot
sudo chmod -R 755 /var/www/html/boot
```

Set SELinux context for HTTP files:

```bash
sudo dnf install -y policycoreutils-python-utils
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/html/boot(/.*)?"
sudo restorecon -R /var/www/html/boot
```

---

## 📦 Step 3 — Copy Boot Artifacts and Ignition Files

```bash
cd /home/user/ocp-bootstrap/
sudo cp rhcos-live-initramfs.x86_64.img rhcos-live-kernel-x86_64 rhcos-live-rootfs.x86_64.img /var/www/html/boot/
sudo cp bootstrap.ign master*.ign worker*.ign /var/www/html/boot/
```

---

## 🧠 Step 4 — Configure PXE Bootloader

```bash
sudo mkdir -p /var/lib/tftpboot/pxelinux.cfg
sudo cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
sudo cp /usr/share/syslinux/menu.c32 /var/lib/tftpboot/
sudo cp /usr/share/syslinux/memdisk /var/lib/tftpboot/
```

Create PXE boot menu configuration file:

```bash
sudo vi /var/lib/tftpboot/pxelinux.cfg/default
```

Example configuration:

```text
DEFAULT rhcos
PROMPT 0
TIMEOUT 30

LABEL rhcos
    KERNEL boot/rhcos-live-kernel-x86_64
    APPEND initrd=boot/rhcos-live-initramfs.x86_64.img rootfs=boot/rhcos-live-rootfs.x86_64.img       ip=dhcp rd.neednet=1 ignition.platform.id=metal       coreos.inst.ignition_url=http://<pxe-server-ip>/boot/bootstrap.ign
```

---

## 🌐 Step 5 — Configure DHCP Server

```dhcp
subnet 192.168.0.0 netmask 255.255.255.0 {
    option routers 192.168.0.1;
    option domain-name-servers 192.168.0.10;
    range 192.168.0.100 192.168.0.200;

    host bootstrap {
        hardware ethernet aa:bb:cc:dd:ee:01;
        fixed-address 192.168.0.100;
        filename "pxelinux.0";
        next-server <pxe-server-ip>;
    }

    host master0 {
        hardware ethernet aa:bb:cc:dd:ee:02;
        fixed-address 192.168.0.101;
        filename "pxelinux.0";
        next-server <pxe-server-ip>;
    }

    host worker0 {
        hardware ethernet aa:bb:cc:dd:ee:03;
        fixed-address 192.168.0.102;
        filename "pxelinux.0";
        next-server <pxe-server-ip>;
    }
}
```

Restart DHCP service:

```bash
sudo systemctl restart dhcpd
```

---

## 🔥 Step 6 — Open Firewall Ports

```bash
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=tftp --permanent
sudo firewall-cmd --reload
```

If SELinux blocks access:

```bash
sudo setsebool -P tftp_anon_write 1
sudo setsebool -P tftp_home_dir 1
```

---

## 🖥️ Step 7 — Boot the Nodes via PXE

1. Power on each node (bootstrap, masters, workers).  
2. Ensure PXE (Network Boot) is the first boot option in BIOS.  
3. The node will:
   - Receive DHCP lease
   - Fetch PXE config (`pxelinux.0`)
   - Download kernel, initramfs, rootfs
   - Fetch and apply its Ignition file

---

## 🚀 Step 8 — Monitor Bootstrap Progress

```bash
./openshift-install --dir <install_dir> wait-for bootstrap-complete --log-level=info
```

After bootstrap is finished:

```bash
./openshift-install --dir <install_dir> wait-for install-complete --log-level=info
```

---

## ✅ Step 9 — Validate Cluster Installation

```bash
export KUBECONFIG=<install_dir>/auth/kubeconfig
oc get nodes
oc get clusterversion
```

Expected output:

```
NAME         STATUS   ROLES           AGE   VERSION
master-0     Ready    master,worker   30m   v4.17.x
master-1     Ready    master,worker   29m   v4.17.x
master-2     Ready    master,worker   29m   v4.17.x
```

---

## 🧾 Notes

- Update `coreos.inst.ignition_url` for each node type.
- All `.ign` and RHCOS files **must be accessible** over HTTP.
- Ensure pull secret and mirror registry configs are embedded in `.ign` files.

---

## 📚 Reference Documentation

- [Preparing PXE assets for OpenShift 4.17](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/installing_an_on-premise_cluster_with_the_agent-based_installer/prepare-pxe-assets-agent)
- [Disconnected Installation Overview](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/installing/disconnected-installation-overview)

---

> 🏁 You have now installed OpenShift 4.17 using PXE Boot on RHEL 9 with pre-generated Ignition and mirrored images.
