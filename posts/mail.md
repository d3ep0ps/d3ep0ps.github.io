# The Mail Stack: Postfix, Dovecot, and the War on Spam

> **Sending email is easy. Getting it delivered is a war.**

> [!IMPORTANT]
> **Mission Briefing**
> Open your terminal now. Do not just read this article. You are not here to copy-paste config files; you are here to build a communication infrastructure from scratch.
> Only proceed if you are ready to troubleshoot, debug, and understand every single line you type.

If configuring Apache is like learning to drive, configuring a Mail Server is like learning to fly a helicopter while people are shooting at you.

It is the most complex, fragile, and reputation-dependent system you will ever build.

The standard advice in 2025 is: **"Don't do it."**
Use Google Workspace. Use Microsoft 365. Use Amazon SES.

For a startup trying to hit the ground running, that is good advice. But for a System Architect, it is a cop-out.

If you don't build a mail server at least once, you don't understand the internet. You don't understand **Identity**, **Trust**, or the sheer volume of automated malice that traverses the globe every second.

In my years at ISP and hosting providers, I didn't just run mail servers; I curated **Reputation**. I watched IP blocks get blacklisted because of one compromised WordPress plugin. I saw legitimate business traffic vanish into the void because of a missing PTR record.

Today, we are going to build a carrier-grade mail stack on our Linux and FreeBSD servers. We are going to do it the "Hosting Provider Way"—not with system users, but with **Virtual Users** in a database, capable of hosting thousands of domains on a single box.

-----

## 1\. The Architecture: Truck Drivers vs. Warehouse Managers

Before we install anything, we need a mental model. Most people confuse SMTP, IMAP, and POP3. Here is how they actually fit together.

1.  **The MTA (Postfix):** Think of Postfix as the **Truck Driver**. Its job is to move mail from Server A to Server B using **SMTP** (Simple Mail Transfer Protocol, Port 25). It doesn't care what's inside the envelope. It doesn't store mail for users to read. It just drives.
2.  **The MDA (Dovecot):** Think of Dovecot as the **Warehouse Manager**. When the truck (Postfix) arrives, it hands the mail to Dovecot (via **LMTP**). Dovecot sorts it into shelves (files on disk) and lets users walk in and check their boxes using **IMAP** (Port 143/993).
3.  **The Database (MySQL):** We are **not** going to use Linux system accounts (`useradd vitaliy`). That doesn't scale. We will store domains, mailboxes, and aliases in MySQL. This is **Virtual Hosting**.
4.  **The Interface:**
      * **PostfixAdmin:** A web panel for us (the admins) to create domains/users in MySQL.
      * **Roundcube:** A webmail client for the users.

-----

## 2\. The Prerequisites: DNS is Destiny

You cannot run a mail server without perfect DNS. If your DNS is sloppy, Gmail and Outlook will reject you instantly.

Go to your DNS Manager (or the BIND server we built in the last article) and ensure you have:

1.  **A Record:** `mail.d3ep0ps.com` -\> `1.2.3.4` (Your Server IP).
2.  **MX Record:** `d3ep0ps.com` -\> `10 mail.d3ep0ps.com`.
3.  **PTR Record (Reverse DNS):**
      * *This is the big one.* You must ask your ISP or Cloud Provider to set the **Reverse DNS** of `1.2.3.4` to `mail.d3ep0ps.com`.
      * **Why?** When you connect to Google, they do a background check. "You claim to be `mail.d3ep0ps.com`. I see your IP is `1.2.3.4`. If I look up `1.2.3.4`, does it resolve back to `mail.d3ep0ps.com`?"
      * **Verify it:** `dig -x 1.2.3.4 +short`. It MUST return `mail.d3ep0ps.com`.
      * **STOP.** Do not proceed until you see that return value. If you skip this, everything else will fail.
      * If the answer is "No" (or a generic `ip-1-2-3-4.aws.com`), you are a spammer. Blocked.

-----

## 3\. Installation: Packages vs. The Architect's Way

### Linux (Debian/Ubuntu)

We need the mail stack and the database connectors.

```bash
sudo apt install postfix postfix-mysql dovecot-core dovecot-imapd dovecot-lmtpd dovecot-mysql
```

*During install, choose "Internet Site" and set your FQDN.*

### FreeBSD (The Architect's Choice)

FreeBSD comes with **Sendmail** in the base system. We need to disable it and install our modern stack.

**The "Easy" Way (Binary Packages):**
If you use standard `pkg`, you often need to install separate database driver packages:

```bash
pkg install postfix postfix-mysql dovecot dovecot-mysql mysql80-server postfixadmin roundcube
```

**The "Right" Way (Ports):**
Here is a trap I have seen many times: pre-built binary packages sometimes lag behind or don't support the newest database versions (like MySQL 8.4). Or worse, a routine `pkg upgrade` replaces your custom binary with a generic one, breaking your mail server at 3 a.m.

To guarantee stability, we compile from **Ports** and **Lock** the package.

```bash
# 1. Compile Postfix with MySQL support enabled
cd /usr/ports/mail/postfix
make config  # [X] Check MYSQL and SASL
make install clean

# 2. Compile Dovecot with MySQL support
cd /usr/ports/mail/dovecot
make config  # [X] Check MYSQL
make install clean

# 3. CRITICAL: Lock the packages
# Prevent 'pkg upgrade' from replacing our custom binaries
pkg lock postfix
pkg lock dovecot
```

**Final Setup:**
Now, disable the base Sendmail and enable our new stack in `/etc/rc.conf`:

```bash
sysrc sendmail_enable="NO"
sysrc sendmail_submit_enable="NO"
sysrc sendmail_outbound_enable="NO"
sysrc sendmail_msp_queue_enable="NO"

sysrc postfix_enable="YES"
sysrc dovecot_enable="YES"
```

-----

## 4\. The Database: Virtual Users

We don't create users in `/etc/passwd`. We create them in SQL.
This allows us to host `ceo@company-a.com` and `support@company-b.com` on the same server, even though they share the same username "admin" or "support".

**PostfixAdmin** handles the schema creation for us.

1.  Create a database `postfix`.
2.  Create a user `postfix_admin` with privileges.
3.  Configure PostfixAdmin (`config.local.php`) to connect to it.
4.  Run the setup wizard in your browser (`http://your-ip/postfixadmin/setup.php`).

Now, instead of `useradd`, you log into a web UI and click "Add Mailbox."

-----

## 5\. Configuration: Postfix (The Transporter)

We need to teach Postfix to stop looking at Unix users and start looking at MySQL.

**File:** `/usr/local/etc/postfix/main.cf` (FreeBSD) or `/etc/postfix/main.cf` (Linux).

```text
# Identity
myhostname = mail.d3ep0ps.com
mydomain = d3ep0ps.com

# The Hand-off (Send mail to Dovecot via LMTP)
virtual_transport = lmtp:unix:private/dovecot-lmtp

# Lookups (Tell Postfix how to read MySQL)
virtual_mailbox_domains = proxy:mysql:/usr/local/etc/postfix/sql/mysql_virtual_domains_maps.cf
virtual_mailbox_maps = proxy:mysql:/usr/local/etc/postfix/sql/mysql_virtual_mailbox_maps.cf
virtual_alias_maps = proxy:mysql:/usr/local/etc/postfix/sql/mysql_virtual_alias_maps.cf

# SECURITY: Prevent Open Relays
# Only allow authenticated users or my network to send mail.
smtpd_relay_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
```

**[Personal Note]:** Back in the ISP days, "Open Relays" were the plague. If you messed up `smtpd_relay_restrictions`, spammers would find you in minutes and pump millions of emails through your server. Your IP reputation would be burned for years.

-----

## 6. Configuration: Postfix Master (The Ports)

We modified `main.cf`, but we also need to open the side doors. By default, Postfix only listens on Port 25.
We need Port 587 (Submission) for our users to send mail securely.

**File:** `/usr/local/etc/postfix/master.cf` (FreeBSD) or `/etc/postfix/master.cf` (Linux).

Find the line for `submission` and uncomment/edit it to look like this:

```text
submission inet n       -       n       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
```

*This tells Postfix: "On port 587, require encryption and require a username/password."*

**Challenge:** Why 587? Why not 25?
*Answer:* ISPs block port 25 for residential/dynamic IPs to stop spam. Port 587 is the industry standard for *submission* (authenticated sending). If you try to configure your iPhone to send via port 25, it probably won't work. Use 587.

-----

## 7. Configuration: Dovecot (The Warehouse)

Dovecot does two things: it authenticates users (checking the password) and it stores the files.

**File:** `/usr/local/etc/dovecot/dovecot.conf`

### Storage: Maildir vs mbox

We *always* use **Maildir**.

  * **mbox:** One giant file for all emails (`/var/mail/vitaliy`). If one bit gets corrupted, you lose everything. Locking is a nightmare.
  * **Maildir:** One file per email (`/var/vmail/d3ep0ps.com/vitaliy/new/17099232.server`). It is faster, safer, and easier to backup.

<!-- end list -->

```text
mail_location = maildir:/var/vmail/%d/%n
```

*%d = domain, %n = username*

### Authentication

We configure Dovecot to talk to the same MySQL database Postfix uses. Once verified, Dovecot gives Postfix the "thumbs up" to accept the email (this is SASL).

-----

## 8. The War on Spam: The Holy Trinity

If you stop here, you can send email, but it will go to Spam folders. To be trusted, you need the Trinity.

### 1\. SPF (Sender Policy Framework)

A DNS TXT record that says: "Only these IPs are allowed to send mail for `d3ep0ps.com`."
`v=spf1 mx ip4:1.2.3.4 -all`

### 2\. DKIM (DomainKeys Identified Mail)

A cryptographic signature on every email header.
We install **Rspamd** (the modern replacement for SpamAssassin/Amavis) or **OpenDKIM**.
The server signs the email with a Private Key. The world verifies it with a Public Key in your DNS.
*If the email is modified in transit, the signature breaks.*

### 3\. DMARC

A policy that tells Google/Yahoo what to do if SPF or DKIM fails.
`_dmarc.d3ep0ps.com TXT "v=DMARC1; p=none; rua=mailto:admin@d3ep0ps.com"`
*Translation: "If the check fails, just tell me. Don't block anything yet."*

**Pro Tip:** Start with `p=none` (Monitor mode). Run it for a week. If legitimate email isn't breaking, verify your reports, THEN switch to `p=reject` (Enforce mode).

-----

## 9. The Encryption Layer: TLS and Let's Encrypt

In the 90s, mail moved in plain text. Today, that is unacceptable.
If you don't encrypt the connection, your passwords are visible on the wire, and GMail will flag your delivery as insecure.

We need **TLS Certificates**.
We used to pay hundreds of dollars for these. Today, we use **Certbot (Let's Encrypt)**.

```bash
# Get the cert
certbot certonly --standalone -d mail.d3ep0ps.com
```

Then we tell Postfix and Dovecot where these keys live.

  * **Postfix:** `smtpd_tls_cert_file` and `smtpd_tls_key_file`.
  * **Dovecot:** `ssl_cert` and `ssl_key`.

-----

## 10. The Ritual: Testing via Telnet and OpenSSL

Real engineers don't test with Outlook. They test with the command line.

### Part 1: The SMTP Handshake (Telnet)

We manually speak SMTP to check if the logic works on the local machine.

```bash
telnet localhost 25
```

**The Conversation:**

```text
Trying 127.0.0.1...
Connected to localhost.
220 mail.d3ep0ps.com ESMTP Postfix
EHLO localhost           <-- You say Hello
250-mail.d3ep0ps.com
250-STARTTLS
MAIL FROM:<test@d3ep0ps.com>   <-- Sender
250 2.1.0 Ok
RCPT TO:<admin@d3ep0ps.com>    <-- Recipient
250 2.1.5 Ok
DATA                     <-- I'm ready to write
Subject: The First Email
Hello World from the command line.
.                        <-- The single dot ends the email
250 2.0.0 Ok: queued as 8327A1
QUIT
```

If you see `queued as...`, congratulations. Your truck driver (Postfix) accepted the package and handed it to the warehouse.

### Part 2: The Encrypted Handshake (OpenSSL)

Telnet cannot speak encryption. To test modern secure ports from the outside world, we use the Swiss Army Knife of cryptography: **OpenSSL**.

**Testing SMTP Submission (Port 587 + STARTTLS):**
Modern clients use this port. They connect in plain text and then shout "STARTTLS" to upgrade.

```bash
openssl s_client -starttls smtp -connect mail.d3ep0ps.com:587
```

*Look for `Verify return code: 0 (ok)` and the certificate chain.*

**Testing IMAP (Port 993 + Implicit TLS):**
IMAP expects encryption immediately. Telnet hangs here. OpenSSL works.

```bash
openssl s_client -connect mail.d3ep0ps.com:993
```

If successful, the session opens, and you will see the Dovecot greeting *inside* the encrypted tunnel:

```text
* OK [CAPABILITY IMAP4rev1 ...] Dovecot ready.
a1 LOGIN admin@d3ep0ps.com MySecretPassword
a1 OK [CAPABILITY ...] Logged in
```

**Why this matters:**
Web-based "SSL Checkers" only tell you if the port is open. `openssl s_client` lets you act like the mail client yourself. You can see exactly *where* the handshake fails—is it an expired cert? A mismatched cipher? Or a firewall dropping the packet after the Hello?

### Part 3: Intentional Sabotage (Learning by Breaking)
You have a working server? Good. Now break it.
1.  Go to `main.cf`. Change `myhostname` to `fake.d3ep0ps.com`.
2.  Run `postfix reload`.
3.  Try to send an email. It fails.
4.  **Check the logs.** You will see a "Helo command rejected" or similar mismatch error.
5.  *Fix it.*

**This is the only way you learn.** If you never see a broken state, you won't know what to look for when it breaks at 3 a.m.

-----

## 11. Operations: Firewall & Logs

### Firewall Checklist
You are running a fortress. Open only what is necessary:
*   **TCP 80/443:** Webmail & PostfixAdmin (HTTP/HTTPS)
*   **TCP 25:** Incoming Mail from the world (SMTP)
*   **TCP 587:** Outgoing Mail from your users (Submission)
*   **TCP 993:** File access for your users (IMAPS)
*   *Close everything else. Do not open Port 143 (Unencrypted IMAP) or Port 110 (POP3).*

**Red Team Yourself:**
From your laptop (not the server), try to telnet to port 143: `telnet mail.d3ep0ps.com 143`.
If it connects, **you failed**. Go back to your firewall settings and kill it. It should timeout or refuse connection.

### When things break (and they will)
If you can't connect, your best friend is the log file.
*   **FreeBSD:** `tail -f /var/log/maillog`
*   **Linux:** `tail -f /var/log/mail.log`

Read it while you try to connect. It will tell you exactly why you were rejected ("Password mismatch", "Relay access denied", "Connection timed out").

-----

## Conclusion

We now have a fully functional hosting stack.

  * **LAMP/FAMP** serving the web.
  * **BIND** serving the names.
  * **Postfix/Dovecot** serving the mail.

We have built a miniature version of a Hosting Provider on a single server.
But this server is getting crowded. The web server runs as `www`. The mail server runs as `vmail`. They share the same library versions, the same `/tmp`, and the same kernel.

If a hacker exploits a vulnerability in WordPress, can they read your emails? On this setup... yes.

It is time to **Isolate**.

In the next article, we will take a chainsaw to this monolith. We will introduce **FreeBSD Jails** and **Linux Chroot/Containers**, learning how to cage these services so they can never touch each other.