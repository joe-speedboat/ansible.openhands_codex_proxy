# Ansible Role: openhands_codex_proxy

Deploy OpenHands on Enterprise Linux 10 behind nginx HTTPS with `codex-as-api` as an OpenAI-compatible Codex OAuth proxy.

This role was refactored from a proven Rocky Linux 10 lab setup for OpenHands behind nginx with `codex-as-api` as Codex OAuth proxy.

## What it installs

- Docker CE and Docker Compose plugin
- nginx reverse proxy with HTTPS and `/runtime/<port>/...` support
- nginx Basic Auth protection for the HTTPS reverse proxy
- `codex-as-api` in a `node:22-alpine` container
- OpenHands in `docker.openhands.dev/openhands/openhands:1.7`
- optional OpenAI Codex CLI via npm
- self-signed TLS certificate with SAN for the VM FQDN
- OpenHands settings preseeded for `custom_openai/gpt-5.5`

## Requirements

- Ansible Core 13.3 or newer (tested with)
- Rocky Linux / RHEL-compatible EL 10 target
- Root privileges via `become: true`
- Network access from target to Docker CE repository, npm, and container registries

## Role Variables

Important defaults are in `defaults/main.yml`.

| Variable | Default | Purpose |
| --- | --- | --- |
| `openhands_codex_proxy_fqdn` | `{{ ansible_fqdn | default(inventory_hostname) }}` | External HTTPS FQDN for nginx and OpenHands runtime URLs |
| `openhands_codex_proxy_project_dir` | `/opt/openhands-codex-proxy` | Deployment root on the target |
| `openhands_codex_proxy_codex_model` | `gpt-5.5` | Model exposed through codex-as-api |
| `openhands_codex_proxy_codex_proxy_port` | `18080` | Docker bridge proxy port |
| `openhands_codex_proxy_openhands_port` | `3000` | OpenHands container port on localhost/bridge |
| `openhands_codex_proxy_install_codex_cli` | `true` | Install `@openai/codex` globally |
| `openhands_codex_proxy_enable_firewall` | `true` | Open http/https in firewalld |
| `openhands_codex_proxy_do_basic_auth` | `true` | Enable nginx Basic Auth on the HTTPS virtual host |
| `openhands_codex_proxy_basic_auth_user` | `open` | Basic Auth username |
| `openhands_codex_proxy_basic_auth_password` | `hands` | Basic Auth password |
| `openhands_codex_proxy_basic_auth_realm` | `OpenHands` | Basic Auth realm shown by clients |
| `openhands_codex_proxy_runtime_access_mode` | `allowlist` | Runtime proxy access mode: `public` or `allowlist` |
| `openhands_codex_proxy_runtime_allowed_cidrs` | `[ '10.0.0.0/8', '192.168.0.0/16', '172.16.0.0/12' ]` | CIDR list rendered as nginx `allow` rules when runtime access mode is `allowlist` |

Override the default Basic Auth password for every non-lab deployment. The shipped `open` / `hands` default is intentionally simple for first-boot lab access, not a production secret.

Basic Auth protects the OpenHands Web UI. The `/runtime/<port>/...` sandbox proxy path explicitly disables inherited Basic Auth so OpenHands agent-server and browser-accessed sandbox previews can work without receiving browser Basic Auth credentials.

> [!WARNING]
> `/runtime/<port>/...` is intentionally reachable without nginx Basic Auth when `openhands_codex_proxy_do_basic_auth` is enabled. This path is used by OpenHands runtime/sandbox traffic and can also be opened by Web UI users for previews. Do not expose this role directly to the public internet unless the host is additionally protected, for example by VPN, trusted source IP filtering, or another auth-aware proxy layer.

For internet-facing or semi-public deployments, prefer restricting runtime preview access to trusted networks:

```yaml
openhands_codex_proxy_runtime_access_mode: allowlist
openhands_codex_proxy_runtime_allowed_cidrs:
  - 10.0.0.0/8
  - 192.168.0.0/16
  - 172.16.0.0/12
```

This renders nginx `allow ...; deny all;` directives inside the `/runtime/<port>/...` location. Keep `public` only for labs, VPN-only hosts, or deployments protected by another edge layer.

## Installation

Install the role using Chris's Galaxy namespace-style role directory:

```bash
ansible-galaxy role install joe-speedboat.openhands_codex_proxy
```
or
```bash
git clone https://github.com/joe-speedboat/ansible.openhands_codex_proxy.git /etc/ansible/roles/joe-speedboat.openhands_codex_proxy
```

## Example Playbook

```yaml
---
- name: Deploy OpenHands Codex proxy
  hosts: openhands_codex_proxy_lab
  become: true
  vars:
    openhands_codex_proxy_fqdn: "{{ inventory_hostname }}"
    openhands_codex_proxy_basic_auth_user: open
    openhands_codex_proxy_basic_auth_password: hands
  roles:
    - joe-speedboat.openhands_codex_proxy
...
```

## Post-install Codex OAuth login

The role starts OpenHands and the proxy without copying any credential into Git or workdir. Real model calls require a Codex OAuth login on the VM:

```bash
sudo env CODEX_HOME=/opt/openhands-codex-proxy/codex /usr/local/bin/codex login --device-auth
sudo docker compose -f /opt/openhands-codex-proxy/docker-compose.yml restart codex-proxy openhands
```

Never commit `/opt/openhands-codex-proxy/codex/auth.json` or any device-auth code.

## Verification

On the target:

```bash
sudo nginx -t
sudo docker compose -f /opt/openhands-codex-proxy/docker-compose.yml ps
curl -skI https://<vm-fqdn>/ | sed -n '1,8p'
curl -skI -u open:hands https://<vm-fqdn>/ | sed -n '1,8p'
curl -sS http://172.17.0.1:18080/health
```

After Codex OAuth login, verify a real `/v1/chat/completions` request and the OpenHands WebGUI flow.

## Notes

- External/browser URLs are HTTPS-only.
- Port 80 is only a redirect to HTTPS.
- `codex-as-api` is published only on Docker bridge gateway `172.17.0.1:18080`.
- Use `custom_openai/<model>` for OpenHands, not `openai/<model>`, to avoid `/v1/responses` API selection.

## License

GPLv3

Copyright (c) Chris Ruettimann <chris@bitbull.ch>
