# ct-trans

Container Transfer Tool — backup, restore, dan migrasi seluruh project Docker Compose (image + volume) antar server dengan mudah.

```
ct-trans backup /path/to/project
ct-trans restore backup.tar.gz
ct-trans vol-backup
ct-trans vol-restore
ct-trans list
ct-trans inspect
```

## Daftar Isi

- [Fitur](#fitur)
- [Dependencies](#dependencies)
- [Instalasi](#instalasi)
- [Usage](#usage)
  - [Backup](#backup)
  - [Restore](#restore)
  - [Vol Backup](#vol-backup)
  - [Vol Restore](#vol-restore)
  - [List](#list)
  - [Inspect](#inspect)
- [Alur Kerja](#alur-kerja)
  - [Backup](#backup-1)
  - [Restore](#restore-1)
  - [Vol Backup](#vol-backup-1)
  - [Vol Backup --bundle](#vol-backup---bundle)
  - [Vol Restore](#vol-restore-1)
  - [Vol Restore --bundle](#vol-restore---bundle)
- [Contoh Lengkap](#contoh-lengkap)
- [Migrasi Antar Server via Tailscale](#migrasi-antar-server-via-tailscale)
- [Struktur Direktori Backup](#struktur-direktori-backup)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Fitur

- **Backup** — backup satu direktori project + seluruh volume Docker Compose jadi satu archive
- **Restore** — restore archive project ke current directory + restore semua volume
- **Vol Backup** — backup semua (atau spesifik) named volume dari compose file
- **Vol Restore** — restore volume dari archive backup, lengkap dengan konfirmasi
- **List** — lihat daftar named volume yang terdefinisi di compose file
- **Inspect** — lihat status dan ukuran volume Docker dari compose file
- **Auto stop/start** — otomatis stop service sebelum vol-backup/vol-restore, start ulang setelah selesai
- **Specific volume** — backup/restore volume tertentu dengan flag `-v`
- **Retention policy** — jaga jumlah backup per volume dengan `--retain N`
- **Timestamped archive** — format nama file `YYYYMMDD-HHMMSS-volume-name.tar.gz`
- **Minimal dependencies** — hanya butuh `bash`, `docker`, `docker compose` (`jq` di-install otomatis jika belum ada)
- **Multi compose support** — bisa pakai compose file dari direktori mana pun via `-f`
- **Smart volume mapping** — restore otomatis mapping compose key → Docker volume name, aman migrasi antar server beda nama direktori

## Dependencies

| Tool | Keterangan |
|------|-----------|
| `bash` ≥ 4.0 | Shell (built-in di semua Linux modern) |
| `docker` | Docker Engine |
| `docker compose` | Docker Compose V2 (plugin `docker compose`) |
| `jq` | JSON processor untuk parsing compose config (di-install otomatis jika belum ada) |
| `alpine` | Image minimal untuk proses backup/restore (di-pull otomatis oleh Docker) |

### Install dependencies

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
> `jq` tidak wajib di-install manual — ct-trans akan menginstallnya otomatis saat dibutuhkan.

## Instalasi

### Via clone (langsung pakai)

```bash
git clone https://github.com/yourusername/ct-trans.git
cd ct-trans
chmod +x ct-trans
./ct-trans --help
```

### Install ke system PATH (opsional)

Agar bisa dipanggil dari mana saja:

```bash
sudo cp ct-trans /usr/local/bin/
# atau
ln -s "$(pwd)/ct-trans" ~/.local/bin/
```

Verifikasi:
```bash
ct-trans --version
```

## Usage

### Backup

Backup satu direktori project + seluruh volume docker-compose menjadi satu archive.

```bash
ct-trans backup /path/to/project
```

Alur:
1. Validasi direktori target
2. Cari `docker-compose.yml` secara recursive di dalamnya (termasuk sub-direktori)
3. Cek container — jika masih berjalan, backup ditolak (harus `docker compose down` manual)
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

### Restore

Restore archive (project + volume) ke current working directory.

```bash
cd /target/location
ct-trans restore ../my-project-backup-20250709-120000.tar.gz
```

Alur:
1. Ekstrak archive ke current directory
2. Cari folder `volumes/` di dalam hasil extract
3. Deteksi compose file di project hasil extract
4. Mapping tiap compose key ke Docker volume name yang benar untuk server target (via `docker compose config`)
5. Restore setiap volume docker dari file `.tar.gz` di `volumes/` ke Docker volume name yang sesuai
6. Selesai — user jalankan `docker compose up -d` manual

> **Kenapa ini penting?** Docker Compose memberi prefix nama volume berdasarkan nama direktori project. Jika direktori berbeda antara server sumber dan tujuan (misal `myapp` → `myapp-clone`), Docker akan membuat volume baru dan data restore tidak terbaca. ct-trans otomatis memetakan compose key ke nama volume yang benar di server target.

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

### Vol Backup

Backup semua (atau spesifik) named volume dari docker compose.

```bash
cd /project/dengan/compose/file
ct-trans vol-backup
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

#### Vol Backup volume spesifik

```bash
ct-trans vol-backup -v postgres_data
```

#### Vol Backup dengan custom backup directory

Direktori output backup dapat ditentukan dengan flag `-d` atau `--backup-dir`. Default: `./backups`.

```bash
ct-trans vol-backup \
  -f docker-compose.prod.yml \
  -d /mnt/backups/docker
```

#### Vol Backup dengan retention policy

Hanya menyisakan 7 backup terakhir per volume:

```bash
ct-trans vol-backup --retain 7
```

#### Vol Backup + bundle project (siap kirim antar server)

Backup volume, lalu zip seluruh direktori project (compose file + `.env` + config + backups) jadi satu archive:

```bash
ct-trans vol-backup --bundle
```

Output:
```
ℹ Creating project bundle: my-web-app-20260626-221138.tar.gz
✓ Bundle created: /path/to/my-web-app-20260626-221138.tar.gz
```

Bundle output dapat diarahkan ke direktori tertentu dengan flag `-o` atau `--output`. Default: parent directory dari project.

```bash
ct-trans vol-backup --bundle -o /mnt/backups/bundles
```

Nama file otomatis: `{nama-dir-project}-{YYYYMMDD}-{HHMMSS}.tar.gz`

### Vol Restore

Restore volume dari archive backup, lengkap dengan konfirmasi.

#### Vol Restore semua volume

```bash
ct-trans vol-restore
```

Akan muncul konfirmasi:

```
⚠ You are about to RESTORE 2 volume(s):
  - 20250626-143022-postgres_data.tar.gz
  - 20250626-143022-redis_data.tar.gz

This will OVERWRITE existing volume data. Continue? [y/N]
```

#### Vol Restore tanpa konfirmasi

```bash
ct-trans vol-restore --force
```

#### Vol Restore volume spesifik

```bash
ct-trans vol-restore -v postgres_data
```

#### Vol Restore dari direktori backup tertentu

```bash
ct-trans vol-restore -d /mnt/backups/docker
```

#### Vol Restore dari bundle

Ekstrak bundle + restore volume langsung dalam satu perintah:

```bash
ct-trans vol-restore --bundle my-web-app-20260626-221138.tar.gz
```

Cocok untuk migrasi antar server — cukup kirim 1 file `.tar.gz`, lalu restore.

### List

#### List named volumes

```bash
ct-trans list
```

Output:
```
ℹ Using compose file: /project/docker-compose.yml

Named Volumes in: docker-compose.yml
────────────────────────────────────────
  postgres_data
  redis_data
```

#### List volume spesifik

```bash
ct-trans list -v postgres_data
```

### Inspect

Lihat status dan ukuran volume Docker yang terdefinisi di compose file.

#### Inspect semua volume

```bash
ct-trans inspect
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

#### Inspect volume spesifik

```bash
ct-trans inspect -v postgres_data
```

Berguna untuk mengecek apakah volume sudah siap sebelum `docker compose up -d`, terutama setelah migrasi antar server.

## Alur Kerja

### Backup
```
ct-trans backup /path/to/project
  │
  ├── Validasi direktori target
  ├── Cari docker-compose.yml secara recursive (termasuk sub-direktori)
  ├── Cek container status
  │     ├── Running → warning, minta user stop manual
  │     └── Stopped → lanjut
  ├── Copy semua file project ke staging area
  ├── Backup named volumes ke staging/volumes/
  ├── Archive staging + compress → {nama}-backup-{timestamp}.tar.gz
  └── Simpan di current directory
```

### Restore
```
ct-trans restore archive.tar.gz
  │
  ├── Ekstrak archive ke current directory
  ├── Cari folder volumes/ di hasil extract
  ├── Deteksi compose file di project hasil extract
  ├── Mapping compose key → Docker volume name untuk server target
  ├── Untuk setiap .tar.gz di volumes/:
  │     └── docker run alpine tar xzf → restore ke volume name yg benar
  └── Selesai (user compose up -d manual)
```

### Vol Backup
```
ct-trans vol-backup
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

### Vol Backup --bundle
```
ct-trans vol-backup --bundle
  │
  ├── [proses backup normal]
  ├── Tentukan project root (parent dari compose file)
  ├── cd ke parent directory
  └── tar czf {nama-project}-{TIMESTAMP}.tar.gz {nama-project}/
      └── Hasil: 1 file siap kirim antar server
```

### Vol Restore
```
ct-trans vol-restore
  │
  ├── Cari archive backup terbaru untuk setiap volume
  ├── Tampilkan konfirmasi (kecuali --force)
  ├── docker compose down
  ├── Untuk setiap volume:
  │     └── docker run alpine tar xzf /backup/archive.tar.gz
  ├── docker compose up -d
  └── Selesai
```

### Vol Restore --bundle
```
ct-trans vol-restore --bundle archive.tar.gz
  │
  ├── Ekstrak archive → project dir
  ├── Deteksi compose file di dalam project dir
  ├── Set backup_dir ke ./backups/ di dalam project dir
  └── [proses restore normal]
```

## Contoh Lengkap

**Skenario: backup database Postgres dari production**

```bash
# 1. Cek volume apa saja yang ada
ct-trans list -f docker-compose.prod.yml

# 2. Vol backup hanya volume postgres
ct-trans vol-backup \
  -f docker-compose.prod.yml \
  -v postgres_data \
  -d /backups/postgres \
  --retain 14

# 3. Vol restore ke environment staging
ct-trans vol-restore \
  -f docker-compose.staging.yml \
  -v postgres_data \
  -d /backups/postgres \
  --force
```

## Migrasi Antar Server via Tailscale

Workflow lengkap pindah project beserta volume ke server lain menggunakan Tailscale + `--bundle`.

### Prasyarat

- Kedua server terhubung dalam satu jaringan Tailscale
- `ct-trans` sudah terinstall di kedua server (lihat [Instalasi](#instalasi))

### Step-by-step

**Server A (sumber):**

```bash
# 1. Masuk ke direktori project
cd /var/www/my-web-app

# 2. Vol backup volume + bundle seluruh project
ct-trans vol-backup --bundle -o /tmp
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
ct-trans vol-restore --bundle my-web-app-20260626-221138.tar.gz --force
```

Selesai. Volume + compose file + config sudah ter-restore di Server B.

### Automation dengan cron + rsync

**Server A — backup otomatis tiap hari:**

```bash
# /etc/cron.daily/backup-webapp
#!/bin/bash
cd /var/www/my-web-app
/usr/local/bin/ct-trans vol-backup --bundle -o /tmp/backups --retain 7
```

**Server B — pull backup via Tailscale:**

```bash
# /etc/cron.daily/pull-backup
#!/bin/bash
rsync -avz --remove-source-files user@server-a-tailscale-ip:/tmp/backups/ /tmp/backups/
```

> Koneksi Tailscale sudah dienkripsi end-to-end, tidak perlu konfigurasi VPN atau firewall tambahan.

## Struktur Direktori Backup

```
backups/
├── 20250626-143022-postgres_data.tar.gz
├── 20250626-143022-redis_data.tar.gz
├── 20250627-091234-postgres_data.tar.gz
└── 20250627-091234-redis_data.tar.gz
```

Format nama: `{YYYYMMDD}-{HHMMSS}-{volume-name}.tar.gz`

## Environment & Exit Codes

- **Exit 0** — sukses
- **Exit 1** — error (compose file tidak ditemukan, volume tidak ada, backup gagal, dll)

## Troubleshooting

### `jq: command not found`
Tidak perlu khawatir — `ct-trans` akan mendeteksi package manager (`apt`/`pacman`/`dnf`/`brew`/`apk`) dan menawarkan instalasi otomatis saat pertama kali dibutuhkan.

Atau install manual:
```bash
sudo apt install jq   # Debian/Ubuntu
sudo pacman -S jq     # Arch
sudo dnf install jq   # Fedora
brew install jq       # macOS
```

### `docker compose` command not found
Gunakan `docker-compose` (V1) atau install plugin V2:
```bash
sudo apt install docker-compose-v2
```

### Volume tidak ter-restore setelah `docker compose up`

Setelah restore dan `docker compose up`, container terasa seperti instalasi baru — biasanya karena **mismatch nama volume Docker**.

**Penyebab:** Docker Compose memberi prefix nama volume berdasarkan nama direktori project. Jika direktori berbeda antara backup dan restore, volume data ter-restore di nama lama sementara compose mencari nama baru.

**Solusi (otomatis):** Gunakan `ct-trans restore` atau `ct-trans vol-restore --bundle` — tool ini otomatis membaca compose file di project target dan memetakan compose key ke nama volume Docker yang benar untuk server tersebut.

**Cek manual:**
```bash
ct-trans inspect
```
Pastikan semua volume berstatus `exists` sebelum menjalankan `docker compose up -d`.

### Backup file owner root
File backup dibuat oleh user root di dalam container Docker. Ini normal karena proses backup berjalan sebagai root di container. File tetap bisa dibaca/dihapus oleh host user sesuai permission direktori backup.

### Permission denied saat akses socket Docker
Pastikan user Anda ada di grup `docker`:
```bash
sudo usermod -aG docker $USER
# Logout lalu login kembali
```

## License

MIT
