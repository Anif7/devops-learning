# File Transfers and Storage Automation with Rclone

Transfer, sync and verify files between local folders and S3-compatible storage using **rclone** and a local **MinIO** server.

- No `rclone.conf`: S3 settings come from environment variables.
- No passwords in code: MinIO logins live in `.env` (git-ignored).

## Requirements

- rclone
- Docker with the Compose plugin (`sudo apt install docker-compose-plugin`)

## Setup

```bash
# 1. Create .env and set your own passwords
cp .env.example .env

# 2. Start MinIO
docker compose up -d --wait

# 3. Create test data (sample-data/ and a 1 GB file in sample-data-large/)
./generate-sample-data.sh
```

MinIO URLs:

- S3 API: http://127.0.0.1:9000
- Web console: http://127.0.0.1:9001

## Terminal Setup

Run these in every new terminal.

**1. Disable the rclone config file**

```bash
export RCLONE_CONFIG=/dev/null
```

**2. Load the MinIO logins from `.env`**

```bash
set -a; source .env; set +a
```

**3. Point rclone at MinIO**

```bash
export RCLONE_S3_PROVIDER=Minio
export RCLONE_S3_ENDPOINT=http://127.0.0.1:9000
export RCLONE_S3_ACCESS_KEY_ID="$MINIO_ROOT_USER"
export RCLONE_S3_SECRET_ACCESS_KEY="$MINIO_ROOT_PASSWORD"
```

**4. Test the connection**

```bash
rclone lsd :s3:
```

`:s3:` tells rclone to build an S3 remote from the `RCLONE_S3_*` variables, without saving anything.

## transfer.sh

```bash
./transfer.sh copy <source> <destination> [rclone flags...]
./transfer.sh sync <source> <destination> [--confirm] [rclone flags...]
```

Every run shows progress, verifies the result with `rclone check`, and writes a log file to `logs/`.

**Copy files to a bucket**

```bash
./transfer.sh copy sample-data :s3:main-bucket/uploads
```

**Copy with extra rclone flags (for example, a bandwidth limit)**

```bash
./transfer.sh copy sample-data-large :s3:main-bucket/large --bwlimit 30M
```

**Preview a sync (nothing is changed)**

```bash
./transfer.sh sync sample-data :s3:main-bucket/site
```

**Run the sync**

```bash
./transfer.sh sync sample-data :s3:main-bucket/site --confirm
```

**Sync with a delete limit and a backup of deleted files**

```bash
MAX_DELETE=10 SYNC_BACKUP_DIR=:s3:main-bucket/backup \
  ./transfer.sh sync sample-data :s3:main-bucket/site --confirm
```

**Check the result**

```bash
echo $?      # 0 = success, 2 = wrong usage, anything else = rclone error
ls logs/
```

## More

- [commands.md](commands.md): all rclone commands used, with explanations
- [results.md](results.md): observations and answers to common questions

## Cleanup

```bash
docker compose down -v     # stop MinIO and delete its data
rm -rf storage-a storage-b downloads logs
```
