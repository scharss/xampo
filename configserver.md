# 🌐 Xampo Server Configuration - Domain & HTTPS Setup

This guide explains how to configure a domain name and HTTPS SSL certificates for your Xampo application running on a VPS.

## 🎯 Objective

Transform your Xampo application from `http://your-vps-ip:8080` to `https://yourdomain.com` with automatic SSL certificates and professional security configuration.

## 📋 Prerequisites

- ✅ Xampo already running on your VPS (`docker-compose up -d`)
- ✅ A registered domain name (e.g., `mydomain.com`)
- ✅ Domain DNS pointing to your VPS IP address
- ✅ VPS with Ubuntu/Debian and sudo access
- ✅ Ports 80 and 443 available

## 🚀 Step-by-Step Configuration

### Step 1: Prepare the Server

**Update system and install required packages:**
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Nginx and Certbot for SSL
sudo apt install nginx python3-certbot-nginx -y

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Step 2: Configure Firewall

**Set up UFW firewall for security:**
```bash
# Allow SSH (CRITICAL - don't lock yourself out!)
sudo ufw allow OpenSSH

# Allow HTTP and HTTPS traffic
sudo ufw allow 'Nginx Full'

# Enable firewall
sudo ufw enable
```

### Step 3: Secure Xampo Configuration

**Modify your Xampo docker-compose.yml for security:**

1. **Navigate to your Xampo directory:**
   ```bash
   cd /path/to/xampo
   ```

2. **Edit docker-compose.yml:**
   ```bash
   sudo nano docker-compose.yml
   ```

3. **Change the web service ports to localhost only:**
   ```yaml
   # BEFORE (INSECURE):
   ports:
     - "8080:80"
   
   # AFTER (SECURE):
   ports:
     - "127.0.0.1:8080:80"
   ```

4. **Apply changes:**
   ```bash
   docker-compose down
   docker-compose up -d
   ```

### Step 4: Create Nginx Configuration

**Create Nginx configuration for your domain:**
```bash
# Create configuration file (replace mydomain.com with your domain)
sudo nano /etc/nginx/sites-available/mydomain.com
```

**Add this configuration:**
```nginx
# HTTP server block (will be upgraded to HTTPS by Certbot)
server {
    listen 80;
    server_name mydomain.com www.mydomain.com;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    
    # Root location - proxy to Xampo
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Handle large file uploads
        client_max_body_size 128M;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # phpMyAdmin location (secure access)
    location /phpmyadmin/ {
        proxy_pass http://127.0.0.1:8081/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Additional security for phpMyAdmin
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
    
    # Deny access to sensitive files
    location ~ /\.ht {
        deny all;
    }
    
    location ~ /\.env {
        deny all;
    }
}
```

### Step 5: Enable Site Configuration

**Activate the configuration:**
```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/mydomain.com /etc/nginx/sites-enabled/

# Remove default configuration
sudo rm -f /etc/nginx/sites-enabled/default

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Step 6: Secure phpMyAdmin Access (Optional)

**Create password protection for phpMyAdmin:**
```bash
# Install apache2-utils for htpasswd
sudo apt install apache2-utils -y

# Create password file (replace 'admin' with your preferred username)
sudo htpasswd -c /etc/nginx/.htpasswd admin

# Enter a strong password when prompted
```

### Step 7: Configure SSL with Let's Encrypt

**Obtain and configure SSL certificates:**
```bash
# Run Certbot (replace mydomain.com with your domain)
sudo certbot --nginx -d mydomain.com -d www.mydomain.com

# Follow the prompts:
# 1. Enter email address
# 2. Agree to terms (A)
# 3. Choose whether to share email (Y/N)
# 4. Select option 2 (Redirect HTTP to HTTPS)
```

### Step 8: Verify Configuration

**Test your setup:**
```bash
# Check Nginx status
sudo systemctl status nginx

# Check Xampo containers
docker-compose ps

# Test SSL configuration
sudo certbot certificates

# Check automatic renewal
sudo certbot renew --dry-run
```

## 🔒 Security Best Practices Applied

### ✅ **Network Security:**
- Firewall configured (UFW)
- Xampo bound to localhost only
- Sensitive ports not exposed

### ✅ **SSL/TLS Security:**
- Automatic SSL certificates
- HTTP to HTTPS redirection
- Modern SSL configuration

### ✅ **Application Security:**
- Security headers configured
- phpMyAdmin password protected
- Sensitive files blocked
- Large file upload limits

### ✅ **Maintenance:**
- Automatic SSL renewal
- Easy configuration updates
- Professional logging

## 🌐 Final URLs

After completing this setup, your application will be available at:

- **Main Application:** `https://mydomain.com`
- **phpMyAdmin:** `https://mydomain.com/phpmyadmin/`
- **SSL Grade:** A+ (verifiable at ssllabs.com)

## 🔧 Troubleshooting

### **Common Issues:**

**1. Domain not accessible:**
```bash
# Check DNS propagation
dig mydomain.com

# Verify Nginx is running
sudo systemctl status nginx

# Check error logs
sudo tail -f /var/log/nginx/error.log
```

**2. SSL certificate issues:**
```bash
# Check certificate status
sudo certbot certificates

# Renew certificates manually
sudo certbot renew

# Check SSL configuration
sudo nginx -t
```

**3. phpMyAdmin not accessible:**
```bash
# Check Xampo containers
docker-compose ps

# Verify phpMyAdmin container
docker-compose logs phpmyadmin
```

## 📊 Performance Optimizations

**Additional Nginx optimizations you can add:**
```nginx
# Add to server block for better performance
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# Enable gzip compression
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;
```

## 🔄 Updates and Maintenance

**Regular maintenance commands:**
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Check SSL certificate renewal
sudo certbot renew

# Restart services if needed
sudo systemctl restart nginx
docker-compose restart
```

## 🎯 Success Verification

Your setup is complete when:
- ✅ `https://mydomain.com` loads your Xampo application
- ✅ HTTP automatically redirects to HTTPS
- ✅ SSL certificate shows as valid (green lock)
- ✅ phpMyAdmin accessible with authentication
- ✅ All containers running properly

---

**🎉 Congratulations! Your Xampo application is now running professionally with domain name and SSL security!**

*For advanced configurations or multiple applications, refer to the enterprise documentation.* 