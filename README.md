# Starlink Remote Access via VPS

## Overview

Starlink uses CGNAT (Carrier Grade NAT), which prevents users from receiving a public IPv4 address. As a result, remote access to services inside the home network (e.g. Home Assistant, NAS, or self-hosted tools) is not possible by default.

This project provides a robust solution by creating a secure VPN tunnel from your home network (via FritzBox) to a VPS with a public IP, and exposing internal services using NGINX Proxy Manager.

This setup has proven to be both reliable and efficient in real-world use. Over the past year, it has run continuously and without interruption and minimal maintenance requirements.

## Architecture

- Home network connected via Starlink and managed by a FritzBox router  
- WireGuard VPN tunnel between FritzBox and VPS  
- NGINX Proxy Manager running on VPS handles reverse proxy + SSL

This diagram shows how NGINX Proxy Manager, your VPS, FritzBox, and internal services work together to securely expose a service to the internet.
~~~
      [Client Browser]
             │
       HTTPS Request
             ▼
     ┌─────────────────┐
     │   VPS Server    │
     │ ─────────────── │
     │    NGINX        │
     │ ─────────────── │
     │    WireGuard    │
     └─────────────────┘
             │
      Forward over VPN
             ▼
     ┌─────────────────┐
     │    FritzBox     │
     │ ─────────────── │
     │    WireGuard    │
     └─────────────────┘
             │
   Internal LAN Routing
             ▼
     ┌─────────────────┐
     │ Internal Server │
     │ ─────────────── │
     │  Local Service  │
     └─────────────────┘
~~~

## Requirements

- Starlink internet connection / CGNAT 
- FritzBox supporting WireGuard  (**FRITZ!OS ≥ 7.5**)  
- VPS with public IPv4 (e.g. Hetzner, Netcup, Ionos)  
- Domain name (optional but recommended) 

---

## VPS WireGuard Key Generation

On the VPS, generate a WireGuard key pair:

~~~
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
~~~

- `privatekey` contains the VPS private key (keep it secret)  
- `publickey` contains the VPS public key (to be shared with FritzBox)

---

## VPS WireGuard Configuration

Install WireGuard:

~~~
sudo apt update && sudo apt install wireguard
~~~

Create VPN config: `/etc/wireguard/wg0.conf`

~~~
[Interface]
Address = 10.0.0.1/24
PrivateKey = <VPS_PRIVATE_KEY>
ListenPort = 51820

[Peer]
PublicKey = <FRITZBOX_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32
~~~

Activate WireGuard tunnel:

~~~
sudo wg-quick up wg0
~~~

Make sure UDP port 51820 is open in your VPS firewall:

~~~
sudo ufw allow 51820/udp
~~~

---

## FritzBox WireGuard Setup

1. Open your FritzBox Web UI  
2. Navigate to: **Internet > Permit Access > VPN (WireGuard)**  
3. Choose **Import existing configuration**  
4. Paste the following configuration:

~~~
[Interface]
PrivateKey = <FRITZBOX_PRIVATE_KEY>
Address = 10.0.0.2/24

[Peer]
PublicKey = <VPS_PUBLIC_KEY>
AllowedIPs = 0.0.0.0/0
Endpoint = <VPS_PUBLIC_IP>:51820
PersistentKeepalive = 25
~~~

After importing, FritzBox will establish and maintain the VPN tunnel to your VPS.

<img width="376" height="42" alt="grafik" src="https://github.com/user-attachments/assets/65f7b5b1-e376-4932-901d-c3f326bfbe94" />


---

## NGINX Proxy Manager Setup (on VPS)

Install Docker and Docker Compose:

~~~
sudo apt install docker.io docker-compose
~~~

Create `docker-compose.yml`:

~~~
version: '3'
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
~~~

Launch NGINX Proxy Manager:

~~~
docker-compose up -d
~~~

---

## Exposing Internal Services via NGINX Proxy Manager

Once the VPN tunnel between your FritzBox and VPS is active, and NGINX Proxy Manager (NPM) is running inside Docker, you can expose internal services over HTTPS.

---

### Accessing the NGINX Proxy Manager Interface

After launching NPM on the VPS, open the following URL in a browser:
`http://<VPS_PUBLIC_IP>`

Default login credentials (if not already configured):

- **Email:** `admin@example.com`
- **Password:** `changeme`

You will be prompted to change these credentials upon first login.

---

### Creating a Proxy Host Entry

1. In the NPM dashboard, navigate to **"Proxy Hosts"**.
2. Click **"Add Proxy Host"** and configure:
   - **Domain Names:** e.g. `home.example.com`
   - **Forward Hostname / IP:** `10.0.0.2` (WireGuard IP of FritzBox)
   - **Forward Port:** `8123` (adjust to your service)
   - Enable **Websockets Support** if required
3. SSL Settings:
   - Check **"Block Common Exploits"**
   - Enable **SSL**
   - Choose **Let's Encrypt Certificate**
   - Set **Force SSL**

Click **Save** to apply.

<img width="494" height="542" alt="grafik" src="https://github.com/user-attachments/assets/16c6ba48-e6e3-4a1d-80ed-e77966c03519" />

---

### DNS Configuration

Make sure your domain's DNS settings point to your VPS IP address:

- Create an **A record** for `home.example.com` → `<VPS_PUBLIC_IP>`

This step is essential for certificate generation and public access.

---

### Testing Public Access

Once DNS has propagated and the proxy is active:
`home.example.com`


You should see your internal service securely loaded via HTTPS.

~~~bash
curl -I https://home.example.com
~~~

### Exposing Internal Services with NGINX Proxy Manager

Once your VPN tunnel between the FritzBox and the VPS is active, and NGINX Proxy Manager (NPM) is running inside Docker on your VPS, you can expose services from your home network securely over HTTPS.

---

#### Accessing the NGINX Proxy Manager Web Interface

To configure reverse proxies, access the NPM dashboard via your browser:
`http://<VPS_PUBLIC_IP>`


On first login, you'll be prompted to create a new admin account. If default credentials apply, they are:

- **Email:** `admin@example.com`
- **Password:** `changeme`

---

#### Creating a Proxy Host

1. Log in to the dashboard and go to **Proxy Hosts**
2. Click **Add Proxy Host**
3. Fill in the configuration:
   - **Domain Names:** e.g. `service.example.com`
   - **Forward Hostname / IP:** `10.0.0.2` (WireGuard IP of FritzBox)
   - **Forward Port:** `8123` (adjust to the internal service)
   - Enable Websockets support if the service requires it
4. Under **SSL**:
   - Select **"Request a Let's Encrypt SSL Certificate"**
   - Enable **Force SSL**
   - Optionally enable **HTTP/2 support**

Save the entry.

---

#### DNS Configuration

In your domain registrar’s DNS settings:

- Create an **A Record** pointing `service.example.com` to your **VPS public IP**

Example:
`service.example.com` → `198.51.100.42`

---

#### Verifying Public Access

Once DNS is updated and the proxy is active:

- Test in a browser:  
  `https://service.example.com`

- Or via terminal:

~~~bash
curl -I https://service.example.com
~~~

You should receive a valid HTTPS response from your internal service through the VPS.
---

## Security Best Practices

- Enforce HTTPS for all exposed services  
- Use authentication in NGINX Proxy Manager  
- Monitor VPN connections and rotate keys periodically  
- Restrict inbound traffic using firewall rules  
- Keep Docker containers and system packages up to date  
- Install `fail2ban` or similar intrusion protection on VPS

---

## License

This project is licensed under the MIT License. See the `LICENSE` file for full terms.

---

## Contributions

Pull requests and suggestions are welcome.  
Please open issues for enhancements or fixes.
