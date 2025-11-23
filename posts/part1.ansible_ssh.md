# The Day-1 Server Security Checklist: Hardening SSH on Linux and FreeBSD

**Stop logging in as root. Stop typing passwords. Stop trusting yourself to “remember” security. Automate it.**


In our previous article, ["Stop Clicking: The Journey to Infrastructure as Code,"](https://d3ep0ps.com/#iac) we learned how to spin up servers automatically using tools like Terraform and Vagrant.

We have built the infrastructure. Now we must secure it.

There is a sobering reality to deploying a server on the public internet: Within minutes of booting up, it will be port-scanned by automated bots looking for default credentials and weak configurations.

This isn't personal. It's automated. And if your defense relies on you manually remembering to edit config files on every new server, you will eventually fail.

Security is not an "add-on" for later. It is a Day-1 requirement that must be baked into your automation.

This article is Part 1 of a two-part series on securing Unix systems. Whether you are deploying Ubuntu Linux or running a hardened FreeBSD jail, the principles of identity are the same. Today, we focus on the **Identity and Access Layer**: securing *who* can log in and *how*, using SSH and Ansible.

## ⚠️ The Rules of Engagement: Read This First

Before we begin, a critical warning. We are about to modify the primary way you access your server (SSH).

**Mistakes here are catastrophic.** A typo in an SSH config file can lock you out completely. There is no "undo" button once the connection is dropped. You will likely have to destroy the server and start over.

Follow these rules to stay safe:

1.  **Never work on a production server first.** Use a local VM (like Vagrant) or a disposable cloud instance to test your playbooks.
2.  **Always keep one root session open.** Before applying changes that restart SSH, keep one active terminal connection open to the server. If you lock yourself out, this existing session might be your only way back in to fix it.
3.  **Test your access immediately.** After applying any changes, open a *new* terminal window and try to log in. Do not close your original session until you have confirmed the new configuration works.
4.  **Use Ansible's safety features.** We will use validation commands in our playbooks to check config file syntax before restarting services. Do not remove them.



## Phase 1: Local Preparation (The Identity Layer)

We cannot automate the creation of your private identity. This is the one manual step you must take on your local machine to establish *who you are*. You have two options for generating keys.

### Option A: The Standard (Ed25519 Software Keys)

If you are still using 2048-bit RSA keys, it is time to upgrade. The modern standard for software-based keys is **Ed25519**. It is smaller, faster, and more secure than older types.

On your local machine (macOS, Linux, or BSD), generate a new key pair:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

### Option B: The Gold Standard (Hardware-Backed FIDO Keys)

For the highest level of security, use a hardware security key like a YubiKey.

Modern versions of OpenSSH support FIDO/U2F tokens natively. When you generate this type of key, the private cryptographic material is generated on the hardware device and **never leaves it**.

To log in, you must have the physical device plugged into your computer *and* physically touch it to confirm your presence. This makes remote theft of your keys virtually impossible.

To generate a hardware-backed key (ensure your token is plugged in):

```bash
# The '-sk' stands for Security Key
ssh-keygen -t ecdsa-sk -C "yubikey_2025"
```

*(Note: On macOS or some Linux distributions, you may need to install specific middleware libraries like `libfido2` for this command to work.)*

### The Importance of Passphrases

Whichever option you choose, the prompt will ask for a passphrase. **Do not skip this.**

An SSH private key without a passphrase is just a file. If someone steals your laptop, they own your servers. A strong passphrase encrypts the key at rest.

### Living with Passphrases (ssh-agent)

Security fights with convenience. Typing a long passphrase every time you SSH is annoying. The solution is the **SSH Agent**, which holds your decrypted key securely in memory for your session.

  * Boot your laptop.
  * Run `ssh-add` and type your passphrase once.
  * Work all day without typing it again.



## The "Bootstrap" Problem: Getting In the First Time

Before we look at the Ansible code, we must address a common chicken-and-egg problem.

To run an Ansible playbook that secures a server, Ansible must first be able to log into that server with existing administrative privileges.

Every environment starts differently:

  * **Local VMs (Vagrant):** Usually provide a `vagrant` user with passwordless sudo pre-configured.
  * **VPS Providers (e.g., DigitalOcean):** Often provide a `root` login, either with a emailed password or an SSH key you selected during creation.
  * **Cloud Platforms (e.g., GCP, AWS):** These platforms use a tool called **cloud-init** that runs at first boot. It reads metadata you provide (like your SSH public key) and configures a default user (e.g., `ubuntu`, `ec2-user`) so you can log in.

**This is your "Bootstrap User."**

You will use this initial, often insecure user exactly once: to run the hardening playbook. The goal of the playbook is to use this temporary access to create your personalized, secure admin user and then slam the door on the insecure bootstrap methods.

In the lab instructions below, you will see where to configure this bootstrap user in Ansible's inventory file so it knows how to make that very first connection.



## Phase 2: The Ansible Approach to User Management

With Ansible, we define the desired state of our users in YAML. This approach works seamlessly across both Linux and FreeBSD. We need a non-root admin user with your Public SSH Key installed.

Hardening SSH is not about paranoia.
It’s about removing human error from the equation.
Systems should fail closed, not fail open. Automating identity is the first step toward predictable, safe infrastructure.


```yaml
# vars/main.yml
admin_user: "d3ep0ps_admin"
# Paste the content of your LOCAL ~/.ssh/id_ed25519.pub (or id_ecdsa_sk.pub) here
admin_ssh_pub_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."

# playbook.yml tasks
- name: Ensure admin user exists
  user:
    name: "{{ admin_user }}"
    # On Linux, 'sudo' group usually grants admin rights.
    # On FreeBSD, you typically add users to the 'wheel' group.
    groups: "{{ 'wheel' if ansible_os_family == 'FreeBSD' else 'sudo' }}"
    shell: /bin/sh
    append: yes
    state: present

- name: Deploy SSH public key for admin user
  authorized_key:
    user: "{{ admin_user }}"
    state: present
    key: "{{ admin_ssh_pub_key }}"
```



## Phase 3: Hardening the Daemon (sshd\_config as Code)

## 🚨 DANGER ZONE: SSH Configuration

**Warning:** This section modifies the SSH daemon's configuration. If you make a mistake and SSH restarts, you will be locked out. Ensure you have followed the "Rules of Engagement" above.

We need to edit `/etc/ssh/sshd_config` to enforce two critical settings:

1.  **`PasswordAuthentication no`**: Reject *any* attempt to log in with a password.
2.  **`PermitRootLogin no`**: Force attackers to guess a valid username *and* have that user's private key.

We use Ansible's `lineinfile` module to safely edit the file, and a **handler** to restart the SSH daemon only if a change occurs.

```yaml
# playbook.yml tasks
- name: Disable Password Authentication
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'
    # Validate command checks syntax BEFORE saving, preventing many lockouts
    validate: '/usr/sbin/sshd -t -f %s'
  notify: Restart SSH

- name: Disable Root Login
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin no'
    validate: '/usr/sbin/sshd -t -f %s'
  notify: Restart SSH

# handlers/main.yml
- name: Restart SSH
  service:
    name: sshd
    state: restarted
```

*(Note the `validate` command. This is a crucial safety feature. It runs `sshd -t` (test mode) against the new config file. If there is a syntax error, Ansible will fail and NOT save the broken file, saving you from a lockout.)*



## 🧪 Hands-on Lab: Lock Down a Server (Safely)

Don't just read this code—run it in a safe environment.

I have created a new GitHub repository, **`security-lab`**, that contains a complete Ansible playbook designed to work against both Linux and FreeBSD targets.

**👉 Clone the Lab:** [github.com/d3ep0ps/security-lab](https://github.com/d3ep0ps/security-lab)

### The Mission

Use a disposable virtual machine (like one created with Vagrant or a fresh cloud instance). Follow the instructions in the repository's `README.md` to configure your inventory and run the playbook.

**Your Goal:** After the playbook runs, verify that password login and root login now fail, and only your new key-based admin user can get in.

## Next Steps

Your server's identity layer is now secure. But the network door—port 22—is still wide open to the internet.

In **Part 2** of this series, we will enter another danger zone: **Building the Perimeter**. We will use Ansible to configure firewalls (UFW on Linux, PF on FreeBSD) and set up active defenses (Fail2Ban) to automatically block malicious IPs.