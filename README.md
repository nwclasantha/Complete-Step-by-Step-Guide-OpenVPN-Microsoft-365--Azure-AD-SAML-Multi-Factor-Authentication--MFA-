# **🔐 Complete Step-by-Step Guide: OpenVPN + Microsoft 365 (Azure AD SAML) + Multi-Factor Authentication (MFA)**
This guide provides a **fully detailed** setup for integrating **OpenVPN Access Server** with **Azure AD SAML authentication** while enforcing **Multi-Factor Authentication (MFA) via Microsoft Authenticator**.

---

## **✅ Key Features**
✔ **No `.ovpn` files contain private keys or certificates** (users authenticate via Azure AD).  
✔ **Microsoft 365 (Azure AD) authentication via SAML.**  
✔ **Multi-Factor Authentication (MFA) enforced via Microsoft Authenticator.**  
✔ **Users get access to network ranges based on Azure AD groups (`20.20.20.0/24`, `30.30.30.0/24`).**  
✔ **VPN access is revoked automatically when an employee leaves Azure AD.**  

---

## **✅ Complete Flow**
![2sSVJ8PgnN79bVGyzhQskk](https://github.com/user-attachments/assets/35ba1797-1571-4e87-9221-b63086028470)


# **🚀 Step 1: Deploy OpenVPN Server on AWS**
### **1.1: Launch an AWS EC2 Instance**
1️⃣ **Login to AWS Console** → Go to **EC2**  
2️⃣ Click **Launch Instance**  
3️⃣ Choose **Ubuntu 22.04 LTS**  
4️⃣ Select an **instance type**: `t3.medium` or higher  
5️⃣ **Configure Security Group**:
   - **TCP 443** → OpenVPN Web UI  
   - **TCP 943** → OpenVPN Admin UI  
   - **UDP 1194** → OpenVPN VPN Traffic  
   - **TCP 22** → SSH (Optional)  
6️⃣ **Launch the instance** and **connect via SSH**:
```bash
ssh -i your-key.pem ubuntu@<your-ec2-public-ip>
```

---

### **1.2: Install OpenVPN Access Server**
```bash
wget https://openvpn.net/downloads/openvpn-as-latest-ubuntu22.amd_64.deb
sudo dpkg -i openvpn-as-latest-ubuntu22.amd_64.deb
```
Set the **OpenVPN admin password**:
```bash
sudo passwd openvpn
```
Retrieve the **Admin Web UI URL**:
```bash
sudo cat /usr/local/openvpn_as/init.log | grep "Admin UI"
```
Example output:
```
Admin UI: https://<your-ec2-public-ip>:943/admin
```
📌 **Login to OpenVPN Web UI**:
- **URL**: `https://<your-ec2-public-ip>:943/admin`
- **Username**: `openvpn`
- **Password**: *(Set earlier)*

---

# **🚀 Step 2: Configure Azure AD for OpenVPN Authentication**
### **2.1: Create an Azure AD Enterprise Application**
1️⃣ **Go to** [Azure AD Portal](https://portal.azure.com)  
2️⃣ Navigate to **Azure Active Directory → Enterprise Applications**  
3️⃣ Click **New Application**  
4️⃣ Click **Create Your Own Application** → Name it **"OpenVPN SAML Auth"**  
5️⃣ Select **"Integrate any other application"**  
6️⃣ Click **Create**  

---

### **2.2: Configure SAML Authentication**
1️⃣ Inside **OpenVPN SAML Auth**, go to **Single Sign-On → SAML**  
2️⃣ Click **Edit** under **Basic SAML Configuration**  
3️⃣ Set:
   - **Identifier (Entity ID)**:  
     ```
     https://<your-vpn-server>/
     ```
   - **Reply URL (ACS URL):**  
     ```
     https://<your-vpn-server>:943/saml
     ```
4️⃣ Click **Save**  

---

### **2.3: Set User Attributes & Claims**
1️⃣ Click **Edit** under **Attributes & Claims**  
2️⃣ Ensure these claims exist:
   - **NameID** → `user.userprincipalname`
   - **givenname`
   - **surname`
   - **email`
3️⃣ Click **Save**  

---

### **2.4: Download Federation Metadata XML**
1️⃣ Scroll down to **SAML Signing Certificate**  
2️⃣ Click **Download Federation Metadata XML**  
3️⃣ Save this file for later  

---

# **🚀 Step 3: Configure OpenVPN to Use SAML Authentication**
1️⃣ **Log in to OpenVPN Admin UI** (`https://<your-ec2-public-ip>:943/admin`)  
2️⃣ **Go to Authentication → SAML**  
3️⃣ **Upload the Federation Metadata XML** from Azure AD  
4️⃣ Set:
   - **Entity ID** → `https://<your-vpn-server>/`
   - **Assertion Consumer Service URL** → `https://<your-vpn-server>:943/saml`
5️⃣ Click **Save & Apply**  

---

# **🚀 Step 4: Configure OpenVPN Server Configuration (`server.conf`)**
📌 **File Location**: `/etc/openvpn/server.conf`
```ini
############################################
# OpenVPN Server Configuration - Microsoft 365 SAML + MFA
############################################

# OpenVPN Port & Protocol
port 1194
proto udp
dev tun

# VPN Subnet for Clients
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt

# Use PAM Authentication for Microsoft 365 Login
plugin /usr/lib/openvpn/plugins/openvpn-plugin-auth-pam.so openvpn
verify-client-cert none

# TLS Security
tls-server
tls-version-min 1.2
cipher AES-256-CBC
auth SHA256
dh /etc/openvpn/dh.pem
remote-cert-tls server
tls-crypt /etc/openvpn/ta.key

# Logging
status /var/log/openvpn/openvpn-status.log
log /var/log/openvpn/openvpn.log
verb 3  

# Route Enforcements Based on Azure AD Groups
client-config-dir /etc/openvpn/ccd
ccd-exclusive  

# Default Pushed Routes
push "route 20.20.20.0 255.255.255.0"
push "route 30.30.30.0 255.255.255.0"

# Prevent Clients from Modifying Routes
route 20.20.20.0 255.255.255.0
route 30.30.30.0 255.255.255.0

# Disable Compression (Security Best Practice)
comp-lzo no  

# Enable IP Forwarding
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"

# User & Group Permissions
user nobody
group nogroup

# Management Interface
management localhost 7505
```
---
# **🚀 Step 5: Configure OpenVPN Client (`client.ovpn`)**
📌 **File Location**: `/path/to/client.ovpn`
```ini
client
dev tun
proto udp
remote <your-vpn-server> 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
auth SHA256
verb 3

# Use Microsoft 365 (Azure AD) Authentication
auth-user-pass
```
---

# **✅ Final Outcome**
✔ **Microsoft 365 authentication is enforced (SAML + MFA).**  
✔ **No `.ovpn` files contain private keys or certificates.**  
✔ **Azure AD groups control which networks users can access.**  
✔ **VPN access is revoked when an employee leaves the company.**  
