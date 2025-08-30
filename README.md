# Red Hat Partner Convergence 2025: Image Mode Demo

This repository demonstrates **Red Hat Enterprise Linux (RHEL) Image Mode**, a modern way to manage the OS lifecycle using container technologies.  

The demo covers how to **build, deploy, update, and roll back** RHEL using a container-native workflow.

---

## 📋 Demo Flow

1. **Initial Deployment**: Build a RHEL 9.6 image with a web server + MariaDB, then deploy it as a VM.  
2. **Image Switch**: Switch the running VM from RHEL 9.6 → RHEL 10 using `bootc switch`.  
3. **Update & Rollback**: Demonstrate `bootc update` and `bootc rollback` with a web server config change.

---

## ⚙️ Demo Preparation

We’ll use:

- **Podman** → build container images  
- **bootc-image-builder** → convert container images into bootable `qcow2` disk images  
- **virt-install / virt-manager** → run the VM  

---

## 🚀 Step-by-Step Guide

### Phase 1: Initial Deployment (RHEL 9.6)

#### 1. Create the Image

`Containerfile.v1`:

```dockerfile
# Pull RHEL 9.6 bootc base image
FROM registry.redhat.io/rhel9/rhel-bootc:9.6

# Update, install base tools, and create a user
RUN dnf -y update &&     dnf -y install tmux mkpasswd &&     pass=$(mkpasswd --method=SHA-512 --rounds=4096 redhat) &&     useradd -m -G wheel bootc-user -p $pass &&     echo "%wheel ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/wheel-sudo &&     dnf clean all && rm -rf /var/cache /var/log/dnf

# Install web server and database
RUN dnf -y install httpd mariadb mariadb-server vim &&     systemctl enable httpd &&     systemctl enable mariadb &&     dnf clean all && rm -rf /var/cache /var/log/dnf

# Copy static files
COPY files/index.html /var/www/html/index.html
COPY files/00-mariadb-tmpfile.conf /usr/lib/tmpfiles.d/

# MOTD messages
RUN echo "This is a RHEL VM installed using a bootable container as an rpm-ostree source!" > /etc/motd.d/10-first-setup.motd &&     echo "This server now supports MariaDB as a database, after last update" > /etc/motd.d/20-upgrade.motd

# Expose ports
EXPOSE 80 3306

# Default command
CMD ["/usr/sbin/init"]
```

Build and push:

```bash
podman build -t quay.io/<username>/imagemode-demo:v1 -f Containerfile.v1 .
podman push quay.io/<username>/imagemode-demo:v1
```

#### 2. Create a Bootable Disk

```bash
sudo podman run --rm --privileged   --security-opt label=type:unconfined_t   --volume .:/output   --volume $(pwd)/files/config.toml:/config.toml   --volume /var/lib/containers/storage:/var/lib/containers/storage   registry.redhat.io/rhel9/bootc-image-builder:9.5   --type qcow2   --local quay.io/<username>/imagemode-demo:v1
```

#### 3. Launch the VM

- Import the generated `.qcow2` with **virt-manager** or **virt-install**.  
- Verify services:

```bash
systemctl status httpd
systemctl status mariadb
bootc status
```

- Open a browser → visit VM IP → confirm **index.html** is served.

---

### Phase 2: Upgrade to RHEL 10 (bootc switch)

#### 1. Create the RHEL 10 Image

`Containerfile.v2`:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest
...
COPY files/index-update.html /var/www/html/index.html
```

Build and push:

```bash
podman build -t quay.io/<username>/imagemode-demo:v2 -f Containerfile.v2 .
podman push quay.io/<username>/imagemode-demo:v2
```

#### 2. Switch the VM

```bash
sudo bootc switch quay.io/<username>/imagemode-demo:v2
sudo reboot
```

Verify:

```bash
bootc status
```

> ✅ System should now run **RHEL 10** with original `/var/www/html` content preserved.

---

### Phase 3: Update & Rollback

In this phase, we demonstrate how **bootc update** and **bootc rollback** work.  
Unlike the earlier Containerfiles, `Containerfile.update` **starts from the v2 image** you already pushed to Quay.io and only applies the necessary changes.  

#### 1. Build the Updated Image

`Containerfile.update`:

```dockerfile
# Start from the previously built RHEL 10 v2 image
FROM quay.io/<username>/imagemode-demo:v2

# Ensure the new directory exists
RUN mkdir -p /usr/share/www/html

# Reconfigure httpd to serve from /usr/share/www/html
RUN sed -i 's|DocumentRoot "/var/www/html"|DocumentRoot "/usr/share/www/html"|' /etc/httpd/conf/httpd.conf && \
    sed -i 's|Directory "/var/www/html"|Directory "/usr/share/www/html"|' /etc/httpd/conf/httpd.conf

# Copy updated static files
COPY files/index-update.html /usr/share/www/html/index.html

# (Optional) MOTD message about the change
RUN echo "Web server document root has been moved to /usr/share/www/html in this update." > /etc/motd.d/30-webroot-change.motd

# Expose required port
EXPOSE 80 3306

# Default command
CMD ["/usr/sbin/init"]
```

Build & push (as `v1` deliberately):

```bash
podman build -t quay.io/<username>/imagemode-demo:v1 -f Containerfile.update .
podman push quay.io/<username>/imagemode-demo:v1
```

#### 2. Demonstrate Update Behavior

```bash
# VM is tracking v2, so this won’t find updates
sudo bootc update
```

Rollback:

```bash
sudo bootc rollback
sudo reboot
```

Now track `v1` → apply the update:

```bash
sudo bootc update
sudo reboot
```

#### 3. Verify

- `bootc status` → should show updated `v1`.  
- Browser → web content now served from `/usr/share/www/html`.  

---



---

## 📂 Repository Layout

```
.
├── Containerfile.v1        # Initial RHEL 9.6 image with httpd + MariaDB
├── Containerfile.v2        # RHEL 10 upgrade image with updated index.html
├── Containerfile.update    # Modified image (httpd config + new content)
├── files/
│   ├── index.html          # Default welcome page for v1
│   ├── index-update.html   # Updated web page for v2 and update image
│   ├── 00-mariadb-tmpfile.conf  # MariaDB tmpfile configuration
│   └── config.toml         # Bootc image builder configuration
└── README.md               # Documentation (this file)
```

This structure helps you identify which containerfile and assets belong to each phase of the demo.

## ✅ Key Takeaways

- RHEL Image Mode enables **container-native OS lifecycle management**.  
- `bootc` commands (`status`, `switch`, `update`, `rollback`) simplify upgrades.  
- `/var` and `/etc` are preserved across image switches, making upgrades safer.  

---
