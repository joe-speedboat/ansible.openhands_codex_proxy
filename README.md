# Ansible Role: openhands_codex_proxy

Deploy OpenHands on Enterprise Linux 10 behind nginx HTTPS with `codex-as-api` as an OpenAI-compatible Codex OAuth proxy.

This role was refactored from a proven Rocky Linux 10 lab setup for OpenHands behind nginx with `codex-as-api` as Codex OAuth proxy.

## What it installs

- Docker CE and Docker Compose plugin
- nginx reverse proxy with HTTPS and `/runtime/<port>/...` support
- `codex-as-api` in a `node:22-alpine` container
- OpenHands in `docker.openhands.dev/openhands/openhands:1.7`
- optional OpenAI Codex CLI via npm
- self-signed TLS certificate with SAN for the VM FQDN
- OpenHands settings preseeded for `custom_openai/gpt-5.5`

## Requirements

- Ansible 2.9 or newer
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

## Installation

Install the role using Chris's Galaxy namespace-style role directory:

```bash
ansible-galaxy clone https://github.com/joe-speedboat/ansible.openhands_codex_proxy.git /etc/ansible/roles/joe-speedboat.openhands_codex_proxy
```

For project-local testing, keep the same role directory name under `./roles/`:

```text
roles/joe-speedboat.openhands_codex_proxy/
```

## Example Playbook

```yaml
---
- name: Deploy OpenHands Codex proxy
  hosts: openhands_codex_proxy_lab
  become: true
  vars:
    openhands_codex_proxy_fqdn: "{{ inventory_hostname }}"
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
