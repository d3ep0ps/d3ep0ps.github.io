# The Bootstrap Paradox: BIND, Glue Records, and the Registry

> **You cannot be found if you do not exist. But you cannot exist until you are found.**

In the previous article, we built a LAMP/FAMP stack. We have a web server listening on an IP address. But users don't type IP addresses; they type names.

To turn `192.0.2.10` into `d3ep0ps.com`, we need DNS.

Most modern engineers treat DNS as a SaaS commodity. You buy a domain on Route53 or GoDaddy, you point the NS records to their clouds, and you forget about it. You become a **Consumer** of DNS.

But to understand how the internet actually works, you must become a **Provider**. You must run your own Authoritative Name Server.

And when you try to do that—when you try to host `d3ep0ps.com` using a nameserver called `ns1.d3ep0ps.com`—you hit a logical wall that has broken the brain of every junior sysadmin since the 1980s: **The Bootstrap Paradox.**

To solve it, we have to look at the layer above us: **The Registry.**



## 1\. The View from the Top: I Managed the Phonebook

Before we install BIND, let me share a perspective most admins never see.

Years ago, one of my duties was to manage the `lviv.ua` zone.
I wasn't just configuring my own domains; I was responsible for the **Registry** that other people relied on. I maintained the database that said, "If you are looking for `company.lviv.ua`, go talk to these nameservers."

From that vantage point, you realize that DNS isn't magic. It is a strict hierarchy of delegation. And that hierarchy relies on a protocol older than the web itself: **WHOIS**.

### WHOIS is Not Just About Privacy

Most people think WHOIS is just a tool to see who owns a domain (or to hide from spam).
But technically, WHOIS (running on **TCP Port 43**) is the database of the Registry.

When you buy a domain, your Registrar sends data to the Registry's WHOIS database.
The Registry then uses that database to **generate the TLD Zone File**.

If your nameservers aren't in the WHOIS database, they aren't in the `.com` (or `.ua`) zone file. And if they aren't there, you don't exist.



## 2\. The Bootstrap Paradox (Glue Records)

Here is the problem.

1.  I buy `d3ep0ps.com`.
2.  I want to host my own DNS. So I build a server at IP `1.2.3.4`.
3.  I name this server `ns1.d3ep0ps.com`.
4.  I tell the Registry: "The nameserver for `d3ep0ps.com` is `ns1.d3ep0ps.com`."

Now, imagine a user trying to visit my site.

1.  **User:** "Hello `.com`, where is `d3ep0ps.com`?"
2.  **Registry (.com):** "Go ask `ns1.d3ep0ps.com`."
3.  **User:** "Great. Where is `ns1.d3ep0ps.com`?"
4.  **Registry:** "It's inside `d3ep0ps.com`. Go ask the nameserver for `d3ep0ps.com`."
5.  **User:** "But... that's the server I'm trying to find\!"

**This is the circular dependency.** You cannot resolve the nameserver's name because it lives inside the domain it is supposed to serve.

### The Solution: Glue Records

To break the loop, the Registry allows you to "cheat."
When you define your nameserver at the Registrar level, you provide **both the Name AND the IP Address**.

The Registry takes that IP and injects it directly into the TLD zone file as an "A Record" alongside the "NS Record." This "glues" the name to the IP, allowing the resolver to skip the lookup and go straight to your IP.

**The Lesson:** If you change the IP of your BIND server, you can't just update your zone file. You must update the **Registrar (WHOIS)**, or your site will go dark.



## 3\. Installation: Linux vs FreeBSD

Now that the Registry knows about us (via Glue Records), we need to set up the software to answer the queries. We will use **BIND** (Berkeley Internet Name Domain), the standard since the dawn of the internet.

### Linux (Debian/Ubuntu)

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc
```

  * **Config:** `/etc/bind/`
  * **Zone Files:** `/var/lib/bind/` (usually)
  * **Security:** Runs as user `bind`, but often sits in the default filesystem namespace unless you configure AppArmor.

### FreeBSD (The Secure Default)

FreeBSD treats BIND with extreme caution.

```bash
sudo pkg update
sudo pkg install bind916 bind916-tools bind916-doc
```

  * **Config:** `/usr/local/etc/namedb/`
  * **The Difference:** FreeBSD strongly encourages (and often defaults) to running BIND in a **chroot jail**.
    It runs inside `/var/named/`.
      * `/etc` inside the jail is actually `/var/named/etc/`.
      * If a hacker exploits a vulnerability in BIND, they find themselves trapped in a directory with nothing but zone files. They cannot see `/etc/passwd` or access the rest of the OS.

To enable it in `/etc/rc.conf`:

```bash
sysrc named_enable="YES"
sysrc named_chrootdir="/var/named"
```



## 4\. Configuration: The Anatomy of Authority

We need to tell BIND two things:

1.  **Global Options:** Who can talk to us?
2.  **The Zone:** What records do we hold?

### named.conf (Global Options)

The most critical setting for a hosting server is **Turning Recursion OFF**.

**The Hosting Provider Nightmare:**
If you allow "Recursion: Yes" and "Allow-query: Any", your server will answer queries for *any* domain (like google.com). Attackers will use your server to launch **DNS Amplification DDoS attacks**.

Your config should look like this (secure):

```text
options {
    directory "/usr/local/etc/namedb/working"; # FreeBSD path
    recursion no;              # DO NOT be a resolver for the world
    allow-transfer { none; };  # Don't let strangers steal your zone
    listen-on-v6 { any; };     # It's 2025. Use IPv6.
};
```

### The Zone File (`d3ep0ps.com.db`)

This is the database of your domain.

```text
$TTL    86400
@       IN      SOA     ns1.d3ep0ps.com. admin.d3ep0ps.com. (
                        2025120401      ; Serial (YYYYMMDDnn)
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL

; Name Servers (NS Records)
@       IN      NS      ns1.d3ep0ps.com.
@       IN      NS      ns2.d3ep0ps.com.

; Glue Records (Self-Reference)
ns1     IN      A       192.0.2.10
ns1     IN      AAAA    2001:db8::10

; Web Server
@       IN      A       192.0.2.10
www     IN      CNAME   @
```

**The Ritual of the Serial Number:**
Notice the serial: `2025120401` (Date + 01).
Every time you edit this file, you **must** increment this number. Secondary nameservers (slaves) look at this number. If it hasn't increased, they will ignore your changes. I have seen outages last for days because an admin forgot to bump the integer.



## 5\. Testing: Beyond Ping

How do you know it works? `ping` is useless here. We need `dig`.

**1. Syntax Check (Before Restarting)**
Never restart BIND without checking your syntax.

```bash
named-checkconf
named-checkzone d3ep0ps.com /usr/local/etc/namedb/master/d3ep0ps.com.db
```

**2. The Local Test**
Ask your localhost specific questions.

```bash
dig @localhost d3ep0ps.com SOA
```

You should see your "AUTHORITY SECTION" with the data you just typed.

**3. The "Trace" (The Real Internet Test)**
This is the ultimate test. It bypasses your cache and walks the tree from the Root (.) to the TLD (.com) to you.

```bash
dig +trace d3ep0ps.com
```

Watch the output.

1.  It asks the Root Servers ("Where is .com?").
2.  It asks the .com Servers ("Where is d3ep0ps?"). -\> **This step works because of the Glue Record\!**
3.  It asks Your Server ("Where is www?").

If the chain breaks at step 2, your Glue Records (WHOIS) are wrong.
If it breaks at step 3, your BIND config/Firewall is wrong.



## Conclusion

We have now solved the paradox.

1.  We told the **Registry** where our server is (Glue).
2.  We told our **Server** who it is (BIND).
3.  The world can now find `d3ep0ps.com`.

>If you like to dive deeper into the subject, I recommend reading the [official BIND documentation](https://www.isc.org/bind/) and the book [DNS and BIND](https://www.oreilly.com/library/view/dns-and-bind/0596100574/) by Cricket Liu, Paul Albitz.

Our server acts as a Web Host (Article 1) and a Nameserver (Article 2).
But if you try to send an email to `admin@d3ep0ps.com`, it will vanish.

Hosting email is the hardest task in system administration. It requires more than just software; it requires **Reputation**.

**Next Up:** "The Mail Stack: Postfix, Dovecot, and the War on Spam."