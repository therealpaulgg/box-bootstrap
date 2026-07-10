# Agent Box bootstrap

A small, public Ansible entry point for provisioning the dedicated two-account
Agent Box topology from a **private** Chezmoi repository.

This repository intentionally does **not** contain dotfiles, BWS project IDs,
Bitwarden item IDs, MCP URLs, OAuth state, GitHub tokens, or GlobalProtect
session data. It only obtains a short-lived clone credential from BWS, clones
the private repository, and runs the private repository's existing Agent Box
playbooks.

It is not a replacement for the private repository's normal Chezmoi and
Ansible workflow on personal or ordinary work machines.

## Account model

```text
azureuser  trusted administrator; sudo; full work Chezmoi profile
agent      non-sudo coding runtime; strict headless agent Chezmoi profile
```

The bootstrap verifies that `agent` is not in `sudo`, `admin`, or `wheel`.

## What the bootstrap automates

1. Installs Chezmoi, BWS CLI, Git, Ansible prerequisites, and the required
   Ansible collection.
2. Prompts privately for **separate** BWS access tokens for `azureuser` and
   `agent`, then stores each token only in that account's `0600` local file.
3. Uses the `azureuser` BWS token to read a caller-selected GitHub token secret
   and clones the private dotfiles repository through a temporary `GIT_ASKPASS`
   helper. The helper and credential are discarded immediately after cloning.
4. Uses the private source to create `agent`, apply `work` to `azureuser` and
   `agent` to `agent`, and invoke the existing private host, workstation, and
   agent provisioning playbooks in their required order.
5. Pauses for the only human authentication steps that should not be automated:
   Bitwarden unlock and Tailscale enrollment.
6. Starts the Agent browser/gateway runtime, publishes it through Tailscale
   Serve, and validates the user boundary, tools, and local browser endpoints.

## Prerequisites

- A clean Ubuntu/Debian Azure VM with the administrator account named
  `azureuser` and working `sudo`.
- A BWS machine account token for **each** Unix account.
- A BWS secret, in the single BWS project available to `azureuser`'s token,
  whose value is a GitHub token allowed to clone the private dotfiles repo.
- The full private dotfiles repository, including its vendor GlobalProtect
  archive if you later opt in to client installation.
- Permission to enroll the host in the intended Tailscale tailnet.

## Run on a fresh VM

SSH as `azureuser`, then:

```sh
sudo apt update
sudo apt install -y ansible git curl ca-certificates

git clone https://github.com/therealpaulgg/box-bootstrap.git ~/box-bootstrap
cd ~/box-bootstrap
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i localhost, -c local agent-box-bootstrap.yml --ask-become-pass
```

The playbook asks for:

- the BWS secret key holding the GitHub clone token;
- a BWS access token for `azureuser`;
- a separate BWS access token for `agent`;
- the Bitwarden item name containing the work MCP gateway fields.

All token prompts are hidden and marked `no_log`. Do not pass any secret via
`-e`, an environment variable in shell history, or a command-line argument.

### Bitwarden checkpoint

When paused, use a second terminal as `azureuser`:

```sh
bw login
bw unlock --raw
```

Paste the resulting `BW_SESSION` only into the bootstrap's hidden prompt. It
is used only to render the two Chezmoi work-derived profiles and is never
written to disk by this bootstrap. The final durable secret mechanism is BWS,
not Bitwarden CLI login state on `agent`.

### Tailscale checkpoint

When paused, use a second terminal:

```sh
sudo tailscale up --hostname=agent-box
sudo tailscale status
```

Complete browser authentication, ensure the host appears in the expected
tailnet, then continue the playbook. The private playbook will publish the
loopback-only gateway on tailnet HTTPS port `443` and noVNC on `8443`.

## GlobalProtect

GlobalProtect is deliberately disabled by default:

```text
install_globalprotect=false
```

The first Linux SAML login is not yet a proven automated/headless path. Do not
copy `~/.GlobalProtect`, browser profiles, OAuth state, or cached credentials
from an existing VM.

After its first-login path has been separately validated, install the pinned
client without automating the login itself:

```sh
ansible-playbook -i localhost, -c local agent-box-bootstrap.yml \
  --ask-become-pass -e install_globalprotect=true
```

## Non-goals

- Provisioning ordinary personal/work machines.
- Persisting a Bitwarden unlock token.
- Copying human desktop/editor/history/browser configuration to `agent`.
- Automating Tailscale browser approval or GlobalProtect SAML/MFA.
- Exposing CDP, VNC, or the gateway directly to the LAN.
