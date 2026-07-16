N8N SELF-HOSTING GUIDE
Complete setup guide for a production-ready n8n server.

WHAT YOU'LL BUILD:

A complete, production-ready n8n server with:
- Secure HTTPS access via Caddy
- Automatic startup if your VM reboots
- Weekly automatic updates
- Backups
- Works with domains and subdomains
- Optimized for 1GB RAM (2GB recommended)

PREREQUISITES:

Before starting:
- An Azure account (free credits available)
- About 2 hours of time
- A domain or subdomain ready

STEP 1: SET UP YOUR AZURE VIRTUAL MACHINE

1.1 Create an Azure Account
1. Go to [portal.azure.com](https://portal.azure.com)
2. Click "Start free" and follow the sign-up process
3. You'll get free credits

1.2 Create Your VM
1. In Azure Portal, search for "Virtual machines" and click "Create > Azure virtual machine"
2. Fill in these exact settings:
   Subscription: Your subscription
   Resource group: Create new → name it "n8n-rg"
   Virtual machine name: n8n-vm
   Region: Choose closest to you (e.g., East US)
   Image: Ubuntu 24.04 LTS
   Size: Standard_B1s (1 vCPU, 1GB RAM)
   Username: azureuser
   Authentication: Password (create strong password and save it)
   Inbound ports: HTTP (80), HTTPS (443), SSH (22)

3. Click "Review + create", then "Create"
4. After creation, go to Networking tab → Add inbound rule
   - Restrict SSH (port 22) to your IP only
   - Find your IP at whatsmyip.com
   - Protocol: TCP, Destination port: 22

1.3 Connect to Your VM
1. Copy your VM's public IP address from Azure Portal
2. Open Terminal (Mac/Linux) or PowerShell (Windows)
3. Connect: ssh azureuser@your-vm-ip
4. Type "yes" when asked about fingerprint
5. Enter your password

You are now inside your VM. The prompt should show: azureuser@n8n-vm:~$

STEP 2: INSTALL REQUIRED SOFTWARE

Copy and paste each command one at a time. Press Enter after each.

2.1 Update System
```bash
sudo apt update && sudo apt upgrade -y

2.2 Install Docker 
sudo apt install -y ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /usr/share/keyrings/docker.asc >/dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

2.3 Add Your User to Docker Group
sudo usermod -aG docker $USER

IMPORTANT: Log out and back in for changes to take effect:
exit
Then reconnect: ssh azureuser@your-vm-ip

2.4 Verify Docker Installation 
docker --version
docker compose version

You should see version numbers, not errors.

STEP 3: CREATE YOUR PROJECT FOLDER STRUCTURE

mkdir -p ~/n8n
cd ~/n8n

STEP 4: CREATE CONFIGURATION FILES

4.1 Create Environment File (.env)
This securely stores your passwords and keys.

nano .env

Copy and paste the .env file from your workspace (with your values).
Replace:
- DOMAIN with your actual domain
- N8N_PASSWORD with a strong password
- N8N_ENCRYPTION_KEY (generate: openssl rand -hex 32)
- ACME_EMAIL with your email

Save: Ctrl+X, Y, Enter

5.2 Create Docker Compose File

nano docker-compose.yml
Copy and paste the docker-compose.yml file from your workspace.
Save: Ctrl+X, Y, Enter

5.3 Create Caddyfile (for HTTPS)

nano Caddyfile
Copy and paste the Caddyfile from your workspace.
Save: Ctrl+X, Y, Enter

STEP 6: CREATE AUTOMATIC BACKUP SYSTEM

6.1 Create Backup Script 

nano backup.sh

Copy and paste the backup.sh script from your workspace.
Make it executable: chmod +x backup.sh

6.2 Schedule Daily Backups with Cron

crontab -e

If asked, choose nano as editor.
Add at the bottom:
0 2 * * * /home/azureuser/n8n/backup.sh >> /home/azureuser/n8n/backups/backup.log 2>&1

Save: Ctrl+X, Y, Enter
This runs backups daily at 2 AM.

STEP 6: CREATE AUTO-STARTUP SERVICE

7.1 Create Systemd Service

sudo nano /etc/systemd/system/n8n.service

Copy and paste the systemd service file from your workspace.
Save: Ctrl+X, Y, Enter

7.2 Enable the Service

sudo systemctl daemon-reload
sudo systemctl enable n8n.service
sudo systemctl start n8n.service

STEP 7: LAUNCH EVERYTHING

cd ~/n8n
docker compose up -d

Wait 2-3 minutes for all containers to start.

STEP 9: VERIFY EVERYTHING WORKS

9.1 Check All Containers Are Running

docker ps

You should see 4 containers (portainer, n8n, caddy, watchtower) all showing "Up" status.

9.2 Check n8n Logs

docker logs n8n

Look for any error messages. It should show the application started.

9.3 Check Caddy SSL Certificate
docker logs caddy

Look for "successfully obtained certificate" messages.

STEP 10: ACCESS YOUR N8N

1. Open your browser
2. Go to: https://your-domain
3. First visit may show SSL warning—click "Advanced" then "Proceed"
4. Login with your credentials from .env file

Congratulations! Your n8n instance is live.

FILE STRUCTURE

Your data is stored at:
- Workflows & credentials: ~/n8n/n8n_data/
- Backups: ~/n8n/backups/
- SSL certificates: ~/n8n/caddy_data/

DAILY MAINTENANCE COMMANDS

Check if everything is running:
cd ~/n8n && docker ps

View n8n logs:
docker logs n8n

View Caddy logs (for SSL issues):
docker logs caddy

Restart everything:
cd ~/n8n && docker compose restart

Stop everything:
cd ~/n8n && docker compose down

Start everything:
cd ~/n8n && docker compose up -d

Manually run a backup:
cd ~/n8n && ./backup.sh

RESTORING FROM BACKUP

If something goes wrong:
1. Stop n8n: cd ~/n8n && docker stop n8n
2. Restore: rm -rf n8n_data && tar -xzf backups/n8n-backup-FILENAME.tar.gz
3. Restart: docker start n8n

TROUBLESHOOTING

Cannot connect to https://your-domain:
- Wait 5 minutes for DNS propagation
- Verify your domain is pointing to your VM's public IP
- Check your VM's public IP: curl ifconfig.me

SSL Certificate Not Working:
- Check logs: docker logs caddy
- Ensure ports 80 and 443 are open in Azure

Containers Not Starting:
- Check logs: docker logs container-name

Out of Memory:
If n8n crashes with 1GB RAM, add swap:
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

IMPORTANT NOTES

1. Never change N8N_ENCRYPTION_KEY after starting—you'll lose access to credentials
2. Backups run automatically daily at 2 AM
3. Updates happen weekly (Watchtower)
4. Your VM will auto-start n8n if rebooted (systemd service)

SUCCESS CHECKLIST

[ ] Created Azure VM
[ ] Installed Docker
[ ] Set up HTTPS with Caddy
[ ] Created auto-backups
[ ] Enabled auto-startup
[ ] Added weekly updates
[ ] Accessed n8n via browser
[ ] Accessed Portainer dashboard

Your n8n is now running securely in production.
Go to https://your-domain and start creating workflows.