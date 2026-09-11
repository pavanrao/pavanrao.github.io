---
title: "Git and GitHub Setup"
date: 2026-01-27T23:04:29Z
summary: "A quick reference guide for setting up Git and connecting to GitHub via SSH on Ubuntu Server."
tags: ["git", "github", "ssh", "ubuntu", "devops"]
categories: ["Coding"]
---

Notes on setting up Git and connecting it to GitHub via SSH on Ubuntu Server.

## 1. Update and Install Git

First, ensure your package list is up to date and install the Git package.

```bash
sudo apt update
sudo apt install git -y
```

## 2. Configure Global Git Settings

Set your identity. This information will be embedded in every commit you make.

```bash
git config --global user.name "Your Name"
git config --global user.email "your_email@example.com"
```

*Optional: Set the default branch name to `main` (the modern standard) to avoid "master/main" conflicts later.*

```bash
git config --global init.defaultBranch main
```

## 3. Generate an SSH Key

Using HTTPS for GitHub on a headless server is inconvenient because it requires frequent credential entry. SSH is the standard, secure method.

Generate a new ED25519 SSH key (the recommended algorithm):

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

* **Press Enter** to accept the default file location (`/home/username/.ssh/id_ed25519`).
* **Passphrase:** You can press Enter for no passphrase (convenient for automation) or type one for extra security.

## 4. Add the SSH Key to the ssh-agent

This ensures the server manages your keys automatically in the background.

Start the agent:

```bash
eval "$(ssh-agent -s)"
```

Add your private key:

```bash
ssh-add ~/.ssh/id_ed25519
```

## 5. Add the Public Key to GitHub

You need to copy the public key to your clipboard or display it to copy manually.

Display the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

**On GitHub:**

1. Log in and go to **Settings**.
2. Click **SSH and GPG keys** in the left sidebar.
3. Click **New SSH key**.
4. **Title:** Give it a descriptive name (e.g., "Ubuntu Server Laptop").
5. **Key:** Paste the output from the `cat` command above.
6. Click **Add SSH key**.

## 6. Verify the Connection

Test that your server can authenticate with GitHub.

```bash
ssh -T git@github.com
```

* You will see a warning: `Are you sure you want to continue connecting (yes/no/[fingerprint])?`
* Type `yes` and press Enter.
* **Success Message:** You should see: *"Hi [username]! You've successfully authenticated, but GitHub does not provide shell access."*

## Quick Commands Cheat Sheet

Here are the commands you will likely use immediately after setup:

| Action | Command |
| --- | --- |
| **Clone a repo** | `git clone git@github.com:username/repo-name.git` |
| **Check status** | `git status` |
| **Stage files** | `git add .` (all files) or `git add filename` |
| **Commit** | `git commit -m "Commit message"` |
| **Push** | `git push origin main` |
