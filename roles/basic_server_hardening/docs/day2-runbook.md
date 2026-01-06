# Basic Server Hardening - Day 2+ Runbook

Operational guide for managing and maintaining server security.

---

## 1. SSH Management

### 1.1 Adding SSH Users

```bash
# Create new user
sudo adduser newuser

# Add to sudo group
sudo usermod -aG sudo newuser

# Add to allowed SSH users (if using AllowUsers)
sudo nano /etc/ssh/sshd_config
# Add: AllowUsers existing-user newuser

# Test config and restart
sudo sshd -t
sudo systemctl restart ssh
```

### 1.2 Changing SSH Port

In your playbook:

```yaml
vars:
  ssh_hardening_port: 2222
  fail2ban_ssh_port: 2222  # Must match!
  firewall_allowed_ports:
    - port: 2222
      proto: tcp
      comment: "SSH Custom Port"
```

Then re-run the playbook.

### 1.3 Managing SSH Keys

```bash
# View authorized keys
cat ~/.ssh/authorized_keys

# Add new key
echo "ssh-rsa AAAA..." >> ~/.ssh/authorized_keys

# Remove a key
nano ~/.ssh/authorized_keys
# Delete the unwanted line

# Set correct permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 1.4 Temporarily Allow Password Authentication

**Only for emergency access!**

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Change to
PasswordAuthentication yes

# Restart SSH
sudo systemctl restart ssh

# DON'T FORGET TO REVERT AFTER USE!
```

---

## 2. Firewall Management

### 2.1 Managing Ports

```bash
# Allow a port
sudo ufw allow 3306/tcp comment 'MySQL'

# Allow from specific IP
sudo ufw allow from 192.168.1.100

# Allow subnet
sudo ufw allow from 192.168.1.0/24 to any port 22

# Delete a rule
sudo ufw status numbered
sudo ufw delete 5

# Deny a port
sudo ufw deny 23/tcp
```

### 2.2 Application Profiles

```bash
# List available profiles
sudo ufw app list

# Allow application
sudo ufw allow 'Nginx Full'

# View app definition
sudo ufw app info 'Nginx Full'
```

### 2.3 Advanced Rules

```bash
# Allow specific IP to specific port
sudo ufw allow from 192.168.1.100 to any port 3306 proto tcp

# Allow port range
sudo ufw allow 6000:6007/tcp

# Limit connections (rate limiting)
sudo ufw limit ssh

# Allow established connections
sudo ufw allow in on eth0 to any port 80
```

### 2.4 Troubleshooting Firewall

```bash
# Check status
sudo ufw status verbose

# Check logs
sudo tail -f /var/log/ufw.log

# Reset firewall (CAREFUL!)
sudo ufw --force reset

# View raw iptables
sudo iptables -L -n -v
```

---

## 3. Fail2Ban Management

### 3.1 Managing Bans

```bash
# Check all jails
sudo fail2ban-client status

# Check SSH jail
sudo fail2ban-client status sshd

# Unban IP
sudo fail2ban-client set sshd unbanip 192.168.1.100

# Ban IP manually
sudo fail2ban-client set sshd banip 192.168.1.200

# Unban all from jail
sudo fail2ban-client unban --all
```

### 3.2 Configuring Jail Settings

Edit `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 7200
findtime = 600
ignoreip = 127.0.0.1/8 ::1 192.168.1.100
```

Reload after changes:

```bash
sudo fail2ban-client reload
```

### 3.3 Monitoring Fail2Ban

```bash
# Watch bans in real-time
sudo tail -f /var/log/fail2ban.log

# Check recently banned IPs
sudo fail2ban-client status sshd | grep "Banned IP"

# View ban list
sudo fail2ban-client get sshd banip

# Statistics
sudo fail2ban-client status sshd
```

### 3.4 Email Notifications

Edit `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
destemail = admin@example.com
sendername = Fail2Ban
action = %(action_mwl)s
```

---

## 4. Auto-Upgrades Management

### 4.1 Configuring Update Behavior

Edit `/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
};

Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
Unattended-Upgrade::Mail "admin@example.com";
```

### 4.2 Monitoring Updates

```bash
# Check update log
sudo cat /var/log/unattended-upgrades/unattended-upgrades.log

# View what will be updated
sudo unattended-upgrade --dry-run

# Check if reboot is required
[ -f /var/run/reboot-required ] && cat /var/run/reboot-required.pkgs
```

### 4.3 Manual Updates

```bash
# Update package list
sudo apt update

# Upgrade all packages
sudo apt upgrade

# Dist upgrade
sudo apt dist-upgrade

# Auto remove
sudo apt autoremove
```

---

## 5. Security Auditing

### 5.1 Check Open Ports

```bash
# From the server
sudo netstat -tulpn | grep LISTEN
sudo ss -tulpn | grep LISTEN

# Using nmap (from another machine)
nmap -sS your-server-ip
nmap -p- your-server-ip
```

### 5.2 Review Authentication Logs

```bash
# Successful logins
sudo grep "Accepted" /var/log/auth.log

# Failed login attempts
sudo grep "Failed password" /var/log/auth.log

# All root login attempts
sudo grep "root" /var/log/auth.log

# Sudo usage
sudo grep "sudo" /var/log/auth.log
```

### 5.3 Check for Suspicious Activity

```bash
# Currently logged in users
who
w

# Login history
last
last -a

# Failed login attempts
sudo lastb

# Check for SUID files
sudo find / -perm -4000 2>/dev/null
```

---

## 6. Troubleshooting

### 6.1 Can't Connect via SSH

**Checklist**:
```bash
# 1. Is SSH running?
sudo systemctl status ssh

# 2. Is the port open in firewall?
sudo ufw status | grep 22  # or your custom port

# 3. Check SSH config
sudo sshd -t
sudo cat /etc/ssh/sshd_config | grep -E "Port|PermitRootLogin|PasswordAuth"

# 4. Check if your IP is banned
sudo fail2ban-client status sshd

# 5. Check logs
sudo tail -100 /var/log/auth.log
```

### 6.2 Firewall Blocking Legitimate Traffic

```bash
# Check which rules are being hit
sudo ufw status verbose

# Check firewall logs
sudo tail -100 /var/log/ufw.log

# Temporarily disable to test
sudo ufw disable
# Test your connection
sudo ufw enable
```

### 6.3 Fail2Ban False Positives

```bash
# Unban yourself
sudo fail2ban-client set sshd unbanip YOUR_IP

# Add to whitelist
sudo nano /etc/fail2ban/jail.local

# Under [sshd], add:
ignoreip = 127.0.0.1/8 ::1 YOUR_IP YOUR_SUBNET/24

# Reload
sudo fail2ban-client reload
```

### 6.4 Auto-Updates Not Working

```bash
# Check service status
sudo systemctl status unattended-upgrades

# Check configuration
dpkg-reconfigure -plow unattended-upgrades

# Test manually
sudo unattended-upgrade --dry-run --debug

# Check logs for errors
sudo tail -100 /var/log/unattended-upgrades/unattended-upgrades.log
```

---

## 7. Disaster Recovery

### 7.1 Lost SSH Access

Via console access:

```bash
# Reset SSH to defaults
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
sudo nano /etc/ssh/sshd_config

# Set temporarily:
Port 22
PermitRootLogin yes
PasswordAuthentication yes

# Restart
sudo systemctl restart ssh

# Fix the issue from remote access
# Then re-run the hardening role
```

### 7.2 Firewall Lockout

Via console access:

```bash
# Disable firewall
sudo ufw disable

# Fix rules
sudo ufw allow 22/tcp
sudo ufw allow from TRUSTED_IP

# Re-enable
sudo ufw enable
```

### 7.3 Fail2Ban Issues

```bash
# Stop Fail2Ban
sudo systemctl stop fail2ban

#Clear bans
sudo rm /var/lib/fail2ban/fail2ban.sqlite3

# Restart
sudo systemctl start fail2ban
```

---

## 8. Best Practices

1. **Always keep console access** - Don't rely solely on SSH
2. **Test changes incrementally** - One change at a time
3. **Keep session open** - When making SSH/firewall changes
4. **Use SSH keys** - Much more secure than passwords
5. **Whitelist your IPs** - Add to Fail2Ban ignore list
6. **Monitor logs** - Check regularly for suspicious activity
7. **Document changes** - Keep notes on custom configurations
8. **Regular audits** - Review security settings monthly
9. **Backup configs** - Before making changes
10. **Update regularly** - Keep all software current

---

## 9. Maintenance Checklist

**Daily**:
- [ ] Check Fail2Ban for unusual ban patterns
- [ ] Review authentication logs

**Weekly**:
- [ ] Check for pending security updates
- [ ] Review firewall rules
- [ ] Check for required reboots

**Monthly**:
- [ ] Audit SSH users and keys
- [ ] Review all firewall rules
- [ ] Check Fail2Ban configuration
- [ ] Test disaster recovery procedures
- [ ] Update security documentation

---

## 10. Advanced Topics

### 10.1 Two-Factor Authentication

Install and configure Google Authenticator:

```bash
sudo apt install libpam-google-authenticator

# Run for each user
google-authenticator

# Edit PAM
sudo nano /etc/pam.d/sshd
# Add: auth required pam_google_authenticator.so

# Edit SSH config
sudo nano /etc/ssh/sshd_config
# Set: ChallengeResponseAuthentication yes

# Restart
sudo systemctl restart ssh
```

### 10.2 Port Knocking

Install knockd for additional security:

```bash
sudo apt install knockd

# Configure
sudo nano /etc/knockd.conf

# Example sequence
[options]
    UseSyslog

[openSSH]
    sequence    = 7000,8000,9000
    seq_timeout = 5
    command     = ufw allow from %IP% to any port 22
    tcpflags    = syn

[closeSSH]
    sequence    = 9000,8000,7000
    seq_timeout = 5
    command     = ufw delete allow from %IP% to any port 22
    tcpflags    = syn
```

### 10.3 Intrusion Detection (AIDE)

```bash
# Install AIDE
sudo apt install aide

# Initialize database
sudo aideinit

# Check for changes
sudo aide --check

# Update database after legitimate changes
sudo aide --update
```

---

## 11. Getting Help

```bash
# SSH help
man sshd_config
sudo sshd -t

# UFW help
man ufw
sudo ufw --help

# Fail2Ban help
man fail2ban
fail2ban-client --help

# System logs
sudo journalctl -u ssh
sudo journalctl -u fail2ban
```

For more information, review the [Day 1 Guide](day1-guide.md) or consult official documentation for each component.
