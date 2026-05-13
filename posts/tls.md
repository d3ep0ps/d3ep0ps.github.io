# TLS From First Principles: How the Internet Learned to Keep Secrets

> **"Encryption is not about hiding something shameful. It is about the right to have a private conversation in a world full of strangers."**

In **The Gateway** article, we configured Traefik to terminate TLS at the edge of our cluster. In the **Mail Server** article, we told Postfix and Dovecot to "use TLS" and moved on. In both cases, we used the technology without truly explaining it.

Today we fix that.

By the end of this article, you will not just know *how to install* a certificate. You will understand *why the whole system works* — what a certificate actually is, who signs it and why that matters, and how a browser and a server negotiate a secret key in plain sight of the entire internet without anyone being able to steal it.

We will then take that understanding and apply it systematically: Apache, Nginx, Postfix, Dovecot, Kubernetes, and GCP. One mental model. Every layer of the stack.

---

## 1. The Problem: The Internet Was Born Trusting

The early internet was built by academics sharing research. Nobody was trying to steal anyone's passwords, because there were almost no passwords worth stealing.

HTTP — the Hypertext Transfer Protocol — was designed in that world. It is a plain-text protocol. Every byte you send over HTTP travels across routers, cables, and switches in a format that any machine along the path can read.

To see how dangerous this is, run this on your local network while someone connects to an HTTP site:

```bash
# As root — capture HTTP traffic on your interface
sudo tcpdump -i eth0 -A 'tcp port 80' | grep -i 'password\|user\|login\|cookie'
```

You will see credentials in cleartext. Not as a theoretical exercise — as a live demonstration. This is how coffee-shop attacks work. This is how hotel Wi-Fi harvesting works. This is why HTTP is effectively dead for anything that matters.

The three threats TLS is designed to defeat are precise:

**Eavesdropping** — A passive attacker reads your traffic without interfering. You never know they were there. Your session cookie, your private message, your API key — all visible.

**Tampering** — An active attacker sits between you and the server (a "Man-in-the-Middle") and modifies the data in transit. They can inject malicious JavaScript into a page, alter a bank transfer amount, or replace a software download with malware — and without TLS, neither you nor the server has any way to detect it.

**Impersonation** — An attacker stands up a server that *claims* to be your bank. Without a way to verify identity, your browser has no mechanism to tell the real server from the fake one.

TLS — Transport Layer Security — solves all three simultaneously. It provides **confidentiality** (encryption defeats eavesdropping), **integrity** (message authentication codes defeat tampering), and **authentication** (certificates defeat impersonation).

---

## 2. How TLS Works: Three Acts

Understanding TLS requires understanding three ideas that build on each other. Each one is simple. Together, they are elegant.

### Act I — The Asymmetric Lock

Classic symmetric encryption uses one key: the same key encrypts and decrypts. If you want to encrypt a message for your friend, you both need the same key — and you face a classic paradox: *how do you securely share the key in the first place?* If you could share a secret over an insecure channel, you wouldn't need the key.

Asymmetric (or "public-key") cryptography breaks this paradox with a clever trick: it uses *two mathematically linked keys*. What one encrypts, only the other can decrypt — and knowing one key tells you nothing useful about the other.

Think of it as a padlock:

- The **public key** is an *open padlock*. You give copies to everyone. Anyone can snap it shut over a message.
- The **private key** is the *only key that opens it*. You keep it secret, always, forever.

If someone wants to send you a secret, they drop it in a box and snap your public padlock shut. Now *only you* can open it. They cannot even re-open what they just locked.

This is how your browser sends a secret to a server without ever transmitting the secret itself.

### Act II — The Handshake

The TLS handshake is the negotiation that happens before a single byte of your actual data is transmitted. It takes milliseconds and accomplishes three things: agreeing on which encryption algorithms to use, authenticating the server's identity, and establishing a shared secret key — all over an unencrypted channel.

Here is the simplified flow of a modern TLS 1.3 handshake:

```
Client                                          Server
  |                                               |
  |──── ClientHello ──────────────────────────>   |
  |     (Cipher suites, Key shares, Random)       |
  |                                               |
  |   <──── ServerHello + Certificate ────────    |
  |     (Chosen suite, Key share,                 |
  |      server's certificate, Finished)          |
  |                                               |
  |  [Client verifies cert and derives secret]    |
  |                                               |
  |──── Finished ──────────────────────────────>  |
  |                                               |
  |  <=== Encrypted Application Data ====>        |
```

The key exchange step deserves a moment of appreciation. Using **Diffie-Hellman key exchange**, the client and server each generate a temporary key pair, exchange public values, and mathematically combine them with their own private value. The result — on both sides independently — is the identical shared secret. An eavesdropper who recorded every byte of this exchange cannot derive the secret, because they never saw either private value.

From this point on, all communication is encrypted with a fast symmetric cipher (typically AES-256-GCM), using the shared secret as the key. Asymmetric cryptography was used only for the handshake; the heavy lifting of bulk data encryption is done symmetrically.

### Act III — The Certificate and the Chain of Trust

The handshake above has one unsolved problem: when the server presents its public key, how does the client know it belongs to the *real* server, and not an impersonator?

This is the job of the **certificate**.

A TLS certificate is a document that binds a public key to an identity (a domain name). Crucially, this document is **digitally signed** — meaning a trusted third party has cryptographically endorsed it.

The structure is a chain:

```
Root CA Certificate
  └── Intermediate CA Certificate (signed by Root)
        └── Your Server Certificate (signed by Intermediate)
```

**Root CAs** (Certificate Authorities) are organisations whose public keys are pre-installed in your operating system and browser — Mozilla, Google, Apple, and Microsoft all maintain a list. You trust them by default, implicitly, because your OS vendor has vetted them.

**Intermediate CAs** exist for security: Root CA private keys are kept offline in vaults. Day-to-day signing is done by Intermediates, which the Root has endorsed.

**Your certificate** is signed by an Intermediate. When your browser receives it, it walks the chain: *"Does Intermediate X trust this cert? Does Root Y trust Intermediate X? Is Root Y in my trust store?"* If all three checks pass, the identity is verified.

Beyond the chain, the browser checks six things for every certificate:

1. **Validity period** — the `Not Before` and `Not After` timestamps. Expired certs are hard-rejected.
2. **Hostname match** — the domain in the cert must match the domain you navigated to (`CN` field or `Subject Alternative Names`).
3. **Signature validity** — the cryptographic signature must verify correctly against the issuer's public key.
4. **Chain completeness** — every link in the chain must be valid and trusted.
5. **Revocation** — the cert must not appear on a CRL (Certificate Revocation List) or OCSP response as revoked.
6. **Certificate Transparency logs** — modern browsers require certs to be logged in public CT logs, making it impossible to issue a certificate for a domain without it being publicly auditable.

Fail any of these, and the browser shows the red warning. All six must pass for the padlock to appear green.

---

## 3. The Certificate Landscape: Who Signs What

Now that we understand what a certificate is, the question becomes: *who should sign yours?*

### Self-Signed Certificates

A self-signed certificate is one where *you* are the issuer. You generate a key pair, create a certificate, and sign it with your own private key. There is no chain of trust — just your word that you are who you say you are.

```bash
# Generate a self-signed certificate valid for 365 days
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/ssl/private/self.key \
  -out /etc/ssl/certs/self.crt \
  -subj "/CN=internal.d3ep0ps.local"
```

Browsers will warn users. External clients will reject them by default. But self-signed certificates are perfectly appropriate in several situations: internal services on a private network, inter-service mTLS inside a Kubernetes cluster (where you control all the clients), and development environments.

The rule of thumb: if you control *all* the clients, self-signed is fine. If any external client will connect, you need a CA-signed certificate.

### Commercial CAs (Paid Certificates)

Before Let's Encrypt existed, the only way to get a browser-trusted certificate was to pay a CA. They still exist, and for some use cases they are still the right answer.

Commercial CAs offer three validation levels:

**Domain Validation (DV)** — The CA verifies that you control the domain (by checking a DNS record or serving a file). This is the same level of validation Let's Encrypt performs. You pay for the convenience of longer validity periods and sometimes for the brand name, but the security level is identical to a free cert.

**Organisation Validation (OV)** — The CA additionally verifies that your organisation legally exists (company registration records, phone verification). The cert contains your company name. Costs more, but provides an auditable business identity.

**Extended Validation (EV)** — The most rigorous vetting. Historically displayed a green address bar with the company name. Modern browsers have largely dropped the visual distinction, which has eroded the value proposition of EV significantly.

The honest answer: for web servers, Let's Encrypt DV certificates are now the professional standard. OV/EV certificates are worth considering for high-value financial or government services where the organisational vetting record matters for compliance.

### Let's Encrypt: The ACME Revolution

Let's Encrypt, launched in 2016, is a non-profit CA that issues DV certificates for free, automatically, at scale. It is now the largest CA in the world by certificates issued, having fundamentally changed the industry.

The automation is powered by the **ACME protocol** (Automatic Certificate Management Environment), now an IETF standard (RFC 8555). ACME works by issuing a **challenge**: the CA tells your server to prove it controls the domain by either:

- **HTTP-01**: Serve a specific file at `http://yourdomain.com/.well-known/acme-challenge/<token>`
- **DNS-01**: Create a DNS TXT record `_acme-challenge.yourdomain.com` with a specific value

Once the challenge is satisfied, the CA issues the certificate. The entire process takes seconds and is fully automated by tools like **Certbot**.

Let's Encrypt certificates are valid for **90 days**. This is a deliberate security design decision, not a limitation. Short-lived certificates limit the damage window of a compromised key — if your private key is stolen, the attacker's window is at most 90 days, not years. They also force automation: you cannot maintain a fleet of 90-day certificates manually, so you are pushed toward proper certificate lifecycle management from day one.

### Comparison at a Glance

| | Self-Signed | Commercial DV | Let's Encrypt | Commercial OV/EV |
|---|---|---|---|---|
| **Cost** | Free | $10–$200/yr | Free | $100–$1000+/yr |
| **Browser trust** | ❌ (warning) | ✅ | ✅ | ✅ |
| **Automation** | Manual | Manual/API | Fully automated | API (varies) |
| **Validity** | Any | 1 year | 90 days | 1–2 years |
| **Identity verified** | None | Domain only | Domain only | Domain + Org |
| **Best for** | Internal/Dev | Legacy systems | Everything public | Finance/Compliance |

---

## 4. TLS Across the Stack

TLS is not just an HTTPS thing. It is the transport security layer for nearly every protocol that carries sensitive data. The mental model is always the same — a handshake, a certificate, a shared secret — but the specifics vary per protocol.

### Web (HTTPS)

When you navigate to `https://d3ep0ps.com`, your browser performs the full handshake described in Section 2. The web server (Apache or Nginx) terminates TLS, decrypts your request, and responds. In a cluster architecture like ours, TLS is terminated at **The Gateway** (Traefik or Nginx), not at each individual service — keeping internal east-west traffic lightweight.

HTTPS operates on **port 443** by convention. Port 80 (HTTP) should be kept alive only to redirect to 443:

```nginx
server {
    listen 80;
    server_name d3ep0ps.com;
    return 301 https://$host$request_uri;
}
```

### Mail (STARTTLS vs. Implicit TLS)

Mail is where TLS gets confusing, because there are two distinct mechanisms and three ports.

**STARTTLS** is an upgrade mechanism. A client connects on a plain-text port (25 for server-to-server SMTP, 587 for submission) and issues a `STARTTLS` command mid-session to upgrade the connection to TLS. The connection starts unencrypted and becomes encrypted.

**Implicit TLS** (sometimes called "TLS-wrapped" or "SMTPS/IMAPS") starts TLS immediately, before any protocol commands are exchanged. This is the modern preferred approach.

| Port | Protocol | TLS Method | Used For |
|---|---|---|---|
| 25 | SMTP | STARTTLS | Server-to-server relay |
| 465 | SMTPS | Implicit TLS | Client submission (legacy, now preferred) |
| 587 | SMTP Submission | STARTTLS | Client submission |
| 993 | IMAPS | Implicit TLS | IMAP client access |
| 995 | POP3S | Implicit TLS | POP3 client access |

We configured both Postfix and Dovecot in the **Mail Server** article. In Section 5.2 below, we will add proper TLS to that stack.

### DNS over TLS (DoT)

In the **BIND DNS** article, we ran an authoritative and recursive resolver. Standard DNS (port 53) is also a plaintext protocol — your DNS queries tell an observer exactly what sites you are visiting, even if those sites are served over HTTPS.

**DNS over TLS** (DoT, port 853) wraps DNS queries in a standard TLS session, providing confidentiality and integrity. **DNSSEC** (which we touched on in the BIND article) is a different and complementary mechanism: DNSSEC signs DNS *responses* to prevent tampering with the data itself, but it does not encrypt them. DoT and DNSSEC solve different problems and should be used together.

Configuring BIND for DoT requires a TLS certificate and a `listen-on` directive with `tls` enabled:

```bash
# /etc/named.conf snippet
options {
    listen-on port 853 tls local-tls { any; };
};

tls local-tls {
    cert-file "/etc/letsencrypt/live/dns.d3ep0ps.com/fullchain.pem";
    key-file  "/etc/letsencrypt/live/dns.d3ep0ps.com/privkey.pem";
};
```

---

## 5. Practice: Let's Encrypt Across the Entire Stack

Theory understood. Now we install.

All four practice sections assume a Debian/Ubuntu-based Linux host unless noted otherwise. The certificate management tool we will use is **Certbot** — the reference ACME client maintained by the Electronic Frontier Foundation.

---

### 5.1 — Web: Apache and Nginx with Certbot

**Install Certbot:**

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx python3-certbot-apache
```

**For Nginx — automatic mode:**

Certbot's `--nginx` plugin reads your existing Nginx config, obtains the certificate, and modifies the config to enable TLS — all in one command:

```bash
sudo certbot --nginx -d d3ep0ps.com -d www.d3ep0ps.com
```

Certbot will ask for your email address (for expiry notices), agree to terms, and handle the HTTP-01 challenge automatically. After success, it rewrites your server block to something equivalent to:

```nginx
server {
    listen 443 ssl;
    server_name d3ep0ps.com www.d3ep0ps.com;

    ssl_certificate     /etc/letsencrypt/live/d3ep0ps.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/d3ep0ps.com/privkey.pem;

    # Certbot adds these for strong TLS configuration:
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Your existing location blocks follow...
}

# HTTP → HTTPS redirect (also added automatically):
server {
    listen 80;
    server_name d3ep0ps.com www.d3ep0ps.com;
    return 301 https://$host$request_uri;
}
```

**For Apache — automatic mode:**

```bash
sudo certbot --apache -d d3ep0ps.com -d www.d3ep0ps.com
```

The `--apache` plugin similarly reads your VirtualHost config and creates a new TLS-enabled VirtualHost automatically.

**Verify the certificate:**

```bash
# Inspect the live certificate from the outside
openssl s_client -connect d3ep0ps.com:443 -servername d3ep0ps.com </dev/null 2>/dev/null \
  | openssl x509 -noout -dates -subject -issuer
```

You should see your domain in the subject, Let's Encrypt as the issuer, and a validity window of ~90 days from today.

**Test automatic renewal:**

Certbot installs a systemd timer (or cron job) that runs twice daily and renews any cert expiring within 30 days. Test that the renewal pipeline works:

```bash
sudo certbot renew --dry-run
```

A successful dry run means your renewal automation is working. Run this after every system change that might affect it (firewall rules, web server restarts, port changes).

**Handle certificate renewal and reloading:**

One critical detail: after a certificate is renewed, the web server must be **reloaded** to pick up the new files.

- If you used the `--nginx` or `--apache` plugins, Certbot handles this reload automatically.
- If you used `certonly` or a manual method, you need a renewal hook, similar to what we will set up for the mail server in the next section:

```bash
# Example of adding a manual reload hook if not using plugins
sudo certbot renew --deploy-hook "systemctl reload nginx"
```

---

### 5.2 — Mail: Postfix + Dovecot

The good news: **you do not need a second certificate**. The same Let's Encrypt certificate issued for your mail domain (`mail.d3ep0ps.com`) covers both Postfix and Dovecot. Issue it once:

```bash
# Use standalone mode if your web server isn't running on this host,
# or --nginx/--apache if it is (adjust the domain to your mail hostname)
sudo certbot certonly --standalone -d mail.d3ep0ps.com
```

The certificate lands at `/etc/letsencrypt/live/mail.d3ep0ps.com/`.

**Configure Postfix (SMTP):**

Edit `/etc/postfix/main.cf` and add or update the TLS directives:

```ini
# /etc/postfix/main.cf

# TLS for incoming connections (STARTTLS on port 25 and 587)
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.d3ep0ps.com/fullchain.pem
smtpd_tls_key_file  = /etc/letsencrypt/live/mail.d3ep0ps.com/privkey.pem
smtpd_tls_security_level = may          # offer TLS, but don't require it for inbound
smtpd_tls_loglevel = 1                  # log TLS handshakes

# TLS for outgoing connections (when Postfix sends to other servers)
smtp_tls_cert_file = /etc/letsencrypt/live/mail.d3ep0ps.com/fullchain.pem
smtp_tls_key_file  = /etc/letsencrypt/live/mail.d3ep0ps.com/privkey.pem
smtp_tls_security_level = may           # use TLS when the remote server supports it
smtp_tls_loglevel = 1

# Disable obsolete protocols
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_protocols  = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
```

Enable the submission port (587) with STARTTLS in `/etc/postfix/master.cf` — uncomment or add:

```
submission inet n - y - - smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
```

Reload Postfix:

```bash
sudo systemctl reload postfix
```

**Configure Dovecot (IMAP/POP3):**

Edit `/etc/dovecot/conf.d/10-ssl.conf`:

```ini
# /etc/dovecot/conf.d/10-ssl.conf

ssl = required                          # require TLS for all connections

ssl_cert = </etc/letsencrypt/live/mail.d3ep0ps.com/fullchain.pem
ssl_key  = </etc/letsencrypt/live/mail.d3ep0ps.com/privkey.pem

# Disable obsolete protocols
ssl_min_protocol = TLSv1.2

# Strong cipher selection
ssl_cipher_list = ECDHE+AESGCM:ECDHE+CHACHA20:!aNULL:!eNULL:!EXPORT
ssl_prefer_server_ciphers = yes
```

Restart Dovecot:

```bash
sudo systemctl reload dovecot
```

**Verify Postfix STARTTLS:**

```bash
openssl s_client -connect mail.d3ep0ps.com:587 -starttls smtp
```

You will see the handshake, the certificate details, and then drop into an SMTP session. Look for `250-STARTTLS` in the server greeting and your Let's Encrypt cert in the certificate chain output.

**Verify Dovecot IMAPS:**

```bash
openssl s_client -connect mail.d3ep0ps.com:993
```

A successful connection shows the certificate and the IMAP greeting (`* OK Dovecot ready`).

**Handle certificate renewal for Postfix and Dovecot:**

Certbot renews certificates automatically, but Postfix and Dovecot need to be reloaded to pick up the new files. Add a renewal hook:

```bash
# /etc/letsencrypt/renewal-hooks/deploy/reload-mail.sh
#!/bin/bash
systemctl reload postfix
systemctl reload dovecot
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-mail.sh
```

Certbot runs all scripts in `renewal-hooks/deploy/` after every successful renewal. The mail stack will always have a fresh certificate without manual intervention.

---

### 5.3 — Kubernetes: cert-manager on GKE

In a Kubernetes cluster, you don't run Certbot. Certificate lifecycle management is handled by **cert-manager** — a Kubernetes-native controller that watches `Certificate` and `Issuer` resources and automates the entire ACME flow, storing the result as a Kubernetes `Secret`.

**Step 1 — Install cert-manager via Helm:**

```bash
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager with CRD installation
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

Verify all three pods are running:

```bash
kubectl get pods -n cert-manager
# NAME                                      READY   STATUS    RESTARTS
# cert-manager-xxxxxxxxx-xxxxx              1/1     Running   0
# cert-manager-cainjector-xxxxxxxxx-xxxxx   1/1     Running   0
# cert-manager-webhook-xxxxxxxxx-xxxxx      1/1     Running   0
```

**Step 2 — Create a ClusterIssuer (staging first):**

Always start with Let's Encrypt's staging environment. The staging CA is not browser-trusted, but its rate limits are far more generous. Test here, then switch to production.

```yaml
# cluster-issuer-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: zhhuta@gmail.com
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          class: nginx   # or 'gce' for GKE's built-in ingress
```

```bash
kubectl apply -f cluster-issuer-staging.yaml
```

Once staging works, create the production issuer:

```yaml
# cluster-issuer-prod.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: zhhuta@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

```bash
kubectl apply -f cluster-issuer-prod.yaml
```

**Step 3 — Annotate your Ingress:**

The simplest way to use cert-manager is to annotate your existing Ingress resource. cert-manager watches for the annotation and automatically creates a `Certificate` resource and triggers the ACME flow:

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: d3ep0ps-ingress
  annotations:
    # Tell cert-manager which issuer to use
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - d3ep0ps.com
    - www.d3ep0ps.com
    secretName: d3ep0ps-tls    # cert-manager will create and populate this Secret
  rules:
  - host: d3ep0ps.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: d3ep0ps-svc
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml
```

**Step 4 — Watch cert-manager work:**

```bash
# Watch the Certificate resource cert-manager creates automatically
kubectl get certificate -w

# Inspect the ACME challenge process
kubectl describe certificaterequest

# Once issued, the Secret is populated — verify it
kubectl get secret d3ep0ps-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 --decode | openssl x509 -noout -dates -subject
```

The `Certificate` resource will transition from `False` to `True` in the `Ready` column within 60–90 seconds on a healthy cluster. cert-manager then monitors the certificate and automatically renews it at 2/3 of its validity window — for a 90-day Let's Encrypt cert, renewal happens at approximately day 60.

---

### 5.4 — GCP: Google-Managed Certificates

GCP offers an alternative to cert-manager for GKE clusters using GKE's native Ingress (backed by a Google Cloud Load Balancer): **Google-managed certificates**.

With managed certs, GCP handles the entire certificate lifecycle — provisioning, renewal, and rotation — with zero configuration on your part beyond pointing at your domain.

**Option A — ManagedCertificate resource on GKE Ingress:**

```yaml
# managed-cert.yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: d3ep0ps-managed-cert
spec:
  domains:
    - d3ep0ps.com
    - www.d3ep0ps.com
```

```bash
kubectl apply -f managed-cert.yaml
```

Reference it from your Ingress annotation:

```yaml
# ingress-gcp.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: d3ep0ps-ingress
  annotations:
    networking.gke.io/managed-certificates: "d3ep0ps-managed-cert"
    kubernetes.io/ingress.class: "gce"    # GKE native ingress — NOT nginx
spec:
  rules:
  - host: d3ep0ps.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: d3ep0ps-svc
            port:
              number: 80
```

GCP provisions the certificate within 15–60 minutes (DNS must resolve to the Load Balancer IP first). Check the status:

```bash
kubectl describe managedcertificate d3ep0ps-managed-cert
# Look for Status.CertificateStatus: Active
```

**Option B — Cloud Run (zero configuration):**

If your service runs on **Cloud Run**, TLS on your custom domain is completely automatic. You map the domain, verify ownership with a DNS record, and GCP handles everything else:

```bash
# Map a custom domain to a Cloud Run service
gcloud run domain-mappings create \
  --service d3ep0ps-service \
  --domain d3ep0ps.com \
  --region us-central1
```

GCP will provision a Google-managed certificate and route HTTPS traffic to your service with no further action required.

**When to use which approach:**

| Scenario | Recommended Approach |
|---|---|
| GKE + nginx Ingress controller | cert-manager + Let's Encrypt |
| GKE + GCE Ingress (Cloud Load Balancer) | Google-managed certificates |
| Cloud Run with custom domain | GCP automatic (domain mapping) |
| Multi-cloud or on-premise Kubernetes | cert-manager + Let's Encrypt |
| Internal services, mTLS between pods | cert-manager with self-signed ClusterIssuer |

The decision comes down to control versus simplicity. cert-manager is more portable, more configurable (supports DNS-01 challenges, wildcard certs, custom CAs), and works identically on any Kubernetes cluster. Google-managed certificates require zero operational knowledge but tie you to GCP's ingress implementation.

---

## 6. Operational Discipline: Never Let a Certificate Expire

A certificate expiry in production is one of the most avoidable incidents in infrastructure. It is also one of the most common. The pattern is always the same: a certificate was renewed manually, the person who knew about it left the team, and nobody noticed until browsers started showing red warnings.

The correct answer is architecture, not vigilance: **automate renewal so humans are never in the loop**.

### Prometheus Alert for Certificate Expiry

Even with automated renewal, you want an early warning system. Add this alert rule to your Prometheus configuration:

```yaml
# prometheus/rules/tls.yml
groups:
- name: tls
  rules:
  - alert: TLSCertificateExpiringSoon
    expr: ssl_certificate_expiry_seconds{} < 86400 * 14
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "TLS certificate expiring soon on {{ $labels.instance }}"
      description: >
        Certificate for {{ $labels.domain }} expires in
        {{ $value | humanizeDuration }}. Renewal may be failing.

  - alert: TLSCertificateExpired
    expr: ssl_certificate_expiry_seconds{} < 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "TLS certificate EXPIRED on {{ $labels.instance }}"
```

This requires the `ssl_exporter` or `blackbox_exporter` (with the `tcp_tls` probe module) to expose the `ssl_certificate_expiry_seconds` metric. The 14-day warning gives you two weeks to investigate a stuck renewal before the cert actually expires.

### Test Your Renewal Pipeline

After any infrastructure change — new firewall rules, server migration, port change — always verify the renewal pipeline:

```bash
# Certbot: dry run renewal for all certs
sudo certbot renew --dry-run

# cert-manager: trigger a manual renewal check
kubectl cert-manager renew --all -n default

# Inspect an individual cert-manager Certificate
kubectl get certificate d3ep0ps-tls -o jsonpath='{.status.conditions[*]}' | jq
```

A renewal dry run exercises the full ACME flow (generating a challenge, serving it, having Let's Encrypt verify it) without actually issuing a certificate. If it passes, your renewal is healthy. Run this regularly, especially after firewall changes.

### The Architecture-Level Answer

Prometheus alerts and dry-run tests are safety nets. The real answer to certificate expiry is making manual intervention structurally impossible:

- On bare metal: Certbot + systemd timer with renewal hooks for every service that consumes the cert
- In Kubernetes: cert-manager's reconciliation loop monitors every `Certificate` resource continuously
- On GCP: Google-managed certificates are renewed automatically by the platform

If you find yourself manually copying certificate files between servers, stop. That process will fail the moment no one is watching. The cert-manager pattern — a controller that watches desired state and continuously reconciles it — is the correct abstraction for certificate management at scale.

---

## 7. What We Built

Starting from a plaintext network and three concrete threats, we traced TLS through its mathematical foundations — asymmetric encryption, the Diffie-Hellman handshake, the chain of trust — and then applied it at every layer of the stack we have built across this series:

- **Apache and Nginx** now terminate HTTPS with automatically-renewed Let's Encrypt certificates
- **Postfix and Dovecot** encrypt both client submission and inter-server relay, using the same certificate with automated reload on renewal
- **Kubernetes (GKE)** manages its own certificate lifecycle through cert-manager, with no human in the renewal loop
- **GCP** offers managed certificates for Cloud Load Balancer and zero-configuration TLS for Cloud Run

The pattern in every case is the same: trust is not assumed, it is cryptographically proven; and renewal is not a calendar event, it is an automated system property.

---

## Next Up

With transport security in place across every layer, the next question is: *what's actually inside your containers?* A TLS certificate proves who you are talking to. It says nothing about whether the code running on that server is trustworthy.

In **Security as Code: SBOM and Supply Chain**, we will look at software supply chain security — generating a Software Bill of Materials, scanning images for known vulnerabilities with `trivy`, signing images with Cosign, and integrating all of it into **The Forge** so that security is enforced at build time, not discovered in production.

---

*Published on [d3ep0ps.com](https://d3ep0ps.com) — From the Command Line to Intelligent Systems.*
