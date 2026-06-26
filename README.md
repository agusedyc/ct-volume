# ct-volume

Docker Volume Backup/Restore Tool — backup dan restore volume Docker berdasarkan docker-compose / compose file dengan mudah.

```
ct-volume backup
ct-volume restore
ct-volume list
```

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

### Backup semua volume

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

### Backup volume spesifik

```bash
ct-volume backup -v postgres_data
```

### Backup dengan custom compose file dan backup directory

```bash
ct-volume backup \
  -f docker-compose.prod.yml \
  -d /mnt/backups/docker
```

### Backup dengan retention policy

Hanya menyisakan 7 backup terakhir per volume:

```bash
ct-volume backup --retain 7
```

### Restore semua volume

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

### Restore tanpa konfirmasi

```bash
ct-volume restore --force
```

### Restore volume spesifik

```bash
ct-volume restore -v postgres_data
```

### Restore dari direktori backup tertentu

```bash
ct-volume restore -d /mnt/backups/docker
```

### List named volumes

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

### List volume spesifik

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
  └── Jika --retain → hapus backup terlama
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
