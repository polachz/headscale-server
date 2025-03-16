# Headscale Server with Headplane Web UI and Traefik Reverse Proxy Implemented Using Podman Rootless Quadlet

Warning: This procedure is intended for operating a private Tailscale server and is shared with the community without any guarantees or liability. Use this guide at your own risk!

This guide assumes the use of the AlmaLinux 9 operating system. If you are using a different distribution, some adjustments may be necessary.

### Install Necessary Packages and Enable Services
```bash
sudo dnf install containers-common podman podman-plugins firewalld
sudo systemctl enable firewalld
sudo systemctl start firewalld
```

### Enable Podman Quadlet Logs Persistent Storage

To enable persistent storage of logs for rootless Podman containers in journalctl, edit the file:

```bash
sudo nano /etc/systemd/journald.conf
```
Ensure the following line is set:
```
Storage=persistent
```
### Allow System Services Port Binding

### Allow System Services Ports Binding

We need to make the server accessible on http/https ports, aka 80/443. We assume that no other
service runs on the server and then is safe to allow biding for non-root processes to priviledge ports.
If not, consider different way how to grant access to these ports for user (non root) processes!!


Create the following file:

```bash
sudo nano /etc/sysctl.d/sys-ports.conf
```
Add the content:
```
net.ipv4.ip_unprivileged_port_start = 80
```
### Allow Necessary Ports on the Firewall

Run the following commands:

```bash
sudo firewall-cmd --permanent --zone=public --add-service=ssh
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --permanent --zone=public --add-port=3478/udp
sudo firewall-cmd --reload
```

Verify the firewall rules are correctly applied:

```bash
firewall-cmd --list-all
```

### Create a User Account for the Headscale Server

```bash
sudo useradd -m headscale
```
### Enable Automatic Start of User Services After Boot

```bash
sudo loginctl enable-linger headscale
```
### Log in as the Server User

**!!!! Important: Always use the machinectl command to switch to the headscale user. Using other methods may cause certain features to malfunction.**

```bash
sudo machinectl shell --uid=headscale
```

### Enable the Podman Socket

```bash
systemctl --user enable podman.socket
systemctl --user start podman.socket
```

### Create and Configure Container Volume Files

Create the volumes folder:

```bash
mkdir ~/volumes
```

Copy the content of the volume_files folder from the Git repository into this directory. Review and configure these files to meet your requirements. The repository contains examples from headscale/headplane.

You should have a structure similar to this:
```
/home/headscale/volumes/traefik/config/traefik.yaml
/home/headscale/volumes/headscale/config/config.yaml
/home/headscale/volumes/headplane/config.yaml
```
Protect the files by setting appropriate permissions to prevent access by other users:

```bash
chmod 640 [file]
```

### Create Quadlet Files

Create the necessary folder:

```bash
mkdir -p ~/.config/containers/systemd/
```

Copy the content of the quadlet folder into this directory. Ensure that all .container and .pod files are located directly in this folder.

You should have to get this:

```
/home/headscale/.config/containers/systemd/headscale.container
/home/headscale/.config/containers/systemd/headplane.container
...
```

### Set correct permissions to protect these files:

```bash
chmod 640 [file]
```

### Transform Quadlet Files into Systemd User Services

```bash
systemctl --user daemon-reload
```

### Optional: Create an Alias for Running Headscale Commands

Add the following alias to the headscale user’s .bashrc file:

```bash
alias hs='podman exec headscale headscale'
```

Once the Headscale server container is running, you can execute Headscale CLI commands like this:

```bash
hs machines list
```


## Run the Headscale Server


```bash
systemctl --user start headscale-pod.service
```

### Verify Services Are Running

Check the status of individual services:

```bash
systemctl --user status traefik.service
systemctl --user status headscale.service
systemctl --user status headplane.service
```

### To view logs for Headscale or Headplane, use:

```bash
journalctl --user -xeu headscale.service
journalctl --user -xeu headplane.service
```

Traefik logs can be found here:

```
/home/headscale/volumes/traefik/logs/traefik.log
/home/headscale/volumes/traefik/logs/access.log
```

## Important: Generate an API Key for Headplane

Once the Headscale server is running, you need to create an API key for Headplane:

```bash
hs apikeys create --expiration 999d
```
This key can also be used to log in to the Headplane Web UI:

https://headplane.example.com/admin
