# Jenkins Lab 08: Configure a Jenkins Static Agent Using SSH

This document is the companion guide for **DevOps Lab #8**, where we configure a permanent Jenkins agent that connects to the Jenkins controller using SSH.

The guide follows the lab demonstrated in the video. It covers SSH key-based authentication, Jenkins SSH credentials, and the four host key verification strategies available in the node configuration shown in the tutorial.

> **Lab environment shown in the video**
>
> - Jenkins controller: Linux VM running Jenkins
> - Jenkins agent: separate Linux VM
> - Agent SSH user: `rohit` (use the Linux account you created on your own agent)
> - Example agent IP address: `192.168.50.129`
> - SSH key algorithm: Ed25519
> - Jenkins node name: `Jenkins agent SSH`
> - Jenkins credential ID: `jenkins-agent-rohit-ssh`
>
> These values are examples from the demonstration environment. Replace them with your own hostname, IP address, username, directories, and credential ID.

---

## Table of Contents

1. [What We Are Building](#1-what-we-are-building)
2. [Prerequisites](#2-prerequisites)
3. [Start the VMs and Open Jenkins](#3-start-the-vms-and-open-jenkins)
4. [Create a Permanent Jenkins Agent](#4-create-a-permanent-jenkins-agent)
5. [Generate an SSH Key Pair on the Controller](#5-generate-an-ssh-key-pair-on-the-controller)
6. [Create an SSH Credential in Jenkins](#6-create-an-ssh-credential-in-jenkins)
7. [Configure the Agent to Launch over SSH](#7-configure-the-agent-to-launch-over-ssh)
8. [Enable Passwordless SSH Authentication](#8-enable-passwordless-ssh-authentication)
9. [The Four Host Key Verification Strategies](#9-the-four-host-key-verification-strategies)
10. [Troubleshooting](#10-troubleshooting)
11. [Security Notes](#11-security-notes)
12. [What You Learned](#12-what-you-learned)

---

## 1. What We Are Building

In the previous lab, a static agent was connected to the Jenkins controller using the inbound connection method. In this lab, we configure another static agent, but this time the controller initiates the connection to the agent over SSH.

```text
              Jenkins Controller
             (Jenkins + SSH client)
                      |
                      | SSH connection (TCP/22)
                      | Authenticates with private key
                      v
                Jenkins Agent
              (sshd + authorized_keys)
                      |
                      v
             Runs pipeline workloads
```

The basic flow is:

1. The controller uses an SSH private key to authenticate as a user on the agent.
2. The agent checks whether the matching public key is authorized for that user.
3. The SSH server host key is checked according to the selected host key verification strategy.
4. If authentication and host verification succeed, Jenkins launches its agent process over SSH and the node becomes online.

**Important distinction:** SSH user authentication and SSH host key verification are two separate checks. The private key and `authorized_keys` enable the controller to authenticate to the agent. The host key verification strategy helps the controller verify the identity of the SSH server it is connecting to.

## 2. Prerequisites

Before starting, make sure:

- Jenkins is installed and accessible on the controller VM.
- The agent VM is running and reachable from the controller over the network.
- An SSH server is installed and running on the agent.
- You know the Linux username on the agent that Jenkins should use.
- The selected agent account has a writable remote root directory for Jenkins.
- Java is installed on the agent and compatible with the Jenkins version you are running.
- You have permission to manage nodes and credentials in Jenkins.

### Check SSH on the agent

On Ubuntu/Debian agents, check the SSH service:

```bash
sudo systemctl status ssh
```

If the OpenSSH server package is not installed, install it:

```bash
sudo apt update
sudo apt install -y openssh-server
```

Start and enable SSH:

```bash
sudo systemctl enable --now ssh
```

Check the agent's IP address:

```bash
ip address
```

Or, on a lab VM where this is sufficient:

```bash
hostname -I
```

From the controller, test basic network and SSH reachability:

```bash
ping <AGENT_IP>
ssh <AGENT_USER>@<AGENT_IP>
```

Replace the placeholders with your actual values. A successful ping is not required on networks that block ICMP; the important point is that the controller can reach the agent's SSH service.

## 3. Start the VMs and Open Jenkins

Start both the Jenkins controller VM and the agent VM. Log in to each and confirm they are running.

Open the Jenkins dashboard in your browser and sign in. In the demonstration, the controller already has a node, and we add a second node configured for SSH.

## 4. Create a Permanent Jenkins Agent

In Jenkins, navigate to **Manage Jenkins → Nodes** (the exact menu wording can vary by Jenkins version), then choose **New Node**.

Configure the node with values appropriate to your environment:

| Setting | Example / guidance |
|---|---|
| Node name | `Jenkins agent SSH` |
| Node type | Permanent Agent |
| Remote root directory | The existing directory on the agent intended for Jenkins work |
| Labels | For example, `ssh-agent` |
| Usage | Use this node as much as possible, or choose a policy appropriate to your jobs |
| Launch method | Launch agents via SSH |
| Host | The agent's IP address or resolvable hostname |
| Credentials | The SSH username and private key credential created below |
| Host Key Verification Strategy | Choose one of the four strategies discussed later |

The remote root directory must exist and be writable by the SSH user. Use the directory you prepared on the agent in your lab; do not copy a path from this guide without checking that it exists in your environment.

At first, the SSH credential will not be available because we have not created it yet. Leave the configuration until the credential is ready, or open the credential setup in another tab.

## 5. Generate an SSH Key Pair on the Controller

The SSH private/public key pair used for this lab is generated on the **Jenkins controller**, because the controller is the client initiating the SSH connection.

First, inspect the SSH directory for the account under which you are configuring the connection. For the controller account used in the demonstration:

```bash
ls -la ~/.ssh
```

Generate an Ed25519 key pair:

```bash
ssh-keygen -t ed25519 -C "jenkins-agent-ssh"
```

Follow the prompts. In the demonstration, no passphrase was set. In production, consider your organization's key-protection and credential-management requirements. A passphrase-protected key requires an appropriate way for Jenkins to unlock it, configured in the credential.

The command creates two files at the selected path. With the default filename, they are:

```text
~/.ssh/id_ed25519       # Private key
~/.ssh/id_ed25519.pub   # Public key
```

If you specify a different filename, use that filename in the following steps.

### Private key vs. public key

- **Private key (`id_ed25519`)**: Keep it secret. Add its contents to the Jenkins SSH credential. Do not copy it into the agent's `authorized_keys` file or commit it to a repository.
- **Public key (`id_ed25519.pub`)**: Install it for the target Linux user on the agent. It is not a secret, but it should still be handled thoughtfully.

You can display the public key with:

```bash
cat ~/.ssh/id_ed25519.pub
```

To inspect the private key for the Jenkins credential, use the exact key file path you generated:

```bash
cat ~/.ssh/id_ed25519
```

Be careful when displaying or copying private key material. Do not paste it into public chat, documentation, screenshots, or source control.

### Why Ed25519?

The `-t ed25519` option tells `ssh-keygen` which key type to generate. Ed25519 is a modern public-key signature system supported by current OpenSSH installations. RSA is another possible key type, but follow your organization's cryptographic policy and compatibility requirements.

## 6. Create an SSH Credential in Jenkins

Jenkins needs the private key and the target Linux username to authenticate to the agent.

In Jenkins, navigate to **Manage Jenkins → Credentials**, select the appropriate store and domain (the demonstration uses the System store and Global domain), and click **Add Credentials**.

Choose:

**Kind:** `SSH Username with private key`

Fill in the fields:

| Field | Example / guidance |
|---|---|
| ID | `jenkins-agent-rohit-ssh` |
| Description | `SSH key for agent user` |
| Username | `rohit` in the demo; use your actual agent Linux username |
| Private Key | Choose the option to enter the key directly, then paste the controller's private key |
| Passphrase | Leave blank only if the key was generated without a passphrase; otherwise enter the key's passphrase |

Save the credential.

### Which username should be entered?

Use the Linux account that Jenkins will log in as **on the agent**, not an arbitrary username and not necessarily the controller's operating-system user. In the video, the target account is `rohit`; your environment may use another dedicated account.

### Why does Jenkins need the private key?

The private key proves the identity of the connecting user. The agent's SSH server checks whether the corresponding public key is authorized for the account. Jenkins stores the private key in its credential store, so a pipeline or node configuration can refer to a credential ID instead of embedding the key in a script.

Return to the node configuration and refresh the credential list if the newly created credential is not shown.

## 7. Configure the Agent to Launch over SSH

Return to **Manage Jenkins → Nodes → Jenkins agent SSH → Configure**.

Configure the launch method as **Launch agents via SSH**, then provide:

- **Host:** the agent's IP address or hostname.
- **Credentials:** the SSH credential created in the previous section.
- **Host Key Verification Strategy:** select the strategy you want to test.
- **Availability and connection settings:** use sensible values for your environment.

Save the configuration, then select the node and choose **Launch agent** or the equivalent action available in your Jenkins version.

In the demonstration, the first connection attempt fails with an authentication error. That is expected at this stage because the controller's public key has not yet been installed for the target user on the agent.

## 8. Enable Passwordless SSH Authentication

The controller's public key must be authorized on the agent for the account that Jenkins will use.

### 8.1 Check the agent's `authorized_keys`

On the **agent**, inspect the authorized keys file for the target account. If the account is `rohit`, for example:

```bash
cat /home/rohit/.ssh/authorized_keys
```

The directory or file may not exist yet. The file should contain the public key authorized to log in as that user.

> `authorized_keys` belongs to the target user on the agent. Ensure ownership and permissions are correct. OpenSSH may reject keys if the home directory or `.ssh` files are writable by inappropriate users.

### 8.2 Copy the public key to the agent

On the controller, run `ssh-copy-id` using the **public key file**:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub <AGENT_USER>@<AGENT_IP>
```

Example in the demonstration environment (substitute your actual username and IP):

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub rohit@192.168.50.129
```

On the first connection, SSH may ask you to confirm the agent's host key and then request the target user's password. Verify the host key fingerprint before accepting it. After successful authentication, `ssh-copy-id` appends the public key to the target user's `~/.ssh/authorized_keys` file on the agent.

**Correction worth remembering:** `ssh-copy-id` copies the **public key**, not the private key. Keep the private key on the controller and store it as a Jenkins credential.

### 8.3 Test passwordless SSH from the controller

Test the connection using the same account and key:

```bash
ssh -i ~/.ssh/id_ed25519 <AGENT_USER>@<AGENT_IP>
```

For example:

```bash
ssh -i ~/.ssh/id_ed25519 rohit@192.168.50.129
```

Once user key-based authentication is correctly configured, the connection should not ask for the agent user's account password. Depending on the SSH setup, the first connection can still prompt for host key verification. Host-key trust and user authentication are separate controls.

Exit the remote shell when finished:

```bash
exit
```

### 8.4 Retry the Jenkins node connection

Return to Jenkins and launch the agent again. With the matching public key installed on the agent and the private key configured in the Jenkins credential, Jenkins should be able to authenticate.

In the recorded demonstration, the node connects successfully using **Non-Verifying Verification Strategy**. The next sections explain the four strategies demonstrated in the video.

---

## 9. The Four Host Key Verification Strategies

When connecting over SSH, two separate questions must be answered:

1. **User authentication:** Can the controller prove it is authorized to log in as the selected agent user? This is the private-key / `authorized_keys` mechanism.
2. **Host verification:** Can the controller verify that the SSH server it reached is the expected agent, rather than an unexpected or impersonated server?

The Jenkins SSH launch configuration provides four strategies in this lab. Their names and availability may depend on the installed Jenkins SSH Build Agents plugin version.

### Strategy 1: Non-Verifying Verification Strategy

**What it does:** The strategy does not validate the SSH server's host key. User authentication can still happen with the configured private key, but the controller is not using the host key to confirm the server's identity.

**How it was tested in the video:**

1. Choose **Non-Verifying Verification Strategy**.
2. Save the node configuration.
3. Launch the agent after configuring key-based user authentication.
4. Confirm that the node comes online.

**Use case:** Convenient for a disposable lab where host verification behavior is being demonstrated.

**Security note:** This strategy removes an important protection against server impersonation and man-in-the-middle attacks. It should not be treated as a secure production default.

### Strategy 2: Manually Provided Key Verification Strategy

**What it does:** You explicitly provide the SSH server's public host key to Jenkins, so Jenkins can compare the key presented during connection with the key supplied in the node configuration.

**Find the host key on the agent:**

The demonstration uses the agent's Ed25519 SSH host public key. On the agent, run:

```bash
sudo cat /etc/ssh/ssh_host_ed25519_key.pub
```

This file is the SSH **server host public key**, not the user's public key from `~/.ssh/id_ed25519.pub`.

You can also inspect the fingerprint:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Copy the host public key as required by the Jenkins strategy's field, then:

1. Disconnect the agent if it is already connected.
2. Choose **Manually Provided Key Verification Strategy**.
3. Enter the agent's host public key in the requested field.
4. Save and launch the agent.
5. Confirm in the node status/logs that the connection succeeds.

**Why this matters:** The controller needs to trust the SSH server host key, which identifies the server. This is different from installing the controller user's public key in the agent's `authorized_keys`.

### Strategy 3: Manually Trusted Key Verification Strategy

**What it does:** Jenkins can ask you to manually trust the agent's host key during the initial connection. The initial key exchange requires a trust decision before the connection can proceed.

**How it was tested in the video:**

1. Disconnect the node.
2. Select **Manually Trusted Key Verification Strategy**.
3. Enable **Require manual verification for initial connection**.
4. Save the configuration and launch the agent.
5. Jenkins presents a host-key trust prompt or pending verification action.
6. Review the presented fingerprint and approve the key only after verifying that it belongs to your agent.
7. Check the node log/status and confirm that the connection succeeds.

The recorded demonstration shows a fingerprint and asks for manual approval. The fingerprint is a short representation of the public host key, calculated with a cryptographic hash such as SHA-256.

**Good practice:** Independently retrieve the fingerprint from the agent and compare it before approving. Do not accept a key simply because Jenkins displays a prompt.

### Strategy 4: Known Hosts File Verification Strategy

**What it does:** Jenkins relies on the controller account's OpenSSH `known_hosts` file to find and validate the agent's host key.

In the demo, the connection initially fails because the expected known-hosts file or entry is not available. The fix is to ensure the controller has a known-hosts entry for the agent.

#### Step 1: Identify the controller account's SSH directory

On the controller, check:

```bash
whoami
echo "$HOME"
ls -la ~/.ssh
```

The `known_hosts` file must be available to the operating-system account under which the SSH agent launcher runs. Do not assume that a file under your interactive shell's home directory is visible to Jenkins if the Jenkins service runs as a different user.

#### Step 2: Create the SSH directory if needed

For the appropriate controller account, ensure `~/.ssh` exists and has suitable permissions:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Use the home directory of the actual Jenkins service account if that is the account the launcher reads.

#### Step 3: Add the agent's host key

You can use `ssh-keyscan` to retrieve a public host key, but it **does not authenticate** that the key is genuinely from the intended host. Verify the fingerprint through a trusted channel before trusting the result.

For example, after verifying the fingerprint independently:

```bash
ssh-keyscan -H <AGENT_IP> >> ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts
```

Alternatively, connect interactively from the controller and verify the fingerprint before accepting it, which normally adds an entry to `known_hosts` for that SSH client account.

Check the stored entry:

```bash
ssh-keygen -F <AGENT_IP> -f ~/.ssh/known_hosts
```

If you connect by hostname in Jenkins, ensure the entry matches the hostname Jenkins uses. A key entry recorded only for an IP address may not match a connection configured with a hostname, or vice versa.

#### Step 4: Retry the Jenkins connection

Select **Known Hosts File Verification Strategy**, save the node configuration, and launch the agent again. In the demonstration, the connection succeeds once Jenkins can find the agent's host key in the controller's known-hosts file.

### Comparison

| Strategy | How the server host key is trusted | Security note |
|---|---|---|
| Non-Verifying | Not verified by this strategy | Least protective; suitable only for controlled demonstrations |
| Manually Provided Key | Key is supplied in the node configuration | Make sure the key came from a trusted source |
| Manually Trusted Key | User manually approves the initial key | Verify the fingerprint independently before approving |
| Known Hosts File | Matches an entry in the controller account's `known_hosts` file | Manage the file for the correct service account and verify entries |

These methods apply to **server host key verification**. They do not replace the private-key credential and `authorized_keys` used for **user authentication**.

---

## 10. Troubleshooting

### Authentication failed

**Likely causes:**
- The public key has not been added to the correct agent user's `authorized_keys`.
- The Jenkins credential contains the wrong private key.
- The username in the Jenkins credential is not the account that owns `authorized_keys`.
- The `.ssh` directory or `authorized_keys` permissions/ownership are invalid.
- The SSH server is not reachable or is configured to disallow the required authentication method.

**Checks:**

```bash
# From controller: test SSH with the intended key
ssh -i ~/.ssh/id_ed25519 <AGENT_USER>@<AGENT_IP>

# On agent: inspect the authorized public keys for that user
cat /home/<AGENT_USER>/.ssh/authorized_keys

# On agent: inspect SSH service status on Ubuntu/Debian
sudo systemctl status ssh
```

Do not copy the controller's private key to the agent. Only the public key belongs in `authorized_keys`.

### The node stays offline or cannot launch

- Check the node configuration's Host and Credentials fields.
- Confirm TCP port 22 is reachable from the controller.
- Verify that the configured remote root directory exists and is writable by the SSH user.
- Make sure a suitable Java runtime is available on the agent.
- Open the node's log in Jenkins and read the earliest relevant error message.

### Known Hosts File strategy says no host is found

- Confirm the `known_hosts` file exists for the operating-system account Jenkins uses.
- Make sure the entry corresponds to the exact hostname or IP configured in the node.
- Verify the host key and fingerprint before adding it.
- Check file ownership and permissions.

### The host key changed

An SSH host key can change if the agent is rebuilt, its host keys are regenerated, or an address is reassigned. Do not blindly remove an old key and accept the new one. First verify why the key changed and confirm the new fingerprint with a trusted source, then update the relevant trusted key or `known_hosts` entry.

### Useful diagnostics

```bash
# Confirm the host can be reached from the controller
ssh -v -i ~/.ssh/id_ed25519 <AGENT_USER>@<AGENT_IP>

# Inspect the agent's SSH service on Ubuntu/Debian
sudo systemctl status ssh

# Show the agent host key fingerprint
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Verbose SSH output can contain usernames, hostnames, paths, and other environment details. Review it before sharing logs publicly.

---

## 11. Security Notes

- **Protect the private key.** Never commit the private key to Git, copy it into `authorized_keys`, or share it in screenshots.
- **Prefer a dedicated agent account.** Give it only the permissions required by your build workloads.
- **Verify server host keys.** Avoid non-verifying host-key strategies for production workloads.
- **Verify fingerprints independently.** Do not blindly trust a new key prompt or add a `ssh-keyscan` result without verification.
- **Keep credentials scoped.** Store keys in Jenkins Credentials and refer to credentials by ID rather than hardcoding secrets in a pipeline.
- **Review account and file permissions.** Ensure that only the intended account and privileged administrators can read private keys and modify authorized keys.
- **Rotate credentials carefully.** Update the Jenkins credential and agent authorization when rotating the key pair.

## 12. What we Learned

By completing this lab, you have:

- Created a permanent Jenkins node using the SSH launch method.
- Generated an Ed25519 user key pair on the controller.
- Stored the private key in a Jenkins **SSH Username with private key** credential.
- Authorized the matching public key on the agent with `ssh-copy-id`.
- Established key-based, passwordless SSH user authentication.
- Tested four Jenkins SSH host key verification strategies:
  1. Non-Verifying Verification Strategy
  2. Manually Provided Key Verification Strategy
  3. Manually Trusted Key Verification Strategy
  4. Known Hosts File Verification Strategy
- Connected the static agent to Jenkins so it can execute pipeline work.
