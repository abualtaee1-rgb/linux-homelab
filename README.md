# Linux Home Lab: Hardened Ubuntu Server

The first project in my DevOps portfolio. I built an Ubuntu Server 26.04 LTS
VM in VirtualBox (bridged networking) and hardened it step by step: key-only
SSH with root login disabled, a default-deny firewall with rate-limited SSH,
a least-privilege sudo rule, and fail2ban. I tested each layer and saved the
evidence in [`docs/evidence`](docs/evidence).

I'm studying DevOps engineering in Sweden, and this is the first step in
building a portfolio of hands-on projects. Below are the problems I hit along
the way and how I fixed them, since that's where I learned the most.

**Skills shown:** Linux administration, SSH hardening, UFW, sudo and least
privilege, fail2ban, Git and GitHub

## Setup

- **Host and hypervisor:** Windows 11, VirtualBox
- **VM:** Ubuntu Server 26.04.1 LTS, 2 GB RAM, 2 CPUs, 25 GB disk. I used
  VirtualBox's default sizing; measured usage (about 17% memory, 15% disk)
  showed it was plenty for a text-only server.
- **Why this OS:** it's the same release used in my course, so the lab
  reinforces what I'm studying. LTS is also what production teams run.
- **Networking:** Bridged adapter, so the VM behaves like a real server on my
  network. The trade-off is that other devices on my home network could reach
  it before hardening. I kept that window short and hardened SSH first. My
  router blocks unsolicited inbound traffic from the internet.
- **Access:** I connect from Windows PowerShell using the built-in OpenSSH
  client and an ed25519 key pair. The private key stays on my PC, protected
  by a passphrase, and the public key is in `authorized_keys` on the VM.

## Hardening steps

### SSH
Configured in `/etc/ssh/sshd_config.d/01-hardening.conf` ([file](docs/evidence/sshd-hardening.conf)):

- **`PermitRootLogin no`**: every Linux server has a `root` account, so attackers never have to guess the username. The Ubuntu default (`prohibit-password`) still allows root to log in with a key. Setting it to `no` removes that route entirely, so admin work happens through a named user and `sudo`.
- **`PasswordAuthentication no`**: removes password guessing as an attack. Only someone holding my private key can log in.
- **`KbdInteractiveAuthentication no`**: SSH can also ask for a password through PAM (keyboard-interactive). Ubuntu already ships with this off, and I set it explicitly so the hardening stays correct even if the default changes or the file is edited later.

### Firewall (UFW)
Default deny for incoming traffic, default allow for outgoing, and `ufw limit` on SSH (22/tcp). The limit rule temporarily blocks an address that makes too many connection attempts in a short time ([output](docs/evidence/ufw-status.txt)).

### Least-privilege sudo
Instead of giving every user full root, I created a `deployers` group and a rule in `/etc/sudoers.d/90-deployers` that lets its members run exactly one command as root: `systemctl status ssh` ([file](docs/evidence/sudoers-90-deployers.txt)). Test user `alice` is a member.

### fail2ban
The `sshd` jail reads from the systemd journal and bans an address after 3 failed logins within 1 minute, for 10 minutes ([config](docs/evidence/fail2ban-sshd.local)). The 1-minute window is short so the demo is quick. A real deployment would use a longer window so slow attackers are still caught.

### Updates
Installed all pending package updates with `apt update` and `apt upgrade`.

## How I verified it

| What | How I tested it | Result |
|---|---|---|
| Passwords are off | `ssh -o PubkeyAuthentication=no user@vm` | `Permission denied (publickey)`, with no password prompt |
| Key login works | Logged in from a second terminal before closing the first | Asked for my key passphrase, then logged in |
| Effective SSH settings | `sudo sshd -T` ([output](docs/evidence/sshd-effective-settings.txt)) | `permitrootlogin no`, `passwordauthentication no`, `kbdinteractiveauthentication no` |
| Firewall | `sudo ufw status verbose`, then a new SSH connection | Default deny incoming, SSH rate-limited, still reachable |
| sudo least privilege | `sudo -l -U alice` ([output](docs/evidence/alice-sudo-rights.txt)), then tested as alice | `status ssh` worked, `stop ssh` was refused |
| fail2ban | Failed 3 logins from my PC ([log](docs/evidence/fail2ban-log-ban-unban.txt)) | Ban logged in the same second as the third failure, and the connection then timed out |
| Settings survive a reboot | Rebooted, then re-ran the checks above | Everything still in force |

## What broke and how I fixed it

### 1. A sudoers rule that was valid but did nothing
I wrote a rule to let the `deployers` group run `systemctl status ssh`, but I
wrote the arguments as `/status /ssh`, as if they were paths. `visudo -c`
reported "parsed OK", so I almost moved on. Running `sudo -l -U alice` showed
the rule exactly as sudo understood it, slashes included, which would never
match a real command. Fixing it and testing as alice confirmed the allowed
command worked and `systemctl stop ssh` was refused.

**Lesson:** a syntax check proves the file parses, not that the rule does what
I meant. I now test the behaviour as the affected user.

### 2. Tuning fail2ban for the wrong threat
My first config was `maxretry = 10` and `findtime = 1m`, because I was thinking
about flood attacks. fail2ban doesn't measure traffic volume. It counts failed
authentication attempts per IP, so those values made brute-forcing easier, and
a flood from many addresses would never trip it. I changed `maxretry` to 3 and
tested from my PC: three failed logins, and the ban was logged the same second
as the third failure. The blocked connection timed out instead of returning
"Permission denied", which showed the firewall was dropping packets before sshd
saw them.

**Lesson:** know what a tool actually measures before tuning it.


### 3. fail2ban counted zero failures on my first test
I tested fail2ban by running `ssh wronguser@<vm-ip>` from the VM itself.
`fail2ban-client status sshd` still showed `Total failed: 0`. I first assumed
the filter wasn't matching the log lines, so I ran `fail2ban-regex` against the
real journal. It matched the "Invalid user" lines fine, so the filter wasn't the
problem. The cause was in `jail.conf`: `ignoreself = true` makes fail2ban skip
the server's own addresses, so failures from the VM never counted. Repeating
the test from my PC (a different address) produced three `Found` lines and a
ban within the same second.

**Lesson:** when a test shows nothing, check where the test came from before
blaming the tool. The test itself can be the problem.

### 4. A config file that was never saved
After editing `01-hardening.conf` in nano, `cat` returned "No such file". I had
exited without saving. I'd have found out much later if I hadn't checked, so I
now verify every change with `cat` and `sshd -T` before reloading. I also keep
a second SSH session open as a way back in until the new settings are proven.
