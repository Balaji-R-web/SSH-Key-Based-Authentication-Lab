# Project 7: SSH Key-Based Authentication Lab

## 1. Project Overview

This project demonstrates how to configure SSH key-based authentication on Linux. SSH keys allow users to connect to remote Linux servers securely without entering their account password every time.

The project covers SSH key generation, public-key installation, file permissions, authentication testing, SSH troubleshooting, and security best practices.

## 2. Objectives

- Generate an SSH key pair using Ed25519.
- Understand public and private SSH keys.
- Configure the `authorized_keys` file.
- Set secure SSH file permissions.
- Test SSH key-based authentication.
- Troubleshoot common SSH connection problems.
- Learn how to manage SSH host fingerprints and known hosts.

## 3. Environment

- Operating system: Amazon Linux 2023 or another Linux distribution
- Access: Linux terminal or AWS EC2 instance
- Protocol: SSH
- Tools: `ssh`, `ssh-keygen`, `ssh-agent`, `ssh-add`

> **Important:** Never upload your private SSH key to GitHub. Commands and sample outputs below are illustrative; your IP address, username, and key fingerprint will differ.

## 4. Step 1: Check SSH Tools

Check whether SSH tools are installed:

```bash
ssh -V
ssh-keygen -V
```

The second command may display usage information or an error because `-V` is not supported by every version of `ssh-keygen`. To check its availability, use:

```bash
command -v ssh
command -v ssh-keygen
command -v ssh-add
```

Example output:

```text
/usr/bin/ssh
/usr/bin/ssh-keygen
/usr/bin/ssh-add
```

**Explanation:** These commands confirm that the SSH client and key-management utilities are available.

## 5. Step 2: Generate an SSH Key Pair

Generate an Ed25519 key pair:

```bash
ssh-keygen -t ed25519 -C "project7-lab"
```

When prompted, press Enter to accept the default file location or specify a separate filename such as:

```text
/home/ec2-user/.ssh/project7_ed25519
```

Set a passphrase for additional protection when possible.

The command creates two files:

- `project7_ed25519` — private key
- `project7_ed25519.pub` — public key

If you accepted the default location, the files will usually be named `id_ed25519` and `id_ed25519.pub`.

Check the files:

```bash
ls -l ~/.ssh/
```

Example output:

```text
-rw-------  id_ed25519
-rw-r--r--  id_ed25519.pub
```

**Explanation:** The private key must remain secret. The public key can be installed on the remote server to authorize access.

## 6. Step 3: Verify the Key Pair

Display the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Example output:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... project7-lab
```

Check the key fingerprint:

```bash
ssh-keygen -lf ~/.ssh/id_ed25519.pub
```

Example output:

```text
256 SHA256:ExampleFingerprintHere project7-lab (ED25519)
```

**Explanation:** A fingerprint provides a short identifier for checking a public key. Share only the public key when configuring remote access.

## 7. Step 4: Configure the Remote Linux Server

You need a client machine and a target server. They can be two EC2 instances or another client and Linux server that you are authorized to access.

On the **client machine**, copy the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the complete output.

On the **target server**, sign in using your existing authorized method. Then run these commands as the target account, replacing `ec2-user` with the correct username if needed:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Open the file:

```bash
vi ~/.ssh/authorized_keys
```

Paste the client's complete public key on a new line. Do not delete any existing authorized keys. Save and exit.

Verify the configuration:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Expected permissions:

```text
drwx------  .ssh
-rw-------  authorized_keys
```

**Explanation:** SSH checks ownership and permissions on key files and related directories. Incorrect permissions can prevent public-key authentication from working.

## 8. Step 5: Test SSH Key Authentication

From the client machine, connect to the target server:

```bash
ssh -i ~/.ssh/id_ed25519 ec2-user@SERVER_IP
```

Replace `SERVER_IP` with the target server's IP address or DNS name, and replace `ec2-user` with the actual target username.

For a custom key filename, use its path instead:

```bash
ssh -i ~/.ssh/project7_ed25519 ec2-user@SERVER_IP
```

Example successful output:

```text
Last login: ...
[ec2-user@target-server ~]$
```

Run a command on the remote server:

```bash
hostname
whoami
```

Example output:

```text
target-server
ec2-user
```

**Explanation:** Successful login confirms that the server accepted the key for the specified account. A first connection may ask you to verify the server's host fingerprint. Confirm it using a trusted source before accepting it.

## 9. Step 6: Use SSH Agent (Optional)

An SSH agent can keep a decrypted private key available for your current login session.

Start the agent:

```bash
eval "$(ssh-agent -s)"
```

Add the private key:

```bash
ssh-add ~/.ssh/id_ed25519
```

List keys currently loaded in the agent:

```bash
ssh-add -l
```

Example output:

```text
256 SHA256:ExampleFingerprintHere project7-lab (ED25519)
```

**Explanation:** The agent helps avoid repeatedly entering the private-key passphrase. Do not add keys you do not trust to the agent.

## 10. Step 7: Simplify Connections with SSH Config (Optional)

On the client machine, open the SSH configuration file:

```bash
vi ~/.ssh/config
```

Add this configuration, replacing the username, host, and key path:

```sshconfig
Host project7-server
    HostName SERVER_IP
    User ec2-user
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Set secure permissions:

```bash
chmod 600 ~/.ssh/config
```

Connect using the alias:

```bash
ssh project7-server
```

**Explanation:** The SSH configuration file stores connection settings so you do not need to enter the full command every time.

## 11. Step 8: Troubleshoot SSH Connections

### Problem A: Permission denied (publickey)

Check the following:

```bash
whoami
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
ssh -v -i ~/.ssh/id_ed25519 ec2-user@SERVER_IP
```

Possible causes:

- The public key is missing from the target account's `authorized_keys`.
- You used the wrong username or private key.
- SSH directory permissions or ownership are incorrect.
- The server does not permit the intended authentication method.

### Problem B: Connection timed out

Check the target IP address and network access. For an AWS EC2 instance, verify:

- The instance is running and reachable.
- Its security group allows inbound TCP port 22 from your client IP, where appropriate.
- Network ACLs, routing, firewalls, and network connectivity allow SSH traffic.

### Problem C: Host key verification failed

The saved host fingerprint may not match the server's current fingerprint.

Check the existing entry:

```bash
ssh-keygen -F SERVER_IP
```

Verify the server's fingerprint through a trusted channel before making changes. If the server was legitimately rebuilt and its identity has changed, remove only the verified obsolete entry:

```bash
ssh-keygen -R SERVER_IP
```

Reconnect and verify the new fingerprint before accepting it.

**Security warning:** Do not blindly remove or accept host keys when a mismatch could indicate a man-in-the-middle attack.

### Problem D: Check SSH service logs on Amazon Linux

On the target server, inspect the logs:

```bash
sudo journalctl -u sshd --since "15 minutes ago"
```

If the service unit or logs differ on your Linux distribution, consult its SSH service documentation.

## 12. Step 9: Security Best Practices

- Keep private keys secret and protect them with a passphrase.
- Never commit private keys, passwords, or other credentials to GitHub.
- Use a separate key for labs or specific environments.
- Restrict access to the private key and SSH configuration.
- Use least-privilege accounts instead of logging in as root.
- Verify server fingerprints before accepting new or changed host keys.
- Restrict network access to SSH wherever practical.
- Do not disable password authentication or restart SSH services on a remote server without validating the configuration and ensuring an alternative access session is available.

## 13. Project Outcome

After completing this project, I can:

- Generate and identify SSH key pairs.
- Configure public-key authentication for a Linux account.
- Apply secure SSH file permissions.
- Connect to a remote Linux server using a private key.
- Use SSH agent and client configuration.
- Troubleshoot common SSH authentication and connectivity issues.
- Follow essential SSH security practices.

## 14. Tools and Technologies

- Linux / Amazon Linux
- OpenSSH
- SSH key pairs (Ed25519)
- Linux file permissions
- AWS EC2 networking (optional)

## 15. Conclusion

This hands-on lab demonstrates SSH key-based authentication and secure remote access to Linux servers. It builds practical knowledge useful for Linux administration, cloud operations, and DevOps engineering.
