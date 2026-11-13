# Secure Linux Mail Server | Postfix, Dovecot & OpenSSL

**Individual System & Network Administration Project** · Asia Pacific University (APU) · 2025

**Author:** Fairouz Abdulrahman Hamed Mousa  
**Module:** CT106-3-2-SNA — System and Network Administrator

## Project Overview

I independently deployed, configured, tested, and troubleshot a Linux-based email server in a virtualized client–server environment. I used **Rocky Linux** to host the mail services and **Ubuntu** with **Mozilla Thunderbird** to verify that two user accounts could send and receive email. I integrated **Postfix** for mail transfer, **Dovecot** for mailbox access, and **OpenSSL** to configure encrypted connections.

The goal was to put practical Linux system administration, network configuration, user management, service management, and mail protocol security into practice—not simply to install software.

> **Scope:** This was a controlled academic lab. The certificate was self-signed, and the configuration should not be treated as a production-ready public mail server.

## What I Implemented

- Prepared a VirtualBox-based **Rocky Linux server** and **Ubuntu client**.
- Verified the server hostname and static IPv4 address, and checked the client's DHCP-assigned address and DNS settings.
- Installed and configured **Postfix** to handle SMTP mail routing and local Maildir delivery.
- Installed and configured **Dovecot** for IMAP access, mailbox storage, and authentication.
- Generated a **self-signed RSA 4096-bit certificate** with OpenSSL and referenced the certificate/key in the mail service configurations.
- Configured the Linux firewall for the relevant mail services and secure ports.
- Created two Linux mail users and configured their accounts in Thunderbird.
- Tested TLS connectivity and demonstrated email delivery **in both directions**.
- Diagnosed and resolved authentication, TLS/connection, and mail destination configuration issues.

## Lab Architecture

```text
              VirtualBox Lab Network — 192.168.200.0/24

   Ubuntu Client                                 Rocky Linux Server
   fairouzclient.bungkus.org                     fairouzserver.bungus.org
   192.168.200.80 (DHCP)                         192.168.200.4
   +--------------------------+                  +--------------------------+
   | Thunderbird              |  IMAPS TCP 993   | Dovecot (IMAP / Maildir) |
   | user1@bungus.org         | <--------------> |                          |
   | user2@bungus.org         |  SMTPS TCP 465   | Postfix (SMTP)           |
   |                          | <--------------> | OpenSSL / TLS certificate|
   +--------------------------+                  +--------------------------+
```

*The hostnames above follow the assignment documentation. The lab was not a public internet email deployment.*

![Rocky Linux server IP verification](01-rocky-server-ip.png)
![Ubuntu client network verification](02-ubuntu-client-network.png)

## Tools and Technologies

| Area | Technology |
|---|---|
| Virtualization | Oracle VirtualBox |
| Server OS | Rocky Linux |
| Client OS | Ubuntu Linux |
| Mail transfer | Postfix (SMTP / SMTPS) |
| Mailbox access | Dovecot (IMAP / IMAPS) |
| Encryption | OpenSSL, SSL/TLS, self-signed X.509 certificate |
| Email client | Mozilla Thunderbird |
| Administration | `systemctl`, `firewall-cmd`, `nmcli`, `nano`, `useradd`, `passwd` |
| Connectivity testing | `openssl s_client` |

## 1. Postfix Mail Transfer Configuration

I installed Postfix and edited `/etc/postfix/main.cf` to set the server identity, local mail domains, trusted network, Maildir mailbox format, and TLS certificate paths. I also worked with `/etc/postfix/master.cf` to configure SMTP and SMTPS services, including the TLS wrapper mode for port **465**.

Selected settings documented in my report:

```ini
myhostname = fairouzserver.bungus.org
mydomain = bungus.org
myorigin = $mydomain
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
home_mailbox = Maildir/
smtpd_tls_cert_file = /etc/pki/tls/certs/fairouzserver.crt
smtpd_tls_key_file = /etc/pki/tls/private/fairouzserver.key
smtpd_tls_security_level = encrypt
```

![Postfix main configuration](03-postfix-configuration.png)

## 2. Dovecot Mailbox and Authentication Configuration

I configured Dovecot's Maildir storage, IMAP/LMTP protocols, authentication mechanisms, and TLS certificate references. The IMAP listener was configured for **143**, and encrypted IMAPS access for **993**.

```ini
# /etc/dovecot/conf.d/10-mail.conf
mail_location = maildir:~/Maildir

# /etc/dovecot/dovecot.conf
protocols = imap lmtp

# /etc/dovecot/conf.d/10-auth.conf
auth_mechanisms = plain login

# /etc/dovecot/conf.d/10-ssl.conf
ssl = required
ssl_cert = </etc/pki/tls/certs/fairouzserver.crt
ssl_key = </etc/pki/tls/private/fairouzserver.key
```

![Dovecot TLS configuration](04-dovecot-tls.png)
![Dovecot IMAP listeners](05-imap-service-ports.png)

## 3. Certificate and Service Management

I generated a self-signed certificate and private key for the lab using OpenSSL. I then restarted, enabled, and checked the status of both services with `systemctl`. The captured status screens show Postfix and Dovecot running.

```bash
sudo openssl req -x509 -newkey rsa:4096 \
  -keyout /etc/pki/tls/private/fairouzserver.key \
  -out /etc/pki/tls/certs/fairouzserver.crt \
  -days 365 -nodes

sudo systemctl restart postfix
sudo systemctl enable postfix
sudo systemctl status postfix

sudo systemctl restart dovecot
sudo systemctl enable dovecot
sudo systemctl status dovecot
```

![OpenSSL certificate generation](06-openssl-certificate.png)
![Postfix service running](07-postfix-running.png)
![Dovecot service running](08-dovecot-running.png)

**Security note:** A self-signed certificate encrypts traffic but does not establish third-party identity trust. Thunderbird required a certificate security exception in this lab.

## 4. Firewall and TLS Connectivity

I configured firewall access for IMAP/POP3 services and added the secure mail ports **993/TCP (IMAPS)** and **465/TCP (SMTPS)**. I used OpenSSL client connection tests against both ports to inspect the TLS connection and certificate information.

```bash
sudo firewall-cmd --add-port=993/tcp
sudo firewall-cmd --add-port=465/tcp

openssl s_client -connect fairouzserver.bungus.org:993
openssl s_client -connect fairouzserver.bungus.org:465
```

![Firewall port configuration](09-firewall-ports.png)
![OpenSSL TLS connectivity tests](10-tls-connectivity.png)

## 5. User Management and Thunderbird Setup

I created two Linux users (`user1` and `user2`), assigned passwords, and configured their email accounts on the Ubuntu client. Both accounts used the same mail server, with **IMAPS port 993** for incoming messages and **SMTPS port 465** for outgoing messages.

```bash
sudo useradd user1 -m -s /bin/bash
sudo passwd user1
sudo useradd user2 -m -s /bin/bash
sudo passwd user2
```

| Thunderbird setting | Value |
|---|---|
| Incoming protocol | IMAP |
| Incoming hostname | `fairouzserver.bungus.org` |
| Incoming port / security | `993` / SSL/TLS |
| Outgoing hostname | `fairouzserver.bungus.org` |
| Outgoing port / security | `465` / SSL/TLS |
| Test accounts | `user1@bungus.org`, `user2@bungus.org` |

![Thunderbird server settings](11-thunderbird-configuration.png)
![Configured Thunderbird mail accounts](12-email-accounts.png)

## 6. Functional Email Testing

I verified end-to-end operation by sending a test message from **user1 to user2**, confirming it appeared in the recipient's mailbox, and then repeating the test **from user2 to user1**. These screenshots are the principal functional evidence for the working lab.

![Email delivery from user1 to user2](13-email-delivery-test.png)
![Reverse-direction email delivery](14-reverse-email-test.png)

## 7. Troubleshooting and Lessons Learned

The project required several rounds of troubleshooting, which I documented rather than omitting them:

| Issue encountered | What I investigated / changed |
|---|---|
| Dovecot did not recognize an email-style username | Used `doveadm user` for diagnosis and added `auth_username_format = %n` in `10-auth.conf` to process the local username portion |
| Connection refused on SMTP port 465 (`errno=111`) | Checked the mail service's TLS certificate/key configuration and corrected the key-related issue documented in the report |
| Mail sending initially failed | Found a missing `$mydomain` entry in Postfix `mydestination` and corrected the configuration |

![Dovecot authentication troubleshooting](15-authentication-troubleshooting.png)
![SMTPS connection troubleshooting](16-connection-troubleshooting.png)
![Postfix destination troubleshooting](17-postfix-troubleshooting.png)

## Results and Skills Demonstrated

**Outcome:** I completed a functioning two-account mail-server lab with evidence of running services, TLS connection checks, Thunderbird configuration, and successful bidirectional message delivery.

This project strengthened my practical skills in **Linux system administration, network services, SMTP/IMAP configuration, TLS certificate management, firewall configuration, account provisioning, log/error-driven troubleshooting, and end-to-end validation**.

## Limitations and Production Considerations

This repository documents an **academic proof of concept**, not a hardened production deployment. It uses a self-signed certificate, local test accounts, and a private virtual network. A real deployment would require trusted certificates, secure SMTP authentication and relay controls, DNS mail records, spam and abuse protections, monitoring, backups, and additional hardening. Some settings shown here are historical lab settings and should be independently reviewed before reuse.

## Repository Contents

- `README.md` — project walkthrough, configurations, testing, and lessons learned
- `01-...png` through `17-...png` — **actual pages/screenshots extracted from my original 56-page assignment**
- `Individual-SNA-Mail-Server-Report.pdf` — original academic documentation

**All screenshot paths are relative to the repository root**, so the images display on GitHub when uploaded alongside this README.

---

*Individual academic project by Fairouz Abdulrahman Hamed Mousa · Asia Pacific University (APU) · CT106-3-2-SNA.*
