# sol1-rustguac

Ansible role to install and configure [rustguac](https://github.com/sol1/rustguac) — a lightweight remote access gateway for SSH, RDP, VNC, and web browser sessions.

## What it does

1. Installs rustguac from the Sol1 apt repo (`packages.sol1.net`, via `sol1-packages_repo`)
2. Deploys `config.toml` with all settings templated
3. Deploys secrets (`env` file for Vault/OIDC)
4. Optionally creates an admin API key for initial setup
5. Optionally installs and configures HAProxy as a TLS-terminating reverse proxy
6. Starts rustguac systemd services (rustguac + guacd)

## Requirements

- Debian 13 (Trixie) or Ubuntu 24.04+
- Root access
- The [`sol1-packages_repo`](https://github.com/sol1-ansible/sol1-packages_repo) role on
  the roles path (deb install method) — rustguac is installed from the
  `packages.sol1.net` apt repo that this role configures.
- The [`sol1-haproxy`](https://github.com/sol1/sol1-haproxy) role on the roles path
  when `rustguac_haproxy_enabled` is set (rustguac contributes its frontend/backend to
  it via `include_role`).
- The [`sol1-hashivault`](https://github.com/sol1/sol1-hashivault) role on the roles
  path when `rustguac_vault_install` is set (rustguac installs/bootstraps a local Vault
  through it and consumes the generated AppRole credentials). See `requirements.yml`.

## Usage

### Minimal (rustguac only)

```yaml
- hosts: gateway
  become: true
  roles:
    - role: sol1-rustguac
      rustguac_admin_name: admin
```

### Production (OIDC + Vault + HAProxy)

```yaml
- hosts: gateway
  become: true
  roles:
    - role: sol1-rustguac
      rustguac_listen_addr: "127.0.0.1:8089"
      rustguac_trusted_proxies: ["127.0.0.1/32"]

      # OIDC
      rustguac_oidc_enabled: true
      rustguac_oidc_issuer_url: "https://your-idp.example.com"
      rustguac_oidc_client_id: rustguac
      rustguac_oidc_client_secret: "{{ vault_oidc_secret }}"
      rustguac_oidc_redirect_uri: "https://console.example.com/auth/callback"
      rustguac_group_role_mappings:
        Admins: admin
        Users: operator

      # Vault address book
      rustguac_vault_enabled: true
      rustguac_vault_addr: "https://vault.example.com:8200"
      rustguac_vault_role_id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      rustguac_vault_secret_id: "{{ vault_secret_id }}"

      # Network allowlists
      rustguac_allowed_ssh_cidrs: ["10.0.0.0/8"]
      rustguac_allowed_rdp_cidrs: ["10.0.0.0/8"]

      # HAProxy
      rustguac_haproxy_enabled: true
      rustguac_haproxy_cert: /etc/ssl/private/console.pem
```

### Full stack (gateway + jumpboxes)

```yaml
- hosts: gateway
  become: true
  roles:
    - role: sol1-rustguac
      rustguac_haproxy_enabled: true
      # ... (as above)

- hosts: jumpboxes
  become: true
  roles:
    - role: sol1-xrdp-desktop
      xrdp_desktop: xfce
      xrdp_enable_h264: true
      xrdp_enable_audio: true
```

See [sol1-xrdp-desktop](https://github.com/sol1-ansible/sol1-xrdp-desktop) for the jumpbox role.

## Variables

### Core

| Variable | Default | Description |
|----------|---------|-------------|
| `rustguac_version` | `latest` | Version to install (`latest` or `1.0.0`) |
| `rustguac_install_method` | `deb` | Install method: `deb` |
| `rustguac_prefix` | `/opt/rustguac` | Install directory |
| `rustguac_listen_addr` | `127.0.0.1:8089` | Listen address (loopback for HAProxy) |
| `rustguac_admin_name` | `""` | Create admin API key with this name (empty = skip) |
| `rustguac_session_max_duration_secs` | `28800` | Max session duration (8h) |

### OIDC

| Variable | Default | Description |
|----------|---------|-------------|
| `rustguac_oidc_enabled` | `false` | Enable OIDC SSO |
| `rustguac_oidc_issuer_url` | `""` | OIDC provider URL |
| `rustguac_oidc_client_id` | `""` | Client ID |
| `rustguac_oidc_client_secret` | `""` | Client secret (stored in env file) |
| `rustguac_oidc_redirect_uri` | `""` | Callback URL |
| `rustguac_group_role_mappings` | `{}` | Group-to-role map |

### Vault

| Variable | Default | Description |
|----------|---------|-------------|
| `rustguac_vault_enabled` | `false` | Enable Vault address book |
| `rustguac_vault_addr` | `""` | Vault server URL |
| `rustguac_vault_role_id` | `""` | AppRole role ID |
| `rustguac_vault_secret_id` | `""` | AppRole secret ID (stored in env file) |

### Network & Security

| Variable | Default | Description |
|----------|---------|-------------|
| `rustguac_trusted_proxies` | `[]` | CIDR list of trusted reverse proxies |
| `rustguac_allowed_ssh_cidrs` | `[]` | Allowed SSH target CIDRs |
| `rustguac_allowed_rdp_cidrs` | `[]` | Allowed RDP target CIDRs |
| `rustguac_allowed_vnc_cidrs` | `[]` | Allowed VNC target CIDRs |

### HAProxy (optional)

| Variable | Default | Description |
|----------|---------|-------------|
| `rustguac_haproxy_enabled` | `false` | Install and configure HAProxy |
| `rustguac_haproxy_cert` | `/etc/ssl/private/rustguac.pem` | TLS certificate path |

### Drive & Recording

| Variable | Default | Description |
|----------|---------|-------------|
| `rustguac_drive_enabled` | `false` | Enable file transfer |
| `rustguac_recording_rotation_enabled` | `true` | Auto-rotate recordings |
| `rustguac_recording_max_disk_percent` | `80` | Disk usage threshold |

## Related

- [rustguac](https://github.com/sol1/rustguac) -- the remote access gateway
- [sol1-xrdp-desktop](https://github.com/sol1-ansible/sol1-xrdp-desktop) -- prepare xrdp jumpboxes with H.264
- [Deployment guide](https://github.com/sol1/rustguac/blob/main/docs/deployment-guide.md) -- production planning

## License

Apache-2.0
