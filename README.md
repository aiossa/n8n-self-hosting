# Self-Hosting SSL enabled N8N on a Linux Server

This guide provides step-by-step instructions to self-host [n8n](https://n8n.io), a free and open-source workflow automation tool, on a Linux server using Docker, Nginx, and Certbot for SSL with a custom domain name.

Youtube Video Explanation: 

## Step 1: Installing Docker

1. **Update the Package Index:**
   ```bash
   sudo apt update

2. **Install Docker:**
    ```bash
    sudo apt install docker.io

3.  **Start Docker:**
    ```bash
    sudo systemctl start docker

4. **Enable Docker to Start at Boot:**
    ```bash
    sudo systemctl enable docker
5. **Prepare folders**

# Create the .n8n directory in your home folder
mkdir -p ~/.n8n

# Set ownership to user ID 1000 (which matches the container user)
sudo chown 1000:1000 ~/.n8n

# Set proper permissions (read/write/execute for owner, read/execute for group)
chmod 755 ~/.n8n

## Step 2: Starting n8n in Docker

Run the following command to start n8n in Docker. Replace your-domain.com with your actual domain name:

    ```
    sudo docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -e N8N_HOST=n8n.n8n-courses.ru \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=https \
  -e WEBHOOK_URL=https://n8n.n8n-courses.ru/ \
  -v ~/.n8n:/home/node/.n8n \
  --user "1000:1000" \
  -it \
  n8nio/n8n

    ```
  
Or if you are using a subdomain, it should look like this:

    ```bash
    sudo docker run -d --restart unless-stopped -it \
    --name n8n \
    -p 5678:5678 \
    -e N8N_HOST="subdomain.your-domain.com" \
    -e WEBHOOK_TUNNEL_URL="https://subdomain.your-domain.com/" \
    -e N8N_EDITOR_BASE_URL="https://subdomain.your-domain.com/" \
    -e WEBHOOK_URL="https://subdomain.your-domain.com/" \
    -v ~/.n8n:/root/.n8n \
    n8nio/n8n
    ```


This command does the following:

- Downloads and runs the n8n Docker image.
- Exposes n8n on port 5678.
- Sets environment variables for the n8n host and webhook tunnel URL.
- Mounts the n8n data directory for persistent storage.
- After executing the command, n8n will be accessible on your-domain.com:5678.

## Step 3: Installing Nginx

Nginx is used as a reverse proxy to forward requests to n8n and handle SSL termination.

1. **Install Nginx:**
    ```bash
    sudo apt install nginx

## Step 4: Configuring Nginx

Configure Nginx to reverse proxy the n8n web interface:

1. **Create a New Nginx Configuration File:**
    ```bash
    sudo nano /etc/nginx/sites-available/n8n.conf

2. **Paste the Following Configuration:**
    ```bash
    server {
       listen 80;
       server_name n8n.n8n-courses.ru;
   
       location / {
           proxy_pass http://localhost:5678;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";
           proxy_set_header Host $host;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_read_timeout  3600s;
           proxy_send_timeout 3600s;
           proxy_buffering off;
       }
   }
    ```
    Replace your-domain.com with your actual domain.

3. **Enable the Configuration:**
    ```bash
    sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/

    If you see the error saying /etc/nginx/sites-enabled/ doesn't exist. Create it by running: sudo mkdir /etc/nginx/sites-enabled/

4. **Test the Nginx Configuration and Restart:**
    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```

## Step 5: Setting up SSL with Certbot

Certbot will obtain and install an SSL certificate from Let's Encrypt.

1. **Install Certbot and the Nginx Plugin:**
    ```bash
    sudo apt install certbot python3-certbot-nginx

2. **Obtain an SSL Certificate:**
    ```bash
    sudo certbot --nginx -d your-domain.com
    // If you have a subdomain then it will be subdomain.your-domain.com

Follow the on-screen instructions to complete the SSL setup.
Once completed, n8n will be accessible securely over HTTPS at your-domain.com.

IMPORTANT: Make sure you follow the above steps in order. Step 5 will modify your /etc/nginx/sites-available/n8n.conf file to something like this:
![image](https://github.com/user-attachments/assets/344187ec-5bcf-4d97-ad35-21b6562182e5)
 

## Important Notes
- Ensure your domain's DNS A record points to your server's IP address.
- Allow ports 80 (HTTP), 443 (HTTPS), and 5678 (n8n) in your server's firewall.
- Nginx handles SSL termination, so it forwards requests to the n8n instance over HTTP internally.

## Why Nginx and Certbot?

**Nginx:** It serves as a reverse proxy, forwarding client requests to n8n running on Docker. This setup enhances security, load balancing, and scalability.

**Certbot:** Certbot is a tool from the Electronic Frontier Foundation (EFF) that automates the process of obtaining and renewing SSL certificates from Let's Encrypt, a free and open Certificate Authority.

By using Nginx and Certbot, you ensure that your n8n instance is securely accessible over the internet with HTTPS.

## Troubleshooting

If you encounter issues with Nginx, check the logs located at /var/log/nginx/error.log for more details.
For Docker-related issues, ensure the Docker service is running: sudo systemctl status docker.
