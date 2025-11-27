

# The Day-1 Server Security Checklist: Building the Perimeter on Linux and FreeBSD

**Go beyond keys. Silence the noise, block the bots, and automate your network defenses.**




In **Part 1** of this series, ["The Day-1 Server Security Checklist: Hardening SSH,"](https://d3ep0ps.com/#part1_ansible_ssh) we secured the **Identity Layer**. We ensured that the only way to log into your server is with a cryptographically secure key, and that the root account is unreachable via SSH.

Your server's front door has an unpickable lock.

But the path to that door is visible to the entire internet. Every second, automated scanners are rattling the handle, testing port 22, flooding your logs with failed authentication attempts. This noise masks real threats and wastes server resources.

It is time to build a wall around the server and post a guard at the gate.

In **Part 2**, we focus on the **Network Perimeter**. We will implement "defense in depth" by changing the default SSH port, deploying a "default-deny" firewall (UFW on Linux, PF on FreeBSD), and setting up active defenses with Fail2Ban, all automated with Ansible.

## ⚠️ The Rules of Engagement: Read This First

Before we begin, a critical warning. We are about to change the port you use to connect to your server and turn on a system designed to block network traffic.

**Mistakes here are catastrophic.** If you change the SSH port in the config but forget to open it in the firewall, or if you enable the firewall before allowing SSH, **you will lock yourself out completely.** There is no "undo" button once the connection is dropped.

Follow these rules to stay safe:

1.  **Never work on a production server first.** Use a local VM (like Vagrant) or a disposable cloud instance to test your playbooks.
2.  **Always keep one root session open.** Before applying changes that restart SSH or enable a firewall, keep one active terminal connection open to the server. If you lock yourself out, this existing session might be your only way back in to fix it.
3.  **Order matters.** In our automation, we must ensure the new SSH port is allowed in the firewall *before* we change the SSH configuration and restart the service.
4.  **Test your access immediately.** After applying any changes, open a *new* terminal window and try to log in on the new port. Do not close your original session until you have confirmed the new configuration works.



## 1\. The Great Debate: Changing the SSH Port

Before we touch a firewall, we must address the most common, and controversial, first step in server hardening: changing the SSH port from the default `22` to something else (e.g., `2222` or `5757`).

Critics argue this is "security through obscurity." They are right. A determined attacker will port-scan your entire server and find SSH wherever you hide it. Changing the port does not stop a targeted attack.

**So why do we do it?**

We do it for **signal-to-noise ratio**.

99% of the attacks hitting your server are dumb scripts scanning the entire IPv4 address space for port 22. By moving your SSH service to a high, non-standard port, you instantly drop off the radar of these low-effort bots.

Your logs go from thousands of failed login attempts per hour to near zero. This means when you *do* see a failed login in your logs, you know it is a real anomaly worth investigating. It turns noise into signal.

### The Ansible Implementation

Changing the port is a simple edit to `/etc/ssh/sshd_config`, followed by a service restart. We use a variable for the port number so it's easy to change later.

```yaml
# playbook.yml snippet
- name: Change SSH port to a non-standard port
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?Port\s+'
    line: "Port {{ ssh_custom_port }}"
    validate: '/usr/sbin/sshd -t -f %s'
  notify: Restart SSH
```



## 2\. The Philosophy of the Firewall: Default Deny

A firewall is your digital border guard. It inspects every network packet trying to enter or leave your server.

The most critical concept in firewall design is the **Default Deny** policy.

  * **Bad Policy (Default Allow):** Everything is allowed in, except for the things I explicitly block. (This is hard to manage and insecure).
  * **Good Policy (Default Deny):** Everything is blocked, except for the specific traffic I explicitly allow.

We will configure our firewalls to drop all incoming connection requests by default, and then drill a single, precise hole for our custom SSH port.



## 3\. The Linux Implementation: UFW (Uncomplicated Firewall)

On Linux (specifically Ubuntu/Debian, our target), the underlying kernel firewall is complex (`nftables` or `iptables`). To make it manageable, we use an abstraction layer called **UFW**.

UFW is designed to be simple and human-readable. It fits the "calm engineering" ethos perfectly.

### The Ansible Approach

Ansible has an excellent built-in module for UFW that makes declarative configuration easy. Notice the order: we allow the custom port *before* enabling the firewall.

```yaml
# roles/firewall_linux/tasks/main.yml
---
- name: Ensure UFW is installed
  apt:
    name: ufw
    state: present

- name: Set default policies (Default Deny)
  community.general.ufw:
    policy: deny
    direction: incoming

- name: Allow outgoing traffic
  community.general.ufw:
    policy: allow
    direction: outgoing

- name: Allow SSH on custom port
  community.general.ufw:
    rule: allow
    port: "{{ ssh_custom_port }}"
    proto: tcp
    comment: "Allow custom SSH access"

# Crucial: Enable the firewall last, after rules are set.
- name: Enable UFW firewall
  community.general.ufw:
    state: enabled
```



## 4\. The FreeBSD Implementation: PF (Packet Filter)

FreeBSD takes a different approach. It doesn't need "uncomplicated" wrappers because its native firewall, **PF (Packet Filter)**, is legendary for its clean, readable, and powerful configuration syntax.

Unlike Linux, where we use an Ansible module to run commands, on FreeBSD we manage the firewall by templating its configuration file directly. This highlights a key difference in managing the two OSs.

### The Ansible Approach

We define our rules in a template file (`pf.conf.j2`) and deploy it to `/etc/pf.conf`.

**The Template (`templates/pf.conf.j2`):**

```pf
# /etc/pf.conf - Managed by Ansible

# Macros
# Define the external interface dynamically
ext_if = "{{ ansible_default_ipv4.interface }}"
# Use our custom port variable
ssh_port = "{{ ssh_custom_port }}"

# Options
set block-policy drop
set skip on lo0

# Normalization
scrub in on $ext_if all fragment reassemble

# --- THE GOLDEN RULE: DEFAULT DENY ---
# Block everything coming in by default
block in all
# Allow all traffic going out
pass out all keep state

# Drill a hole: Allow SSH on our custom port
pass in on $ext_if proto tcp from any to any port $ssh_port keep state
```

**The Playbook Task:**
We use the `template` module to deploy the file and a handler to reload PF. We must also ensure the PF service is enabled in `/etc/rc.conf`.

```yaml
# roles/firewall_freebsd/tasks/main.yml
---
- name: Enable PF in rc.conf
  service:
    name: pf
    enabled: yes

- name: Deploy PF configuration template
  template:
    src: pf.conf.j2
    dest: /etc/pf.conf
    mode: '0600'
    # CRITICAL: Validate syntax before saving to prevent lockouts!
    validate: 'pfctl -nf %s'
  notify: Reload PF
```



## 5\. Active Defense: Fail2Ban

We have moved SSH and built a firewall. Our logs are quieter, and only one port is open.

But attackers can still find that port and start trying keys. A static firewall rule allows them to try forever. We need active defense. We need a bouncer.

**Fail2Ban** is that bouncer. It monitors log files (like authentication logs) looking for patterns of failure. If an IP address fails to log in too many times in a short period, Fail2Ban dynamically updates the firewall (UFW or PF) to ban that IP address for a set amount of time.

### The Ansible Approach

We install Fail2Ban and deploy a local configuration file that defines our "jail" rules.

```yaml
# roles/fail2ban/tasks/main.yml
---
- name: Ensure Fail2Ban is installed
  package:
    name: fail2ban
    state: present

- name: Deploy Fail2Ban jail configuration
  template:
    src: jail.local.j2
    dest: /etc/fail2ban/jail.local
  notify: Restart Fail2Ban
```

In our template, we tell Fail2Ban to monitor SSH on our new *custom port*.



## 🧪 Hands-on Lab: Building the Perimeter (Safely)

We are about to change SSH ports and enable firewalls. **This is high-risk.**

Do not run this on a production server without testing first. I have created a new GitHub repository, **`perimeter-lab`**, with a complete Ansible playbook that safely orchestrates all these steps on Linux and FreeBSD Vagrant boxes.

**👉 Clone the Lab:** [github.com/d3ep0ps/security-lab](https://github.com/d3ep0ps/security-lab)

This lab will:

1.  Spin up local Linux and FreeBSD VMs.
2.  Change their SSH port to `2222`.
3.  Configure and enable the appropriate firewall (UFW or PF), ensuring the new port is open.
4.  Install and configure Fail2Ban to watch the new port for intruders.

## The End State

By combining Part 1 and Part 2, you have transformed a vulnerable default installation into a hardened Unix bastion.

  * **Identity:** Only key-based authentication is allowed.
  * **Obscurity:** The SSH service is hidden on a non-standard port, silencing 99% of noise.
  * **Perimeter:** A "default-deny" firewall blocks all other access.
  * **Active Defense:** Any IP that finds the open port and abuses it is automatically banned.

This is the baseline security posture for any internet-facing system.