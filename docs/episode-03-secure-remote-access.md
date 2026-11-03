# Episode 03 — Secure remote access with Tailscale and SSH keys

**Project:** Lenovo ThinkPad L580 homelab · Ubuntu Server · MSI Windows laptop  
**Date documented:** November 2026  
**Goal:** Securely administer my home server from another network without exposing SSH directly to the public internet.

> **Learning mindset:** Configure one thing at a time, understand every command, test both successful and rejected connections, and keep a recovery path open.

## 1. The problem I wanted to solve

In [Episode 02](episode-02-lenovo-l580.md), I transformed a laptop with a damaged screen into a headless Ubuntu Server accessible via SSH on my **home Wi-Fi**.

The next question was: **How can I reach that same server from a café, hotel, or another country?**

My original SSH setup depended on the Lenovo's local address, which is only reachable on my home network. Instead of forwarding TCP port **22** through my home router, I chose a private mesh VPN:

- **Tailscale** for private connectivity between devices, built on WireGuard.
- **OpenSSH** for the actual remote shell.
- An **Ed25519 SSH key**, protected by a passphrase, instead of SSH password login.

**Architecture:**

```text
MSI laptop (Windows / PowerShell)
       |
       |  Tailscale private network (encrypted)
       |  + SSH with an Ed25519 key
       v
Lenovo homelab (Ubuntu Server / OpenSSH)
       |
       +-- Home Wi-Fi / router
       +-- Future Docker services: Jellyfin, Immich, Nextcloud
```

The MSI and Lenovo can be on **different networks**, provided both have internet access and are authorized to communicate through the same tailnet. No public SSH port forwarding is needed for this design.

**Important distinction:** This project uses **ordinary OpenSSH over Tailscale**, **not** the separate optional product called *Tailscale SSH*. I have not enabled `tailscale set --ssh`.

## 2. My starting point

Before this episode, the following was already working:

- Ubuntu Server installed on the Lenovo and running with the lid closed.
- Wi-Fi configured with Netplan.
- OpenSSH installed and accessible locally from my MSI.
- Terminal access from Windows PowerShell.

When I connected after completing the configuration, the Ubuntu welcome screen identified the system as **Ubuntu 26.04.1 LTS**.

My Linux account is `adir` and the server hostname is `lenovo-homelab`. For reproducibility, the code examples below use `YOUR_TAILSCALE_IP` instead of publishing the actual private network address.

## 3. Installing and logging in to Tailscale on Ubuntu

### Install the client

The [official Tailscale Linux guide](https://tailscale.com/docs/install/linux) provides this installation method:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

- `curl` downloads the installation script using HTTPS.
- `-f` fails on HTTP errors, `-sS` keeps output concise while displaying errors, and `-L` follows redirects.
- `| sh` runs the downloaded script. **Only do this when you trust the source**; the signed APT repository is an alternative.

> This is the official documented installation command, included for reproducibility. The exact command used for my original Tailscale installation was not recorded in these notes.

To join the private network:

```bash
sudo tailscale up
```

Tailscale displayed an authentication URL, which I opened on the MSI to approve the Lenovo.

### Difficulty encountered: authentication link returned HTTP 404

The first link led to this message:

```text
Error 404
This authentication link could not be located. It may have expired.
```

**What I did:** I returned to the Lenovo's terminal and requested a fresh login link:

```bash
sudo tailscale login
```

I opened the **newly generated** URL and completed the authentication. It worked.

**What I learned:** A failed or expired browser authentication URL does not automatically mean the Tailscale client is incorrectly installed. Before reinstalling, try a fresh login link and inspect the service status.

For further troubleshooting, useful checks are:

```bash
sudo systemctl is-active tailscaled   # Expected: active
tailscale status                      # List peers / connection state
```

If reauthentication is genuinely required, the official documentation also describes:

```bash
# Only when necessary; can interrupt current Tailscale connectivity
sudo tailscale up --force-reauth
```

**Never publish a live authentication URL or an authentication key in screenshots or GitHub commits.**

## 4. Verifying the private Tailscale connection

On the Lenovo:

```bash
tailscale ip -4
```

This prints its private Tailscale IPv4 address, generally in the `100.x.y.z` range. Then:

```bash
tailscale status
```

My result showed the `lenovo-homelab` Linux device as part of the tailnet.

On my **MSI**, I installed the Windows Tailscale client and signed in to the **same tailnet**.

My first SSH test from PowerShell:

```powershell
ssh adir@YOUR_TAILSCALE_IP
```

**Result: successful.** The familiar `adir@lenovo-homelab:~$` prompt appeared, now reached using the Tailscale address instead of the home LAN address.

At this stage, SSH password authentication was still available. I wanted to secure it further.

## 5. Creating an Ed25519 SSH key on Windows

I generated a **separate key dedicated to this Lenovo** on the MSI, in **PowerShell** (not inside a terminal already connected to Ubuntu):

```powershell
ssh-keygen -t ed25519 -a 100 -f "$env:USERPROFILE\.ssh\id_ed25519_lenovo"
```

| Part | Why it matters |
|---|---|
| `ssh-keygen` | Generates an SSH key pair |
| `-t ed25519` | Uses the Ed25519 key algorithm |
| `-a 100` | Applies 100 key-derivation rounds to protect the private key passphrase |
| `-f ...` | Saves this key under a dedicated filename in my Windows `.ssh` directory |

During key generation, I chose a **strong passphrase**.

Two files are created on Windows:

```text
C:\Users\<WINDOWS_USER>\.ssh\
├── id_ed25519_lenovo       # PRIVATE KEY: never share
└── id_ed25519_lenovo.pub   # PUBLIC KEY: safe to install on the server
```

The **passphrase protects the private key stored on my MSI**. It is **not** the same as the password of my Ubuntu account.

> If the key file already exists, do not overwrite it without checking whether it is used elsewhere.

## 6. Adding the public key to the Lenovo

From **Windows PowerShell on the MSI**, I used this command:

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519_lenovo.pub" | ssh adir@YOUR_TAILSCALE_IP "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

The command prompted me for my **Ubuntu password**, because I was using the existing password-based SSH connection for the last time to copy the new public key.

What each part does:

| Part | Purpose |
|---|---|
| `Get-Content ... .pub` | Reads the *public* key file on Windows |
| `| ssh adir@...` | Sends that public key to a command running on Ubuntu |
| `umask 077` | Sets restrictive default permissions for newly created files |
| `mkdir -p ~/.ssh` | Creates the remote SSH directory if necessary |
| `cat >> ~/.ssh/authorized_keys` | Appends the public key, without deleting any previously authorized keys |

A manual alternative is to paste the public key into `~/.ssh/authorized_keys` on Ubuntu and set appropriate permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

**Do not copy the private key to the server and do not include it in GitHub.**

## 7. Proving that SSH key authentication works

I opened a **new PowerShell session on the MSI** and deliberately prevented the SSH client from falling back to password authentication:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_lenovo" `
  -o IdentitiesOnly=yes `
  -o PreferredAuthentications=publickey `
  -o PasswordAuthentication=no `
  -o KbdInteractiveAuthentication=no `
  adir@YOUR_TAILSCALE_IP
```

*PowerShell note:* The backtick (`` ` ``) continues a command on the following line. It must be the final character on each continued line. Alternatively, paste all options on a single line.

**Observed result:**

```text
Enter passphrase for key '...id_ed25519_lenovo':
Welcome to Ubuntu 26.04.1 LTS
adir@lenovo-homelab:~$
```

**Why this was convincing:** The client explicitly disabled password methods. A successful login meant my private key was accepted by OpenSSH.

Security reminder: on the very first SSH connection, verify the server host-key fingerprint using a trusted channel before accepting it. The Lenovo can display its Ed25519 host-key fingerprint with:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

## 8. Hardening the OpenSSH server

**Safety principle:** Keep an existing SSH session open until both a new key-based login succeeds and the server's effective configuration has been checked.

On Ubuntu, I used a separate OpenSSH configuration snippet:

```bash
sudo nano /etc/ssh/sshd_config.d/00-keys-only.conf
```

Its intended configuration was:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

| Directive | Purpose |
|---|---|
| `PubkeyAuthentication yes` | Allow authentication using public keys |
| `PasswordAuthentication no` | Disable ordinary SSH password authentication |
| `KbdInteractiveAuthentication no` | Disable keyboard-interactive authentication methods |
| `PermitRootLogin no` | Block direct root login over SSH |

**Validate before applying:**

```bash
sudo sshd -t
```

No output usually means the configuration syntax is valid.

**Check the effective configuration:**

```bash
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

Expected values:

```text
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
permitrootlogin no
```

Ubuntu can include additional configuration snippets, so reading the **effective** configuration is more reliable than assuming a file was applied.

**Apply the changes:**

```bash
sudo systemctl reload ssh
```

### Positive test: key-based login

From the MSI, I connected again with:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_lenovo" -o IdentitiesOnly=yes adir@YOUR_TAILSCALE_IP
```

**Observed result: success.** The server asked for the **private key passphrase**, then showed the Ubuntu welcome screen and shell prompt.

### Negative test: deny password-only authentication

I also ran a client test with public-key authentication deliberately disabled:

```bash
ssh -o PubkeyAuthentication=no \
  -o PreferredAuthentications=password \
  -o KbdInteractiveAuthentication=no \
  adir@YOUR_TAILSCALE_IP
```

**Observed result:**

```text
Permission denied (publickey).
```

**A testing detail worth documenting:** In the screenshot, I ran this negative test **from the Lenovo shell to its own Tailscale IP**, not from a separate Windows PowerShell session. It demonstrates the observed rejection for that connection, but I should repeat the same negative test from the MSI as an independent final check.

Similarly, my screenshot did **not** capture the complete output of `sshd -T`; the values above are the intended configuration and should be checked when auditing the server.

## 9. What is working, and what still needs testing

### Confirmed in this episode

- [x] Tailscale successfully authenticated on the Lenovo after an initial 404 error.
- [x] The Lenovo received a private Tailscale IP address.
- [x] The MSI connected to the Lenovo through its Tailscale IP.
- [x] An Ed25519 key pair was generated and the public key installed on Ubuntu.
- [x] Key-only SSH client authentication succeeded.
- [x] A new SSH connection worked after the server hardening procedure.
- [x] A password-only SSH attempt from the Lenovo shell was rejected.

### Still to verify

- [ ] Repeat the negative password-only test **from the MSI**.
- [ ] Confirm the effective settings with `sudo sshd -T` and save a sanitized record.
- [ ] Test SSH from a truly separate network, such as the MSI connected to my phone's cellular hotspot.
- [ ] Check router settings to confirm there is **no public port forwarding** to SSH.
- [ ] Review Tailscale access policies, device approval, and identity-provider MFA.
- [ ] Store a secure, encrypted backup of the SSH private key.
- [ ] Reboot Ubuntu after pending system updates and verify remote access again.

**External-network test procedure (not yet recorded as complete):**

1. Leave the Lenovo at home, connected to its normal Wi-Fi and power.
2. Connect the MSI to a mobile hotspot using **cellular data**, not the home Wi-Fi.
3. Ensure Tailscale is connected on the MSI.
4. Run these commands in **MSI PowerShell**:

```powershell
tailscale ping YOUR_TAILSCALE_IP
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_lenovo" adir@YOUR_TAILSCALE_IP
```

Success here would demonstrate that the setup also works from another network. Do not mark this step as completed until tested.

## 10. Troubleshooting cheatsheet

| Symptom | Likely cause | Safe next step |
|---|---|---|
| Tailscale login page returns `404` | Expired or replaced authorization link | Run `sudo tailscale login` and open the *new* URL |
| `tailscale status` shows the device offline | Network, service or authentication issue | Check Wi-Fi, `sudo systemctl status tailscaled`, and reauthentication state |
| SSH connects with a password but not a key | Public key missing, wrong identity or permissions | Check `~/.ssh/authorized_keys`, key path and SSH verbose logs |
| `Permission denied (publickey)` after hardening | The client lacks an accepted public key | Keep a rescue SSH session open; check the key, permissions and server configuration |
| A password-only connection is refused | Expected after password auth is disabled | Confirm key-based login remains functional |
| Cannot connect from another network | Tailscale is not online or network policies block traffic | Check both peers with `tailscale status` and `tailscale ping` |

Additional checks (do not paste complete sensitive logs into a public issue):

```bash
sudo systemctl status ssh --no-pager
sudo journalctl -u ssh -n 50 --no-pager
sudo sshd -t
```

On **Windows PowerShell**, `ssh -v ...` can help diagnose authentication. Redact IP addresses, usernames, paths and other personal information before posting logs publicly.

## 11. Security decisions and limitations

- **No need to forward TCP port 22 publicly:** the private Tailscale path can reach OpenSSH without it. This project did not add any public port-forwarding rules; existing router rules should still be audited.
- **OpenSSH and Tailscale are separate layers:** Tailscale handles private network connectivity; OpenSSH handles SSH identity and shell access.
- **The Ed25519 private key stays on the MSI:** only the `.pub` file is copied to Ubuntu.
- **A passphrase matters:** losing an unprotected private key could expose the server to unauthorized key-based login if the attacker can also reach it.
- **The SSH key does not replace Tailscale access controls:** access to the tailnet should be restricted by authenticated accounts, trusted devices and least-privilege grants.
- **No blanket guarantee of access everywhere:** captive portals, unusual firewalls, account expiry and power/network outages at home can interrupt connections.
- **Firmware and reboot maintenance remain pending:** Ubuntu displayed `System restart required` and noted firmware updates. Those should be reviewed separately, with a recovery plan.

## 12. Next steps for my homelab

- [ ] Finish the separate-network test and lock down Tailscale access policies.
- [ ] Configure a secure backup and recovery method for the SSH private key.
- [ ] Install Docker Engine and Docker Compose.
- [ ] Deploy Jellyfin with read-only access to my existing movie library.
- [ ] Later, deploy Immich and Nextcloud with appropriate writable storage and backups.
- [ ] Publish the next episode documenting containers, volumes, networking and troubleshooting.

> **My biggest takeaway:** Security is not a single command. It is a series of layers and tests. I learned to verify that the **right connection succeeds**, the **wrong one fails**, and my recovery path remains available.

## References

- [Tailscale — Install on Linux](https://tailscale.com/docs/install/linux)
- [Tailscale — SSH over Tailscale vs. Tailscale SSH](https://tailscale.com/docs/reference/ssh-over-tailscale)
- [Tailscale — Access control and grants](https://tailscale.com/docs/features/access-control)
- [Ubuntu Server — OpenSSH](https://ubuntu.com/server/docs/how-to/security/openssh-server/)
- [OpenSSH manual — sshd_config](https://man.openbsd.org/sshd_config)
