# Walkthrough for Securing Nextcloud on CasaOS

This guide details the complete process for installing Nextcloud on CasaOS, integrating it with Nginx Proxy Manager, and securing it with a Cloudflare Tunnel and optimal HTTPS settings.

---

### **1. Requirements**
* A server running a Linux distribution (or Windows with WSL2).
* Docker installed and running.
* A registered domain name managed through Cloudflare.

---

### **2. Install CasaOS**
1.  Open a terminal (use the WSL terminal if on Windows).
2.  Run the official installation script:
    ```bash
    wget -qO- https://get.casaos.io | sudo bash
    ```
3.  Once complete, access your CasaOS dashboard using the server's local IP address provided in the terminal.

---

### **3. Install Core Applications**
For each of the following, go to the CasaOS **App Store** and search for the application by name.

1.  **Nextcloud**: Install the version provided by **BigBearCasaOS**. This version includes the necessary SMB client.
2.  **Nginx Proxy Manager**: Install the official version.
3.  **Cloudflared**: Install the `cloudflared` app provided by **Cloudflare Inc**.

---

### **4. Configure Cloudflared Tunnel**
1.  Open the `cloudflared` application from the CasaOS dashboard.
2.  Follow the UI instructions to log in to your Cloudflare account and create a new tunnel.
3.  When defining the **Public Hostname** for your Nextcloud service (e.g., `nextcloud.yourdomain.com`), configure it to point to your **Nginx Proxy Manager** instance:
    * **Service Type**: `HTTP`
    * **URL**: `http://<NPM-Container-IP>:<NPM-Port>` (e.g., `http://172.17.0.4:80`). You can find the IP in the Nginx Proxy Manager app settings within CasaOS.
    * Normally the port 80 and 443 for web HTTP and HTTPS, and for Nginx Proxy Manager panel dashboard port is 81.

---

### **5. Setup Nginx Proxy Manager**
1.  Open Nginx Proxy Manager. The default credentials are:
    * **Email**: `admin@example.com`
    * **Password**: `changeme`
2.  Change the default credentials immediately upon first login.
3.  Navigate to **Hosts -> Proxy Hosts** and click **Add Proxy Host**.

#### **Tab: Details**
* **Domain Names**: Your full Nextcloud domain (e.g., `nextcloud.yourdomain.com`).
* **Scheme**: `http`
* **Forward Hostname / IP**: The IP address of your network.
* **Forward Port**: The port for the Nextcloud container, which is `7580` from default installation config.
* **Enable** `Websocket Support`.

#### **Tab: Advanced**
* Paste the following custom Nginx configuration. This adds security headers and ensures the correct client information is passed to Nextcloud.
    ```nginx
    # Security Headers
    add_header Strict-Transport-Security "max-age=15552000; includeSubDomains; preload" always;
    add_header Content-Security-Policy "upgrade-insecure-requests";

    # Proxy Headers to pass correct client IP and protocol info
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_set_header Host $host;

    # Cloudflare Specific Headers
    proxy_set_header CF-Connecting-IP $remote_addr;
    proxy_set_header CF-Visitor "{\"scheme\":\"https\"}";

    # Redirect any internal HTTP requests to HTTPS
    proxy_redirect http:// https://;
    ```
4.  Click **Save**.

---

### **6. Setup Nextcloud `config.php`**
This is the most critical part for resolving the "insecure access" errors.

1.  Open the CasaOS **Files** app.
2.  Navigate to and edit the following file: `/DATA/AppData/big-bear-nextcloud/html/config/config.php`
3.  Carefully add or modify the following parameters within the `$CONFIG` array:

    ```php
    <?php
    $CONFIG = array (
      // ... other existing config lines ...

      // In 'trusted_domains', add your public domain.
      'trusted_domains' => [
        'localhost',
        'nextcloud.yourdomain.com',
      ],

      // Add IPs of all proxies. A robust list includes the NPM container, the Docker host, and the server's LAN IP.
      'trusted_proxies' => [
        '172.17.0.x', // Your Nginx Proxy Manager Container IP
        '172.18.0.1', // Your Docker Host IP (typically the gateway) from nextcloud nextwork
        '192.168.x.x', // Your server's LAN IP
      ],

      // Define which headers to check for the original client IP.
      'forwarded_for_headers' => [
          'HTTP_X_FORWARDED_FOR',
          'HTTP_CF_CONNECTING_IP'
      ],
      
      // -- The following settings force Nextcloud to generate correct HTTPS URLs --
      'overwritehost' => 'nextcloud.yourdomain.com',
      'overwrite.cli.url' => 'https://nextcloud.yourdomain.com',
      'overwritewebroot' => '/',
      'cookies_samesite' => 'None', // Use 'Lax' or 'None', 'None' is often better for complex proxy setups.

      // -- Optional but Recommended Settings --
      'default_phone_region' => 'ID', // Your country code (e.g., US, GB, ID)
      'log_rotate_size' => 104857600, // 100 MB log rotation
      'maintenance_window_start' => 1, // Start maintenance tasks at 1 AM

      // ... other existing config lines ...
    );
    ```
4.  Save the file.

---

### **7. Final Nextcloud Environment Variable**
This step provides a direct override to ensure Nextcloud knows it's behind an HTTPS proxy.

1.  On the CasaOS dashboard, find your Nextcloud app.
2.  Click the three-dots icon and select **Settings**.
3.  Scroll down and add a new **Environment Variable**:
    * **Name**: `OVERWRITEPROTOCOL`
    * **Value**: `https`
4.  Click **Save**. The container will restart.

---

### **8. Configure Cloudflare SSL/TLS**
For maximum security, configure your Cloudflare domain settings as follows:

1.  Log in to Cloudflare and select your domain.
2.  Navigate to the **SSL/TLS** section.
3.  On the **Overview** tab, set the SSL/TLS encryption mode to **Full (strict)**.
4.  Navigate to the **Edge Certificates** sub-menu:
    * Enable **Always Use HTTPS**.
    * Enable **HTTP Strict Transport Security (HSTS)** and configure it:
        * **Max Age**: 6 months or more.
        * **Enable** `Apply HSTS policy to subdomains`.
        * **Enable** `Preload`.
        * **Enable** `No-Sniff Header`.
    * Enable **Opportunistic Encryption**.
    * Enable **TLS 1.3**.
    * Enable **Automatic HTTPS Rewrites**.

Your Nextcloud instance should now be fully accessible via HTTPS with all security warnings resolved. Congratulations on a successful and secure setup!
