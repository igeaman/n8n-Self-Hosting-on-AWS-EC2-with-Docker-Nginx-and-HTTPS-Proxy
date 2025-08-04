# How to Self-Host n8n on AWS EC2 with Docker, Nginx, and HTTPS

This tutorial guides you through setting up the n8n community version on an Amazon EC2 instance. We will use Docker to manage the n8n application and a PostgreSQL database, Nginx as a reverse proxy, and Certbot to secure the connection with an SSL certificate from Let's Encrypt.

This guide assumes you have a registered domain name and an AWS account.

### Step 1: Launch an AWS EC2 Instance

This phase creates the virtual server where n8n will run.

1.  **Log in to the AWS Console** and navigate to the EC2 dashboard.
2.  Click **"Launch instances"**.
3.  **Name** your instance (e.g., `n8n-server`).
4.  Choose an **Ubuntu Server** AMI (e.g., Ubuntu Server 22.04 LTS). Select an instance type like `t2.micro` or `t3.micro` which are free-tier eligible.
5.  **Create a key pair** (e.g., `n8n-key`). This is a `.pem` file you will use to connect to your server. Download and save it securely.
6.  **Configure Security Group**: Create a new security group with the following inbound rules:
    * **SSH (Port 22)**: Set the source to "My IP" for security.
    * **HTTP (Port 80)**: Set the source to "Anywhere".
    * **HTTPS (Port 443)**: Set the source to "Anywhere".
7.  Click **"Launch instance"**.

### Step 2: Set Up DNS for Your Domain

You need to point your domain name to the public IP address of your EC2 instance.

1.  In your domain registrar's DNS management console, create a new **`A` record**.
2.  Set the **Host/Name** to `n8n` (or your desired subdomain).
3.  Set the **Value** to the public IPv4 address of your EC2 instance. This IP can be found in the EC2 dashboard.

### Step 3: Connect to EC2 and Install Docker

Now, you will connect to your server and install the necessary software.

1.  Open a terminal and use your downloaded key pair to connect via SSH. Replace the file path and IP address with your own.
    ```bash
    ssh -i /path/to/n8n-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
    ```
2.  Install Docker and Docker Compose:
    ```bash
    sudo apt update
    sudo apt install docker.io -y
    sudo apt install docker-compose -y
    sudo usermod -aG docker ubuntu
    newgrp docker
    ```

### Step 4: Deploy n8n with Docker Compose

This step uses Docker Compose to run n8n and its database.

1.  Create a project directory and a `docker-compose.yml` file:
    ```bash
    mkdir n8n-project && cd n8n-project
    nano docker-compose.yml
    ```
2.  Paste the following configuration into the file. Replace `n8n.yourdomain.com` with your actual domain.
    ```yaml
    version: '3.7'
    services:
      n8n:
        image: n8nio/n8n:latest
        restart: always
        ports:
          - '5678:5678'
        environment:
          - DB_TYPE=postgresdb
          - DB_POSTGRESDB_HOST=db
          - DB_POSTGRESDB_DATABASE=n8n
          - DB_POSTGRESDB_USER=n8n
          - DB_POSTGRESDB_PASSWORD=n8npass
          - N8N_HOST=n8n.yourdomain.com
          - N8N_PROTOCOL=http
          - WEBHOOK_URL=https://n8n.topperkid.com
          - N8N_EDITOR_BASE_URL=https://n8n.topperkid.com/
        volumes:
          - n8n_data:/home/node/.n8n
      db:
        image: postgres:14
        restart: always
        environment:
          - POSTGRES_USER=n8n
          - POSTGRES_PASSWORD=n8npass
          - POSTGRES_DB=n8n
        volumes:
          - postgres_data:/var/lib/postgresql/data
    
    volumes:
      n8n_data:
      postgres_data:
    ```
3.  Save and close the file (Ctrl+X, Y, Enter).
4.  Start the containers:
    ```bash
    docker-compose up -d
    ```

### Step 5: Configure Nginx and SSL with WebSocket Support

This is the most critical step for security and real-time functionality.

1.  Install Nginx and Certbot:
    ```bash
    sudo apt install nginx certbot python3-certbot-nginx -y
    ```
2.  Create an Nginx configuration file for your domain:
    ```bash
    sudo nano /etc/nginx/sites-available/n8n.conf
    ```
3.  Add this basic configuration, replacing `n8n.yourdomain.com` with your domain. This sets up a temporary HTTP listener for Certbot.
    ```nginx
    server {
        listen 80;
        server_name n8n.yourdomain.com;
    
        location / {
            proxy_pass http://localhost:5678;
        }
    }
    ```
4.  Enable the Nginx configuration, then test and reload Nginx:
    ```bash
    sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
    sudo nginx -t
    sudo systemctl reload nginx
    ```
5.  Run Certbot to get an SSL certificate. This will automatically update your Nginx configuration.
    ```bash
    sudo certbot --nginx -d n8n.yourdomain.com
    ```
    Follow the prompts. Certbot will create a new configuration that handles HTTPS and redirects HTTP traffic.
6.  **Add WebSocket Support**: Certbot's automatic configuration is great, but it doesn't include the necessary headers for n8n's WebSockets. You must manually add them.
    * Open your Nginx configuration file again:
        ```bash
        sudo nano /etc/nginx/sites-available/n8n.conf
        ```
    * Find the `server` block that listens on `443 ssl`. Inside its `location /` block, add the following three lines to properly handle WebSocket traffic.
    
    ```nginx
    server {
        server_name n8n.yourdomain.com;
    
        location / {
            proxy_pass http://localhost:5678;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            # Crucial for WebSockets
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
        # ... other Certbot-managed SSL settings ...
    }
    ```
7.  Test and reload Nginx one last time to apply these changes:
    ```bash
    sudo nginx -t
    sudo systemctl reload nginx
    ```

### Step 6: Access n8n

You can now open a web browser and navigate to `https://n8n.yourdomain.com`. You will see the n8n welcome screen, where you can create your first owner account and start building workflows.
