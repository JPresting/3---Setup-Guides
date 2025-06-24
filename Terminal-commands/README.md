# Terminal Commands Reference Guide

Essential terminal commands for managing Linux servers - structured for beginners to advanced users.

## 1. First Steps - Basic Navigation

### Essential Commands You'll Use Every Day
```bash
# Show where you are
pwd

# List files and folders
ls
ls -la          # Detailed view with hidden files

# Navigate directories
cd foldername   # Enter folder
cd ..           # Go back one level
cd ~            # Go to home directory

# Create folders and files
mkdir foldername
touch filename.txt

# Edit files
nano filename.txt
```

### File Operations
```bash
# Copy, move, delete
cp file.txt backup.txt
mv file.txt newname.txt
rm file.txt
rm -rf foldername    # Delete folder (CAREFUL!)

# View file contents
cat file.txt         # Show entire file
tail -f file.txt     # Follow file updates (for logs)
```

---

## 2. System Preparation - Always Do This First

### Update Your System
```bash
# ALWAYS run these first on a new VM
sudo apt update     # Update package list
sudo apt upgrade    # Install updates
```

### Install Essential Tools
```bash
# Basic tools you'll need
sudo apt install curl wget git nano htop net-tools -y
```

### Check System Status
```bash
# See what's running
free -h             # Memory usage
df -h               # Disk space
top                 # Running processes
```

---

## 3. Understanding sudo

### When to Use sudo
```bash
# Installing software
sudo apt install package-name

# Editing system files
sudo nano /etc/nginx/nginx.conf

# Managing services
sudo systemctl start nginx

# Docker commands (unless user in docker group)
sudo docker ps
```

### When NOT to Use sudo
```bash
# Working in your home directory
cd ~/myproject
mkdir myfolder
nano myfile.txt
```

---

## 4. Network Basics - Make Sure VM Can Communicate

### Check What Ports Are Open
```bash
# See what's listening
sudo netstat -tlnp
sudo netstat -tlnp | grep :80    # Check specific port

# Alternative (modern command)
sudo ss -tlnp
```

### Test Network Connectivity
```bash
# Basic connectivity
ping google.com
ping -c 4 google.com  # Send only 4 packets

# Test websites
curl -I http://google.com
wget --spider http://google.com
```

### Check Internal Services
```bash
# Test if something is running locally
curl localhost:3000
curl localhost:80
```

---

## 5. Firewall Management

### Understanding the Flow
```
Internet → Oracle Cloud Security Lists → VM → iptables → Container
```

- **Oracle Cloud Security Lists**: Cloud firewall (external)
- **iptables**: VM firewall (internal)
- **Both must allow traffic** for it to work

### Check Current Firewall Rules
```bash
# See what's allowed through iptables
sudo iptables -L
sudo iptables -L INPUT -n -v  # Detailed view
```

### Open Ports (iptables)
```bash
# Allow specific port
sudo iptables -I INPUT -p tcp --dport 3000 -j ACCEPT

# Save rules so they persist after reboot
sudo mkdir -p /etc/iptables
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### Oracle Cloud Security Lists
**Must be done in Oracle Cloud Console:**
1. **Networking → Virtual Cloud Networks**
2. **Select your VCN → Security Lists**
3. **Add Ingress Rules** for ports you need

---

## 6. Service Management

### Control System Services
```bash
# Check if service is running
sudo systemctl status nginx

# Start/Stop services
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx

# Enable auto-start on boot
sudo systemctl enable nginx
```

### View Service Logs
```bash
# Real-time logs
sudo journalctl -u nginx -f

# Recent logs
sudo journalctl -u nginx --since "1 hour ago"
```

---

## 7. Installing Common Software

### Web Server (Nginx)
```bash
# Install
sudo apt install nginx -y

# Start and enable
sudo systemctl start nginx
sudo systemctl enable nginx

# Test configuration
sudo nginx -t

# Reload after config changes
sudo systemctl reload nginx
```

### Docker
```bash
# Install Docker
sudo apt install docker.io -y

# Start and enable
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group (avoid sudo)
sudo usermod -aG docker $USER
# Log out and back in for this to take effect
```

### SSL Certificates (Certbot)
```bash
# Install Certbot
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot

# Get certificate
sudo certbot certonly --nginx -d yourdomain.com
```

---

## 8. Docker Management

### Basic Container Operations
```bash
# See what's running
sudo docker ps
sudo docker ps -a     # Include stopped containers

# Run a container
sudo docker run -d --name myapp -p 3000:8080 image-name

# Stop/Start containers
sudo docker stop myapp
sudo docker start myapp

# View logs
sudo docker logs myapp
sudo docker logs -f myapp    # Follow logs
```

### Image Management
```bash
# Download images
sudo docker pull nginx
sudo docker pull ubuntu

# See downloaded images
sudo docker images

# Remove old images
sudo docker image prune -af
```

---

## 9. File and Configuration Management

### Nginx Configuration
```bash
# Main config file
sudo nano /etc/nginx/nginx.conf

# Create site config
sudo nano /etc/nginx/sites-available/mysite.conf

# Enable site
sudo ln -s /etc/nginx/sites-available/mysite.conf /etc/nginx/sites-enabled/

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

### Important Log Locations
```bash
# System logs
sudo tail -f /var/log/syslog

# Nginx logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Auth logs (login attempts)
sudo tail -f /var/log/auth.log
```

---

## 10. Automation with Cron

### Edit Cron Jobs
```bash
# User cron jobs
crontab -e

# System cron jobs
sudo crontab -e

# List current jobs
crontab -l
```

### Cron Schedule Examples
```bash
# Run every day at 2 AM
0 2 * * * /path/to/script.sh

# Run every Sunday at 4 AM  
0 4 * * 0 /path/to/script.sh

# Run every 15 minutes
*/15 * * * * /path/to/script.sh
```

---

## 11. Monitoring and Troubleshooting

### System Resource Usage
```bash
# Memory and CPU
free -h
top
htop              # Better interface

# Disk usage
df -h             # Overall disk space
du -sh folder/    # Specific folder size

# System uptime and load
uptime
```

### Process Management
```bash
# Find processes
ps aux | grep nginx
pgrep nginx

# Kill processes
kill PID
kill -9 PID       # Force kill
killall nginx
```

### Network Troubleshooting
```bash
# Test specific ports
telnet localhost 3000
nc -zv localhost 3000

# Check routing
ip route
ping 8.8.8.8      # Test external connectivity
```

---

## 12. Emergency Commands

### When Things Go Wrong
```bash
# Restart services
sudo systemctl restart nginx
sudo systemctl restart docker

# Restart containers
sudo docker restart container-name

# Check what's using a port
sudo lsof -i :3000
sudo netstat -tlnp | grep :3000

# System restart (last resort)
sudo reboot
```

### Quick Health Check
```bash
# Run these to check overall system health
free -h                    # Memory
df -h                      # Disk
sudo docker ps            # Containers
sudo systemctl status nginx  # Web server
sudo iptables -L INPUT -n -v  # Firewall
```

---

## 13. Advanced Topics

### Archive and Backup
```bash
# Create backups
tar -czf backup.tar.gz folder/
zip -r backup.zip folder/

# Extract backups
tar -xzf backup.tar.gz
unzip backup.zip
```

### Certificate Management
```bash
# List SSL certificates
sudo certbot certificates

# Renew certificates
sudo certbot renew
sudo certbot renew --dry-run  # Test renewal
```

---

## Quick Reference Card

### Daily Commands
```bash
sudo apt update && sudo apt upgrade  # Update system
sudo docker ps                       # Check containers
sudo systemctl status nginx         # Check web server
sudo tail -f /var/log/nginx/error.log  # Watch logs
df -h                               # Check disk space
```

### Emergency Fixes
```bash
sudo systemctl restart nginx        # Fix web server
sudo docker restart container      # Fix container
sudo reboot                        # Fix everything (nuclear option)
```

**Remember:** Start simple, work your way up. AI is very helpful here. As long as you provide it with the context of your pwd and already installed instances - you should be good.
