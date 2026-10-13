# Ansible: Splunk Universal Forwarder on Linux clients

![ansible-lint](https://github.com/antonvikstrom/ansible-splunk-forwarder/actions/workflows/ansible-lint.yml/badge.svg)

Ansible playbooks that take a Linux machine in my home lab from a clean snapshot to sending logs to Splunk, with one command per step and no manual changes.

This is part of my home lab. See [building-secure-home-lab](https://github.com/antonvikstrom/building-secure-home-lab) for the network and the SIEM.

## What it does

- `bootstrap.yml` logs in with the local admin account once and creates an `ansible` user with sudo rights and key-only login.
- `install_uf.yml` installs or upgrades the Splunk Universal Forwarder, sends to the Splunk server on port 9997, and monitors `/var/log/auth.log` (as `linux_secure`) and `/var/log/syslog` (as `syslog`) into the `linux` index.
- A second run reports `changed=0`. The playbooks only change what is missing.

## Setup

| Machine | Role | IP |
|---|---|---|
| ansible-ctl | Control node (Ubuntu 24.04 LXC on Proxmox) | 10.1.1.30 |
| nix-client01 | Target (Ubuntu) | 10.1.3.101 |
| splunk-siem | Splunk Enterprise | 10.1.1.10 |

pfSense only allows SSH from ansible-ctl to the target.

## Usage

```bash
# Put the Splunk UF .deb in files/ (not in git, see files/README.md)
ansible-vault create group_vars/linux_clients/vault.yml   # see vault.yml.example
ansible-playbook bootstrap.yml -k -K                       # once per new machine
ansible-playbook install_uf.yml --ask-vault-pass
```

## Decisions

- **A separate control node instead of my Mac.** It sits in the server zone, so the firewall rule is one host and one port.
- **A dedicated `ansible` user with key login.** Not root, and not my own account. Easy to see in the logs what Ansible did.
- **The sudo password in Ansible Vault.** It is never typed in or stored in plain text. The vault file itself is not in the repo either, since it can be cracked offline if the vault password is weak.
- **A deploy key for GitHub.** The control node can push to this repo only, not to my whole account.
- **ansible-lint in GitHub Actions.** Every push is checked against the `production` profile.

## Problems I ran into

- **sudo-rs.** Newer Ubuntu uses sudo-rs, and Ansible timed out waiting for a sudo prompt it didn't recognize. Classic sudo is still installed as `sudo.ws`, so I set `become_exe = sudo.ws` in `ansible.cfg`.
- **Wrong documentation.** My notes said the client was `linux-client01` at 10.1.3.150. The first connection test showed it was really `nix-client01` at 10.1.3.101. I fixed the notes, not the machine. Keeping the inventory in code helps prevent this.
- **The forwarder stopped after an upgrade.** The new version waited for license acceptance and never started, so no data arrived. Searching with `_index_earliest` (arrival time) instead of `earliest` showed it right away, since old syslog lines keep their old timestamps.
- **The upgrade failed on a clean snapshot.** Splunk will not migrate while it is running. The playbook now does a dry run of the install, stops the forwarder only if an upgrade is needed, and accepts the license before starting it again. These tasks must run before handlers would, so they have a `noqa` comment explaining why.
- **Local lint passed, CI failed.** Ubuntu's ansible-lint was older than the one in GitHub Actions and missed the `partial-become` rule. CI is the version that counts.

## Next

- Windows clients over WinRM (forwarder and Sysmon)
- Move the tasks into a role when there is more than one thing to automate
