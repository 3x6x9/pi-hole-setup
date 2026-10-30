# How to set up Pi-hole on a Raspberry Pi

[Pi-hole](https://github.com/pi-hole/pi-hole)

This is a manual for a fresh installation of Debian 13 on a Raspberry Pi 4 Model B (4 GB).

## Prerequisites

- Raspberry Pi and power supply
- At least a 16 GB microSD card
- Ethernet cable
- [Pi-hole prerequisites](https://docs.pi-hole.net/main/prerequisites/)

## Install the OS

1. Connect the microSD card to a computer.
2. Download and open the [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
3. Go through the setup:
   1. Choose **Raspberry Pi OS Lite (64-bit)**.
      - A desktop environment is not needed.
      - The current OS is Debian Trixie (Debian 13).
   2. Choose a hostname. Check it later with `hostnamectl`.
   3. Choose a city for the time zone. Check it later with `timedatectl status`.
   4. Choose a username and password (for example, `user1`).
   5. Skip Wi-Fi setup.
   6. Enable remote access with SSH over port 22.
   7. Raspberry Pi Connect is not needed.

## Connect Pi to router

1. Connect the Pi to the router with an Ethernet cable. A wired connection is preferred over Wi-Fi.
2. Log in to the router's settings page.
3. Reserve a static IPv4 address for the Pi (for example, `192.168.2.100`).

## Pi Setup

<details open>
<summary>Common and useful Linux commands</summary>

```sh
# CAUTION: Execute commands with root privileges using sudo.
sudo

# Print the current working directory
pwd

# Change the current directory
cd /home
cd ..

# List files and directories in the current directory
ls -lisa

# Create a new directory
mkdir folder

# Create a new file
touch file.md

# Remove files and directories
rm file.md

# Display file contents
cat file.md

# Copy files and directories
cp file.md file1.md

# Restart or shut down
sudo reboot
sudo shutdown -h now

# Raspberry Pi configuration text UI
sudo raspi-config

# Show system information
htop

ss -tlnp
netstat -tlnp
lsblk

# Update and upgrade apt packages
sudo apt update && sudo apt upgrade -y

# Install an apt package
sudo apt install <package>

# Remove an apt package
sudo apt remove <package>
sudo apt purge <package>

sudo apt purge '~c'

# Remove unused packages
sudo apt autoremove --purge -y
sudo apt autoclean

# Cancel
# Press Ctrl+C

# Quit a text UI (not nano or vim)
# Press Esc

# Get the current CPU temperature in Celsius
vcgencmd measure_temp

# Check the current time and date
date

# Useful Pi-hole commands
pihole -up
pihole -r
pihole -d
pihole -g
```
</details>

### Connect via SSH

Open a terminal on any device (for example, Termux on Android).

```sh
ssh 192.168.X.X -l <username>
# Example:
ssh 192.168.2.100 -l user1
```

### Install updates

On first startup, run:

1. `sudo apt update && sudo apt upgrade -y`
2. `sudo reboot`

### Harden SSH Config

1. Open the SSH server configuration:
   ```sh
   sudo nano /etc/ssh/sshd_config
   ```
2. Keep the defaults, but set:
   ```text
   PermitRootLogin no
   X11Forwarding no
   ```
   Lines beginning with `#` are comments and are ignored.
3. Save with `Ctrl+X`, then `Y`, then `Enter`.
4. Restart SSH (or reboot):
   ```sh
   sudo systemctl restart ssh
   ```

### Install ufw (Uncomplicated firewall)

1. Install UFW:
   ```sh
   sudo apt install ufw -y
   ```
2. Configure UFW:
   ```sh
   sudo ufw default deny incoming
   sudo ufw default allow outgoing

   # Allow SSH
   sudo ufw allow 22/tcp

   # Pi-hole
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw allow 53/tcp
   sudo ufw allow 53/udp
   sudo ufw allow 67/tcp
   sudo ufw allow 67/udp
   sudo ufw allow 123/udp
   sudo ufw allow 546:547/udp
   ```
3. Enable UFW and check the configuration:
   ```sh
   sudo ufw enable # Confirm when prompted
   sudo ufw status verbose
   ```

### Install unattended-upgrades

1. Install unattended-upgrades:
   ```sh
   sudo apt install unattended-upgrades -y
   ```
2. Configure it:
   1. Run `sudo dpkg-reconfigure --priority=low unattended-upgrades` and choose **Yes**.
   2. Open `/etc/apt/apt.conf.d/50unattended-upgrades`:
      ```sh
      sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
      ```
   3. Keep the defaults, but ensure these settings are present:
      ```text
      Unattended-Upgrade::AutoFixInterruptedDpkg "true";
      Unattended-Upgrade::Remove-Unused-Dependencies "true";
      ```
   4. Save with `Ctrl+X`, then `Y`, then `Enter`.
3. Check that it is running:
   ```sh
   systemctl status unattended-upgrades
   ```

### Install fail2ban

1. Install fail2ban:
   ```sh
   sudo apt install fail2ban -y
   ```
2. Create and edit the local configuration:
   ```sh
   sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
   sudo nano /etc/fail2ban/jail.local
   ```
3. Keep the defaults, but set `bantime = 1h` in `[DEFAULT]` and set the following in `[sshd]`:
   ```ini
   [sshd]
   enabled = true
   filter = sshd
   ```
   Save with `Ctrl+X`, then `Y`, then `Enter`.
4. Enable fail2ban and check its status:
   ```sh
   sudo systemctl enable --now fail2ban
   sudo fail2ban-client status
   sudo fail2ban-client status sshd
   ```

### Install sysstat

1. Install sysstat:
   ```sh
   sudo apt install sysstat -y
   ```
2. Enable sysstat:
   ```sh
   sudo systemctl enable --now sysstat
   ```
3. See the [sysstat repository](https://github.com/sysstat/sysstat).

### Install rsyslog and configure persistent logs

1. Install and enable rsyslog:
   ```sh
   sudo apt install rsyslog -y
   sudo systemctl enable --now rsyslog
   ```
2. Create the journald configuration directories and persistent journal:
   ```sh
   sudo mkdir -p /etc/systemd/journald.conf.d
   sudo mkdir -p /var/log/journal
   sudo systemd-tmpfiles --create --prefix=/var/log/journal
   ```
3. Configure journald limits:
   ```sh
   sudo tee /etc/systemd/journald.conf.d/00-journal-size.conf <<'EOF'
   [Journal]
   Storage=persistent
   SystemMaxUse=200M
   SystemMaxFiles=5
   RuntimeMaxUse=100M
   MaxRetentionSec=30day
   EOF
   ```
4. Restart journald and flush the journal:
   ```sh
   sudo systemctl restart systemd-journald
   sudo journalctl --flush
   ```
5. Check service status:
   ```sh
   systemctl status rsyslog
   systemctl status systemd-journald
   systemctl status logrotate.timer
   ```
6. Check logs:
   ```sh
   journalctl -xe
   journalctl -u ssh
   journalctl -u fail2ban
   ```
7. Check disk usage periodically:
   ```sh
   df -h /var # Disk space
   ncdu / # Find large directories
   ls -lh /var/log/*.log
   ```
8. Verify log rotation and journal usage:
   ```sh
   cat /etc/logrotate.d/rsyslog
   journalctl --disk-usage
   ls -ld /var/log/journal
   ```
9. If the journal is too large, trim it manually:
   ```sh
   journalctl --vacuum-size=150M
   ```
10. View recent logs:
    ```sh
    journalctl -u ssh -n 20
    tail -n 20 /var/log/syslog
    tail -n 20 /var/log/auth.log
    ```

### Disable unused services

Review enabled services and disable any that are not needed:

```sh
sudo systemctl list-unit-files --type=service --state=enabled
sudo systemctl disable --now bluetooth
```

## Install Pi-hole

### Installation

Follow the [Pi-hole basic installation guide](https://docs.pi-hole.net/main/basic-install/). To start the installer:

```sh
curl -sSL https://install.pi-hole.net | bash
```

### Post-installation

See the [Pi-hole post-installation guide](https://docs.pi-hole.net/main/post-install/).

#### Set up the Pi's network settings

On newer Raspberry Pi OS releases (Debian 12 and later), networking may be handled by NetworkManager rather than `dhcpcd`. Check the services:

```sh
systemctl status dhcpcd # Expected to be not found
systemctl status NetworkManager # Expected to be running
```

#### Set up DNS to use localhost

1. Check the current connection and DNS settings:
   ```sh
   nmcli connection show # For example: Wired connection 1
   nmcli device show eth0 | grep -E 'IP4|IP6|DNS'
   ```
   Note the current IPv6 DNS address, such as `fd5h:p09f:1778:1::1`.
2. Open the NetworkManager text UI:
   ```sh
   sudo nmtui
   ```
3. Select **Edit a connection**, then select the active connection (for example, **Wired connection 1**).
4. Leave the IPv4 and IPv6 configurations set to **Automatic**.
5. In **Show IPv4 Config**, keep the defaults and set:
   - **DNS servers** to `127.0.0.1`.
   - **Ignore automatically obtained DNS parameters**.
6. In **Show IPv6 Config**, choose a ULA address (`fdxx::/8`) using the current prefix and a custom suffix. For example, use `fd5h:p09f:1778:1::53` based on the prefix from `IP6.DNS[1]`.
7. Keep the other defaults and set:
   - **Addresses** to `fd5h:p09f:1778:1::53/64`.
8. Reboot to apply the changes:
   ```sh
   sudo reboot
   ```
9. Verify the network and DNS settings:
   ```sh
   ip addr show eth0
   ip route show
   ip -6 route show
   cat /etc/resolv.conf
   nmcli device show eth0 | grep -E 'IP4|IP6|DNS'
   ```
   Confirm that `IP4.DNS[1]` is `127.0.0.1`, `IP6.ADDRESS[1]` is the new `fd5h:p09f:1778:1::53/64`, and `IP6.DNS[1]` is `fd5h:p09f:1778:1::53`.

Pi-hole will listen on the local IPv6 addresses. Because the configured IPv6 address is static, it can be used in the router settings.

In the router settings, set custom DNS servers:

1. IPv4 primary/secondary: `192.168.2.100`
2. IPv6 primary/secondary: `fd5h:p09f:1778:1::53`

Reboot the router. Confirm that devices on the same Wi-Fi network receive `192.168.2.100` and `fd5h:p09f:1778:1::53` as their DNS servers automatically.

### Updating

Run `pihole -up`. See the [Pi-hole update guide](https://docs.pi-hole.net/main/update/).

### Uninstall

1. Set the router's DNS settings back to automatic/default.
2. If DHCP was disabled on the router, set it back to automatic as well.
3. Run `pihole uninstall`.
4. See the [Pi-hole uninstall guide](https://docs.pi-hole.net/main/uninstall/).

### Telekom Smart 3 router

1. Set the IPv4 address.
2. Set the gateway to the router's local address.
3. Disable IPv4 DHCP on the router.
4. Enable DHCP in the Pi-hole admin UI.
5. Set `::1` for IPv6.

## Configure Pi-hole

1. In **Settings → DNS**, choose upstream DNS servers (for example, Cloudflare DNSSEC) and check the four IPv4 and IPv6 boxes.
2. Add blocklists:
   1. Open [my blocklist Gist](https://gist.github.com/3x6x9/b7b11eaeb67b0ab572ee5fac783f0a79).
   2. Copy all or some of the entries under **blocklists**.
   3. Paste the entries into **Add a new subscribed list**, then select **Add blocklist**.
   4. Go to **Tools → Update Gravity**.

## Install Unbound

Follow the [Unbound guide for Pi-hole](https://docs.pi-hole.net/guides/dns/unbound/).
