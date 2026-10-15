# How to set up the Raspberry Pi to use Pi-hole

This is a manual for a freshly new installed Debian 13 on a Raspberry Pi 4 Model B 4GB.

## Prerequisites

- Raspberry Pi + power supply
- at least 16 GB microSD card
- Ethernet cable
- https://docs.pi-hole.net/main/prerequisites/

## Install the OS

1. Connect microSD card to a computer
2. Download the Raspberry Pi Imager
    - https://www.raspberrypi.com/software/
3. Go through the setup
    - Choose Raspberry Pi OS Lite (64-bit)
        - no desktop environment is needed
        - current OS: Debian Trixie (Debian 13)
    - Choose hostname
        - Check later `hostnamectl`
    - Choose City for timezone settings
        - Check later `timedatectl status`
    - Choose a username and password (e.g. user1)
    - Skip Wi-Fi Setup
    - Enable Remote Access
        - SSH connection over port 22
    - New Raspberry Pi Connect not needed

## Connect Pi to router

1. Connect the Pi to the Router via Ethernet cable
    - wired connection to the router is preferred over Wi-Fi
2. Login to the routers settings page
3. Reserve a static IPv4 for the Pi (e.g. 192.168.2.100)

## Connect to Pi via SSH

1. Common and useful linux commands
   ```
   # CAUTION - Execute a command with root privilages
   sudo

   # Print  current working directory
   pwd

   # Change the current directory
   cd /home
   cd ..

   # List files and directories in the current directory
   ls -lisa

   # Create new directory
   mkdir folder

   # Create new file
   touch file.md

   # Remove files and directories
   rm file.md

   # Concatenates and displays file content
   cat file.md

   # Copy file and directories
   cp file.md file1.md

   # Restart/Shutdown
   sudo reboot
   sudo shutdown -h now

   # Raspi Config text UI
   sudo raspi-config

   # Show System Information
   htop

   ss -tlnp
   netstat -tlnp
   lsblk

   # Update & Upgrade apt repository
   sudo apt update && sudo apt upgrade -y

   # Install new apt package
   sudo apt install <package>

   # Remove apt package
   sudo apt remove <package>
   sudo apt purge <package>

   sudo apt purge '~c'

   # Remove unused packages
   sudo apt autoremove --purge -y
   sudo apt autoclean

   # Cancel
   Ctrl + C

   # text ui quit (not nano or vim)
   Esc
   ```

2. Open Terminal on any device (possible on Android via e.g. Termux, or on iOS via Shortcuts)
    - `ssh 192.168.X.X -l <username>` e.g. ssh 192.168.2.100 -l user1

3. On first startup run
    - `sudo apt update && sudo apt upgrade -y`
    - `sudo reboot`

4. Harden SSH Config
    - `sudo nano /etc/ssh/sshd_config`
        - use defaults, but (`#`or `//` usually means comment out = getting ignored)
          ```
          PermitRootLogin no
          X11Forwarding no
          ```
        - Press to safe changes: `Ctrl+X -> Y -> Enter`
    - `sudo systemctl restart ssh # or sudo reboot`

5. Install firewall (Uncomplicated firewall)
    - `sudo apt install ufw -y`
    - Config UFW
      ```
      sudo ufw default deny incoming
      sudo ufw default allow outgoing

      # Allow SSH
      sudo ufw allow 22/tcp

      # Pi-hole needs
      sudo ufw allow 80/tcp
      sudo ufw allow 443/tcp
      sudo ufw allow 53/tcp
      sudo ufw allow 53/udp
      sudo ufw allow 67/tcp
      sudo ufw allow 67/udp
      sudo ufw allow 123/udp
      sudo ufw allow 546:547/udp
      ```
    - Enable and check config
      ```
      sudo ufw enable # Press Yes
      sudo ufw status verbose
      ```
6. Install unattended-upgrades
    - `sudo apt install unattended-upgrades -y`
    - Config unattended-upgrades
        - `sudo dpkg-reconfigure --priority=low unattended-upgrades # text UI`
            - Yes
        - `sudo nano /etc/apt/apt.conf.d/50unattended-upgrades`
            - use defaults, but
              ```
              Unattended-Upgrade::Origins-Pattern {
                "origin=Debian,codename=${distro_codename},label=Debian";
                "origin=Debian,codename=${distro_codename},label=Debian-Security";
                "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
              };
              
              Unattended-Upgrade::AutoFixInterruptedDpkg "true";
              Unattended-Upgrade::Remove-Unused-Dependencies "true";
              ```
            - Press to safe changes: `Ctrl+X -> Y -> Enter`
    - Check if it's running
        - `systemctl status unattended-upgrades`


7. Install fail2ban
    - `sudo apt install fail2ban -y`
    - Config fail2ban
        - `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`
        - `sudo nano /etc/fail2ban/jail.local`
            - in `[DEFAULT]`, use defaults, but
                - `bantime = 1h`
            - in `[sshd]`, use defaults, but
              ```
              enabled = true
              filter = sshd
              ```
            - Press to safe changes: `Ctrl+X -> Y -> Enter`
    - Enable and check config
      ```
      sudo systemctl enable --now fail2ban
      sudo fail2ban-client status
      sudo fail2ban-client status sshd
      ```

8. Install sysstat
    - `sudo apt install sysstat -y`
    - Enable sysstat
        - `sudo systemctl enable --now sysstat`
    - Checkout the repo: https://github.com/sysstat/sysstat

9. Install rsyslog
    - `sudo apt install rsyslog -y`
    - Enable rsyslog
        - `sudo systemctl enable --now rsyslog`
    - Config Logs
      ```
      sudo mkdir -p /etc/systemd/journald.conf.d
      sudo mkdir -p /var/log/journal

      sudo systemd-tmpfiles --create --prefix=/var/log/journal
      ```
        - in one go:
          ```
          sudo tee /etc/systemd/journald.conf.d/00-journal-size.conf << 'EOF'
          [Journal]
          Storage=persistent
          SystemMaxUse=200M
          SystemMaxFiles=5
          RuntimeMaxUse=100M
          MaxRetentionSec=30day
          EOF
          ```
        - `sudo systemctl restart systemd-journald # or sudo reboot`
        - `sudo journalctl --flush`
        - Check status
            - `systemctl status rsyslog`
        - Check logs
          ```
          journalctl -xe
          journalctl -u ssh
          journalctl -u fail2ban
          ```
        - Check periodically
          ```
          df -h /var # disk space
          ncdu / # find big directories
          ls -lh /var/log/*.log
          ```
        - Verify
          ```
          # rsyslog running?
          systemctl status rsyslog

          systemctl status systemd-journald
          systemctl status logrotate.timer

          # Check
          cat /etc/logrotate.d/rsyslog

          # journald persistent and sized?
          journalctl --disk-usage
          ls -ld /var/log/journal

          # if too big manually trim
          journalctl --vacuum-size=150M   

          # see some logs
          journalctl -u ssh -n 20
          tail -n 20 /var/log/syslog
          tail -n 20 /var/log/auth.log
          ```

10. Disable unused services
   ```
   sudo systemctl list-unit-files --type=service --state=enabled

   sudo systemctl disable --now bluetooth
   ```

## Install Pi-hole

1. Installation
    - `curl -sSL https://install.pi-hole.net | bash # YOLO - text UI`
    - https://docs.pi-hole.net/main/basic-install/

2. Post Installation
    - https://docs.pi-hole.net/main/post-install/
    - Set Pi's network settings
        - Check Pi's network settings
            - On newer Raspberry Pi OS (Debian 12+), networking may be handled by NetworkManager, not dhcpcd
          ```
          systemctl status dhcpcd # should be not found
          systemctl status NetworkManager # should be running
          ```
        - Setup DNS to localhost
            - Check first
                - `nmcli connection show # e.g. Wired connection 1`
                - `nmcli device show eth0 | grep -E 'IP4|IP6|DNS'`
                    - Note: the IP6.DNS[1] shows e.g. fd5h:p09f:1778:1::1
            - `sudo nmtui # text UI`
                - Edit connection
                - Select active connection e.g. Wired connection 1
                    - Leave IPv4/IPv6 Configuration Automatic
                    - Show IPv4 Config
                        - use defaults, only set
                            1. DNS servers to `127.0.0.1`
                            2. check: `Ignore automatically obtained DNS paramters`
                    - Show IPv6 Config
                        - Choose one of the ULA addresses (fdxx - or take the current prefix of the IP6.DNS[1])
                            - take the `fd5h:p09f:1778:1::1` and choose a custom suffix e.g. `fd5h:p09f:1778:1::53`
                        - use defaults, only set
                            1. Addresses to `fd5h:p09f:1778:1::53/64`
            - Restart connection
              ```
              sudo reboot
              ```
            - Verify
              ```
              ip addr show eth0
              ip route show
              ip -6 route show

              cat /etc/resolv.conf
              nmcli device show eth0 | grep -E 'IP4|IP6|DNS'

              # IP4.DNS[1] should be 127.0.0.1
              # IP6.ADDRESS[1] should be the new fd5h:p09f:1778:1::53/64
              # IP6.DNS[1] should be fd5h:p09f:1778:1::53
              ```
    - Pi-hole will listen now on all local IPv6 addresses
        - but we set a IPv6 static lease
        - the fixed IPv6 will not change and can be therefore used in the router settings
    - Go to router settings
        - Set custom DNS
            1. IPv4 Primary: `192.168.2.100`
            2. IPv6 Primary: `fd5h:p09f:1778:1::53`
        - Reboot the router
        - Verify that on all local devices connected to the same Wi-Fi have now the DNS settings on automatic which
          should be now `192.168.2.100` and `fd5h:p09f:1778:1::53`

3. Updating
    - `pihole -up`
    - https://docs.pi-hole.net/main/update/

4. Uninstall
    - Make sure that the DNS settings in the Router are back to automatic/default
    - If DHCP was disabled on router, also set this back to automatic
    - `pihole uninstall`
    - https://docs.pi-hole.net/main/uninstall/

## Configure Pi-hole

1. Settings
    1. DNS
        1. Choose Upstream DNS Servers (e.g. Cloudflare (DNSSEC))
            - Check the 4 Checkboxes for IPv4 and IPv6

2. Lists
   1. Checkout my Gist: https://gist.github.com/3x6x9/b7b11eaeb67b0ab572ee5fac783f0a79
      1. Copy everything under blocklists (or only some selection)
      2. Paste everything into the `Add a new subscribed list`
      3. Press `Add blocklist`
      4. Go to Tools → Update Gravity

## Install Unbound

Follow the Introductions on https://docs.pi-hole.net/guides/dns/unbound/
