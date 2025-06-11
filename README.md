To join your **Fedora 41** desktop (`hostname.domain.com`) to the **Windows Server 2022 Active Directory** (`domain.com`), follow these steps:

---

### **1. Ensure System Requirements**
Before starting, ensure the following:
- **Network connectivity**: Verify `hostname.domain.com` can reach `DC.domain.com` (`xxx.xxx.xxx.xxx`).
- **Time synchronization**: Ensure both machines have synchronized time using NTP.
- **Administrator credentials**: You have the **Administrator** account credentials to join the domain.
- **AD User account**: You want to log in using `username`.

---

### **2. Configure Network and Hostname**
Ensure your Fedora desktop has the correct hostname:
```bash
hostnamectl set-hostname hostname.domain.com
```
Then, verify it:
```bash
hostnamectl
```

---

### **3. Configure DNS to Use the Domain Controller**
Edit the DNS settings to point to your **Active Directory DNS** (`dc.domain.com`):

1. Open NetworkManager configuration:
   ```bash
   nmcli con show
   ```
2. Find your active connection name (e.g., `ens33`).
3. Set the DNS server:
   ```bash
   nmcli con mod ens33 ipv4.dns "xxx.xxx.xxx.xxx"
   ```
4. Apply changes:
   ```bash
   nmcli con up ens33
   ```
5. Verify DNS:
   ```bash
   nslookup dc.domain.com
   ```

---

### **4. Install Required Packages**
Install the required software for domain authentication:
```bash
sudo dnf install -y realmd sssd adcli oddjob oddjob-mkhomedir samba-common-tools krb5-workstation authselect
```

Enable and start the necessary services:
```bash
sudo systemctl enable --now oddjobd
```

---

### **5. Discover the Active Directory Domain**
Test domain discovery:
```bash
sudo realm discover domain.com
```
If successful, it should return details about your domain.

---

### **6. Join Fedora to Active Directory**
Use the `administrator` credentials to join:
```bash
sudo realm join --user=Administrator domain.com
```
You'll be prompted for the **Administrator** password. If successful, Fedora is now part of the domain.

Verify:
```bash
realm list
```

---

### **7. Configure SSSD for Authentication**
Modify `/etc/sssd/sssd.conf`:
```bash
sudo nano /etc/sssd/sssd.conf
```
Ensure it contains:
```ini
[sssd]
domains = domain.com
config_file_version = 2
services = nss, pam

[domain/domain.com]
id_provider = ad
access_provider = ad
auth_provider = ad
chpass_provider = ad
enumerate = true
cache_credentials = true
use_fully_qualified_names = False
fallback_homedir = /home/%u
default_shell = /bin/bash
ldap_id_mapping = True
```
Save and exit.

Restart SSSD:
```bash
sudo systemctl restart sssd
```
Enable it on startup:
```bash
sudo systemctl enable sssd
```

---

### **8. Allow AD Users to Log In**
Verify the AD user `username`:
```bash
id username
```
If it returns a UID and GID, authentication is working.

To allow AD users to log in:
```bash
sudo realm permit --all
```

If home directories should be created automatically, enable it:
```bash
sudo authselect select sssd with-mkhomedir --force
```

Restart services:
```bash
sudo systemctl restart sssd
```

---

### **9. Test Login**
Log in as `username`:
```bash
su - username
```
Alternatively, try logging in through the GUI.

---

### **10. Verify Authentication**
Check AD authentication:
```bash
getent passwd username
```
Check group memberships:
```bash
groups username
```

---

### **11. Ensure Fedora Respects AD Authentication**
If Fedora is still using local authentication, enforce domain authentication:
```bash
sudo pam-auth-update --enable mkhomedir
```
Restart the system and test logging in.

---

### **12. Optional: Enable Sudo for AD Users**
If you want `username` to have sudo access, add it to the `wheel` group:
```bash
sudo usermod -aG wheel username
```
Or add a rule:
```bash
echo "username ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/username
```

---

### **Final Steps**
- **Reboot** Fedora:  
  ```bash
  sudo reboot
  ```
- **Try logging in using `username` via GUI or SSH.**
- **Test authentication by running:**  
  ```bash
  sudo -i
  ```

---

This setup ensures that `hostname.domain.com` is fully integrated into `domain.com`, allowing `username` to log in seamlessly using **Active Directory credentials**.
