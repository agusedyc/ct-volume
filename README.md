# CTool

Container Transfer Tool — backup, restore, and migrate entire Docker Compose projects (image + volume) between servers with ease.

[**English**](#english-version) · [**Bahasa Indonesia**](#versi-bahasa-indonesia)

---

## English Version

- [Features](#features)
- [Dependencies](#dependencies)
- [Installation](#installation)
- [Usage](#usage)
  - [Backup](#backup)
  - [Restore](#restore)
  - [Vol Backup](#vol-backup)
  - [Vol Restore](#vol-restore)
  - [List](#list)
  - [Inspect](#inspect)
- [Workflow](#workflow)
  - [Backup](#backup-1)
  - [Restore](#restore-1)
  - [Vol Backup](#vol-backup-1)
  - [Vol Backup --bundle](#vol-backup---bundle-1)
  - [Vol Restore](#vol-restore-1)
  - [Vol Restore --bundle](#vol-restore---bundle-1)
- [Complete Example](#complete-example)
- [Cross-Server Migration via Tailscale](#cross-server-migration-via-tailscale)
- [Backup Directory Structure](#backup-directory-structure)
- [Environment & Exit Codes](#environment--exit-codes)
- [Troubleshooting](#troubleshooting)
- [License](#license)

### Features

- **Backup** — backup a project directory + all Docker Compose volumes into a single archive
- **Restore** — restore project archive to current directory + restore all volumes
- **Vol Backup** — backup all (or specific) named volumes from compose file
- **Vol Restore** — restore volumes from backup archive, with confirmation prompt
- **List** — list named volumes defined in compose file
- **Inspect** — view Docker volume status and size from compose file
- **Cleanup** — remove all Docker resources (containers, volumes, images) from a project
- **Auto stop/start** — automatically stop services before vol-backup/vol-restore, restart after completion
- **Specific volume** — backup/restore specific volumes with `-v` flag
- **Retention policy** — keep N backups per volume with `--retain N`
- **Timestamped archive** — filename format `YYYYMMDD-HHMMSS-volume-name.tar.gz`
- **Minimal dependencies** — only requires `bash`, `docker`, `docker compose` (`jq` auto-installed if missing)
- **Multi compose support** — use compose files from any directory via `-f`
- **Smart volume mapping** — automatic compose key → Docker volume name mapping, safe migration between servers with different directory names

### Dependencies

| Tool | Description |
|------|-------------|
| `bash` ≥ 4.0 | Shell (built-in on all modern Linux) |
| `docker` | Docker Engine |
| `docker compose` | Docker Compose V2 (plugin `docker compose`) |
| `jq` | JSON processor for parsing compose config (auto-installed if missing) |
| `alpine` | Minimal image for backup/restore process (auto-pulled by Docker) |

#### Install dependencies

**Ubuntu / Debian:**
```bash
sudo apt update && sudo apt install -y docker.io docker-compose-v2 jq
```

**Arch Linux:**
```bash
sudo pacman -S docker docker-compose jq
```

**Fedora:**
```bash
sudo dnf install docker docker-compose jq
```

**macOS (Homebrew):**
```bash
brew install docker docker-compose jq
```

> Make sure Docker daemon is running and your user has access to the Docker socket (member of the `docker` group).
> `jq` does not need to be installed manually — CTool will install it automatically when needed.

### Installation

#### Via clone (run directly)

```bash
git clone https://github.com/yourusername/ctool.git
cd ctool
chmod +x ctool
./ctool --help
```

#### Install to system PATH (optional)

To call it from anywhere:

```bash
sudo cp ctool /usr/local/bin/
# or
ln -s "$(pwd)/ctool" ~/.local/bin/
```

Verify:
```bash
ctool --version
```

### Usage

#### Backup

Backup a project directory + all docker-compose volumes into a single archive.

```bash
ctool backup /path/to/project
```

If containers are running, a confirmation prompt will ask to stop them automatically. Use `--restart` to restart services after backup completes:

```bash
ctool backup /path/to/project --restart
```

Workflow:
1. Validate target directory
2. Search for `docker-compose.yml` recursively (including subdirectories)
3. Check containers — if running, confirm auto-stop (or abort if declined)
4. Backup all named volumes to `volumes/` folder (temporary)
5. Archive all project files + `volumes/` folder → `{dir-name}-backup-{timestamp}.tar.gz`
6. Output saved to current working directory

Output:
```
ℹ Using compose file: /path/to/project/docker-compose.yml
ℹ Copying project files...
ℹ Backing up volume: postgres_data
✓ Volume backup: postgres_data.tar.gz
ℹ Creating archive: my-project-backup-20250709-120000.tar.gz
✓ Backup created: .../my-project-backup-20250709-120000.tar.gz (1.2M)
```

Archive structure:
```
my-project-backup-20250709-120000.tar.gz
└── my-project/
    ├── docker-compose.yml
    ├── ... (all project files)
    └── volumes/
        ├── postgres_data.tar.gz
        └── redis_data.tar.gz
```

#### Restore

Restore archive (project + volume) to current working directory.

```bash
cd /target/location
ctool restore ../my-project-backup-20250709-120000.tar.gz
```

Workflow:
1. Extract archive to current directory
2. Find `volumes/` folder in extracted output
3. Detect compose file in extracted project
4. Map each compose key to the correct Docker volume name for the target server (via `docker compose config`)
5. Restore each docker volume from `.tar.gz` files in `volumes/` to the correct Docker volume name
6. Done — offer automatic `docker compose up -d`, or manual

> **Why this matters:** Docker Compose prefixes volume names based on the project directory name. If the directory differs between source and destination servers (e.g. `myapp` → `myapp-clone`), Docker creates new volumes and restored data is not picked up. CTool automatically maps compose keys to the correct volume name on the target server.

Output:
```
ℹ Extracting archive: my-project-backup-20250709-120000.tar.gz
ℹ Found volumes in: my-project/volumes/
ℹ Resolving volume mapping from compose file: my-project/docker-compose.yml
ℹ Restoring volume: new-server_myapp_postgres_data ← postgres_data.tar.gz
✓ Volume restored: new-server_myapp_postgres_data
ℹ Restoring volume: new-server_myapp_redis_data ← redis_data.tar.gz
✓ Volume restored: new-server_myapp_redis_data
✓ Restore finished. 2 volume(s) restored to /target/location/my-project
ℹ Run 'docker compose up -d' manually inside .../my-project when ready.
```

#### Vol Backup

Backup all (or specific) named volumes from docker compose.

```bash
cd /project/with/compose/file
ctool vol-backup
```

Output:
```
ℹ Using compose file: /project/with/compose/file/docker-compose.yml
ℹ Backup directory: /project/with/compose/file/backups
ℹ Stopping services...
✓ Services stopped.
ℹ Backing up volume: postgres_data → 20250626-143022-postgres_data.tar.gz
✓ Backup complete: 20250626-143022-postgres_data.tar.gz (1.2M)
ℹ Backing up volume: redis_data → 20250626-143022-redis_data.tar.gz
✓ Backup complete: 20250626-143022-redis_data.tar.gz (4.5K)
✓ Services started.
✓ Backup summary: 2 file(s), total 1.3M
  Location: /home/user/project/backups
```

##### Vol Backup specific volume

```bash
ctool vol-backup -v postgres_data
```

##### Vol Backup with custom backup directory

Output backup directory can be specified with `-d` or `--backup-dir` flag. Default: `./backups`.

```bash
ctool vol-backup \
  -f docker-compose.prod.yml \
  -d /mnt/backups/docker
```

##### Vol Backup with retention policy

Keep only the last 7 backups per volume:

```bash
ctool vol-backup --retain 7
```

##### Vol Backup + bundle project (ready to ship between servers)

Backup volumes, then zip the entire project directory (compose file + `.env` + config + backups) into a single archive:

```bash
ctool vol-backup --bundle
```

Output:
```
ℹ Creating project bundle: my-web-app-20260626-221138.tar.gz
✓ Bundle created: /path/to/my-web-app-20260626-221138.tar.gz
```

Bundle output can be directed to a specific directory with `-o` or `--output` flag. Default: parent directory of the project.

```bash
ctool vol-backup --bundle -o /mnt/backups/bundles
```

Filename format: `{project-dir-name}-{YYYYMMDD}-{HHMMSS}.tar.gz`

#### Vol Restore

Restore volumes from backup archive, with confirmation prompt.

##### Vol Restore all volumes

```bash
ctool vol-restore
```

Shows confirmation:

```
⚠ You are about to RESTORE 2 volume(s):
  - 20250626-143022-postgres_data.tar.gz
  - 20250626-143022-redis_data.tar.gz

This will OVERWRITE existing volume data. Continue? [y/N]
```

##### Vol Restore without confirmation

```bash
ctool vol-restore --force
```

##### Vol Restore specific volume

```bash
ctool vol-restore -v postgres_data
```

##### Vol Restore from specific backup directory

```bash
ctool vol-restore -d /mnt/backups/docker
```

##### Vol Restore from bundle

Extract bundle + restore volumes in a single command:

```bash
ctool vol-restore --bundle my-web-app-20260626-221138.tar.gz
```

Perfect for cross-server migration — just send 1 `.tar.gz` file, then restore.

#### List

##### List named volumes

```bash
ctool list
```

Output:
```
ℹ Using compose file: /project/docker-compose.yml

Named Volumes in: docker-compose.yml
────────────────────────────────────────
  postgres_data
  redis_data
```

##### List specific volume

```bash
ctool list -v postgres_data
```

#### Inspect

View Docker volume status and size defined in compose file.

##### Inspect all volumes

```bash
ctool inspect
```

Output:
```
ℹ Using compose file: /project/docker-compose.yml

Volume                    Docker Name                               Status    Size
──────────────────────────────────────────────────────────────────────────────────
postgres_data             myapp_postgres_data                       exists    1.2G
redis_data                myapp_redis_data                          exists    4.5K
uploads                   myapp_uploads                             missing   -
```

##### Inspect specific volume

```bash
ctool inspect -v postgres_data
```

Useful to check whether volumes are ready before `docker compose up -d`, especially after cross-server migration.

#### Cleanup

Remove all Docker resources (containers, volumes, images) from a compose project.

```bash
ctool cleanup /path/to/project
```

Shows warning and confirmation before execution:

```
ℹ Using compose file: /path/to/project/docker-compose.yml

⚠ You are about to COMPLETELY REMOVE all Docker resources for:
   /path/to/project

This will:
  - Stop and remove all containers
  - Remove all volumes (DATA will be LOST!)
  - Remove all images used by services
  - Remove orphan containers

Are you sure? [y/N]
```

### Workflow

#### Backup
```
ctool backup /path/to/project
  │
  ├── Validate target directory
  ├── Search for docker-compose.yml recursively (including subdirectories)
  ├── Check container status
  │     ├── Running → confirm auto-stop (Y → down, N → abort)
  │     └── Stopped → proceed
  ├── Copy all project files to staging area
  ├── Backup named volumes to staging/volumes/
  ├── Archive staging + compress → {name}-backup-{timestamp}.tar.gz
  └── Save to current directory
```

#### Restore
```
ctool restore archive.tar.gz
  │
  ├── Extract archive to current directory
  ├── Find volumes/ folder in extracted output
  ├── Detect compose file in extracted project
  ├── Map compose key → Docker volume name for target server
  ├── For each .tar.gz in volumes/:
  │     └── docker run alpine tar xzf → restore to correct volume name
  └── Done — offer "${_COMPOSE_CMD} up -d" immediately, or manual
```

#### Vol Backup
```
ctool vol-backup
  │
  ├── Read compose file → extract named volumes
  ├── Check for running containers
  │     ├── Yes → docker compose down
  │     └── No → proceed
  ├── For each volume:
  │     └── docker run alpine tar czf /backup/TIMESTAMP-volume.tar.gz
  ├── If previously running → docker compose up -d
  ├── If --retain → remove oldest backups
  └── If --bundle → zip project dir → {dir}-{TIMESTAMP}.tar.gz
```

#### Vol Backup --bundle
```
ctool vol-backup --bundle
  │
  ├── [normal backup process]
  ├── Determine project root (parent of compose file)
  ├── cd to parent directory
  └── tar czf {project-name}-{TIMESTAMP}.tar.gz {project-name}/
      └── Result: 1 file ready to ship between servers
```

#### Vol Restore
```
ctool vol-restore
  │
  ├── Find latest backup archive for each volume
  ├── Show confirmation (unless --force)
  ├── docker compose down
  ├── For each volume:
  │     └── docker run alpine tar xzf /backup/archive.tar.gz
  ├── docker compose up -d
  └── Done
```

#### Vol Restore --bundle
```
ctool vol-restore --bundle archive.tar.gz
  │
  ├── Extract archive → project dir
  ├── Detect compose file inside project dir
  ├── Set backup_dir to ./backups/ inside project dir
  └── [normal restore process]
```

#### Cleanup
```
ctool cleanup /path/to/project
  │
  ├── Validate target directory
  ├── Search for docker-compose.yml recursively
  ├── Show data loss warning
  ├── User confirmation
  │     ├── Y → docker compose down -v --rmi all --remove-orphans
  │     └── N → abort
  └── Done
```

### Complete Example

**Scenario: backup Postgres database from production**

```bash
# 1. Check what volumes exist
ctool list -f docker-compose.prod.yml

# 2. Vol backup only postgres volume
ctool vol-backup \
  -f docker-compose.prod.yml \
  -v postgres_data \
  -d /backups/postgres \
  --retain 14

# 3. Vol restore to staging environment
ctool vol-restore \
  -f docker-compose.staging.yml \
  -v postgres_data \
  -d /backups/postgres \
  --force
```

### Cross-Server Migration via Tailscale

Complete workflow to migrate a project with volumes to another server using Tailscale + `--bundle`.

#### Prerequisites

- Both servers connected in the same Tailscale network
- `ctool` installed on both servers (see [Installation](#installation))

#### Step-by-step

**Server A (source):**

```bash
# 1. Enter project directory
cd /var/www/my-web-app

# 2. Vol backup volumes + bundle entire project
ctool vol-backup --bundle -o /tmp
# Output: /tmp/my-web-app-20260626-221138.tar.gz
```

**Transfer via Tailscale:**

```bash
# From Server A → send to Server B (via rsync over Tailscale)
rsync -avz --progress /tmp/my-web-app-*.tar.gz user@server-b:/tmp/

# Or from Server B → pull from Server A
ssh user@server-a-tailscale-ip
rsync -avz --progress user@server-a:/tmp/my-web-app-*.tar.gz /tmp/
```

**Server B (destination):**

```bash
# 3. Vol restore directly from bundle
cd /tmp
ctool vol-restore --bundle my-web-app-20260626-221138.tar.gz --force
```

Done. Volumes + compose file + config are restored on Server B.

#### Automation with cron + rsync

**Server A — automated daily backup:**

```bash
# /etc/cron.daily/backup-webapp
#!/bin/bash
cd /var/www/my-web-app
/usr/local/bin/ctool vol-backup --bundle -o /tmp/backups --retain 7
```

**Server B — pull backup via Tailscale:**

```bash
# /etc/cron.daily/pull-backup
#!/bin/bash
rsync -avz --remove-source-files user@server-a-tailscale-ip:/tmp/backups/ /tmp/backups/
```

> Tailscale connections are end-to-end encrypted, no need for additional VPN or firewall configuration.

### Backup Directory Structure

```
backups/
├── 20250626-143022-postgres_data.tar.gz
├── 20250626-143022-redis_data.tar.gz
├── 20250627-091234-postgres_data.tar.gz
└── 20250627-091234-redis_data.tar.gz
```

Filename format: `{YYYYMMDD}-{HHMMSS}-{volume-name}.tar.gz`

### Environment & Exit Codes

- **Exit 0** — success
- **Exit 1** — error (compose file not found, volume does not exist, backup failed, etc.)

### Troubleshooting

#### `jq: command not found`

No need to worry — `ctool` will detect the package manager (`apt`/`pacman`/`dnf`/`brew`/`apk`) and offer automatic installation when first needed.

Or install manually:
```bash
sudo apt install jq   # Debian/Ubuntu
sudo pacman -S jq     # Arch
sudo dnf install jq   # Fedora
brew install jq       # macOS
```

#### `docker compose` command not found

Use `docker-compose` (V1) or install the V2 plugin:
```bash
sudo apt install docker-compose-v2
```

#### Volumes not restored after `docker compose up`

After restore and `docker compose up`, the container feels like a fresh install — this is usually caused by **Docker volume name mismatch**.

**Cause:** Docker Compose prefixes volume names based on the project directory name. If the directory differs between backup and restore, volume data is restored under the old name while compose looks for the new name.

**Solution (automatic):** Use `ctool restore` or `ctool vol-restore --bundle` — this tool automatically reads the compose file from the target project and maps compose keys to the correct Docker volume name for that server.

**Manual check:**
```bash
ctool inspect
```
Make sure all volumes show `exists` status before running `docker compose up -d`.

#### Backup file owner root

Backup files are created by the root user inside the Docker container. This is normal as the backup process runs as root inside the container. Files remain readable/deletable by the host user according to backup directory permissions.

#### Permission denied when accessing Docker socket

Make sure your user is in the `docker` group:
```bash
sudo usermod -aG docker $USER
# Logout then login again
```

### License

MIT

---

## Versi Bahasa Indonesia

- [Fitur](#fitur)
- [Dependencies](#dependencies-1)
- [Instalasi](#instalasi)
- [Usage](#usage-1)
  - [Backup](#backup-2)
  - [Restore](#restore-2)
  - [Vol Backup](#vol-backup-2)
  - [Vol Restore](#vol-restore-2)
  - [List](#list-1)
  - [Inspect](#inspect-1)
- [Alur Kerja](#alur-kerja)
  - [Backup](#backup-3)
  - [Restore](#restore-3)
  - [Vol Backup](#vol-backup-3)
  - [Vol Backup --bundle](#vol-backup---bundle-2)
  - [Vol Restore](#vol-restore-3)
  - [Vol Restore --bundle](#vol-restore---bundle-2)
- [Contoh Lengkap](#contoh-lengkap)
- [Migrasi Antar Server via Tailscale](#migrasi-antar-server-via-tailscale)
- [Struktur Direktori Backup](#struktur-direktori-backup)
- [Environment & Exit Codes](#environment--exit-codes-1)
- [Troubleshooting](#troubleshooting-1)
- [License](#license-1)

### Fitur

- **Backup** — backup satu direktori project + seluruh volume Docker Compose jadi satu archive
- **Restore** — restore archive project ke current directory + restore semua volume
- **Vol Backup** — backup semua (atau spesifik) named volume dari compose file
- **Vol Restore** — restore volume dari archive backup, lengkap dengan konfirmasi
- **List** — lihat daftar named volume yang terdefinisi di compose file
- **Inspect** — lihat status dan ukuran volume Docker dari compose file
- **Cleanup** — hapus seluruh resource Docker (container, volume, image) dari suatu project
- **Auto stop/start** — otomatis stop service sebelum vol-backup/vol-restore, start ulang setelah selesai
- **Specific volume** — backup/restore volume tertentu dengan flag `-v`
- **Retention policy** — jaga jumlah backup per volume dengan `--retain N`
- **Timestamped archive** — format nama file `YYYYMMDD-HHMMSS-volume-name.tar.gz`
- **Minimal dependencies** — hanya butuh `bash`, `docker`, `docker compose` (`jq` di-install otomatis jika belum ada)
- **Multi compose support** — bisa pakai compose file dari direktori mana pun via `-f`
- **Smart volume mapping** — restore otomatis mapping compose key → Docker volume name, aman migrasi antar server beda nama direktori

### Dependencies

| Tool | Keterangan |
|------|-----------|
| `bash` ≥ 4.0 | Shell (built-in di semua Linux modern) |
| `docker` | Docker Engine |
| `docker compose` | Docker Compose V2 (plugin `docker compose`) |
| `jq` | JSON processor untuk parsing compose config (di-install otomatis jika belum ada) |
| `alpine` | Image minimal untuk proses backup/restore (di-pull otomatis oleh Docker) |

#### Install dependencies

**Ubuntu / Debian:**
```bash
sudo apt update && sudo apt install -y docker.io docker-compose-v2 jq
```

**Arch Linux:**
```bash
sudo pacman -S docker docker-compose jq
```

**Fedora:**
```bash
sudo dnf install docker docker-compose jq
```

**macOS (Homebrew):**
```bash
brew install docker docker-compose jq
```

> Pastikan Docker daemon sudah berjalan dan user Anda memiliki akses ke socket Docker (anggota grup `docker`).
> `jq` tidak wajib di-install manual — CTool akan menginstallnya otomatis saat dibutuhkan.

### Instalasi

#### Via clone (langsung pakai)

```bash
git clone https://github.com/yourusername/ctool.git
cd ctool
chmod +x ctool
./ctool --help
```

#### Install ke system PATH (opsional)

Agar bisa dipanggil dari mana saja:

```bash
sudo cp ctool /usr/local/bin/
# atau
ln -s "$(pwd)/ctool" ~/.local/bin/
```

Verifikasi:
```bash
ctool --version
```

### Usage

#### Backup

Backup satu direktori project + seluruh volume docker-compose menjadi satu archive.

```bash
ctool backup /path/to/project
```

Jika container masih berjalan, akan muncul konfirmasi untuk stop otomatis. Gunakan `--restart` untuk menyalakan ulang services setelah backup selesai:

```bash
ctool backup /path/to/project --restart
```

Alur:
1. Validasi direktori target
2. Cari `docker-compose.yml` secara recursive di dalamnya (termasuk sub-direktori)
3. Cek container — jika masih berjalan, konfirmasi auto-stop (atau batal jika tidak setuju)
4. Backup semua named volume ke folder `volumes/` (sementara)
5. Archive seluruh file project + folder `volumes/` → `{nama-dir}-backup-{timestamp}.tar.gz`
6. Hasil disimpan di current working directory

Output:
```
ℹ Using compose file: /path/to/project/docker-compose.yml
ℹ Copying project files...
ℹ Backing up volume: postgres_data
✓ Volume backup: postgres_data.tar.gz
ℹ Creating archive: my-project-backup-20250709-120000.tar.gz
✓ Backup created: .../my-project-backup-20250709-120000.tar.gz (1.2M)
```

Archive structure:
```
my-project-backup-20250709-120000.tar.gz
└── my-project/
    ├── docker-compose.yml
    ├── ... (semua file project)
    └── volumes/
        ├── postgres_data.tar.gz
        └── redis_data.tar.gz
```

#### Restore

Restore archive (project + volume) ke current working directory.

```bash
cd /target/location
ctool restore ../my-project-backup-20250709-120000.tar.gz
```

Alur:
1. Ekstrak archive ke current directory
2. Cari folder `volumes/` di dalam hasil extract
3. Deteksi compose file di project hasil extract
4. Mapping tiap compose key ke Docker volume name yang benar untuk server target (via `docker compose config`)
5. Restore setiap volume docker dari file `.tar.gz` di `volumes/` ke Docker volume name yang sesuai
6. Selesai — tawarkan `docker compose up -d` otomatis, atau manual

> **Kenapa ini penting?** Docker Compose memberi prefix nama volume berdasarkan nama direktori project. Jika direktori berbeda antara server sumber dan tujuan (misal `myapp` → `myapp-clone`), Docker akan membuat volume baru dan data restore tidak terbaca. CTool otomatis memetakan compose key ke nama volume yang benar di server target.

Output:
```
ℹ Extracting archive: my-project-backup-20250709-120000.tar.gz
ℹ Found volumes in: my-project/volumes/
ℹ Resolving volume mapping from compose file: my-project/docker-compose.yml
ℹ Restoring volume: new-server_myapp_postgres_data ← postgres_data.tar.gz
✓ Volume restored: new-server_myapp_postgres_data
ℹ Restoring volume: new-server_myapp_redis_data ← redis_data.tar.gz
✓ Volume restored: new-server_myapp_redis_data
✓ Restore finished. 2 volume(s) restored to /target/location/my-project
ℹ Run 'docker compose up -d' manually inside .../my-project when ready.
```

#### Vol Backup

Backup semua (atau spesifik) named volume dari docker compose.

```bash
cd /project/dengan/compose/file
ctool vol-backup
```

Output:
```
ℹ Using compose file: /project/dengan/compose/file/docker-compose.yml
ℹ Backup directory: /project/dengan/compose/file/backups
ℹ Stopping services...
✓ Services stopped.
ℹ Backing up volume: postgres_data → 20250626-143022-postgres_data.tar.gz
✓ Backup complete: 20250626-143022-postgres_data.tar.gz (1.2M)
ℹ Backing up volume: redis_data → 20250626-143022-redis_data.tar.gz
✓ Backup complete: 20250626-143022-redis_data.tar.gz (4.5K)
✓ Services started.
✓ Backup summary: 2 file(s), total 1.3M
  Location: /home/user/project/backups
```

##### Vol Backup volume spesifik

```bash
ctool vol-backup -v postgres_data
```

##### Vol Backup dengan custom backup directory

Direktori output backup dapat ditentukan dengan flag `-d` atau `--backup-dir`. Default: `./backups`.

```bash
ctool vol-backup \
  -f docker-compose.prod.yml \
  -d /mnt/backups/docker
```

##### Vol Backup dengan retention policy

Hanya menyisakan 7 backup terakhir per volume:

```bash
ctool vol-backup --retain 7
```

##### Vol Backup + bundle project (siap kirim antar server)

Backup volume, lalu zip seluruh direktori project (compose file + `.env` + config + backups) jadi satu archive:

```bash
ctool vol-backup --bundle
```

Output:
```
ℹ Creating project bundle: my-web-app-20260626-221138.tar.gz
✓ Bundle created: /path/to/my-web-app-20260626-221138.tar.gz
```

Bundle output dapat diarahkan ke direktori tertentu dengan flag `-o` atau `--output`. Default: parent directory dari project.

```bash
ctool vol-backup --bundle -o /mnt/backups/bundles
```

Nama file otomatis: `{nama-dir-project}-{YYYYMMDD}-{HHMMSS}.tar.gz`

#### Vol Restore

Restore volume dari archive backup, lengkap dengan konfirmasi.

##### Vol Restore semua volume

```bash
ctool vol-restore
```

Akan muncul konfirmasi:

```
⚠ You are about to RESTORE 2 volume(s):
  - 20250626-143022-postgres_data.tar.gz
  - 20250626-143022-redis_data.tar.gz

This will OVERWRITE existing volume data. Continue? [y/N]
```

##### Vol Restore tanpa konfirmasi

```bash
ctool vol-restore --force
```

##### Vol Restore volume spesifik

```bash
ctool vol-restore -v postgres_data
```

##### Vol Restore dari direktori backup tertentu

```bash
ctool vol-restore -d /mnt/backups/docker
```

##### Vol Restore dari bundle

Ekstrak bundle + restore volume langsung dalam satu perintah:

```bash
ctool vol-restore --bundle my-web-app-20260626-221138.tar.gz
```

Cocok untuk migrasi antar server — cukup kirim 1 file `.tar.gz`, lalu restore.

#### List

##### List named volumes

```bash
ctool list
```

Output:
```
ℹ Using compose file: /project/docker-compose.yml

Named Volumes in: docker-compose.yml
────────────────────────────────────────
  postgres_data
  redis_data
```

##### List volume spesifik

```bash
ctool list -v postgres_data
```

#### Inspect

Lihat status dan ukuran volume Docker yang terdefinisi di compose file.

##### Inspect semua volume

```bash
ctool inspect
```

Output:
```
ℹ Using compose file: /project/docker-compose.yml

Volume                    Docker Name                               Status    Size
──────────────────────────────────────────────────────────────────────────────────
postgres_data             myapp_postgres_data                       exists    1.2G
redis_data                myapp_redis_data                          exists    4.5K
uploads                   myapp_uploads                             missing   -
```

##### Inspect volume spesifik

```bash
ctool inspect -v postgres_data
```

Berguna untuk mengecek apakah volume sudah siap sebelum `docker compose up -d`, terutama setelah migrasi antar server.

#### Cleanup

Hapus seluruh resource Docker (container, volume, image) dari suatu project compose.

```bash
ctool cleanup /path/to/project
```

Akan muncul peringatan dan konfirmasi sebelum eksekusi:

```
ℹ Using compose file: /path/to/project/docker-compose.yml

⚠ You are about to COMPLETELY REMOVE all Docker resources for:
   /path/to/project

This will:
  - Stop and remove all containers
  - Remove all volumes (DATA will be LOST!)
  - Remove all images used by services
  - Remove orphan containers

Are you sure? [y/N]
```

### Alur Kerja

#### Backup
```
ctool backup /path/to/project
  │
  ├── Validasi direktori target
  ├── Cari docker-compose.yml secara recursive (termasuk sub-direktori)
  ├── Cek container status
  │     ├── Running → konfirmasi auto-stop (Y → down, N → batal)
  │     └── Stopped → lanjut
  ├── Copy semua file project ke staging area
  ├── Backup named volumes ke staging/volumes/
  ├── Archive staging + compress → {nama}-backup-{timestamp}.tar.gz
  └── Simpan di current directory
```

#### Restore
```
ctool restore archive.tar.gz
  │
  ├── Ekstrak archive ke current directory
  ├── Cari folder volumes/ di hasil extract
  ├── Deteksi compose file di project hasil extract
  ├── Mapping compose key → Docker volume name untuk server target
  ├── Untuk setiap .tar.gz di volumes/:
  │     └── docker run alpine tar xzf → restore ke volume name yg benar
  └── Selesai — tawarkan "${_COMPOSE_CMD} up -d" langsung, atau manual
```

#### Vol Backup
```
ctool vol-backup
  │
  ├── Baca compose file → extract named volumes
  ├── Cek apakah ada container running
  │     ├── Ya → docker compose down
  │     └── Tidak → lanjut
  ├── Untuk setiap volume:
  │     └── docker run alpine tar czf /backup/TIMESTAMP-volume.tar.gz
  ├── Jika sebelumnya running → docker compose up -d
  ├── Jika --retain → hapus backup terlama
  └── Jika --bundle → zip project dir → {dir}-{TIMESTAMP}.tar.gz
```

#### Vol Backup --bundle
```
ctool vol-backup --bundle
  │
  ├── [proses backup normal]
  ├── Tentukan project root (parent dari compose file)
  ├── cd ke parent directory
  └── tar czf {nama-project}-{TIMESTAMP}.tar.gz {nama-project}/
      └── Hasil: 1 file siap kirim antar server
```

#### Vol Restore
```
ctool vol-restore
  │
  ├── Cari archive backup terbaru untuk setiap volume
  ├── Tampilkan konfirmasi (kecuali --force)
  ├── docker compose down
  ├── Untuk setiap volume:
  │     └── docker run alpine tar xzf /backup/archive.tar.gz
  ├── docker compose up -d
  └── Selesai
```

#### Vol Restore --bundle
```
ctool vol-restore --bundle archive.tar.gz
  │
  ├── Ekstrak archive → project dir
  ├── Deteksi compose file di dalam project dir
  ├── Set backup_dir ke ./backups/ di dalam project dir
  └── [proses restore normal]
```

#### Cleanup
```
ctool cleanup /path/to/project
  │
  ├── Validasi direktori target
  ├── Cari docker-compose.yml recursive
  ├── Tampilkan peringatan data akan hilang
  ├── Konfirmasi user
  │     ├── Y → docker compose down -v --rmi all --remove-orphans
  │     └── N → batal
  └── Selesai
```

### Contoh Lengkap

**Skenario: backup database Postgres dari production**

```bash
# 1. Cek volume apa saja yang ada
ctool list -f docker-compose.prod.yml

# 2. Vol backup hanya volume postgres
ctool vol-backup \
  -f docker-compose.prod.yml \
  -v postgres_data \
  -d /backups/postgres \
  --retain 14

# 3. Vol restore ke environment staging
ctool vol-restore \
  -f docker-compose.staging.yml \
  -v postgres_data \
  -d /backups/postgres \
  --force
```

### Migrasi Antar Server via Tailscale

Workflow lengkap pindah project beserta volume ke server lain menggunakan Tailscale + `--bundle`.

#### Prasyarat

- Kedua server terhubung dalam satu jaringan Tailscale
- `ctool` sudah terinstall di kedua server (lihat [Instalasi](#instalasi))

#### Step-by-step

**Server A (sumber):**

```bash
# 1. Masuk ke direktori project
cd /var/www/my-web-app

# 2. Vol backup volume + bundle seluruh project
ctool vol-backup --bundle -o /tmp
# Output: /tmp/my-web-app-20260626-221138.tar.gz
```

**Transfer via Tailscale:**

```bash
# Dari Server A → kirim ke Server B (via rsync over Tailscale)
rsync -avz --progress /tmp/my-web-app-*.tar.gz user@server-b:/tmp/

# Atau dari Server B → tarik dari Server A
ssh user@server-a-tailscale-ip
rsync -avz --progress user@server-a:/tmp/my-web-app-*.tar.gz /tmp/
```

**Server B (tujuan):**

```bash
# 3. Vol restore langsung dari bundle
cd /tmp
ctool vol-restore --bundle my-web-app-20260626-221138.tar.gz --force
```

Selesai. Volume + compose file + config sudah ter-restore di Server B.

#### Automation dengan cron + rsync

**Server A — backup otomatis tiap hari:**

```bash
# /etc/cron.daily/backup-webapp
#!/bin/bash
cd /var/www/my-web-app
/usr/local/bin/ctool vol-backup --bundle -o /tmp/backups --retain 7
```

**Server B — pull backup via Tailscale:**

```bash
# /etc/cron.daily/pull-backup
#!/bin/bash
rsync -avz --remove-source-files user@server-a-tailscale-ip:/tmp/backups/ /tmp/backups/
```

> Koneksi Tailscale sudah dienkripsi end-to-end, tidak perlu konfigurasi VPN atau firewall tambahan.

### Struktur Direktori Backup

```
backups/
├── 20250626-143022-postgres_data.tar.gz
├── 20250626-143022-redis_data.tar.gz
├── 20250627-091234-postgres_data.tar.gz
└── 20250627-091234-redis_data.tar.gz
```

Format nama: `{YYYYMMDD}-{HHMMSS}-{volume-name}.tar.gz`

### Environment & Exit Codes

- **Exit 0** — sukses
- **Exit 1** — error (compose file tidak ditemukan, volume tidak ada, backup gagal, dll)

### Troubleshooting

#### `jq: command not found`

Tidak perlu khawatir — `ctool` akan mendeteksi package manager (`apt`/`pacman`/`dnf`/`brew`/`apk`) dan menawarkan instalasi otomatis saat pertama kali dibutuhkan.

Atau install manual:
```bash
sudo apt install jq   # Debian/Ubuntu
sudo pacman -S jq     # Arch
sudo dnf install jq   # Fedora
brew install jq       # macOS
```

#### `docker compose` command not found

Gunakan `docker-compose` (V1) atau install plugin V2:
```bash
sudo apt install docker-compose-v2
```

#### Volume tidak ter-restore setelah `docker compose up`

Setelah restore dan `docker compose up`, container terasa seperti instalasi baru — biasanya karena **mismatch nama volume Docker**.

**Penyebab:** Docker Compose memberi prefix nama volume berdasarkan nama direktori project. Jika direktori berbeda antara backup dan restore, volume data ter-restore di nama lama sementara compose mencari nama baru.

**Solusi (otomatis):** Gunakan `ctool restore` atau `ctool vol-restore --bundle` — tool ini otomatis membaca compose file di project target dan memetakan compose key ke nama volume Docker yang benar untuk server tersebut.

**Cek manual:**
```bash
ctool inspect
```
Pastikan semua volume berstatus `exists` sebelum menjalankan `docker compose up -d`.

#### Backup file owner root

File backup dibuat oleh user root di dalam container Docker. Ini normal karena proses backup berjalan sebagai root di container. File tetap bisa dibaca/dihapus oleh host user sesuai permission direktori backup.

#### Permission denied saat akses socket Docker

Pastikan user Anda ada di grup `docker`:
```bash
sudo usermod -aG docker $USER
# Logout lalu login kembali
```

### License

MIT
