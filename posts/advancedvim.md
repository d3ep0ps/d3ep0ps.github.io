# The Sysadmin’s Editor in the Age of Cloud and IaC

> Why we still need Vim and Neovim when VS Code exists. Moving from basic edits to engineering workflows.


## The Circle Completed

In the last two articles—our "Day-1 Server Security Checklist" [Part 1](#part1_ansible_ssh) and [Part 2](#part2_ansible_ssh)—we locked down our servers. We edited SSH configurations, modified firewall rule sets, and created Fail2Ban jail definitions.

It was crucial, foundational work. But if you followed along, you noticed something: **security is mostly about editing text files.**

As we look ahead to Infrastructure as Code (IaC), the volume of text we manage is about to explode. We won't just be tweaking a few config lines; we will be writing hundreds of lines of Terraform HCL, complex Kubernetes YAML manifests, and intricate Ansible playbooks.

This brings us full circle to one of our very first topics: ["Mastering Text Editing on Unix."](#text_with_vi)

Back then, we covered the survival skills needed to edit a file in `vi` without panicking. But survival is no longer the goal. Engineering efficiency is. To manage modern infrastructure at scale, we need to upgrade our relationship with our primary tool: the editor.

## The "VS Code" Question

Before we dive back into the terminal, we have to address the elephant in the room.

A reasonable engineer might ask: *"Why bother with Vim in 2025? Why not just use VS Code? The Google Cloud Console even has a great web-based IDE built right in. Aren't these better tools?"*

The answer is yes—sometimes.

VS Code is a phenomenal piece of engineering. For developing a large application locally on your laptop, it's often the right choice. Cloud-based web IDEs are excellent for quick edits in a specific environment.

**But System Engineering doesn't just happen on your laptop.**

During an outage at 2 AM, no one launches VS Code.  
You SSH into a remote server, open Vim, and fix the problem.  
When everything is burning, your editor needs to be where the problem is — in the terminal, on the server, instantly available.


It happens on remote jump boxes. It happens inside ephemeral Docker containers debugging a crash. It happens over high-latency SSH connections during a 3 a.m. outage. It happens in environments where GUIs don't exist and dozens of megabytes of RAM for an Electron app aren't available.

In those critical moments, your fancy local IDE can't help you. You need something lightweight, ubiquitous, and powerful that lives directly in the terminal where the work is happening.

That tool is **Vim** (or its modern successor, **Neovim**).

The goal isn't to reject modern IDE features. The goal is to bring those features *into the terminal* so you have the same power, regardless of where you are working.

Here is how modern terminal editing has evolved to meet the demands of Infrastructure as Code.

-----

## 1\. Beyond Syntax Highlighting: Understanding Your Configs

When your infrastructure is defined by code, a typo isn't just a syntax error; it's a failed deployment, a security hole, or costly downtime. Relying on your eyes to catch every missing bracket in a 500-line YAML file is a recipe for disaster.

Modern development environments solved this problem with the **Language Server Protocol (LSP)**.

An LSP acts as a backend brain for your editor. It understands the deep structure and rules of a specific language. When you connect your editor to a language server for Terraform, for example, it doesn’t just colorize the text. It understands resources, data sources, and variables.

For the modern sysadmin using a tool like Neovim, this is transformative:

  * **Real-time Validation:** Your editor flags a syntax error or an invalid resource type *as you type it*, long before you run `terraform plan` or `kubectl apply`.
  * **Intelligent Autocomplete:** Forget tabbing through generic words. LSP provides context-aware suggestions for Kubernetes resource properties, Ansible module parameters, and Terraform resource attributes.
  * **Documentation on Hover:** Unsure what a specific field does in a Kubernetes manifest? Hover over it, and the documentation appears right in your terminal.

This isn't just convenience; it's error prevention at the source.

## 2\. Seeing the Structure: Treesitter

Traditional text editors use regular expressions for syntax highlighting, which is why they often get confused by complex nested structures. They see text, not meaning.

**Treesitter** changes this. It’s a tool integrated into modern editors like Neovim that builds a concrete syntax tree of your code in real-time as you edit. It understands that this block of text isn't just indented lines; it's a JSON object, a YAML array, or a Bash function.

This unlocks powerful capabilities:

  * **Precision Highlighting:** Variables, keywords, strings, and comments are colored accurately based on their semantic meaning, not just their pattern.
  * **Structural Selection:** Instead of manually counting lines to select a block, you can instantly select an entire function, a complete JSON object, or a full YAML section with a single command.

## 3\. Fuzzy Finding: Navigating the Monorepo

Modern infrastructure code rarely lives in a single file. It lives in giant git repositories, sprawling across dozens of directories containing roles, modules, manifests, and variable files.

Trying to navigate this by typing out full paths like `:e infrastructure/live/prod/us-east-1/eks/main.tf` is unbearably slow and mentally taxing.

The solution is **fuzzy finding**. Tools like **Telescope** (for Neovim) or **fzf** integrate deeply with your editor.

  * Need to open a specific file? Hit a hotkey, type a few letters of its name (e.g., "prod eks main"), and it's there.
  * Need to find every occurrence of an IP address across your entire repository? Live grep for it and jump instantly to any match.

This turns repository navigation from a memory test into a muscle-memory reflex.

## The Modern Toolchain: Neovim vs. Vim

## When NOT to Use Vim or Neovim

A balanced perspective matters.  
Vim and Neovim give you superpowers in terminal environments — especially for IaC, remote debugging, and low-latency work on servers.  
But they are not the best tool for every scenario.

If you are building a large application locally — React, Go microservices, Python APIs — then VS Code or JetBrains IDEs are often the right choice. They offer deeper integrations for full application development workflows.

Use the best tool for the job.  
For system engineering, infrastructure, Kubernetes, Terraform, and automation — a terminal-first editor keeps you effective everywhere, including places where GUI tools cannot follow.

Vim remains the undisputed king of quick edits on remote servers. It is everywhere, and the basic muscle memory you learned in our first article will serve you forever.

But for your daily driver—your primary environment for crafting infrastructure code—**Neovim** has emerged as the modern champion.

Neovim is a fork of Vim built from the ground up to support modern features like LSP, Treesitter, and asynchronous plugins as first-class citizens. It is not about abandoning Vim's modal editing philosophy, which is as powerful as ever. It's about empowering that philosophy with the modern engine needed for today's complex configurations.

## Conclusion: Editing *Is* Engineering

When configuration is code, the quality of your editing environment directly impacts the stability of your infrastructure.

We don't have to choose between the ubiquity of the terminal and the power of a modern IDE. By adopting modern tools like Neovim, we get both. We gain the ability to work efficiently and precisely on any server, anywhere, at any time.

In the next stages of our journey, we will rely heavily on these skills as we begin building complex automated systems.



### 🧪 Updated Hands-on Lab

To help you make this transition, I have significantly updated our **vim-lab** repository.

**👉 [github.com/d3ep0ps/vim-lab](https://github.com/d3ep0ps/vim-lab)**

It now includes:

  * **Starter Configurations:** "Sane default" config files for both classic `vim` and modern `neovim` to get you started with a usable environment immediately.
  * **Extended Cheatsheet:** A new guide covering intermediate power moves like macros, registers, and window splits.

Go clone it, install the configs, and start building your muscle memory.