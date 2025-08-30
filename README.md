Red Hat Partner Convergence 2025: Image Mode Demo
This README file outlines the steps for a demo showcasing Red Hat Enterprise Linux (RHEL) Image mode, a modern approach to managing the operating system lifecycle using container technologies. The demo will illustrate how to build, deploy, and manage RHEL using a container-native workflow, including updates and rollbacks.

Demo Preparation
The demo will use a series of container builds to create a bootable RHEL image, which will then be deployed as a virtual machine (VM). We'll use Podman for building the images and the bootc-image-builder to create a qcow2 disk image for the VM. The demo flow is as follows:

Initial Deployment: Build a RHEL 9.6 image with a web server and MariaDB, then deploy it as a VM.

Image Switch: Build a new RHEL 10 image with an updated web page and use bootc switch to change the running VM's operating system.

Image Update and Rollback: Push a new version of the RHEL 10 image and demonstrate how bootc update and bootc rollback commands work.

Step-by-Step Demo Guide
Phase 1: Initial Deployment with RHEL 9.6
Build the Initial Image: We'll use Containerfile.v1 to build our first bootable container image based on RHEL 9.6. This image will include httpd and mariadb-server and will serve a welcome page.

# Pull RHEL 9.6 bootc base image
FROM registry.redhat.io/rhel9/rhel-bootc:9.6

# Update, install base tools, and create the bootc-user with a hashed password
RUN dnf -y update && \
    dnf -y install tmux mkpasswd && \
    pass=$(mkpasswd --method=SHA-512 --rounds=4096 redhat) && \
    useradd -m -G wheel bootc-user -p $pass && \
    echo "%wheel ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/wheel-sudo && \
    dnf clean all && rm -rf /var/cache /var/log/dnf

# Install and enable web server and database components
RUN dnf -y install httpd mariadb mariadb-server vim && \
    systemctl enable httpd && \
    systemctl enable mariadb && \
    dnf clean all && rm -rf /var/cache /var/log/dnf

# Copy static files after all RUN commands
COPY files/index.html /var/www/html/index.html
COPY files/00-mariadb-tmpfile.conf /usr/lib/tmpfiles.d/

# Set up MOTD files
RUN echo "This is a RHEL VM installed using a bootable container as an rpm-ostree source!" > /etc/motd.d/10-first-setup.motd && \
    echo "This server now supports MariaDB as a database, after last update" > /etc/motd.d/20-upgrade.motd

# Expose the necessary ports
EXPOSE 80 3306

# Default command for bootc
CMD ["/usr/sbin/init"]

Build the Image: Run the following command to build the image and tag it as quay.io/username/imagemode-demo:v1.

podman build -t quay.io/username/imagemode-demo:v1 -f Containerfile.v1 .

Push the Image: Push the newly built image to a container registry like Quay.io.

podman push quay.io/username/imagemode-demo:v1

Create a qcow2 Disk Image: We'll use the bootc-image-builder container to create a bootable qcow2 disk image from the container image. This process is how you convert a container image into an installable operating system.

sudo podman run --rm --privileged \
  --security-opt label=type:unconfined_t \
  --volume .:/output \
  --volume $(pwd)/files/config.toml:/config.toml \
  --volume /var/lib/containers/storage:/var/lib/containers/storage \
  registry.redhat.io/rhel9/bootc-image-builder:9.5 \
  --type qcow2 \
  --local quay.io/username/imagemode-demo:v1

Create and Load a VM: Use the generated qcow2 image to create and boot a new VM. Tools like virt-manager or virt-install can be used for this. You can import the existing disk image to create the VM.

Verify the VM: After the VM boots, verify the initial configuration.

Web Server: Check the web server content by accessing the VM's IP address in a web browser. The page should show the content from index.html.

Services: Verify that httpd and mariadb services are running. You can check their status using systemctl.

bootc Status: Run bootc status to check the current deployment and confirm it's based on quay.io/username/imagemode-demo:v1.

Phase 2: Upgrading to RHEL 10 with bootc switch
Build the New Image: Use Containerfile.v2 to build an image based on RHEL 10. This image has an updated index.html file (index-update.html).

# Pull RHEL 10 bootc base image
FROM registry.redhat.io/rhel10/rhel-bootc:latest
...
# Copy updated index.html
COPY files/index-update.html /var/www/html/index.html
...

Build the Image: Build this new image and tag it as v2.

podman build -t quay.io/username/imagemode-demo:v2 -f Containerfile.v2 .

Push the Image: Push the new image to your registry.

podman push quay.io/username/imagemode-demo:v2

Switch the Running VM: On the running RHEL 9.6 VM, use the bootc switch command to change the system to track the new RHEL 10 image.

sudo bootc switch quay.io/username/imagemode-demo:v2

Reboot and Verify: Reboot the VM to apply the changes. After the reboot, you'll be running the RHEL 10 version.

bootc Status: Check bootc status to see that the active deployment is now the v2 image.

Web Server Content: Access the web page again. The content should not have changed, as bootc switch preserves the contents of the /var directory, where the index.html file is located in this example. This is a key feature of bootc that differentiates its updates from an application-level update. The /etc and /var directories are preserved during the switch.

Phase 3: Web Server Configuration Change and Update/Rollback
Build the Updated Image: Use Containerfile.update. This Containerfile makes a crucial change: it modifies the httpd.conf file to serve content from /usr/share/www/html instead of the default /var/www/html.

# Configure httpd to serve from /usr/share/www/html
RUN sed -i 's|DocumentRoot "/var/www/html"|DocumentRoot "/usr/share/www/html"|' /etc/httpd/conf/httpd.conf && \
sed -i 's|Directory "/var/www/html"|Directory "/usr/share/www/html"|' /etc/httpd/conf/httpd.conf
...
# Copy static files after all RUN commands
COPY files/index-update.html /usr/share/www/html/index.html

Build and Push: Build this new image and push it to the registry as quay.io/username/imagemode-demo:v1. This is a deliberate change to show how bootc update works.

podman build -t quay.io/username/imagemode-demo:v1 -f Containerfile.update .
podman push quay.io/username/imagemode-demo:v1

Demonstrate bootc update Failure: On the running VM, which is currently on v2, try to run bootc update. It will fail because the VM is tracking the v2 image tag, but we just pushed the update to the v1 tag. bootc update only checks for updates on the currently tracked image.

sudo bootc update
# Expected output: Fails or says no updates available.

Rollback to Previous Image: To get to the v1 image, we can use bootc rollback. This command reverts the system to the previous deployment.

sudo bootc rollback

Reboot and Verify Rollback: Reboot the VM.

bootc Status: After reboot, bootc status should show that you're now running the original v1 image.

Perform the Update: Now that the VM is on the v1 deployment, we can perform the update. Use the bootc update command. This will fetch the latest image tagged v1 from the registry, which is the one we just pushed with the web server configuration change.

sudo bootc update

Reboot and Verify Changes: Reboot the VM to apply the update.

Web Server Content: Access the web page. The content should now be updated because the underlying operating system and the web server's document root have been changed as part of the new image. The new version of httpd.conf from the image has been applied, and the new index-update.html file is being served from its new location.
