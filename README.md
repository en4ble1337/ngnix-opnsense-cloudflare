# Complete Nginx Proxy Manager Setup with Cloudflare Origin Certificates and OPNsense

This comprehensive guide walks you through setting up Nginx Proxy Manager with Cloudflare Origin Certificates for secure internal and external access to your homelab services, using OPNsense as your firewall.

## Prerequisites

- **Docker and Docker Compose** installed on your server
- **Cloudflare account** with a domain configured
- **OPNsense firewall** managing your network
- **Basic understanding** of networking concepts

## Table of Contents

1. [Docker Compose Setup](#docker-compose-setup)
2. [Initial Nginx Proxy Manager Configuration](#initial-nginx-proxy-manager-configuration)
3. [Cloudflare Origin Certificate Generation](#cloudflare-origin-certificate-generation)
4. [Uploading Certificate to Nginx Proxy Manager](#uploading-certificate-to-nginx-proxy-manager)
5. [Cloudflare DNS Configuration](#cloudflare-dns-configuration)
6. [OPNsense Internal DNS Configuration](#opnsense-internal-dns-configuration)
7. [OPNsense Port Forwarding Setup](#opnsense-port-forwarding-setup)
8. [Creating Proxy Hosts](#creating-proxy-hosts)
9. [Testing and Verification](#testing-and-verification)
10. [Troubleshooting](#troubleshooting)

## Docker Compose Setup

### Step 1: Create Docker Compose File

Create a `docker-compose.yml` file with the official configuration:

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      # These ports are in format :
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '81:81' # Admin Web Port
      # Add any other Stream port you want to expose
      # - '21:21' # FTP

    #environment:
      # Uncomment this if you want to change the location of
      # the SQLite DB file within the container
      # DB_SQLITE_FILE: "/data/database.sqlite"

      # Uncomment this if IPv6 is not enabled on your host
      # DISABLE_IPV6: 'true'

    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

### Step 2: Deploy the Container

```bash
docker compose up -d
```

### Step 3: Verify Container Status

```bash
docker compose ps
```

## Initial Nginx Proxy Manager Configuration

### Step 1: Access Admin Interface

- Navigate to `http://YOUR_SERVER_IP:81`
- **Default credentials:**
  - Email: `admin@example.com`
  - Password: `changeme`

### Step 2: Initial Setup

1. Log in with default credentials
2. **Immediately change** your email and password when prompted
3. Complete the initial setup wizard

## Cloudflare Origin Certificate Generation

### Step 1: Access Cloudflare Dashboard

1. Log in to your Cloudflare account
2. Select your domain (e.g., `yourdomain.com`)
3. Navigate to **SSL/TLS** → **Origin Server**

### Step 2: Create Origin Certificate

1. Click **Create Certificate**
2. Choose **"Generate private key and CSR with Cloudflare"**
3. Select **RSA (2048)** as the private key type
4. In the **Hostnames** field, add:
   - `yourdomain.com` (your main domain)
   - `*.yourdomain.com` (wildcard for all subdomains)
5. Set **Certificate Validity** to **15 years** (maximum available)
6. Click **Create**

### Step 3: Download Certificate Files

1. Choose **PEM** as the key format
2. Copy the **Origin Certificate** content and save as `certificate.pem`
3. Copy the **Private Key** content and save as `privatekey.pem`
4. **⚠️ Important:** Save these files securely - you cannot retrieve the private key later

## Uploading Certificate to Nginx Proxy Manager

### Step 1: Access SSL Certificates

1. Go to Nginx Proxy Manager dashboard
2. Navigate to **SSL Certificates** in the left menu
3. Click **Add SSL Certificate**
4. Select **Custom** from the dropdown

### Step 2: Upload Certificate

1. **Certificate:** Paste the content from `certificate.pem`
2. **Certificate Key:** Paste the content from `privatekey.pem`
3. **Intermediate Certificate:** Leave blank
4. **Name:** Give it a descriptive name (e.g., "Cloudflare Origin - yourdomain.com")
5. Click **Save**

## Cloudflare DNS Configuration

### Step 1: Configure SSL/TLS Settings

1. In Cloudflare dashboard, go to **SSL/TLS** → **Overview**
2. Set encryption mode to **Full (strict)**
3. Go to **SSL/TLS** → **Edge Certificates**
4. Enable **Always Use HTTPS**
5. Set **Minimum TLS Version** to 1.2 or higher

### Step 2: Set Up DNS Records

For each service you want to expose:

1. Go to **DNS** → **Records**
2. Add **A record:**
   - **Type:** A
   - **Name:** Your subdomain (e.g., `chat`)
   - **Content:** Your public IP address
   - **Proxy status:** **Proxied** (orange cloud enabled)
   - **TTL:** Auto

## OPNsense Internal DNS Configuration

### Step 1: Access Unbound DNS Overrides

1. Log in to OPNsense web interface
2. Navigate to **Services** → **Unbound DNS** → **Overrides**

### Step 2: Add Host Override

1. In the **Host Overrides** section, click the **"+"** button
2. Configure the override:
   - **Enabled:** ✅ Check this box
   - **Host:** Your subdomain (e.g., `chat`)
   - **Domain:** Your domain (e.g., `yourdomain.com`)
   - **Type:** **A**
   - **IP Address:** Your Nginx Proxy Manager's internal IP (e.g., `192.168.1.100`)
   - **Description:** Optional description
3. Click **Save**
4. Apply changes when prompted

### Step 3: Verify DNS Resolution

Test from a local device:

```bash
nslookup chat.yourdomain.com
```

Should return your internal Nginx Proxy Manager IP address.

## OPNsense Port Forwarding Setup

### Step 1: Create NAT Port Forward Rules

Navigate to **Firewall** → **NAT** → **Port Forward**

### Step 2: Add HTTPS Rule (Port 443)

1. Click the **"+"** button to add a new rule
2. Configure the rule:
   - **Interface:** WAN
   - **TCP/IP Version:** IPv4
   - **Protocol:** TCP
   - **Source:** Any
   - **Destination:** WAN address
   - **Destination Port Range:** HTTPS (443)
   - **Redirect Target IP:** Your NPM internal IP (e.g., `192.168.1.100`)
   - **Redirect Target Port:** HTTPS (443)
   - **Filter rule association:** Add associated filter rule
   - **Description:** "NPM HTTPS Access"
3. Click **Save**

### Step 3: Add HTTP Rule (Port 80) - Optional

1. Create another rule with the same settings but:
   - **Destination Port Range:** HTTP (80)
   - **Redirect Target Port:** HTTP (80)
   - **Description:** "NPM HTTP Access"

### Step 4: Apply Changes

1. Click **Apply changes**
2. Optionally reset state table: **Firewall** → **Diagnostics** → **States** → **Reset State Table**

## Creating Proxy Hosts

### Step 1: Add New Proxy Host

1. In Nginx Proxy Manager, go to **Proxy Hosts**
2. Click **Add Proxy Host**

### Step 2: Configure Details Tab

1. **Domain Names:** `chat.yourdomain.com`
2. **Scheme:** `http` or `https` (depending on your backend service)
3. **Forward Hostname/IP:** Internal IP of your service (e.g., `192.168.1.200`)
4. **Forward Port:** Port your service runs on (e.g., `3000`)
5. **Cache Assets:** Enable if desired
6. **Block Common Exploits:** ✅ Enable
7. **Websockets Support:** ✅ Enable if your service needs it

### Step 3: Configure SSL Tab

1. **SSL Certificate:** Select your Cloudflare Origin certificate
2. **Force SSL:** ✅ Enable
3. **HTTP/2 Support:** ✅ Enable
4. **HSTS Enabled:** ✅ Enable (optional)
5. **HSTS Subdomains:** ✅ Enable (optional)

### Step 4: Save Configuration

Click **Save** to create the proxy host.

## Testing and Verification

### Step 1: Internal Access Test

From a device on your local network:

1. Access `https://chat.yourdomain.com`
2. **Expected:** Certificate warning (this is normal for internal access)
3. Click **Advanced** → **Proceed to site** to continue
4. Verify your service loads correctly

### Step 2: External Access Test

From outside your network (mobile data, different internet connection):

1. Access `https://chat.yourdomain.com`
2. **Expected:** Valid certificate, no warnings
3. Verify your service loads correctly

### Step 3: DNS Resolution Verification

**Internal DNS Test:**
```bash
nslookup chat.yourdomain.com
# Should return: 192.168.1.100 (your NPM internal IP)
```

**External DNS Test:**
```bash
nslookup chat.yourdomain.com 8.8.8.8
# Should return: your public IP address
```

## Troubleshooting

### Common Issues and Solutions

#### Certificate Warnings on Internal Access

**Issue:** Browser shows certificate warnings when accessing internally.

**Solution:** This is expected behavior. Cloudflare Origin Certificates are only trusted by Cloudflare's proxy. Internal access bypasses the proxy, so browsers show warnings.

**Workarounds:**
- Click "Advanced" → "Proceed to site"
- Add certificate to browser's trust store
- Accept the risk for internal access

#### External Access Not Working

**Issue:** Cannot access service from outside the network.

**Check:**
1. **Port forwarding rules:** Ensure ports 80 and 443 are forwarded to NPM
2. **Cloudflare proxy status:** Must be "Proxied" (orange cloud)
3. **Firewall rules:** Check that associated filter rules were created
4. **Public IP:** Verify Cloudflare DNS points to correct public IP

#### DNS Resolution Issues

**Issue:** Domain not resolving to correct IP addresses.

**Solutions:**
1. **Clear DNS cache:** `ipconfig /flushdns` (Windows) or `sudo dscacheutil -flushcache` (macOS)
2. **Check OPNsense overrides:** Verify host override is enabled and correct
3. **Test with different DNS servers:** Use `nslookup domain.com 8.8.8.8`

#### Service Not Loading

**Issue:** Domain resolves but service doesn't load.

**Check:**
1. **Backend service status:** Ensure your service is running
2. **NPM proxy host configuration:** Verify forward IP and port are correct
3. **Internal networking:** Test direct access to service IP:port
4. **NPM logs:** Check logs for error messages

### Let's Encrypt Alternative (If Needed)

If you encounter issues with Cloudflare Origin Certificates, you can fall back to Let's Encrypt:

1. **Disable** "Use a DNS Challenge" in NPM
2. Ensure ports 80 and 443 are accessible from the internet
3. Use HTTP challenge instead of DNS challenge
4. This bypasses the DNS plugin dependency conflicts

### Useful Commands

**Check NPM container status:**
```bash
docker compose ps
docker compose logs app
```

**Test port connectivity:**
```bash
telnet your-public-ip 443
telnet your-npm-internal-ip 443
```

**Check OPNsense NAT rules:**
Navigate to **Firewall** → **NAT** → **Port Forward** to verify rules are active.

## Security Considerations

1. **Regular Updates:** Keep Nginx Proxy Manager updated
2. **Strong Passwords:** Use strong passwords for NPM admin interface
3. **Firewall Rules:** Only open necessary ports
4. **Certificate Monitoring:** Monitor certificate expiration (15 years for Origin Certificates)
5. **Access Logs:** Regularly review access logs for suspicious activity

## Conclusion

This setup provides:

- **Secure external access** through Cloudflare's proxy with valid certificates
- **Internal access** to services using the same domain names
- **Long-term certificates** (15 years) with no renewal headaches
- **Centralized proxy management** through Nginx Proxy Manager
- **Split-horizon DNS** for optimal routing

The combination of Cloudflare Origin Certificates, Nginx Proxy Manager, and OPNsense provides a robust, secure, and maintainable solution for homelab service access.

[1] https://pplx-res.cloudinary.com/image/private/user_uploads/48397835/fb3a71b8-e90e-4aa9-bd17-f23d0eed40ba/image.jpg
[2] https://pplx-res.cloudinary.com/image/private/user_uploads/48397835/c5bae736-aba3-4909-bc64-7f21583aebfa/image.jpg
[3] https://pplx-res.cloudinary.com/image/private/user_uploads/48397835/0e8f055d-e150-4821-828c-064805adcf74/image.jpg
[4] https://nginxproxymanager.com/setup/
[5] https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/
[6] https://www.zenarmor.com/docs/network-security-tutorials/how-to-setup-unbound-dns-on-opnsense
[7] https://www.zenarmor.com/docs/network-security-tutorials/how-to-configure-opnsense-nat
[8] https://community.cloudflare.com/t/origin-certificate-and-nginx-proxy-manager/674782
[9] https://homenetworkguy.com/how-to/configure-split-dns-opnsense-using-unbound/
[10] https://www.wundertech.net/how-to-port-forward-in-opnsense/
[11] https://nginxproxymanager.com/guide/
[12] https://forums.truenas.com/t/how-to-add-cloudflare-ssl-certificates-to-nginx-proxy-manager/23834
[13] https://www.rapidseedbox.com/blog/nginx-proxy-manager
[14] https://www.reddit.com/r/unRAID/comments/173po72/anyone_have_an_easy_guide_to_follow_to_setup/
[15] https://www.youtube.com/watch?v=kR01Is8Xa2M
[16] https://documentation.breadnet.co.uk/kubernetes/nginx-ingress/nginx-ingress-with-cloudflare-origin-server-ssl-tls/
[17] https://github.com/NginxProxyManager/nginx-proxy-manager
[18] https://www.reddit.com/r/unRAID/comments/kniuok/howto_add_a_wildcard_certificate_in_nginx_proxy/
[19] https://www.youtube.com/watch?v=GarMdDTAZJo
[20] https://community.cloudflare.com/t/origin-certificate-and-nginx-proxy-manager/674782/2
[21] https://gist.github.com/prateekrajgautam/75afbaa9bcda8eb1dfb6b5ceecd25e8c
