# ct-volume

Docker Volume Backup/Restore Tool — backup dan restore volume Docker berdasarkan docker-compose / compose file dengan mudah.

```
ct-volume backup
ct-volume restore
ct-volume list
ct-volume dir-backup /path/to/project
ct-volume dir-restore backup.tar.gz
```

## Daftar Isi

- [Fitur](#fitur)
- [Dependencies](#dependencies)
- [Instalasi](#instalasi)
- [Usage](#usage)
  - [Backup](#backup)
  - [Restore](#restore)
  - [List](#list)
- [Alur Kerja](#alur-kerja)
  - [Backup](#backup-1)
  - [Backup --bundle](#backup---bundle)
  - [Restore](#restore-1)
  - [Restore --bundle](#restore---bundle)
- [Contoh Lengkap](#contoh-lengkap)
- [Dir Backup](#dir-backup-2)
- [Dir Restore](#dir-restore-2)
- [Migrasi Antar Server via Tailscale](#migrasi-antar-server-via-tailscale)
- [Struktur Direktori Backup](#struktur-direktori-backup)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Fitur

- **Backup** — backup semua (atau spesifik) named volume dari compose file
- **Restore** — restore volume dari archive backup, lengkap dengan konfirmasi
- **List** — lihat daftar named volume yang terdefinisi di compose file
- **Auto stop/start** — otomatis stop service sebelum backup/restore, start ulang setelah selesai
- **Specific volume** — backup/restore volume tertentu dengan flag `-v`
- **Retention policy** — jaga jumlah backup per volume dengan `--retain N`
- **Timestamped archive** — format nama file `YYYYMMDD-HHMMSS-volume-name.tar.gz`
- **Minimal dependencies** — hanya butuh `bash`, `docker`, `docker compose`, `jq`
- **Multi compose support** — bisa pakai compose file dari direktori mana pun via `-f`
- **Dir Backup** — backup satu direktori project + volume compose jadi satu archive
- **Dir Restore** — restore archive ke current directory + restore semua volume

## Dependencies

| Tool | Keterangan |
|------|-----------|
| `bash` ≥ 4.0 | Shell (built-in di semua Linux modern) |
| `docker` | Docker Engine |
| `docker compose` | Docker Compose V2 (plugin `docker compose`) |
| `jq` | JSON processor untuk parsing compose config |
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

## Instalasi

### Via clone (langsung pakai)

```bash
git clone https://github.com/yourusername/ct-volume.git
cd ct-volume
chmod +x ct-volume
./ct-volume --help
```

### Install ke system PATH (opsional)

Agar bisa dipanggil dari mana saja:

```bash
sudo cp ct-volume /usr/local/bin/
# atau
ln -s "$(pwd)/ct-volume" ~/.local/bin/
```

Verifikasi:
```bash
ct-volume --version
```

## Usage

### Backup

#### Backup semua volume

```bash
cd /project/dengan/compose/file
ct-volume backup
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

#### Backup volume spesifik

```bash
ct-volume backup -v postgres_data
```

#### Backup dengan custom backup directory

Direktori output backup dapat ditentukan dengan flag `-d` atau `--backup-dir`. Default: `./backups`.

```bash
ct-volume backup \
  -f docker-compose.prod.yml \
  -d /mnt/backups/docker
```

#### Backup dengan retention policy

Hanya menyisakan 7 backup terakhir per volume:

```bash
ct-volume backup --retain 7
```

#### Backup + bundle project (siap kirim antar server)

Backup volume, lalu zip seluruh direktori project (compose file + `.env` + config + backups) jadi satu archive:

```bash
ct-volume backup --bundle
```

Output:
```
ℹ Creating project bundle: my-web-app-20260626-221138.tar.gz
✓ Bundle created: /path/to/my-web-app-20260626-221138.tar.gz
```

Bundle output dapat diarahkan ke direktori tertentu dengan flag `-o` atau `--output`. Default: parent directory dari project.

```bash
ct-volume backup --bundle -o /mnt/backups/bundles
```

Nama file otomatis: `{nama-dir-project}-{YYYYMMDD}-{HHMMSS}.tar.gz`

### Restore

#### Restore semua volume

```bash
ct-volume restore
```

Akan muncul konfirmasi:

```
⚠ You are about to RESTORE 2 volume(s):
  - 20250626-143022-postgres_data.tar.gz
  - 20250626-143022-redis_data.tar.gz

This will OVERWRITE existing volume data. Continue? [y/N]
```

#### Restore tanpa konfirmasi

```bash
ct-volume restore --force
```

#### Restore volume spesifik

```bash
ct-volume restore -v postgres_data
```

#### Restore dari direktori backup tertentu

```bash
ct-volume restore -d /mnt/backups/docker
```

#### Restore dari bundle

Ekstrak bundle + restore volume langsung dalam satu perintah:

```bash
ct-volume restore --bundle my-web-app-20260626-221138.tar.gz
```

Cocok untuk migrasi antar server — cukup kirim 1 file `.tar.gz`, lalu restore.

### Dir Backup

Backup satu direktori project + seluruh volume docker-compose menjadi satu archive.

```bash
ct-volume dir-backup /path/to/project
```

Alur:
1. Validasi direktori target
2. Cari `docker-compose.yml` di dalamnya
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

### Dir Restore

Restore archive (project + volume) ke current working directory.

```bash
cd /target/location
ct-volume dir-restore ../my-project-backup-20250709-120000.tar.gz
```

Alur:
1. Ekstrak archive ke current directory
2. Cari folder `volumes/` di dalam hasil extract
3. Restore setiap volume docker dari file `.tar.gz` di `volumes/`
4. Selesai — user jalankan `docker compose up -d` manual

Output:
```
ℹ Extracting archive: my-project-backup-20250709-120000.tar.gz
ℹ Restoring volume: postgres_data
✓ Volume restored: postgres_data
✓ Restore finished. 2 volume(s) restored to /target/location/my-project
ℹ Run 'docker compose up -d' manually inside .../my-project when ready.
```

### List

#### List named volumes

```bash
ct-volume list
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
ct-volume list -v postgres_data
```

## Alur Kerja

### Backup
```
ct-volume backup
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

### Backup --bundle
```
ct-volume backup --bundle
  │
  ├── [proses backup normal]
  ├── Tentukan project root (parent dari compose file)
  ├── cd ke parent directory
  └── tar czf {nama-project}-{TIMESTAMP}.tar.gz {nama-project}/
      └── Hasil: 1 file siap kirim antar server
```

### Restore
```
ct-volume restore
  │
  ├── Cari archive backup terbaru untuk setiap volume
  ├── Tampilkan konfirmasi (kecuali --force)
  ├── docker compose down
  ├── Untuk setiap volume:
  │     └── docker run alpine tar xzf /backup/archive.tar.gz
  ├── docker compose up -d
  └── Selesai
```

### Restore --bundle
```
ct-volume restore --bundle archive.tar.gz
  │
  ├── Ekstrak archive → project dir
  ├── Deteksi compose file di dalam project dir
  ├── Set backup_dir ke ./backups/ di dalam project dir
  └── [proses restore normal]
```

### Dir Backup
```
ct-volume dir-backup /path/to/project
  │
  ├── Validasi direktori target
  ├── Cari docker-compose.yml di dalamnya
  ├── Cek container status
  │     ├── Running → warning, minta user stop manual
  │     └── Stopped → lanjut
  ├── Copy semua file project ke staging area
  ├── Backup named volumes ke staging/volumes/
  ├── Archive staging + compress → {nama}-backup-{timestamp}.tar.gz
  └── Simpan di current directory
```

### Dir Restore
```
ct-volume dir-restore archive.tar.gz
  │
  ├── Ekstrak archive ke current directory
  ├── Cari folder volumes/ di hasil extract
  ├── Untuk setiap .tar.gz di volumes/:
  │     └── docker run alpine tar xzf → restore ke docker volume
  └── Selesai (user compose up -d manual)
```

## Contoh Lengkap

**Skenario: backup database Postgres dari production**

```bash
# 1. Cek volume apa saja yang ada
ct-volume list -f docker-compose.prod.yml

# 2. Backup hanya volume postgres
ct-volume backup \
  -f docker-compose.prod.yml \
  -v postgres_data \
  -d /backups/postgres \
  --retain 14

# 3. Restore ke environment staging
ct-volume restore \
  -f docker-compose.staging.yml \
  -v postgres_data \
  -d /backups/postgres \
  --force
```

## Migrasi Antar Server via Tailscale

Workflow lengkap pindah project beserta volume ke server lain menggunakan Tailscale + `--bundle`.

### Prasyarat

- Kedua server terhubung dalam satu jaringan Tailscale
- `ct-volume` sudah terinstall di kedua server

### Step-by-step

**Server A (sumber):**

```bash
# 1. Masuk ke direktori project
cd /var/www/my-web-app

# 2. Backup volume + bundle seluruh project
ct-volume backup --bundle -o /tmp
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
# 3. Restore langsung dari bundle
cd /tmp
ct-volume restore --bundle my-web-app-20260626-221138.tar.gz --force
```

Selesai. Volume + compose file + config sudah ter-restore di Server B.

### Automation dengan cron + rsync

**Server A — backup otomatis tiap hari:**

```bash
# /etc/cron.daily/backup-webapp
#!/bin/bash
cd /var/www/my-web-app
/usr/local/bin/ct-volume backup --bundle -o /tmp/backups --retain 7
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
```bash
sudo apt install jq   # Debian/Ubuntu
sudo pacman -S jq     # Arch
```

### `docker compose` command not found
Gunakan `docker-compose` (V1) atau install plugin V2:
```bash
sudo apt install docker-compose-v2
```

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
