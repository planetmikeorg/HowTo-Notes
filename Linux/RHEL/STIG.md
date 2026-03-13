# RHEL STIG Implementation Guide

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [STIG Automation Tools](#stig-automation-tools)
- [Manual STIG Implementation](#manual-stig-implementation)
- [STIG Categories](#stig-categories)
- [Version-Specific Differences](#version-specific-differences)
- [Verification and Compliance Scanning](#verification-and-compliance-scanning)
- [Common Issues and Remediation](#common-issues-and-remediation)
- [Best Practices](#best-practices)

---

## Overview

Security Technical Implementation Guides (STIGs) are configuration standards developed by DISA (Defense Information Systems Agency) for securing information systems. RHEL STIGs provide security hardening requirements for Red Hat Enterprise Linux systems.

### STIG Severity Levels
- **CAT I (High)**: Critical vulnerabilities that can lead to immediate compromise
- **CAT II (Medium)**: Vulnerabilities that can lead to compromise but require additional conditions
- **CAT III (Low)**: Vulnerabilities with minimal security impact

### Current STIG Versions
- **RHEL 8**: Version 1, Release 12 (as of 2026)
- **RHEL 9**: Version 1, Release 2 (as of 2026)
- **RHEL 10**: Beta/Preview (STIG in development)

---

## Prerequisites

### System Requirements
```bash
# Check RHEL version
cat /etc/redhat-release

# Check system architecture
uname -m

# Verify subscription status
subscription-manager status

# Check available disk space (at least 2GB free recommended)
df -h

# Check current SELinux status
getenforce
```

### Required Packages
```bash
# RHEL 8/9/10
sudo dnf install -y \
  aide \
  audit \
  openscap \
  openscap-scanner \
  scap-security-guide \
  policycoreutils-python-utils \
  rsyslog \
  firewalld \
  chrony \
  usbguard

# Additional for RHEL 8
sudo dnf install -y policycoreutils-python

# Additional for RHEL 9/10
sudo dnf install -y python3-policycoreutils
```

### Backup Before Starting
```bash
# Create system backup
sudo tar czf /root/pre-stig-backup-$(date +%Y%m%d).tar.gz \
  /etc \
  /boot/grub2 \
  /var/log

# Backup specific configurations
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.pre-stig
sudo cp -a /etc/pam.d /etc/pam.d.pre-stig
sudo cp -a /etc/login.defs /etc/login.defs.pre-stig
sudo cp -a /etc/audit/auditd.conf /etc/audit/auditd.conf.pre-stig

# Create snapshot if using LVM
sudo lvcreate -L 5G -s -n pre-stig-snapshot /dev/rhel/root
```

---

## STIG Automation Tools

### OpenSCAP with SSG (Recommended)

#### Install and Configure
```bash
# Install OpenSCAP and SCAP Security Guide
sudo dnf install -y openscap-scanner scap-security-guide

# List available profiles
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel$(rpm -E %rhel)-ds.xml

# RHEL 8
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml

# RHEL 9
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

#### Scan System for STIG Compliance
```bash
# RHEL 8 - Full STIG scan
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results /tmp/stig-scan-results.xml \
  --report /tmp/stig-scan-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml

# RHEL 9 - Full STIG scan
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results /tmp/stig-scan-results.xml \
  --report /tmp/stig-scan-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Generate remediation script
sudo oscap xccdf generate fix \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --output /tmp/stig-remediation.sh \
  /tmp/stig-scan-results.xml
```

#### Apply Automated Remediation
```bash
# Review the remediation script first
less /tmp/stig-remediation.sh

# Make backup before applying
sudo cp /tmp/stig-remediation.sh /root/stig-remediation-$(date +%Y%m%d).sh

# Apply remediations (CAREFULLY - test on non-production first)
sudo bash /tmp/stig-remediation.sh

# Or apply with OpenSCAP directly
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --remediate \
  --results /tmp/stig-remediation-results.xml \
  --report /tmp/stig-remediation-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel$(rpm -E %rhel)-ds.xml
```

### Ansible Automation

#### Install Ansible and RHEL System Roles
```bash
# Install Ansible
sudo dnf install -y ansible-core rhel-system-roles

# For RHEL 8
sudo dnf install -y ansible

# Verify installation
ansible --version
```

#### Using Ansible Lockdown Project
```bash
# Clone RHEL 8 STIG role
git clone https://github.com/ansible-lockdown/RHEL8-STIG.git
cd RHEL8-STIG

# Clone RHEL 9 STIG role
git clone https://github.com/ansible-lockdown/RHEL9-STIG.git
cd RHEL9-STIG

# Review default variables
cat defaults/main.yml

# Create inventory
cat > inventory.ini << 'EOF'
[stig_hosts]
localhost ansible_connection=local
EOF

# Run playbook in check mode first
ansible-playbook -i inventory.ini site.yml --check

# Apply STIG
sudo ansible-playbook -i inventory.ini site.yml
```

#### Custom Ansible Playbook Example
```yaml
---
- name: Apply RHEL STIG
  hosts: all
  become: yes
  vars:
    stig_profile: stig
    
  tasks:
    - name: Install OpenSCAP packages
      dnf:
        name:
          - openscap-scanner
          - scap-security-guide
        state: present

    - name: Run STIG compliance scan
      command: >
        oscap xccdf eval
        --profile xccdf_org.ssgproject.content_profile_{{ stig_profile }}
        --results /tmp/stig-scan-{{ ansible_date_time.epoch }}.xml
        /usr/share/xml/scap/ssg/content/ssg-rhel{{ ansible_distribution_major_version }}-ds.xml
      register: scan_result
      failed_when: false

    - name: Generate remediation script
      command: >
        oscap xccdf generate fix
        --profile xccdf_org.ssgproject.content_profile_{{ stig_profile }}
        --output /tmp/stig-remediation-{{ ansible_date_time.epoch }}.sh
        /tmp/stig-scan-{{ ansible_date_time.epoch }}.xml
```

---

## Manual STIG Implementation

### Account and Authentication (CAT I & II)

#### Password Policy
```bash
# /etc/security/pwquality.conf
cat > /etc/security/pwquality.conf << 'EOF'
# RHEL 8/9/10 Password Requirements
minlen = 15
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
difok = 8
maxrepeat = 3
maxclassrepeat = 4
gecoscheck = 1
dictcheck = 1
usercheck = 1
enforcing = 1
retry = 3
EOF

# RHEL 8/9/10 - Configure pam_faillock
# Edit /etc/security/faillock.conf
cat > /etc/security/faillock.conf << 'EOF'
# Lock account after failed attempts
deny = 3
fail_interval = 900
unlock_time = 900
silent
audit
even_deny_root
EOF

# Configure PAM for password requirements
# /etc/pam.d/system-auth
sudo authselect select sssd with-faillock --force
```

#### Account Lockout (RHEL 8)
```bash
# /etc/pam.d/password-auth
auth        required      pam_faillock.so preauth silent audit deny=3 unlock_time=900 fail_interval=900
auth        sufficient    pam_unix.so try_first_pass
auth        [default=die] pam_faillock.so authfail audit deny=3 unlock_time=900 fail_interval=900
account     required      pam_faillock.so

# Apply to system-auth as well
sudo cp /etc/pam.d/password-auth /etc/pam.d/system-auth
```

#### Account Lockout (RHEL 9/10)
```bash
# RHEL 9/10 uses authselect
sudo authselect select sssd with-faillock without-nullok --force

# Verify configuration
sudo authselect current
```

#### Password Aging
```bash
# /etc/login.defs
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   60/' /etc/login.defs
sudo sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/' /etc/login.defs
sudo sed -i 's/^PASS_MIN_LEN.*/PASS_MIN_LEN    15/' /etc/login.defs
sudo sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE   7/' /etc/login.defs

# Set for existing users
for user in $(awk -F: '($3>=1000)&&($3!=65534) {print $1}' /etc/passwd); do
  sudo chage -M 60 -m 1 -W 7 "$user"
done

# Set for root
sudo chage -M 60 -m 1 -W 7 root
```

#### Disable Inactive Accounts
```bash
# Set inactive account lock period to 35 days
sudo useradd -D -f 35

# Apply to existing users
for user in $(awk -F: '($3>=1000)&&($3!=65534) {print $1}' /etc/passwd); do
  sudo chage -I 35 "$user"
done
```

### SSH Hardening (CAT I & II)

#### SSH Configuration
```bash
# Backup original
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Apply STIG settings
sudo tee /etc/ssh/sshd_config.d/00-stig.conf << 'EOF'
# STIG SSH Configuration

# Protocol and Encryption
Protocol 2
Ciphers aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512,hmac-sha2-256
KexAlgorithms ecdh-sha2-nistp521,ecdh-sha2-nistp384,ecdh-sha2-nistp256,diffie-hellman-group-exchange-sha256

# Authentication
PermitRootLogin no
PermitEmptyPasswords no
PermitUserEnvironment no
HostbasedAuthentication no
IgnoreRhosts yes
PubkeyAuthentication yes
PasswordAuthentication yes
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication yes
UsePAM yes

# Security Settings
StrictModes yes
X11Forwarding no
PrintLastLog yes
Compression no
ClientAliveInterval 600
ClientAliveCountMax 0
LoginGraceTime 60
MaxAuthTries 3
MaxSessions 10
MaxStartups 10:30:60

# Logging
SyslogFacility AUTHPRIV
LogLevel VERBOSE

# Banner
Banner /etc/issue
EOF

# Verify configuration
sudo sshd -t

# Restart SSH service
sudo systemctl restart sshd
```

#### RHEL 9/10 SSH Updates
```bash
# RHEL 9/10 deprecated some older algorithms
# Additional secure ciphers for RHEL 9/10
sudo tee -a /etc/ssh/sshd_config.d/00-stig.conf << 'EOF'

# RHEL 9/10 Enhanced Security
PubkeyAcceptedKeyTypes ssh-ed25519,ecdsa-sha2-nistp521,ecdsa-sha2-nistp384,ecdsa-sha2-nistp256,rsa-sha2-512,rsa-sha2-256
HostKeyAlgorithms ssh-ed25519,ecdsa-sha2-nistp521,ecdsa-sha2-nistp384,ecdsa-sha2-nistp256,rsa-sha2-512,rsa-sha2-256
EOF
```

### Audit Configuration (CAT II)

#### Configure Auditd
```bash
# RHEL 8/9/10 - /etc/audit/auditd.conf
sudo tee /etc/audit/auditd.conf << 'EOF'
log_file = /var/log/audit/audit.log
log_format = ENRICHED
log_group = root
priority_boost = 4
flush = INCREMENTAL_ASYNC
freq = 50
num_logs = 5
disp_qos = lossy
dispatcher = /sbin/audispd
name_format = HOSTNAME
max_log_file = 10
max_log_file_action = ROTATE
space_left = 75
space_left_action = email
action_mail_acct = root
admin_space_left = 50
admin_space_left_action = halt
disk_full_action = HALT
disk_error_action = HALT
tcp_listen_queue = 5
tcp_max_per_addr = 1
use_libwrap = yes
tcp_client_max_idle = 0
enable_krb5 = no
krb5_principal = auditd
EOF

# Enable and start auditd
sudo systemctl enable auditd
sudo systemctl start auditd
```

#### Audit Rules
```bash
# RHEL 8/9/10 - Create comprehensive audit rules
sudo tee /etc/audit/rules.d/stig.rules << 'EOF'
## STIG Audit Rules

# Remove any existing rules
-D

# Buffer Size
-b 8192

# Failure Mode (2 = panic)
-f 2

# Audit system calls
-a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time-change
-a always,exit -F arch=b32 -S adjtimex -S settimeofday -S stime -k time-change
-a always,exit -F arch=b64 -S clock_settime -k time-change
-a always,exit -F arch=b32 -S clock_settime -k time-change
-w /etc/localtime -p wa -k time-change

# User and group changes
-w /etc/group -p wa -k identity
-w /etc/passwd -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/security/opasswd -p wa -k identity

# Network environment changes
-a always,exit -F arch=b64 -S sethostname -S setdomainname -k system-locale
-a always,exit -F arch=b32 -S sethostname -S setdomainname -k system-locale
-w /etc/issue -p wa -k system-locale
-w /etc/issue.net -p wa -k system-locale
-w /etc/hosts -p wa -k system-locale
-w /etc/sysconfig/network -p wa -k system-locale

# SELinux changes
-w /etc/selinux/ -p wa -k MAC-policy
-w /usr/share/selinux/ -p wa -k MAC-policy

# Login/logout events
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock/ -p wa -k logins

# Session initiation
-w /var/run/utmp -p wa -k session
-w /var/log/wtmp -p wa -k logins
-w /var/log/btmp -p wa -k logins

# Discretionary access control changes
-a always,exit -F arch=b64 -S chmod -S fchmod -S fchmodat -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b32 -S chmod -S fchmod -S fchmodat -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b64 -S chown -S fchown -S fchownat -S lchown -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b32 -S chown -S fchown -S fchownat -S lchown -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b64 -S setxattr -S lsetxattr -S fsetxattr -S removexattr -S lremovexattr -S fremovexattr -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b32 -S setxattr -S lsetxattr -S fsetxattr -S removexattr -S lremovexattr -S fremovexattr -F auid>=1000 -F auid!=4294967295 -k perm_mod

# Unauthorized file access attempts
-a always,exit -F arch=b64 -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EACCES -F auid>=1000 -F auid!=4294967295 -k access
-a always,exit -F arch=b32 -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EACCES -F auid>=1000 -F auid!=4294967295 -k access
-a always,exit -F arch=b64 -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EPERM -F auid>=1000 -F auid!=4294967295 -k access
-a always,exit -F arch=b32 -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EPERM -F auid>=1000 -F auid!=4294967295 -k access

# Privileged commands
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/bin/sudoedit -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/bin/chsh -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/bin/newgrp -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/bin/chcon -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/sbin/semanage -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/sbin/setsebool -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/sbin/setfiles -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/sbin/userhelper -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-priv_change
-a always,exit -F path=/usr/bin/passwd -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-passwd
-a always,exit -F path=/usr/sbin/unix_chkpwd -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-passwd
-a always,exit -F path=/usr/bin/gpasswd -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-passwd
-a always,exit -F path=/usr/bin/chage -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-passwd
-a always,exit -F path=/usr/sbin/usermod -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged-usermod

# File deletion by users
-a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -F auid>=1000 -F auid!=4294967295 -k delete
-a always,exit -F arch=b32 -S unlink -S unlinkat -S rename -S renameat -F auid>=1000 -F auid!=4294967295 -k delete

# Sudoers file changes
-w /etc/sudoers -p wa -k scope
-w /etc/sudoers.d/ -p wa -k scope

# Kernel module loading/unloading
-w /sbin/insmod -p x -k modules
-w /sbin/rmmod -p x -k modules
-w /sbin/modprobe -p x -k modules
-a always,exit -F arch=b64 -S init_module -S delete_module -k modules

# Make configuration immutable
-e 2
EOF

# Load audit rules
sudo augenrules --load

# Verify rules loaded
sudo auditctl -l

# Restart auditd
sudo service auditd restart
```

### SELinux Configuration (CAT I)

#### Enable and Configure SELinux
```bash
# Check current SELinux status
getenforce
sestatus

# RHEL 8/9/10 - Set to Enforcing mode
sudo setenforce 1

# Make persistent
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config

# Set targeted policy
sudo sed -i 's/^SELINUXTYPE=.*/SELINUXTYPE=targeted/' /etc/selinux/config

# Verify configuration
cat /etc/selinux/config

# Label filesystem (if needed, requires reboot)
sudo touch /.autorelabel
sudo reboot
```

#### SELinux Denials Handling
```bash
# Install troubleshooting tools
sudo dnf install -y setroubleshoot-server

# Check for denials
sudo ausearch -m avc -ts recent

# Generate policy from denials
sudo ausearch -m avc -ts recent | audit2allow -M local_policy
sudo semodule -i local_policy.pp

# View loaded modules
sudo semodule -l
```

### Firewall Configuration (CAT II)

#### Configure Firewalld
```bash
# RHEL 8/9/10 - Enable firewalld
sudo systemctl enable firewalld
sudo systemctl start firewalld

# Set default zone to drop
sudo firewall-cmd --set-default-zone=drop

# Create custom zone for trusted services
sudo firewall-cmd --permanent --new-zone=trusted-services
sudo firewall-cmd --permanent --zone=trusted-services --add-service=ssh
sudo firewall-cmd --permanent --zone=trusted-services --add-source=10.0.0.0/8

# Configure public zone (minimal services)
sudo firewall-cmd --permanent --zone=public --remove-service=dhcpv6-client
sudo firewall-cmd --permanent --zone=public --remove-service=ssh

# Reload firewall
sudo firewall-cmd --reload

# Verify configuration
sudo firewall-cmd --list-all-zones
```

### File System and Kernel (CAT II & III)

#### File System Configuration
```bash
# /etc/fstab - Add security options
# Add these options to existing entries or create separate partitions

# /tmp with nosuid,nodev,noexec
sudo tee -a /etc/fstab << 'EOF'
tmpfs /tmp tmpfs defaults,nosuid,nodev,noexec 0 0
EOF

# /var/tmp
sudo tee -a /etc/fstab << 'EOF'
/tmp /var/tmp none bind,nosuid,nodev,noexec 0 0
EOF

# /dev/shm
sudo mount -o remount,nosuid,nodev,noexec /dev/shm

# Update fstab for /dev/shm
sudo sed -i 's|\(tmpfs[[:space:]]\+/dev/shm[[:space:]]\+tmpfs[[:space:]]\+\)defaults|\1defaults,nosuid,nodev,noexec|' /etc/fstab

# Apply changes
sudo systemctl daemon-reload
```

#### Kernel Parameters
```bash
# /etc/sysctl.d/99-stig.conf
sudo tee /etc/sysctl.d/99-stig.conf << 'EOF'
# STIG Kernel Parameters

# Network security
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_syncookies = 1

# IPv6 (if not used, disable)
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# Kernel security
kernel.randomize_va_space = 2
kernel.exec-shield = 1
kernel.kptr_restrict = 1
kernel.dmesg_restrict = 1
kernel.yama.ptrace_scope = 1
fs.suid_dumpable = 0

# Core dumps
fs.suid_dumpable = 0
kernel.core_uses_pid = 1
EOF

# Apply kernel parameters
sudo sysctl -p /etc/sysctl.d/99-stig.conf

# Verify
sudo sysctl -a | grep -E 'net.ipv4|kernel.randomize'
```

#### Disable Core Dumps
```bash
# /etc/security/limits.conf
echo "* hard core 0" | sudo tee -a /etc/security/limits.conf

# systemd
sudo mkdir -p /etc/systemd/coredump.conf.d
sudo tee /etc/systemd/coredump.conf.d/disable.conf << 'EOF'
[Coredump]
Storage=none
EOF

# Disable ABRT (Automatic Bug Reporting Tool)
sudo systemctl disable abrtd
sudo systemctl stop abrtd
```

### USB Storage Restriction (CAT II)

#### Disable USB Storage
```bash
# RHEL 8/9/10
echo "install usb-storage /bin/true" | sudo tee /etc/modprobe.d/usb-storage.conf

# Blacklist USB storage
echo "blacklist usb-storage" | sudo tee -a /etc/modprobe.d/usb-storage.conf

# Update initramfs
sudo dracut -f

# Or use USBGuard (recommended)
sudo dnf install -y usbguard

# Generate initial policy
sudo usbguard generate-policy > /etc/usbguard/rules.conf

# Enable and start
sudo systemctl enable usbguard
sudo systemctl start usbguard

# Allow specific devices
sudo usbguard allow-device <device-id>
```

---

## STIG Categories

### Category I (Critical) Controls

1. **Disk encryption must be used** (FIPS 140-2 compliant)
2. **SELinux must be enabled** in enforcing mode
3. **Root login via SSH must be disabled**
4. **Empty passwords must not be permitted**
5. **Default umask must be 077 or more restrictive**
6. **AIDE must be installed** and configured
7. **Audit system must be configured** to collect all required events
8. **Multi-factor authentication required** for privileged accounts

### Category II (Medium) Controls

1. Password complexity requirements
2. Account lockout after failed attempts
3. SSH protocol and cipher configuration
4. File system mount options
5. Kernel parameter hardening
6. Firewall configuration
7. Log aggregation and forwarding
8. Time synchronization (chronyd)
9. Package verification
10. USB storage restrictions

### Category III (Low) Controls

1. Login banners
2. Session timeout settings
3. Command history configuration
4. Remove unnecessary packages
5. Disable unnecessary services

---

## Version-Specific Differences

### RHEL 8 vs RHEL 9

| Feature | RHEL 8 | RHEL 9 | Notes |
|---------|--------|--------|-------|
| **Default Python** | Python 3.6 | Python 3.9 | Python 2 removed in RHEL 9 |
| **PAM Configuration** | Manual or authconfig | authselect required | authselect is mandatory |
| **Crypto Policy** | DEFAULT | DEFAULT:SHA1 deprecated | RHEL 9 removes weak crypto |
| **Firewall Backend** | iptables/nftables | nftables only | iptables-nft wrapper available |
| **SELinux Policy** | Targeted (v31) | Targeted (v35) | Enhanced policy in RHEL 9 |
| **Audit Daemon** | 2.x | 3.x | New features in 3.x |
| **OpenSSH** | 8.0 | 8.7+ | RSA-SHA1 deprecated |
| **FIPS Mode** | fips=1 kernel param | fips-mode-setup command | Easier FIPS enablement |
| **Kernel Version** | 4.18 | 5.14+ | Major kernel update |
| **systemd** | 239 | 250+ | New features and fixes |
| **GRUB** | GRUB2 | GRUB2 (BLS) | Boot Loader Specification |

### RHEL 8 Specific Commands

```bash
# PAM configuration (RHEL 8)
sudo authconfig --updateall

# Python 3.6
python3.6 --version

# Older crypto policies
sudo update-crypto-policies --set DEFAULT

# Enable FIPS (RHEL 8)
sudo dracut -f
sudo grubby --update-kernel=ALL --args="fips=1"
```

### RHEL 9 Specific Commands

```bash
# PAM configuration (RHEL 9 - mandatory)
sudo authselect select sssd with-faillock without-nullok --force
sudo authselect apply-changes

# Python 3.9+
python3 --version

# Enhanced crypto policies
sudo update-crypto-policies --set DEFAULT
sudo update-crypto-policies --show

# Enable FIPS (RHEL 9 - simplified)
sudo fips-mode-setup --enable
sudo reboot

# Check FIPS status
sudo fips-mode-setup --check

# Verify nftables
sudo nft list ruleset
```

### RHEL 9 Crypto Policy Changes

```bash
# RHEL 9 removes SHA-1 signatures by default
# If needed for legacy compatibility (NOT RECOMMENDED)
sudo update-crypto-policies --set DEFAULT:SHA1

# View current policy
sudo update-crypto-policies --show

# Custom crypto policy
sudo update-crypto-policies --set CUSTOM
```

### RHEL 10 (Preview/Beta)

**Note:** RHEL 10 is in development. STIG not yet finalized.

**Expected Changes:**
- Python 3.11+
- Kernel 6.x
- Enhanced security features
- Improved container security
- Updated SELinux policies
- Podman 5.x
- systemd 252+

```bash
# Check RHEL 10 version
cat /etc/redhat-release

# RHEL 10 will likely maintain RHEL 9 patterns
# Use authselect
sudo authselect select sssd with-faillock --force

# FIPS mode
sudo fips-mode-setup --enable
```

---

## Verification and Compliance Scanning

### OpenSCAP Verification

```bash
# Full compliance scan
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results /tmp/compliance-scan-$(date +%Y%m%d).xml \
  --report /tmp/compliance-report-$(date +%Y%m%d).html \
  /usr/share/xml/scap/ssg/content/ssg-rhel$(rpm -E %rhel)-ds.xml

# Generate score
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  /usr/share/xml/scap/ssg/content/ssg-rhel$(rpm -E %rhel)-ds.xml 2>&1 | grep "Score"

# Check specific rule
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --rule xccdf_org.ssgproject.content_rule_<rule-id> \
  /usr/share/xml/scap/ssg/content/ssg-rhel$(rpm -E %rhel)-ds.xml
```

### AIDE Verification

```bash
# Initialize AIDE database
sudo aide --init

# Move database to production location
sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

# Check file integrity
sudo aide --check

# Update database after legitimate changes
sudo aide --update

# Schedule daily checks
sudo tee /etc/cron.daily/aide << 'EOF'
#!/bin/bash
/usr/sbin/aide --check | /usr/bin/mail -s "AIDE Report: $(hostname)" root
EOF

sudo chmod +x /etc/cron.daily/aide
```

### Manual Verification Checklist

```bash
# Verify SELinux
getenforce  # Should return "Enforcing"
sestatus

# Verify auditd
sudo systemctl status auditd
sudo auditctl -l | wc -l  # Should show numerous rules

# Verify firewalld
sudo systemctl status firewalld
sudo firewall-cmd --list-all

# Verify SSH configuration
sudo sshd -T | grep -E 'permitrootlogin|permitemptypasswords|protocol|ciphers'

# Verify password policy
sudo pwscore <<< "testpassword"  # Should fail
grep -E 'minlen|dcredit|ucredit' /etc/security/pwquality.conf

# Verify kernel parameters
sudo sysctl net.ipv4.ip_forward
sudo sysctl kernel.randomize_va_space

# Verify mounted filesystems
mount | grep -E '/tmp|/var/tmp|/dev/shm'

# Verify USB storage disabled
lsmod | grep usb_storage  # Should return nothing

# Verify file permissions
sudo find / -xdev -type f -perm -002 ! -type l -ls 2>/dev/null

# Verify unowned files
sudo find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null
```

---

## Common Issues and Remediation

### SSH Connection Issues After STIG

**Problem:** Cannot connect via SSH after applying STIG

**Solutions:**
```bash
# Check SSH is running
sudo systemctl status sshd

# Check firewall allows SSH
sudo firewall-cmd --list-services
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload

# Check SSH configuration
sudo sshd -T | grep -i permitrootlogin

# Check PAM configuration
sudo authselect current

# Check SELinux denials
sudo ausearch -m avc -ts recent | grep sshd

# Emergency access via console if needed
# Temporarily allow root login
sudo sed -i 's/^PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config.d/00-stig.conf
sudo systemctl restart sshd
```

### Account Lockout Issues

**Problem:** Users locked out due to faillock

**Solutions:**
```bash
# Check failed attempts
sudo faillock --user <username>

# Unlock user
sudo faillock --user <username> --reset

# View faillock configuration
cat /etc/security/faillock.conf

# Adjust settings if too restrictive (carefully)
sudo vi /etc/security/faillock.conf
# Increase unlock_time or deny attempts
```

### Audit Log Full

**Problem:** System halts due to audit log full

**Solutions:**
```bash
# Check disk space
df -h /var/log/audit

# Check auditd configuration
cat /etc/audit/auditd.conf | grep -E 'space_left|disk_full_action'

# Rotate logs manually
sudo service auditd rotate

# Adjust log retention
sudo vi /etc/audit/auditd.conf
# Increase num_logs or adjust max_log_file

# Change action (NOT RECOMMENDED for production)
# disk_full_action = SUSPEND  # Instead of HALT
```

### SELinux Denials Blocking Applications

**Problem:** Applications fail due to SELinux denials

**Solutions:**
```bash
# Identify denials
sudo ausearch -m avc -ts recent

# Generate policy module
sudo ausearch -m avc -ts recent | audit2allow -M myapp
sudo semodule -i myapp.pp

# Or set specific context
sudo semanage fcontext -a -t httpd_sys_content_t "/path/to/app(/.*)?"
sudo restorecon -Rv /path/to/app

# Last resort: Create permissive domain (NOT RECOMMENDED)
sudo semanage permissive -a httpd_t
```

### System Boot Issues

**Problem:** System won't boot after STIG application

**Solutions:**
```bash
# Boot into emergency mode
# At GRUB, press 'e' and add to kernel line:
systemd.unit=emergency.target

# Or:
systemd.unit=rescue.target

# Check recent changes
journalctl -xb -p err

# Common issues:
# 1. FIPS mode with missing libraries
# 2. SELinux relabeling needed
# 3. Audit rules preventing boot

# Disable SELinux temporarily if needed
# Add to kernel parameters: selinux=0

# Disable audit if needed
# Add to kernel parameters: audit=0
```

### Performance Degradation

**Problem:** System slow after STIG

**Solutions:**
```bash
# Check audit performance impact
sudo systemctl status auditd
sudo auditctl -l | wc -l

# Optimize audit rules
# Remove overly verbose rules from /etc/audit/rules.d/stig.rules

# Check SELinux performance
sudo seinfo --all

# Check for excessive logging
du -sh /var/log/audit/

# Tune kernel parameters
sudo sysctl vm.swappiness=10

# Review running services
systemctl list-units --type=service --state=running
```

---

## Best Practices

### 1. Pre-Implementation

```bash
# Complete checklist before starting:
- [ ] Full system backup created
- [ ] LVM snapshot created (if applicable)
- [ ] Configuration backups saved
- [ ] Documentation reviewed
- [ ] Testing environment validated
- [ ] Maintenance window scheduled
- [ ] Rollback plan documented
- [ ] Team notified
```

### 2. Phased Implementation

```bash
# Phase 1: Non-disruptive changes
- Install packages
- Configure audit rules
- Enable SELinux (permissive first)
- Configure file system parameters

# Phase 2: Service configuration
- SSH hardening
- Firewall configuration
- Authentication changes (test thoroughly)

# Phase 3: Restrictive changes
- Account policies
- USB restrictions
- Core dump disabling
- Enable SELinux enforcing

# Phase 4: Validation
- Run compliance scans
- Test all services
- Verify user access
- Monitor logs
```

### 3. Documentation

```bash
# Document everything:
cat > /root/STIG-implementation-log.txt << 'EOF'
STIG Implementation Log
Date: $(date)
RHEL Version: $(cat /etc/redhat-release)
Implemented by: [Name]

Changes made:
- [List each change with timestamp]
- [Include any deviations from standard]
- [Note any exceptions granted]

Testing performed:
- [List tests run]
- [Document results]

Issues encountered:
- [List problems and resolutions]

Next actions:
- [Scheduled rescans]
- [Monitoring requirements]
EOF
```

### 4. Monitoring

```bash
# Set up continuous monitoring
# Daily compliance checks
cat > /etc/cron.daily/stig-check << 'EOF'
#!/bin/bash
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results /var/log/stig-daily-$(date +%Y%m%d).xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel$(rpm -E %rhel)-ds.xml 2>&1 | \
  mail -s "Daily STIG Scan $(hostname)" root
EOF

chmod +x /etc/cron.daily/stig-check

# Log monitoring
sudo dnf install -y logwatch
sudo systemctl enable --now logwatch.timer

# Security advisories
sudo dnf install -y yum-plugin-security
sudo dnf updateinfo list security
```

### 5. Maintenance

```bash
# Regular tasks:
# 1. Weekly: Review audit logs
sudo aureport -au -i

# 2. Monthly: Update system
sudo dnf update --security

# 3. Quarterly: Re-run compliance scan
sudo oscap xccdf eval --profile stig ...

# 4. Annually: Review and update policies
# - Review DISA STIG updates
# - Update local documentation
# - Retrain staff if needed
```

### 6. Exception Handling

```bash
# Document exceptions properly
cat > /root/STIG-exceptions.txt << 'EOF'
STIG Exception Documentation

Exception ID: EX-001
Rule: RHEL-08-010370
Description: SSH X11 forwarding required
Business Justification: Required for remote graphical applications
Compensating Controls: VPN required, limited to specific users
Approved by: [Name/Title]
Date: [Date]
Review date: [Date + 1 year]
EOF
```

---

## Additional Resources

### Official Documentation
- DISA STIGs: https://public.cyber.mil/stigs/
- Red Hat Security Guide: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/
- OpenSCAP Project: https://www.open-scap.org/
- SCAP Security Guide: https://github.com/ComplianceAsCode/content

### Tools
- OpenSCAP: https://www.open-scap.org/
- Ansible Lockdown: https://github.com/ansible-lockdown/
- AIDE: https://aide.github.io/
- USBGuard: https://usbguard.github.io/

### Commands Quick Reference
```bash
# Check STIG version
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel$(rpm -E %rhel)-ds.xml | grep -i version

# List all profiles
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel$(rpm -E %rhel)-ds.xml | grep -A 1 Profile

# Export configuration
oscap xccdf export-oval-variables --profile stig

# View rule details
oscap info --fetch-remote-resources ssg-rhel$(rpm -E %rhel)-ds.xml

# Generate guide
oscap xccdf generate guide --profile stig /usr/share/xml/scap/ssg/content/ssg-rhel$(rpm -E %rhel)-ds.xml > /tmp/stig-guide.html
```