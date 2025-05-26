# SSH Remote Server Setup

This project demonstrates how to provision a remote Linux server on AWS, configure secure SSH access using two distinct locally generated key pairs, and optionally enhance server security with tools like Fail2Ban. It is designed to build foundational skills in Linux server setup, SSH key management, and remote access hardening.

---
## Project Overview
> This Project is inspired by [roadmap.sh](https://roadmap.sh/projects/ssh-remote-server-setup)
- Cloud Provider
  - AWS EC2 (Amazon Linux, Free Tier `t2.micro`)

- Focus Areas:
  - SSH Configuration with Two Key Pairs
  - Server Provisioning and Firewall Configuration
  - SSH Aliasing for Simplified Access
  - Brute-Force Protection with Fail2Ban

---
## Local Machine Configurations (Client)
> Before connecting to the remote server, ensure the local Linux machine is properly configured. 
### Install & Enable SSH (*if not already*)
```bash
# install latest available version of packages & installs the OpenSSH server 
sudo apt update && sudo apt install openssh-server 
# start SSH service immediately & ensure SSH starts automatically on boot
sudo sysstemctl start ssh && sudo systemctl enable ssh
```
### Configure Firewall (UFW)
If it is not installed yet, install latest available version of packages & install UFW.
```bash
sudo apt update && sudo apt install ufw
```
Ensure baseline behavior of your firewall, by denying all unsolicited incoming traffic by default and allowing all outgoing traffic.
Together, this setup implements a default-deny posture, which is a core principle of network security.
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
Allow SSH & Enable UFW.
```bash
sudo ufw allow ssh
sudo ufw enable
```
This guarantees that only allowed services (life SSH) are reachable - while everything else remains secure and closed on your local machine.

### Optional: Enable Logging & Check Status
<details> 
  <summary>Expand here...</summary>
  
  Enable Logging:
  ```bash
  sudo ufw logging on
  ```
  Logs will go to `/var/log/ufw.log`. This will allow you to monitor firewall activity. 

  Check Status:
  ```bash
  sudo ufw status verbose
  ```
</details>

---

## Provisioning AWS EC2 Instance

### Launch the Server
  - Launch an EC2 instance using Amazon Linux 2023 AMI (`t2.micro`).
  - Allow only port 22 (SSH) via Security Group.
  - Generate and download an AWS Key Pair (`aws_key.pem`) for the initial login. 
### Connect to the EC2 Instance
```bash
# ensure initial key pair downloaded from AWS has restricted permissions
chmod 400 aws.key.pem
# then connect to the newly created EC2 instance 
ssh -i ~/.ssh/aws.key.pem ec2-user@<your-ec2-ip>
```
---
## SSH Key-Based Authentication
### Generate Two Key Pairs (Locally)
```bash
ssh-keygen -t ed25519 -f ~/.ssh/my_first_key -C "first-key"
ssh-keygen -t ed25519 -f ~/.ssh/my_second_key -C "second-key"
```
### Add Public Keys to EC2 Instance
On the local machine, copy each (`.pub`) key and append them to the remote server. 
> Inside the EC2 Instance (*after logging in*)
```bash
mkdir -p ~/.ssh    # create directory if it does not exist
nano ~/.ssh/authorized_keys    # paste both keys on sepearate lines
```
Then, set proper file permissions.
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```
Now disconnect from the server, and test new key-based logins from local machine.
```bash
ssh -i ~/.ssh/my_first_key ec2-user@<EC2-Public-IP>
ssh -i ~/.ssh/my_second_key ec2-user@<EC2-Public-IP>
```
#### SSH Config for Aliases (Locally)

Update or create `~/.ssh/config`:

```sshconfig
Host roadmapsh-test-server
    HostName <your-ec2-ip>
    User ec2-user
    IdentityFile ~/.ssh/my_first_key
    IdentityFile ~/.ssh/my_second_key
```

Now connect with:

```bash
ssh roadmapsh-test-server
```

---
## Install Fail2Ban (on EC2)
Update system and install:
```bash
sudo yum update -y
sudo yum install fail2ban -y
```
Create local config:
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Edit `/etc/fail2ban/jail.local`:
- Apply global settings:
```ini
[DEFAULT] 
# duration an IP is banned 
bantime = 3600    # 1 hour (3600 seconds)

# time window to observe failures
findtime = 604800    # 1 week (604800 seconds)

# number of failures allowed before banning
maxretry = 3

# email address to send alerts
destemail = <Your_Email_Address>

# ignore your own IP address
ignoreip = <Your_IP_Address>
```
- Configure SSH service:
```ini
[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = /var/log/secure  # for Amazon Linux
maxretry = 3
```
Restart the service (Fail2Ban) to apply changes:
```bash
sudo systemctl restart fail2ban
```
### Optional: Check Logs & Verify Status
<details> 
  <summary>Expand here...</summary>
  
  Tail logs for debug/troubleshooting:
  ```bash
  sudo less /var/log/fail2ban.log
  ``` 

  Check Status of a specific jail (e.g., SSH):
  ```bash
  sudo fail2ban-client status sshd
  ```
</details>



