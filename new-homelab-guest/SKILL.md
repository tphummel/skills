---
name: new-homelab-guest
description: Add a new LXC guest to the homelab across both terraform and ansible repos
argument-hint: [guest-name]
---

# Add a new LXC guest to the homelab

You are adding a new Proxmox LXC guest named `$ARGUMENTS` to the homelab infrastructure.

## Step 0: Locate repos

Before doing anything else, identify the terraform and ansible repos. Check in this order:

1. Look for `./terraform/` and `./ansible/` relative to the current working directory.
2. If not found, search `~/Code` for the homelab terraform repo: `fd -t d terraform ~/Code --max-depth 3` then verify it contains `main.tf` with proxmox LXC resources (`rg 'proxmox_lxc' <path>/main.tf`).
3. Search `~/Code` for the ansible repo: `fd -t d ansible ~/Code --max-depth 3` then verify it contains homelab playbooks (`rg 'hummel.casa' <path>/*.yml --max-count 1`). There may be multiple ansible dirs; pick the one with active playbooks.

Set `$TERRAFORM_REPO` and `$ANSIBLE_REPO` to the absolute paths of those directories. Use these variables for all subsequent file paths.

---

## Information to gather from the user

Ask the user for all of these before writing any files:

### Infrastructure (Terraform)

#### MAC address and IP address (gather first)

Before asking other questions, scan `$TERRAFORM_REPO/main.tf` to find all MAC addresses (`0A:BC:DE:00:00:XX`) and IP addresses (`10.20.71.XXX`) currently in use. Determine the next available values (highest in-use + 1 for each).

Ask the user for MAC and IP together in a single question using `AskUserQuestion`. Present:
- **Option 1**: The next sequential MAC and IP (e.g., `0A:BC:DE:00:00:62 / 10.20.71.157`) with a description noting these are the next available
- **Option 2**: "I'll specify my own" — so the user can type custom values via the "Other" free-text input

The user will frequently want to specify their own values rather than using the next sequential ones. After receiving the user's choice, verify the chosen MAC and IP are not already in use by grepping `$TERRAFORM_REPO/main.tf` for both values. If either is taken, inform the user and ask again.

#### Remaining infrastructure settings
1. **Proxmox target node** (available: pve, pve2, pve4, pve5, pve6, pve7, pve8, pve9)
2. **CPU cores** (typically 1)
3. **RAM in MiB** (common: 256, 512, 1024, 2048, 4096, 16384)
4. **Root filesystem size** (common: 5G, 8G, 10G, 12G, 16G, 24G, 25G, 100G)
5. **NAS mount needed?** (mountpoint for nas-homelab-guests at /mnt/nas)

### Configuration (Ansible)
6. **Does this guest run a container?** If so, what image, tag, and port? (podman is the default runtime; only use Docker if explicitly requested)
7. **Does it need Caddy reverse proxy with TLS?** (most guests do)
8. **What port does the service expose?** (for Caddy target_port)
9. **Does it need Node.js?** If so, which LTS version?
10. **Any special packages, config files, or environment variables?**

---

## Part 1: Terraform

All file paths are relative to `$TERRAFORM_REPO/`.

### Step 1: Add guest definition to main.tf

Add a new entry inside the `locals.guests` block, before the closing `}`. Follow the existing format:

```hcl
guest_name = {
  mac              = "0A:BC:DE:00:00:XX"
  dhcp_reservation = "10.20.71.XXX"
  domain           = "guest-name.hummel.casa"
  target_node      = "pveX"
  cpu_cores        = 1
  ram_mib          = 2048
  root_fs_size     = "16G"
  playbook         = "guest-name.yml"
}
```

Use underscores in the HCL key name (e.g., `my_guest`), hyphens in the domain (e.g., `my-guest.hummel.casa`).

### Step 2: Create guest-name.tf

The file must contain these four resources:

1. `random_password.guest_name` - 30 char password with `_%@` specials
2. `proxmox_lxc.guest_name` - LXC container with:
   - `depends_on = [null_resource.guest_name_unifi_client]`
   - `ostemplate = local.debian_12_bookworm_lxc_template`
   - NAS mountpoint block (if requested)
   - SSH public keys: `lxc_guest_ssh`, `atlantis_root_2`, `gitea_hummel_casa`, `olivetin`
   - File + remote-exec provisioners for ansible-pull setup
   - SSH connection block using `var.atlantis_ansible_ssh_private_key_file`
   - `lifecycle { ignore_changes = [ostemplate] }`
3. `cloudflare_record.guest_name` - A record for the domain
4. `null_resource.guest_name_unifi_client` - UniFi DHCP reservation with:
   - Create provisioner: login, create client, poll for confirmation (120s timeout, 30s propagation wait)
   - Destroy provisioner: login, forget-sta by MAC
   - Both use `--insecure` flag for curl

**IMPORTANT**: Read an existing guest .tf file (like `$TERRAFORM_REPO/ntfy.tf`) before writing to ensure you match the exact current pattern, including the UniFi polling logic.

### Step 3: Terraform git workflow

Terraform changes always go to a feature branch, never directly to main:

```bash
cd $TERRAFORM_REPO
git checkout main && git pull origin main
git checkout -b add-guest-name-guest
git add main.tf guest-name.tf
git commit -m "add guest-name guest"
git push -u origin add-guest-name-guest
```

---

## Part 2: Ansible

All file paths are relative to `$ANSIBLE_REPO/`.

### Step 1: Create guest-name.yml

Read an existing playbook (e.g., `$ANSIBLE_REPO/verdaccio.yml`) to get the current `dns_api_token` value — do not hardcode it, copy it from there.

```yaml
---
- hosts: localhost
  become: yes
  gather_facts: yes
  vars:
    domain: 'guest-name.hummel.casa'
    dns_api_token: '<copy from existing playbook>'
    tls_email: 'tls@hummel.casa'
    # If container-based, add container vars:
    container:
      name: guest-name
      description: "Description of the service"
      image_name: 'dockerhub.hummel.casa/org/image'
      image_tag: 'v1.0.0'
      exposed_port: '8080'
  roles:
    - role: ansible-role-caddy-tls-dns
      vars:
        caddy_domain: '{{ domain }}'
        caddy_tls_email: '{{ tls_email }}'
        caddy_dns_api_token: '{{ dns_api_token }}'
        caddy_target_port: "{{ container.exposed_port }}"
    - role: geerlingguy.node_exporter
  tasks:
    # ... service-specific tasks
  handlers:
    # ... if needed
```

### Common ansible patterns

**Podman-based service (default)**: Use `$ANSIBLE_REPO/verdaccio.yml` as the reference pattern. The key elements are:

1. **Install podman** via apt (no role needed, just a task):
   ```yaml
   - name: Install podman
     apt:
       name:
         - podman
       state: present
       update_cache: yes
   ```

2. **Systemd service file** using podman directly (no daemon, no socket):
   ```yaml
   - name: Create systemd service file for {{ container.name }}
     copy:
       content: |
         [Unit]
         Description={{ container.description }}
         After=network-online.target
         Wants=network-online.target

         [Service]
         Restart=on-failure
         RestartSec=5s
         ExecStartPre=-/usr/bin/podman rm -f {{ container.name }}
         ExecStart=/usr/bin/podman run --name {{ container.name }} \
           --security-opt=no-new-privileges \
           --pids-limit=100 \
           --memory=512m \
           --memory-swap=512m \
           -p 127.0.0.1:{{ container.exposed_port }}:{{ container.exposed_port }} \
           -v /path/on/host:/path/in/container:ro \
           -v /path/on/host:/path/in/container \
           -e TZ=America/Los_Angeles \
           {{ container.image_name }}:{{ container.image_tag }}
         ExecStop=/usr/bin/podman stop -t 10 {{ container.name }}
         ExecStopPost=/usr/bin/podman rm -f {{ container.name }}
         TimeoutStopSec=30

         [Install]
         WantedBy=multi-user.target
       dest: /etc/systemd/system/{{ container.name }}.service
       mode: 0644
     notify:
       - Reload systemd
       - Start service
   ```

3. **Handlers** for reload, restart, and image cleanup:
   ```yaml
   handlers:
     - name: Reload systemd
       systemd:
         daemon_reload: yes

     - name: Start service
       systemd:
         name: "{{ container.name }}"
         enabled: yes
         state: restarted
       notify:
         - Cleanup old images

     - name: Cleanup old images
       listen: Start service
       shell: |
         set -euo pipefail
         repo="{{ container.image_name }}"
         keep_tag="{{ container.image_tag }}"
         keep_ref="${repo}:${keep_tag}"
         echo "==> Evaluating images for repo: ${repo}"
         podman image ls --format '{{'{{'}}.Repository{{'}}'}}:{{'{{'}}.Tag{{'}}'}}' "${repo}" || exit 0
         echo "==> Keeping: ${keep_ref}"
         old_images=$(podman image ls --format '{{'{{'}}.Repository{{'}}'}}:{{'{{'}}.Tag{{'}}'}}' "${repo}" \
           | awk -v keep="${keep_ref}" '$0 != keep && $0 !~ /<none>:/ {print $0}')
         if [ -z "$old_images" ]; then
           echo "==> No old images found to remove."
         else
           echo "==> Will remove:"
           echo "$old_images"
           echo "$old_images" | xargs -r -n1 podman image rm -f
         fi
         echo "==> Pruning dangling layers..."
         podman image prune -f --filter "dangling=true"
       args:
         executable: /bin/bash
       register: cleanup_result
       changed_when: true
       notify:
         - Show cleanup output

     - name: Show cleanup output
       debug:
         var: cleanup_result.stdout_lines
   ```

**Docker-based service (only if explicitly requested)**: Use the `geerlingguy.docker` role and read an existing Docker-based playbook (e.g., `$ANSIBLE_REPO/ntfy.yml`, `$ANSIBLE_REPO/mealie.yml`) for the pattern. Do NOT use Docker unless the user specifically asks for it.

**Node.js service** (requires `curl` installed first):
```yaml
- name: Install required packages
  apt:
    name:
      - curl
    state: present

- name: Check current Node.js version
  command: node --version
  register: node_version
  ignore_errors: yes
  changed_when: false

- name: Remove Debian default nodejs if not vXX
  apt:
    name: nodejs
    state: absent
    purge: yes
  when: node_version.rc == 0 and not node_version.stdout.startswith('vXX')

- name: Install Node.js XX LTS repository
  shell: |
    curl -fsSL https://deb.nodesource.com/setup_XX.x | bash -
  when: node_version.rc != 0 or not node_version.stdout.startswith('vXX')

- name: Install Node.js XX
  apt:
    name: nodejs
    state: latest
    update_cache: yes
```

**Note**: Debian 12 ships Node 18 without npm. The nodesource package bundles npm. You must purge the Debian nodejs first or apt won't upgrade, and `curl` is not installed by default in LXC containers.

**Conditional global npm install**:
```yaml
- name: Check if tool-name is installed
  command: which tool-name
  register: tool_check
  ignore_errors: yes
  changed_when: false

- name: Install tool-name globally if not on PATH
  command: npm install -g tool-name@latest
  when: tool_check.rc != 0
```

**Docker image registries**:
- `dockerhub.hummel.casa/` for DockerHub images
- `ghcr.hummel.casa/` for GitHub Container Registry images
- `registry.hummel.casa/` for custom images

### Step 2: Add Prometheus targets

Edit `$ANSIBLE_REPO/prometheus.yml` to add the new guest to both scrape target lists:

1. **node_exporter targets** — add `"guest-name.hummel.casa:9100"` to the `node_exporter` job's targets list (around line 62-96)
2. **caddy targets** — add `"guest-name.hummel.casa:2019"` to the `caddy` job's targets list (around line 101-130)

Read the file first to find the exact insertion points. Add new entries at the end of each target list, before the closing `]`.

### Step 3: Add link to start page

Edit `$ANSIBLE_REPO/templates/start_website_index.html.j2` to add a link for the new guest. Choose the appropriate section:

- **Apps** — user-facing applications (around line 286)
- **Ops Apps** — infrastructure/monitoring tools (around line 308)
- **Automation Tools** — automation services (around line 344)
- **Gaming** — game servers (around line 368)

Add a `<dt>` entry in the chosen section's `<dl>`:
```html
<dt><a href="https://guest-name.hummel.casa">Guest Display Name</a></dt>
```

Ask the user which section the new guest belongs in.

### Step 4: Ansible git workflow

Ansible playbooks are committed and pushed directly to main:

```bash
cd $ANSIBLE_REPO
git add guest-name.yml prometheus.yml templates/start_website_index.html.j2
git commit -m "add guest-name guest playbook"
git push origin main
```
