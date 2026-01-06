# Basic Server Hardening - Day 1 Guide

Quick start guide after deploying the `devops_store.basic_server_hardening` role.

---

## 1. Post-Deployment Verification

### 1.1 Verify SSH Configuration

```bash
# Check SSH service status
sudo systemctl status ssh

# Test SSH configuration
sudo sshd -t

# View SSH configuration
sudo cat /etc/ssh/sshd_config | grep -E "Port|PermitRootLogin|PasswordAuthentication"
```

### 1.2 Verify Firewall

```bash
# Check UFW status
sudo ufw status verbose

# List all rules
sudo ufw status numbered

# Check default policies
sudo ufw show raw
```

### 1.3 Verify Fail2Ban

```bash
# Check Fail2Ban status
sudo systemctl status fail2ban

# Check active jails
sudo fail2ban-client status

# Check SSH jail specifically
sudo fail2ban-client status sshd
```

### 1.4 Verify Auto-Upgrades

```bash
# Check if unattended-upgrades is running
sudo systemctl status unattended-upgrades

# Test configuration
sudo unattended-upgrade --dry-run --debug

# Check configuration
cat /etc/apt/apt.conf.d/50unattended-upgrades
```

---

## 2. Testing Your Setup

### 2.1 Test SSH Access

**IMPORTANT**: Keep your current session open while testing!

```bash
# From another terminal/machine, test SSH connection
ssh user@your-server

# If you changed the SSH port
ssh -p 2222 user@your-server

# Test with verbose output
ssh -v user@your-server
```

### 2.2 Test Firewall Rules

```bash
# From another machine, test open ports
nmap your-server-ip

# Test HTTP (if configured)
curl http://your-server-ip

# Test HTTPS (if configured)
curl https://your-server-ip
```

### 2.3 Test Fail2Ban

```bash
# Try failed SSH logins (from another machine)
# After several failures, check if IP is banned

# Check banned IPs
sudo fail2ban-client status sshd

# View ban log
sudo tail -f /var/log/fail2ban.log
```

---

## 3. Understanding What Was Changed

### 3.1 SSH Hardening

**Changes made**:
- Root login disabled (unless configured otherwise)
- Password authentication disabled (unless configured otherwise)
- SSH port possibly changed
- Maximum authentication attempts limited
- Specific users/groups may be allowed

**Configuration file**: `/etc/ssh/sshd_config`

### 3.2 UFW Firewall

**Changes made**:
- UFW enabled and active
- Default policy: deny incoming, allow outgoing
- Allowed ports configured (SSH, HTTP, HTTPS, custom)

**Check rules**:
```bash
sudo ufw status
```

### 3.3 Fail2Ban Protection

**Changes made**:
- Fail2Ban installed and active
- SSH jail enabled
- Ban time, find time, and max retries configured
- Email notifications (if configured)

**Check jails**:
```bash
sudo fail2ban-client status
```

### 3.4 Automatic Updates

**Changes made**:
- Unattended-upgrades installed
- Security updates automatically installed
- Automatic reboot (if configured)
- Email notifications (if configured)

**Check configuration**:
```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

---

## 4. Common First-Day Tasks

### 4.1 Update SSH Config File

If you need to adjust SSH settings:

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Test configuration
sudo sshd -t

# Restart SSH (KEEP CURRENT SESSION OPEN!)
sudo systemctl restart ssh
```

### 4.2 Add Firewall Rules

```bash
# Allow a new port
sudo ufw allow 3306/tcp comment 'MySQL'

# Allow from specific IP
sudo ufw allow from 192.168.1.100 to any port 22

# Delete a rule
sudo ufw delete allow 80/tcp

# Reload firewall
sudo ufw reload
```

### 4.3 Manage Fail2Ban

```bash
# Unban an IP
sudo fail2ban-client set sshd unbanip 192.168.1.100

# Ban an IP manually
sudo fail2ban-client set sshd banip 192.168.1.200

# Reload Fail2Ban configuration
sudo fail2ban-client reload

# Restart Fail2Ban
sudo systemctl restart fail2ban
```

### 4.4 Check Automatic Updates

```bash
# View update log
sudo cat /var/log/unattended-upgrades/unattended-upgrades.log

# View packages that will be updated
sudo unattended-upgrade --dry-run

# Manually trigger update (for testing)
sudo unattended-upgrade --debug
```

---

## 5. Security Health Check

Run these commands to verify your security posture:

```bash
# 1. SSH is hardened
sudo sshd -t && echo "SSH config OK"

# 2. Firewall is active
sudo ufw status | grep "Status: active"

# 3. Fail2Ban is running
sudo systemctl is-active fail2ban

# 4. Auto-updates are configured
sudo systemctl is-enabled unattended-upgrades

# 5. No root login via SSH
sudo grep "^PermitRootLogin no" /etc/ssh/sshd_config

# 6. Password authentication disabled (if configured)
sudo grep "^PasswordAuthentication no" /etc/ssh/sshd_config

# 7. Check failed login attempts
sudo journalctl -u ssh | grep "Failed password" | tail -20

# 8. Check Fail2Ban bans
sudo fail2ban-client status sshd
```

---

## 6. What If Something Goes Wrong?

### 6.1 Locked Out of SSH

**Prevention**:
- Always keep one SSH session open when making changes
- Test new SSH configuration before logging out
- Have console access (via hosting provider) as backup

**Recovery** (if you have console access):
```bash
# Via console, edit SSH config
sudo nano /etc/ssh/sshd_config

# Set temporarily
PermitRootLogin yes
PasswordAuthentication yes

# Restart SSH
sudo systemctl restart ssh

# Log in and fix the issue
```

### 6.2 Firewall Blocking Access

**Recovery** (via console access):
```bash
# Disable firewall temporarily
sudo ufw disable

# Add necessary rules
sudo ufw allow 22/tcp

# Re-enable firewall
sudo ufw enable
```

### 6.3 Fail2Ban Banned Your IP

```bash
# Via console or another IP
sudo fail2ban-client set sshd unbanip YOUR_IP

# Add your IP to ignore list
sudo nano /etc/fail2ban/jail.local

# Add under [sshd]
ignoreip = 127.0.0.1/8 YOUR_IP
```

---

## 7. Local Client Configuration

### 7.1 Update SSH Config (if port changed)

On your local machine, edit `~/.ssh/config`:

```bash
Host myserver
    HostName your-server-ip
    Port 2222  # New SSH port
    User your-username
    IdentityFile ~/.ssh/id_rsa
```

### 7.2 Use SSH Config

```bash
# Now you can simply use
ssh myserver

# Instead of
ssh -p 2222 user@your-server-ip
```

---

## 8. Monitoring and Logs

### 8.1 Important Log Files

```bash
# SSH authentication logs
sudo tail -f /var/log/auth.log

# Fail2Ban logs
sudo tail -f /var/log/fail2ban.log

# UFW logs
sudo tail -f /var/log/ufw.log

# System logs
sudo journalctl -f

# Unattended upgrades log
sudo tail -f /var/log/unattended-upgrades/unattended-upgrades.log
```

### 8.2 Real-Time Monitoring

```bash
# Watch failed SSH attempts
sudo tail -f /var/log/auth.log | grep "Failed password"

# Watch Fail2Ban activity
sudo tail -f /var/log/fail2ban.log | grep Ban

# Watch firewall blocks
sudo tail -f /var/log/ufw.log
```

---

## 9. Tips and Best Practices

1. **Always test before logging out** - Keep a session open while testing
2. **Use SSH keys** - Much more secure than passwords
3. **Change default SSH port** - Reduces automated attacks
4. **Monitor logs regularly** - Check for suspicious activity
5. **Keep allow list updated** - Add trusted IPs to Fail2Ban ignore list
6. **Document changes** - Keep notes on custom configurations
7. **Test disaster recovery** - Ensure console access works
8. **Update regularly** - Re-run the role periodically
9. **Backup configurations** - Save copies of config files
10. **Use strong passwords** - For any remaining password authentication

---

## 10. Next Steps

- Review [`day2-runbook.md`](day2-runbook.md) for advanced management
- Set up monitoring and alerting
- Configure email notifications for security events
- Plan regular security audits
- Document your server's security baseline
- Set up automated configuration backups

---

## 11. Quick Reference

| Component | Status Command | Config File | Log File |
|-----------|----------------|-------------|----------|
| SSH | `systemctl status ssh` | `/etc/ssh/sshd_config` | `/var/log/auth.log` |
| UFW | `ufw status` | `/etc/ufw/` | `/var/log/ufw.log` |
| Fail2Ban | `fail2ban-client status` | `/etc/fail2ban/jail.local` | `/var/log/fail2ban.log` |
| Updates | `systemctl status unattended-upgrades` | `/etc/apt/apt.conf.d/50unattended-upgrades` | `/var/log/unattended-upgrades/` |
