To join your **Fedora 41** desktop (`argos.sleepy-puppy.com`) to the **Windows Server 2022 Active Directory** (`sleepy-puppy.com`), follow these steps:

---

### **1. Ensure System Requirements**
Before starting, ensure the following:
- **Network connectivity**: Verify `argos.sleepy-puppy.com` can reach `oenomaus.sleepy-puppy.com` (`192.168.3.30`).
- **Time synchronization**: Ensure both machines have synchronized time using NTP.
- **Administrator credentials**: You have the **Administrator** account credentials to join the domain.
- **AD User account**: You want to log in using `r.edwards`.

---

### **2. Configure Network and Hostname**
Ensure your Fedora desktop has the correct hostname:
```bash
hostnamectl set-hostname argos.sleepy-puppy.com
```
Then, verify it:
```bash
hostnamectl
```

---

### **3. Configure DNS to Use the Domain Controller**
Edit the DNS settings to point to your **Active Directory DNS** (`oenomaus.sleepy-puppy.com`):

1. Open NetworkManager configuration:
   ```bash
   nmcli con show
   ```
2. Find your active connection name (e.g., `ens33`).
3. Set the DNS server:
   ```bash
   nmcli con mod ens33 ipv4.dns "192.168.3.30"
   ```
4. Apply changes:
   ```bash
   nmcli con up ens33
   ```
5. Verify DNS:
   ```bash
   nslookup oenomaus.sleepy-puppy.com
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
sudo realm discover sleepy-puppy.com
```
If successful, it should return details about your domain.

---

### **6. Join Fedora to Active Directory**
Use the `administrator` credentials to join:
```bash
sudo realm join --user=Administrator sleepy-puppy.com
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
domains = sleepy-puppy.com
config_file_version = 2
services = nss, pam

[domain/sleepy-puppy.com]
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
Verify the AD user `r.edwards`:
```bash
id r.edwards
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
Log in as `r.edwards`:
```bash
su - r.edwards
```
Alternatively, try logging in through the GUI.

---

### **10. Verify Authentication**
Check AD authentication:
```bash
getent passwd r.edwards
```
Check group memberships:
```bash
groups r.edwards
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
If you want `r.edwards` to have sudo access, add it to the `wheel` group:
```bash
sudo usermod -aG wheel r.edwards
```
Or add a rule:
```bash
echo "r.edwards ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/r.edwards
```

---

### **Final Steps**
- **Reboot** Fedora:  
  ```bash
  sudo reboot
  ```
- **Try logging in using `r.edwards` via GUI or SSH.**
- **Test authentication by running:**  
  ```bash
  sudo -i
  ```

---

This setup ensures that `argos.sleepy-puppy.com` is fully integrated into `sleepy-puppy.com`, allowing `r.edwards` to log in seamlessly using **Active Directory credentials**.
