# quadlet-gitlab

Quadlet setup for [GitLab Community Edition](https://gitlab.com/gitlab-org/gitlab) (`docker.io/gitlab/gitlab-ce:latest`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `gitlab.container` | Quadlet unit file |
| `gitlab.env` | Default environment variables |
| `gitlab.override.env.template` | Template for local overrides (external URL, Omnibus config) |
| `gitlab-backup.service` | Systemd service: backs up GitLab data and config |
| `gitlab-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/gitlab -s /usr/sbin/nologin gitlab

REPO_URL=https://github.com/mkoester/quadlet-gitlab.git
REPO=~gitlab/quadlet-gitlab
```

```sh
# 2. Enable linger
sudo loginctl enable-linger gitlab

# 3. Clone this repo into the service user's home
sudo -u gitlab git clone $REPO_URL $REPO

# 4. Create quadlet and data directories
sudo -u gitlab mkdir -p ~gitlab/.config/containers/systemd
sudo -u gitlab mkdir -p ~gitlab/{config,logs,data}

# 5. Create .override.env from template and set the external URL
sudo -u gitlab cp $REPO/gitlab.override.env.template $REPO/gitlab.override.env
sudo -u gitlab nano $REPO/gitlab.override.env

# 6. Symlink all quadlet files from the repo
sudo -u gitlab ln -s $REPO/gitlab.container ~gitlab/.config/containers/systemd/gitlab.container
sudo -u gitlab ln -s $REPO/gitlab.env ~gitlab/.config/containers/systemd/gitlab.env
sudo -u gitlab ln -s $REPO/gitlab.override.env ~gitlab/.config/containers/systemd/gitlab.override.env

# 7. Reload and start
sudo -u gitlab XDG_RUNTIME_DIR=/run/user/$(id -u gitlab) systemctl --user daemon-reload
sudo -u gitlab XDG_RUNTIME_DIR=/run/user/$(id -u gitlab) systemctl --user start gitlab

# 8. Verify (first boot runs gitlab-ctl reconfigure — allow a few minutes)
sudo -u gitlab XDG_RUNTIME_DIR=/run/user/$(id -u gitlab) systemctl --user status gitlab
```

## Configuration

### Environment variables

`gitlab.env` contains non-sensitive defaults:

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Container timezone |

`gitlab.override.env` (created from the template) holds instance-specific values:

| Variable | Description |
|---|---|
| `GITLAB_OMNIBUS_CONFIG` | Inline Ruby config — sets `external_url`, reverse proxy settings, SSH port, and timezone |

The minimum recommended `GITLAB_OMNIBUS_CONFIG` when running behind a reverse proxy:

```
GITLAB_OMNIBUS_CONFIG=external_url 'https://gitlab.example.com'; nginx['listen_port'] = 80; nginx['listen_https'] = false; gitlab_rails['gitlab_shell_ssh_port'] = 2222; gitlab_rails['time_zone'] = 'Europe/Berlin'
```

> **Note:** `nginx['listen_port']` and `nginx['listen_https']` configure GitLab's *built-in* Nginx (an Omnibus component), not the external reverse proxy (Caddy). These settings tell the internal Nginx to accept plain HTTP on port 80 and let Caddy handle TLS.

For more complex configuration, edit `~gitlab/config/gitlab.rb` directly after the first boot (GitLab creates it on first run). Restart after changes:

```sh
sudo -u gitlab XDG_RUNTIME_DIR=/run/user/$(id -u gitlab) systemctl --user restart gitlab
```

## Reverse proxy (Caddy)

Add the following snippet to your Caddyfile. Replace `gitlab.example.com` as needed:

```
gitlab.example.com {
    reverse_proxy localhost:8929
}
```

And add a DNS A/CNAME record for `gitlab.example.com` pointing to your server.

After editing the Caddyfile, reload Caddy:

```sh
sudo systemctl reload caddy
```

## SSH git access

GitLab's SSH daemon listens on port 22 inside the container, mapped to port **2222** on the host (rootless Podman cannot bind privileged ports). Git clone URLs will use this port:

```
git clone ssh://git@gitlab.example.com:2222/user/repo.git
```

To use standard SSH syntax without specifying the port, add this to `~/.ssh/config` on client machines:

```
Host gitlab.example.com
    Port 2222
```

### Firewall (firewalld)

Port 2222 must be opened in firewalld to allow inbound SSH git connections:

```sh
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload
```

Works similarly with other firewall solutions.

## Backup

GitLab's built-in backup tool (`gitlab-backup create`) archives repos, uploads, and the database into a tar file at `/var/opt/gitlab/backups/` (= `~gitlab/data/backups/` on the host). The config directory (`~gitlab/config/`) is backed up separately since it contains secrets not included in `gitlab-backup`. Requires `rsync` to be installed on the host. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup (group, backup user, SSH key).

```sh
# 1. Create backup staging directories (owned by gitlab, readable by backup-readers group)
sudo mkdir -p /var/backups/gitlab/{backups,config}
sudo chown -R gitlab:backup-readers /var/backups/gitlab
sudo chmod -R 750 /var/backups/gitlab

# 2. Symlink the backup service and timer from the repo
sudo -u gitlab mkdir -p ~gitlab/.config/systemd/user
sudo -u gitlab ln -s $REPO/gitlab-backup.service ~gitlab/.config/systemd/user/gitlab-backup.service
sudo -u gitlab ln -s $REPO/gitlab-backup.timer ~gitlab/.config/systemd/user/gitlab-backup.timer

# 3. Enable and start the timer
sudo -u gitlab XDG_RUNTIME_DIR=/run/user/$(id -u gitlab) systemctl --user daemon-reload
sudo -u gitlab XDG_RUNTIME_DIR=/run/user/$(id -u gitlab) systemctl --user enable --now gitlab-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@gitlab-host:/var/backups/gitlab/ /path/to/local/backup/gitlab/
```

This pulls both `backups/` (GitLab data archives) and `config/` (gitlab.rb and secrets) into the local backup directory.

## Notes

- Port `8929` is bound to `127.0.0.1` only — Caddy handles TLS termination.
- Port `2222` is exposed on all interfaces for SSH git access (remember to set up your firewall).
- **First boot is slow**: GitLab runs `gitlab-ctl reconfigure` on startup, which can take 5–10 minutes. `TimeoutStartSec=1800` is set accordingly.
- GitLab Omnibus runs as root inside the container. With rootless Podman, container root maps to the `gitlab` service user on the host — no `UserNS` override is needed.
- GitLab retains backup archives for 7 days by default (`backup_keep_time`). Adjust in `gitlab.rb` if needed.
- The config directory (`~gitlab/config/`) contains `gitlab-secrets.json`. Guard backup access accordingly.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u gitlab XDG_RUNTIME_DIR=/run/user/$(id -u gitlab) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning) for the one-time system setup). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u gitlab XDG_RUNTIME_DIR=/run/user/$(id -u gitlab) systemctl --user enable --now podman-image-prune@30.timer
  ```
