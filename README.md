Ansible Role: RustFS Server Installation and Configuration
=========

This role installs and configures [RustFS](https://rustfs.com) — a high-performance, S3-compatible distributed object storage system written in Rust — on a Linux server.

Requirements
------------

The target host must have access to GitHub releases (github.com) to download the RustFS binaries.

Role Variables
--------------

Available variables are listed below along with their default values (see `defaults/main.yml`).

### Installation options

Whether to install the RustFS server, the CLI client (`rc`), or both:

```yaml
rustfs_install_server: true
rustfs_install_client: true
```

### Version

RustFS server and CLI versions to install:

```yaml
rustfs_server_version: "1.0.0-beta.7"
rustfs_cli_version: "v0.1.20"
```

### System user and directories

Unix user/group that runs the RustFS service:

```yaml
rustfs_group: rustfs
rustfs_user: rustfs
```

Configuration directory, data directories, and log directory:

```yaml
rustfs_etc_dir: /etc/rustfs
rustfs_log_dir: /var/log/rustfs

rustfs_server_make_datadirs: true
rustfs_server_datadirs:
  - /var/lib/rustfs
```

Binary installation paths:

```yaml
rustfs_server_bin: /usr/local/bin/rustfs
rustfs_client_bin: /usr/local/bin/rc
```

### Network

Server listen address (`rustfs_server_addr` — empty means all interfaces), S3 API port, and web console port:

```yaml
rustfs_server_port: "9000"
rustfs_console_port: "9001"
rustfs_server_addr: ""
```

### Credentials

Admin (root) user credentials for the RustFS server:

```yaml
rustfs_root_user: ""
rustfs_root_password: ""
```

### Console and logging

```yaml
rustfs_console_enable: true
rustfs_obs_logger_level: "error"
```

### Build variant

Use musl (static) builds for maximum compatibility. Set to `false` for glibc builds:

```yaml
rustfs_use_musl: true
```

### Download URLs

Base URLs for downloading release artifacts:

```yaml
rustfs_download_base_url: "https://github.com/rustfs/rustfs/releases/download"
rustfs_cli_download_base_url: "https://github.com/rustfs/cli/releases/download"
```

### Client configuration

CLI alias name and TLS certificate validation:

```yaml
rustfs_alias: "myrustfs"
rustfs_validate_certificate: true
```

Connection retry settings for post-install configuration:

```yaml
rustfs_client_connect_retries: 30
rustfs_client_connect_delay: 5
```

### Buckets

Buckets to create after the server is installed. Each bucket requires a `name`:

```yaml
rustfs_buckets: []
# rustfs_buckets:
#   - name: bucket1
```

### TLS

To enable TLS, set `rustfs_enable_tls` to `true` and provide the private key and public certificate content in `rustfs_tls_key` and `rustfs_tls_cert` variables.

```yaml
rustfs_enable_tls: false
rustfs_tls_cert: ""
rustfs_tls_key: ""
rustfs_tls_path: "{{ rustfs_etc_dir }}/ssl"
```

When TLS is enabled, the role:
- Creates the TLS directory (`/etc/rustfs/ssl` by default)
- Copies the certificate as `rustfs_cert.pem` and the key as `rustfs_key.pem`
- Sets `RUSTFS_TLS_PATH` in the environment file
- Configures the server URL to use `https://`

Cert and key content can be loaded from files using Ansible tasks:

```yaml
- name: Load tls key and cert from files
  set_fact:
    rustfs_tls_key: "{{ lookup('file', 'certificates/' + inventory_hostname + '_private.key') }}"
    rustfs_tls_cert: "{{ lookup('file', 'certificates/' + inventory_hostname + '_public.crt') }}"
```

### Extra configuration

Additional environment variables and CLI options passed to the RustFS server:

```yaml
rustfs_server_env_extra: ""
rustfs_server_opts: ""
```

Development Environment (uv)
----------------------------

This role uses [uv](https://docs.astral.sh/uv/) to manage Python tooling:

```shell
# Setup
uv sync

# Linting
uv run ansible-lint .
uv run yamllint .

# Molecule test (Docker required)
uv run molecule test -s default
```

Or use Makefile shortcuts:

```shell
make sync       # install dependencies
make lint       # run ansible-lint + yamllint
make molecule   # run molecule test (default scenario)
```

Example Playbook
----------------

Minimal playbook to install and configure RustFS:

```yaml
---
- name: Install and configure RustFS Server
  hosts: rustfs
  become: true
  gather_facts: true
  roles:
    - role: ricsanfre.rustfs
      rustfs_root_user: "rustfsadmin"
      rustfs_root_password: "supers1cret0"
```

Playbook with custom data directories and buckets:

```yaml
---
- name: Install and configure RustFS Server
  hosts: rustfs
  become: true
  gather_facts: true
  roles:
    - role: ricsanfre.rustfs
      rustfs_root_user: "rustfsadmin"
      rustfs_root_password: "supers1cret0"
      rustfs_server_datadirs:
        - /data/rustfs1
        - /data/rustfs2
      rustfs_buckets:
        - name: backups
        - name: media
```

Playbook with TLS enabled using self-signed certificates:

```yaml
---
- name: Install and configure RustFS Server with TLS
  hosts: rustfs
  become: true
  gather_facts: true
  vars:
    server_hostname: rustfs.example.com
    ssl_key_size: 4096
    ssl_certificate_provider: selfsigned

  pre_tasks:
    - name: Generate self-signed SSL certificates for rustfs
      include_tasks: generate_selfsigned_cert.yml
      args:
        apply:
          delegate_to: localhost
          become: false
    - name: Load tls key and cert
      set_fact:
        rustfs_tls_key: "{{ lookup('file', 'certificates/' + inventory_hostname + '_private.key') }}"
        rustfs_tls_cert: "{{ lookup('file', 'certificates/' + inventory_hostname + '_public.crt') }}"

  roles:
    - role: ricsanfre.rustfs
      rustfs_root_user: "rustfsadmin"
      rustfs_root_password: "supers1cret0"
      rustfs_enable_tls: true
      rustfs_validate_certificate: false
      rustfs_buckets:
        - name: bucket1
```

Where `generate_selfsigned_cert.yml` contains the tasks for generating a private key and self-signed certificate:

```yaml
---
- name: Create private certificate
  community.crypto.openssl_privatekey:
    path: "certificates/{{ inventory_hostname }}_private.key"
    size: "{{ ssl_key_size | int }}"
    mode: "0644"

- name: Create CSR
  community.crypto.openssl_csr:
    path: "certificates/{{ inventory_hostname }}_cert.csr"
    privatekey_path: "certificates/{{ inventory_hostname }}_private.key"
    common_name: "{{ server_hostname }}"

- name: Create certificates for keystore
  community.crypto.x509_certificate:
    csr_path: "certificates/{{ inventory_hostname }}_cert.csr"
    path: "certificates/{{ inventory_hostname }}_public.crt"
    privatekey_path: "certificates/{{ inventory_hostname }}_private.key"
    provider: selfsigned
```

License
-------

MIT

Author Information
------------------

Created by Ricardo Sanchez (ricsanfre)
