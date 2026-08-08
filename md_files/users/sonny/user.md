# User: sonny

language: ko

User-specific rules and preferences for sonny.
Global rules are in `md_files/agent.md`.

## Active Servers (auto-generated)

<!-- alias:project_servers:start -->
Active servers for user `sonny` on project `ultralytics` (filtered from `users.yaml` by `server_active_status=true` and project in `git_available_list`). For ssh user/key, IP, port, mount paths, alias_dir — read `users.yaml` → `users.sonny.servers.{name}` directly per-need (token-efficient).

| Server | push (this project) | sync_hub | GPU ids | Containers (this user) |
|--------|:-------------------:|:--------:|:-------:|------------------------|
| `local_pc` | – | – | 0 | `base_sonny` |
| `deep` | ✓ | ✓ | 0 | `base_sonny` |
| `n2` | – | – | 0 | `base_sonny` |
| `th1` | – | – | 0 | `base_sonny` |
| `ada2` | – | – | 0, 1 | `base_sonny` |
| `z3` | – | – | 0, 1 | `base_sonny` |

**Server usage rules**:

- Cross-server git / docker / rsync operations always go through the `git` / `docker` / `artifact_sync` / `run_scan` skills. Do not hand-roll `ssh ... 'git pull'` / `ssh ... 'docker exec'` (known-host, agent-forwarding, and chown gotchas).
- Training (RL/SL long-run) belongs on GPU servers. This project's push servers (`git_server_allow_push`) and the `sync_hub=true` server are for push / result collection; spread training load across the other GPU servers — see `md_files/code_style.md` → §12.5 Tests vs Experiments.
- Detailed info (ssh user / key, IP, port, source_mnt_path, alias_dir) is not duplicated here. Read `users.yaml` → `users.sonny.servers.{name}` directly when needed.
<!-- alias:project_servers:end -->

## Project Docker Containers (auto-generated)

<!-- alias:project_docker:start -->
Docker images for project `ultralytics` and user `sonny`. Generated from `users.yaml` by `setup.sh` (each user can have different images per project).

| Image | Source | Containers (this user) |
|-------|--------|------------------------|
| `base` (default) | `sonny0714/base:22.04` (linux/amd64)<br>`sonny0714/base:pi` (linux/arm64) | `base_sonny` (CPU only) |

**Container naming** (suffix `_sonny`):

- base image → `base_sonny` (CPU only, no GPU)
- other image, push server of the image's owning project → `<img>_test_sonny` (GPU 0)
- other image, per GPU → `<img>_<gpu_id>_sonny` (e.g. `<img>_0_sonny`, `<img>_1_sonny`)

This project has no user-specific images mapped — use the default `base_sonny` container.

For full Docker usage rules see `md_files/code_style.md` → §12.1 Container Discipline.
<!-- alias:project_docker:end -->
