# Self-Hosting n8n on Your Subdomain Using AWS Lightsail VPS: A Detailed Guide

This document provides a comprehensive, step-by-step guide to setting up n8n on a sub-domain (e.g., `n8n.yourdomain.com`) using an AWS Lightsail Virtual Private Server (VPS). It incorporates all the best practices, troubleshooting steps, and configurations discussed previously.

**Goal:** To host n8n securely with persistent data, accessible via HTTPS on `n8n.yourdomain.com` (or your chosen subdomain).

---

### Prerequisites

Before you begin, ensure you have the following:

1.  **AWS Account:** An active AWS account.
2.  **Domain Name:** A registered domain (e.g., `yourdomain.com`) where you can manage DNS records.
3.  **Basic Linux/SSH Knowledge:** Familiarity with connecting to a Linux server via SSH and executing basic commands.
4.  **Recommended Lightsail Plan:** For stable n8n operation, a Lightsail instance with at least **2GB RAM** (e.g., the $10 USD/month plan) is highly recommended. 1GB RAM might be sufficient for very light use, but 512MB RAM is generally insufficient and can lead to performance issues and unresponsiveness.

---

### Step 1: Create an AWS Lightsail Instance

1.  **Log in to AWS Lightsail:** Go to [lightsail.aws.amazon.com](https://lightsail.aws.amazon.com/).
2.  **Create Instance:** Click **"Create instance"**.
3.  **Choose Instance Location:** Select a region geographically close to your target users or API endpoints for better performance (e.g., "Frankfurt", "Mumbai", or a region in the Americas).
4.  **Pick Instance Image (Blueprint):**
    * Go to the **"OS Only"** tab.
    * Select **"Ubuntu"** (preferably the latest LTS version, e.g., Ubuntu 22.04 LTS or 24.04 LTS).
5.  **Choose Instance Plan:**
    * Select a plan with **at least 2 GB RAM**. The $10 USD/month plan (2 GB RAM, 2 vCPUs, 60 GB SSD) is a good starting point for most startups.
6.  **Identify Your Instance:** Give your instance a memorable name, e.g., `n8n-server`.
7.  **Create SSH Key Pair:** If prompted, create a new SSH key pair and **download the `.pem` file immediately**. This file is crucial for connecting to your server via SSH. Store it securely.
8.  **Create Instance:** Click **"Create instance"**.
9.  **Attach a Static IP Address:**
    * Once your instance is created and running, go to the **"Networking"** tab in the Lightsail dashboard.
    * Click **"Create static IP"**.
    * Select the instance you just created and give the static IP a name (e.g., `n8n-server-static-ip`).
    * Click **"Create and attach"**.
    * **Note down this Static IP address.** This is the public IP that your subdomain will point to.

---

### Step 2: Set up DNS Records for Your Subdomain

This step tells the internet that your subdomain (`n8n.yourdomain.com`) points to your Lightsail server's IP address.

**Important:** If your main domain (`yourdomain.com`) is already active and managed by an external DNS provider (like Hostinger), you will only add a record for the **subdomain** to avoid affecting your existing website.

1.  **Go to Your DNS Provider's Control Panel:** Log in to your DNS provider's account (e.g., Hostinger, Cloudflare) and navigate to the DNS management/Zone Editor section for your domain (`yourdomain.com`).
2.  **Add an A Record:**
    * **Type:** `A`
    * **Name/Host/Subdomain:** Enter `n8n` (this creates `n8n.yourdomain.com`).
    * **Value/Points to:** Enter your Lightsail instance's **static IP address** (the one you noted down in Step 1.9).
    * **TTL (Time To Live):** You can leave this at the default value, or set it to a lower value (e.g., 300 seconds or 5 minutes) for faster propagation during initial setup.
3.  **Save the Record:** Confirm and save the new DNS record.
4.  **Wait for DNS Propagation:** DNS changes can take some time to propagate globally (usually a few minutes, but can be up to 24-48 hours). You can check the propagation status using tools like `https://dnschecker.org`. Do **NOT** proceed until `n8n.yourdomain.com` resolves to your Lightsail Static IP.

---

### Step 3: Connect to Your Lightsail Instance via SSH

You can use the Lightsail browser-based SSH client or your local terminal.

**Option A: Lightsail Browser-Based SSH (Easiest for quick access)**

1.  On the Lightsail dashboard, go to the **"Instances"** tab.
2.  Click on your instance name (e.g., `n8n-server`).
3.  Click the orange **"Connect using SSH"** button. A new browser window will open with a terminal.

**Option B: Local Terminal (Recommended for regular use)**

1.  **Set correct permissions for your `.pem` key:**
    ```bash
    chmod 400 /path/to/your-key.pem
    ```
    (Replace `/path/to/your-key.pem` with the actual path to your downloaded private key).
2.  **Connect via SSH:**
    ```bash
    ssh -i /path/to/your-key.pem ubuntu@YOUR_LIGHTSAIL_STATIC_IP
    ```
    * `ubuntu` is the default username for Ubuntu instances on Lightsail.
    * `YOUR_LIGHTSAIL_STATIC_IP` is your Lightsail instance's static IP address.

---

### Step 4: Prepare Your Lightsail Instance (Install Docker and Docker Compose)

This step installs Docker, which will run n8n in an isolated container, and Docker Compose, which helps manage the container.

1.  **Update System Packages:**
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
2.  **Install Docker Engine:**
    ```bash
    # Install packages to allow apt to use a repository over HTTPS
    sudo apt install ca-certificates curl gnupg -y

    # Create directory for Docker's GPG key
    sudo install -m 0755 -d /etc/apt/keyrings

    # Download and add Docker's official GPG key
    curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    # Set appropriate permissions for the GPG key
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    # Add the Docker repository to Apt sources
    echo \
      "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
      "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    # Update the apt package index with the new repository
    sudo apt update

    # Install Docker Engine, CLI, Containerd, and Docker Compose Plugin
    sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
    ```
3.  **Verify Docker Installation:**
    ```bash
    sudo docker run hello-world
    ```
    You should see a message indicating Docker is working.
4.  **Add `ubuntu` User to the `docker` Group:** This allows you to run Docker commands without `sudo`.
    ```bash
    sudo usermod -aG docker ubuntu
    exit # Log out of SSH
    ```
    **Reconnect to your Lightsail instance via SSH** after running `exit` for the changes to take effect.

---

### Step 5: Configure Nginx as a Reverse Proxy with SSL (Let's Encrypt)

This step sets up Nginx to serve `n8n.yourdomain.com` via HTTPS and forward requests to your n8n Docker container.

1.  **Install Nginx & Certbot:**
    ```bash
    sudo apt install nginx certbot python3-certbot-nginx -y
    ```
2.  **Configure Nginx for n8n:**
    * Open the Nginx configuration file for your subdomain:
        ```bash
        sudo nano /etc/nginx/sites-available/n8n.yourdomain.com
        ```
    * **Delete all existing content in the file.**
    * **Paste the following complete and correct configuration:**

        ```nginx
        # HTTP server block: Redirects all HTTP requests to HTTPS
        server {
            listen 80;
            listen [::]:80;
            server_name n8n.yourdomain.com;

            # Redirect all HTTP requests to HTTPS
            return 301 https://$host$request_uri;
        }

        # HTTPS server block: Serves n8n with SSL
        server {
            listen 443 ssl http2;
            listen [::]:443 ssl http2; # For IPv6 support if your server has it

            server_name n8n.yourdomain.com;

            # SSL configuration (Certbot paths)
            ssl_certificate /etc/letsencrypt/live/[n8n.yourdomain.com/fullchain.pem](https://n8n.yourdomain.com/fullchain.pem);
            ssl_certificate_key /etc/letsencrypt/live/[n8n.yourdomain.com/privkey.pem](https://n8n.yourdomain.com/privkey.pem);
            ssl_trusted_certificate /etc/letsencrypt/live/[n8n.yourdomain.com/chain.pem](https://n8n.yourdomain.com/chain.pem);

            # Recommended SSL settings (from Certbot)
            include /etc/letsencrypt/options-ssl-nginx.conf;
            ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

            # Proxy settings for n8n
            location / {
                proxy_pass http://localhost:5678; # This forwards traffic to n8n's Docker port
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;

                # WebSocket support (crucial for n8n's UI functionality)
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";

                # Proxy timeouts (important for n8n startup and long-running operations)
                proxy_read_timeout 300;
                proxy_connect_timeout 300;
                proxy_send_timeout 300;
            }

            # Optional: Increase client body size if you expect large file uploads
            client_max_body_size 50M;
        }
        ```
    * **Save and Exit `nano`:** Press `Ctrl + O`, then `Enter`, then `Ctrl + X`.
3.  **Create Symbolic Link to Enable Configuration:**
    ```bash
    sudo ln -s /etc/nginx/sites-available/n8n.yourdomain.com /etc/nginx/sites-enabled/
    ```
    * **If you get "File exists" error:** This means the symlink already exists. Remove the old one first:
        ```bash
        sudo rm /etc/nginx/sites-enabled/n8n.yourdomain.com
        ```
        Then, run the `sudo ln -s ...` command again.
4.  **Generate SSL Certificate with Certbot:**
    ```bash
    sudo certbot --nginx -d n8n.yourdomain.com
    ```
    * Follow the prompts (enter email, agree to terms, **select "2: Redirect"** for HTTP to HTTPS).
    * If Certbot complains about `ssl-dhparams.pem` or you get an error in the next step, generate it manually:
        ```bash
        sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
        ```
        (This can take a few minutes.) Then re-run `sudo certbot --nginx -d n8n.yourdomain.com`.
5.  **Test Nginx Configuration:**
    ```bash
    sudo nginx -t
    ```
    * **Expected Output:** "syntax is ok" and "test is successful".
    * **If errors occur:** Carefully review the Nginx configuration you pasted in Step 5.2. Make sure it's identical and that all paths (especially to `.pem` files) are correct. Re-run Certbot or generate `ssl_dhparams.pem` if indicated.
6.  **Restart Nginx:**
    ```bash
    sudo systemctl restart nginx
    ```

---

### Step 6: Configure Lightsail Firewall

This is crucial to allow external internet traffic to reach your Nginx server.

1.  **Go to Lightsail Dashboard:** In your web browser, navigate to the Lightsail console.
2.  **Go to "Instances":** Click on the **"Instances"** tab in the left-hand navigation.
3.  **Click Instance Name:** Click on the **name** of your `n8n-server` instance (e.g., `n8n-server`).
4.  **Go to "Networking" Tab:** On the instance details page, click the **"Networking"** tab.
5.  **Scroll to "Firewall":** Scroll down to the "Firewall" section.
6.  **Add/Verify Rules:** Ensure the following rules are present. If any are missing, click **"+ Add rule"** to add them.
    * **SSH:** Application `SSH`, Protocol `TCP`, Port `22`, Restricted to `Any IPv4 address`. (Usually default)
    * **HTTP:** Application `HTTP`, Protocol `TCP`, Port `80`, Restricted to `Any IPv4 address`.
    * **HTTPS:** Application `HTTPS`, Protocol `TCP`, Port `443`, Restricted to `Any IPv4 address`.
7.  **Save:** Click the orange **"Save"** button at the bottom.

---

### Step 7: Deploy n8n with Docker Compose

This step runs the n8n application as a Docker container with persistent storage.

1.  **Create n8n Directory & Navigate:**
    ```bash
    mkdir ~/n8n
    cd ~/n8n
    ```
2.  **Create `docker-compose.yml` File:**
    ```bash
    nano docker-compose.yml
    ```
3.  **Paste the following content. **CRITICAL: You MUST change all placeholder values as instructed below!**

    ```yaml
    version: '3.8'

    services:
      n8n:
        image: n8nio/n8n
        restart: unless-stopped
        ports:
          - '5678:5678' # Expose port 5678 for internal access by Nginx
        volumes:
          - n8n_data:/home/node/.n8n # Persistent storage for n8n data
        environment:
          # General settings for n8n (important for webhooks and UI links)
          - N8N_HOST=n8n.yourdomain.com # <--- CHANGE TO YOUR SUBDOMAIN (e.g., n8n.yourdomain.com)
          - N8N_PORT=5678
          - N8N_PROTOCOL=https
          - NODE_ENV=production
          - TZ=Asia/Karachi # <--- IMPORTANT: Adjust to your specific timezone (e.g., America/New_York, Europe/London, Asia/Kolkata etc.)

          # Webhook URLs (critical for external services to call n8n workflows)
          - WEBHOOK_URL=https://n8n.yourdomain.com/ # <--- CHANGE TO YOUR SUBDOMAIN
          - WEBHOOK_TUNNEL_URL=https://n8n.yourdomain.com/ # <--- CHANGE TO YOUR SUBDOMAIN

          # Editor Base URL (ensures correct internal links in the n8n UI)
          - N8N_EDITOR_BASE_URL=https://n8n.yourdomain.com/ # <--- CHANGE TO YOUR SUBDOMAIN

          # Security Settings (HIGHLY RECOMMENDED FOR PRODUCTION)
          - N8N_BASIC_AUTH_ACTIVE=true
          - N8N_BASIC_AUTH_USER=your_n8n_user # <--- CHANGE THIS! Choose a secure username
          - N8N_BASIC_AUTH_PASSWORD=your_strong_password # <--- CHANGE THIS! Choose a very strong password
          - N8N_ENCRYPTION_KEY=your_very_long_random_string_for_encryption # <--- CRITICAL: CHANGE THIS! Generate with 'openssl rand -hex 32'

          # Optional: Resolve the 'N8N_RUNNERS_ENABLED' warning
          - N8N_RUNNERS_ENABLED=true

          # Database (Optional, but recommended for production instead of default SQLite)
          # Uncomment and configure if you want to use an external PostgreSQL or MySQL database
          # - DB_TYPE=postgresdb
          # - DB_POSTGRESDB_HOST=your_postgres_host
          # - DB_POSTGRESDB_PORT=5432
          # - DB_POSTGRESDB_DATABASE=n8n
          # - DB_POSTGRESDB_USER=n8nuser
          # - DB_POSTGRESDB_PASSWORD=your_postgres_password

    volumes:
      n8n_data: # Define the Docker volume for persistent storage
    ```
    * **Generate `N8N_ENCRYPTION_KEY`:** Open another SSH session to your Lightsail instance (or temporarily disconnect and reconnect to avoid disturbing your current session) and run:
        ```bash
        openssl rand -hex 32
        ```
        Copy the resulting 64-character hexadecimal string and paste it as the value for `N8N_ENCRYPTION_KEY` in your `docker-compose.yml`. **Store this key securely! Losing it means losing access to encrypted credentials.**
    * **Save and Exit `nano`:** Press `Ctrl + O`, then `Enter`, then `Ctrl + X`.
4.  **Start n8n with Docker Compose:**
    ```bash
    docker compose up -d
    ```
5.  **Verify n8n is Running:**
    ```bash
    docker ps
    ```
    You should see your `n8n` container listed with a `Up` status.
6.  **Check n8n Container Logs (Optional, for debugging):**
    ```bash
    docker compose logs -f n8n
    ```
    (Press `Ctrl + C` to exit the log view.) Look for "n8n ready on 0.0.0.0, port 5678" to confirm it started successfully.

---

### Step 8: Access n8n in Your Browser

Open your web browser and navigate to:

`https://n8n.yourdomain.com` (or your chosen subdomain)

You should be prompted for the basic authentication username and password you set in the `docker-compose.yml`. After that, you'll reach the n8n sign-up page to create your first n8n user account.

---

### Post-Setup & Maintenance

* **Closing SSH Terminal:** You can safely close your SSH terminal. n8n will continue to run in the background thanks to `docker compose up -d` and `restart: unless-stopped`.
* **Managing n8n Container (from `~/n8n` directory):**
    * `docker compose stop n8n`: To stop the n8n service.
    * `docker compose start n8n`: To start the n8n service.
    * `docker compose restart n8n`: To restart the n8n service.
    * `docker compose down`: To stop and remove the n8n container (but your `n8n_data` volume with data will persist).
* **Deleting Old Instance:** Once your new n8n instance is confirmed to be fully functional and stable for a few days/weeks, you can delete your old Lightsail instance from the Lightsail Instances tab. Keep the snapshot for a recovery period.
* **Database Migration (Later):** If you started with SQLite, you can migrate to an external PostgreSQL database later when your needs grow. This involves exporting data, configuring n8n for PostgreSQL in `docker-compose.yml`, and importing the data.
* **Certbot Renewals:** Let's Encrypt certificates are valid for 90 days. Certbot usually sets up a cron job or systemd timer to automatically renew them. You can test this: `sudo certbot renew --dry-run`.
* **Monitoring:** Regularly check your Lightsail instance's metrics (CPU, RAM) in the dashboard to anticipate when you might need to scale up your plan.
* **Security:** Keep your system packages updated (`sudo apt update && sudo apt upgrade -y`). Use strong, unique passwords and consider enabling MFA for your AWS account.

Congratulations! You now have a self-hosted n8n instance running securely on your Lightsail VPS

### Need Help? 
Feel free to contact me at <a href="https://alimazhar.dev/" target="_blank">alimazhar.dev</a> if you are facing any issues 🙂
