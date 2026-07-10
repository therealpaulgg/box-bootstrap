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

1. Installs pinned Chezmoi, BWS CLI, Git, Ansible prerequisites, and the
   required Ansible collection.
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
- One cloud-provided **initial access** path (for example Azure serial console,
  Azure Bastion, or temporary SSH). Tailscale is configured by this bootstrap,
  so it cannot be the mechanism that starts the first interactive session.

## Run on a fresh VM

Use the cloud's initial access path as `azureuser`, then:

```sh
sudo apt update
sudo apt install -y ansible git curl ca-certificates

git clone https://github.com/therealpaulgg/box-bootstrap.git ~/box-bootstrap
cd ~/box-bootstrap
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i localhost, -c local agent-box-bootstrap.yml --ask-become-pass
```

The playbook asks for:

- the BWS project ID holding the GitHub clone token when the token can access
  multiple projects (otherwise leave it blank);
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
written to disk by this bootstrap. The `agent` profile is rendered through
`azureuser`'s unlocked CLI into the agent destination, so no Bitwarden login
database, browser state, or session token is copied into the agent account.
The final durable secret mechanism is BWS, not Bitwarden CLI login state.

### Tailscale checkpoint

Tailscale is installed and its daemon is started in the bootstrap's first
phase, before any private source or profile configuration. When paused, use a
second initial-access terminal:

```sh
sudo tailscale up --hostname=agent-box
sudo tailscale status
```

Complete browser authentication, ensure the host appears in the expected
tailnet, then continue the playbook. The private playbook will publish the
loopback-only gateway on tailnet HTTPS port `443` and noVNC on `8443`.

## GlobalProtect

GlobalProtect is installed by default from the pinned vendor archive in the
private dotfiles source. The bootstrap configures the client, the dedicated
`azureuser` Google Chrome SAML browser, and its XDG callback handlers.

The first Okta/MFA sign-in remains interactive. Complete it through the noVNC
desktop; do not copy `~/.GlobalProtect`, browser profiles, OAuth state, or
cached credentials from another VM.

To omit GlobalProtect for a disposable bootstrap run:

```sh
ansible-playbook -i localhost, -c local agent-box-bootstrap.yml \
  --ask-become-pass -e install_globalprotect=false
```

## Refresh the agent Chezmoi profile

Run this **from an `azureuser` login**, never from an `agent` shell. The `agent`
account is intentionally non-sudo; running `sudo` after switching to that user
will prompt for an unusable agent password.

The agent owns a separate source checkout at
`/home/agent/.local/share/chezmoi`. First refresh that checkout using its
BWS-backed Git credential, then use `azureuser`'s temporary Bitwarden session
to render the agent profile:

```sh
# Optional but normally desired: fetch the current private dotfiles source.
sudo -u agent git -C /home/agent/.local/share/chezmoi pull --ff-only

# Bitwarden unlock is needed only while templates are rendered; it is not saved.
export BW_SESSION="$(bw unlock --raw)"

# Preview the result before changing the agent home directory.
sudo env \
  BITWARDENCLI_APPDATA_DIR="$HOME/.config/Bitwarden CLI" \
  BW_SESSION="$BW_SESSION" \
  chezmoi \
    --config /home/agent/.config/chezmoi/chezmoi.toml \
    --destination /home/agent \
    diff

# Apply after reviewing the diff. Do not add --force for normal refreshes.
sudo env \
  BITWARDENCLI_APPDATA_DIR="$HOME/.config/Bitwarden CLI" \
  BW_SESSION="$BW_SESSION" \
  chezmoi \
    --config /home/agent/.config/chezmoi/chezmoi.toml \
    --destination /home/agent \
    apply

# Root rendered the destination, so return all resulting files to agent.
sudo chown -R agent:agent /home/agent
unset BW_SESSION
```

This deliberately uses `azureuser`'s Bitwarden CLI application data only for
template rendering. It does not copy a Bitwarden session or login database to
`agent`; that account continues to retrieve durable secrets through BWS.

## Non-goals

- Provisioning ordinary personal/work machines.
- Persisting a Bitwarden unlock token.
- Copying human desktop/editor/history/browser configuration to `agent`.
- Automating Tailscale browser approval or GlobalProtect SAML/MFA.
- Exposing CDP, VNC, or the gateway directly to the LAN.
