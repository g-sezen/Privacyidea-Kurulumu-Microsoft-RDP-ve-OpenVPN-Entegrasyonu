# PrivacyIDEA Installation with Microsoft RDP and OpenVPN Integration
In this article, we will go over the steps to set up a two-factor authentication (2FA) system using PrivacyIDEA and OpenVPN.

The Linux distribution used is Ubuntu 24.04.1 LTS.

### PrivacyIDEA Installation

#### Time Zone Setup

Set the time zone:
```bash
sudo timedatectl set-timezone Europe/Istanbul
```

#### Installing Necessary Packages

1. Install Python and Virtualenv:
```bash
sudo apt install python3-pip
sudo pip install virtualenv --break-system-packages
```

2. Create the PrivacyIDEA environment:
```bash
sudo virtualenv -p /usr/bin/python3 /opt/privacyidea
cd /opt/privacyidea
source bin/activate
```

3. Install PrivacyIDEA packages:
```bash
sudo pip install privacyidea --break-system-packages
sudo chown -R $(whoami) /opt/privacyidea/
```

4. Download and add the GPG key:
```bash
wget https://lancelot.netknights.it/NetKnights-Release.asc
sudo mv NetKnights-Release.asc /etc/apt/trusted.gpg.d/
sudo add-apt-repository http://lancelot.netknights.it/community/noble/stable
sudo apt update
sudo apt install privacyidea-apache2
```

#### Web Interface Settings

1. Create an admin user:
```bash
sudo pi-manage admin add admin admin@localhost
```

2. To set the interface language, edit the file:
```bash
sudo nano /opt/privacyidea/lib/python3.12/site-packages/privacyidea/webui/login.py
```

Activate only the English language:

Note: English was preferred because the local language (TR) translation was not very successful.

```python
# DEFAULT_LANGUAGE_LIST = ['en', 'de', 'nl', 'zh Hant', 'fr', 'es', 'tr', 'cs', 'it']  #:
DEFAULT_LANGUAGE_LIST = ['en']  #:
```

3. Restart the Apache server:
```bash
sudo systemctl restart apache2
```

#### MySQL Database Setup

1. Create a database on MySQL:
```sql
CREATE DATABASE rdp_users CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
USE rdp_users;
GRANT ALL PRIVILEGES ON rdp_users.* TO 'pi'@'localhost';
FLUSH PRIVILEGES;
```

2. Add the users table:
```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(255) NOT NULL UNIQUE,
  firstname VARCHAR(255),
  lastname VARCHAR(255),
  email VARCHAR(255),
  mobile VARCHAR(20),
  description TEXT,
  enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

3. Get the user credentials for MySQL access from the following file:
```bash
sudo nano /etc/privacyidea/pi.cfg
```

#### PrivacyIDEA Web Interface Settings

1. Define a new SQL Resolver:
   - Resolver Name: `mysql-resolver`
   - Driver: `mysql-pymysql`
   - Server: `127.0.0.1`
   - Port: `3306`
   - Database: `rdp_users`
   - User: `pi`
   - Password: `EnQaAwbrs2iP`
   - Mapping: `{ "username": "username", "surname": "lastname", "givenname": "firstname", "email": "email", "mobile": "mobile", "description": "description", "userid": "id" }`

2. Create a new realm and set it as default.
3. Register the user from the Users section.
4. Go to the Tokens section. Select TOTP from the Enroll token section.
5. Choose the realm you just created.
6. Enter the username you created on MySQL as the Username.
7. Leave the Pin field blank.
8. You can configure a container by choosing Assign the token to a container.
9. Enroll the token by clicking Enroll Token and assign it to the user.
10. Note: The username on the RDP server you are connecting to must match the username you defined in PrivacyIDEA.
11. Note: This setup was performed on Windows 2022.
12. Complete the setup by following the images below.

![Image 1](images/privacyidea-credential-provider-1.png)
![Image 2](images/privacyidea-credential-provider-2.png)
![Image 3](images/privacyidea-credential-provider-3.png)
![Image 4](images/privacyidea-credential-provider-4.png)
![Image 5](images/privacyidea-credential-provider-5.png)
![Image 6](images/privacyidea-credential-provider-6.png)
![Image 7](images/privacyidea-credential-provider-7.png)
![Image 8](images/privacyidea-credential-provider-8.png)
![Image 9](images/privacyidea-credential-provider-9.png)
![Image 10](images/privacyidea-credential-provider-10.png)

---

After completing these steps, your system will be more secure with PrivacyIDEA and RDP integrated for two-factor authentication.

---

### OpenVPN 2FA Setup

#### OpenVPN Installation

1. Set the time zone:
```bash
sudo timedatectl set-timezone Europe/Istanbul
```

2. Install OpenVPN:
```bash
wget https://raw.githubusercontent.com/angristan/openvpn-install/refs/heads/master/openvpn-install.sh
chmod +x openvpn-install.sh
sudo ./openvpn-install.sh
```

3. Add a new user on the Linux server:
```bash
sudo adduser gokhan
```

4. Add the created user to OpenVPN so they can connect:
```bash
sudo ./openvpn-install.sh
- Clientname: `gokhan`
- Password: `Add a passwordless client`
```

5. Note: The username on the VPN server you are connecting to must match the username you defined in PrivacyIDEA.

#### PrivacyIDEA PAM Integration

1. Install the PrivacyIDEA PAM module:
```bash
git clone https://github.com/privacyidea/privacyidea-pam
cd privacyidea-pam
wget https://github.com/nlohmann/json/releases/download/v3.11.3/json.hpp
sudo apt install make g++ libcurl4-openssl-dev libssl-dev libpam0g-dev
make
make install
sudo mkdir /lib64/security/
sudo mv pam_privacyidea.so /lib64/security
```

2. Edit the PAM file:
```bash
sudo nano /etc/pam.d/openvpn
```
File contents:
```plaintext
auth    required                        pam_unix.so     try_first_pass
auth    [success=1 default=die]         /lib64/security/pam_privacyidea.so      url=https://yourdomain.org prompt=privacyIDEA_Authentication
auth    requisite                       pam_deny.so
auth    required                        pam_permit.so
account sufficient                      pam_permit.so
session sufficient                      pam_permit.so
```

3. Edit the OpenVPN server configuration file:
```bash
sudo nano /etc/openvpn/server.conf
```
Settings:
```plaintext
verify-client-cert none
username-as-common-name
reneg-sec 0
plugin /usr/lib/x86_64-linux-gnu/openvpn/plugins/openvpn-plugin-auth-pam.so "openvpn login USERNAME password PASSWORD 'please enter otp:' OTP"
script-security 3
```

4. Restart the OpenVPN server:
```bash
sudo systemctl restart openvpn@server.service
```

5. Add the following lines to the OpenVPN client configuration file:
```bash
auth-user-pass
static-challenge "privacyIDEA_Authentication" 1
setenv FRIENDLY_NAME "privacyIDEA_Authentication"
```

---

After completing these steps, your system will be more secure with PrivacyIDEA and OpenVPN integrated for two-factor authentication.

--- 

Let me know if you need any further assistance!
