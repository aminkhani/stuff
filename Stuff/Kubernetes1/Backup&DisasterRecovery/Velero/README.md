# 🛡️ Kubernetes Backup Guide

This document provides clear instructions for setting up and managing Kubernetes backups using **Minio** and **Velero**, along with an overview of their advantages and disadvantages.

## 🚀 1. Setup Minio for Backup Storage

### 📁 Step 1: Start a Minio Server and Create a Bucket

Use this Docker Compose file:

```yaml
# compose.yml

version: '3'
services:
  minio:
    image: quay.io/minio/minio
    container_name: minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password
    ports:
      - "9000:9000"  # S3 API
      - "9001:9001"  # Web UI
    volumes:
      - minio-data:/data
volumes:
  minio-data:
```

#### 🌐 Access the Minio Web UI:

- Go to `http://localhost:9001`
- Login with credentials: `admin / password`
- Create a bucket named `velero-backups`.

## ⚙️ 2. Install Velero

### 🔑 Step 1: Create Minio Credentials

Create a file named **`credentials-velero`**:

```bash
# credentials-velero
cat <<EOF > credentials-velero
[default]
aws_access_key_id=admin
aws_secret_access_key=password
EOF
```

### 📥 Step 2: Install Velero CLI

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.15.2/velero-v1.15.2-linux-amd64.tar.gz
tar xvf velero-v1.15.2-linux-amd64.tar.gz
cd velero-v1.15.2-linux-amd64
cp velero /usr/local/bin/
```

### 🚢 Step 3: Deploy Velero to Kubernetes

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.1 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=minio,s3Url=http://192.168.7.253:9000,s3ForcePathStyle="true" \
  --use-volume-snapshots=false
```


## 💾 3. Backup Kubernetes Resources

### 📸 Step 1: Take a Backup

Backup all namespaces:

```bash
velero backup create BACKUP_NAME --include-namespaces '*' --wait
```

### ✅ Verify the Backup

```bash
velero get backup
```

## ♻️ 4. Restore Kubernetes Resources

### 🔁 Step 1: Restore from Backup

```bash
velero restore create --from-backup BACKUP_NAME
```

### ✅ Verify the Restore

```bash
velero restore get
```

## ⏰ 5. Schedule Regular Backups

Set up a scheduled daily backup at 07:00 UTC:

```bash
velero schedule create daily-backup --schedule "0 7 * * *"
```

## ✅ Pros of Using Velero + Minio

- 🧩 **Easy Setup:** Simple, scriptable deployment process.
- 💸 **Cost-Effective:** Minio is a lightweight, self-hosted S3-compatible option.
- 🧠 **Intelligent Automation:** Supports scheduled and manual backups.
- 🔀 **Flexibility:** Works with on-prem and cloud environments.
- 📈 **Scalable:** Scales easily as your cluster grows.

## ⚠️ Cons of Using Velero + Minio

- ⚙️ Initial Complexity: Requires understanding of Kubernetes, S3, and YAML.
- 🔐 Credential Management: Secrets and credentials must be handled securely.
- 🧹 Maintenance Overhead: Periodic testing and cleanup needed.
- 📦 Storage Monitoring: Persistent volumes in Minio may grow rapidly.

## 📝 Final Recommendations

- 🔍 Test backups and restores regularly in a staging environment.
- 🔐 Store credentials in a secure location (e.g., use Kubernetes Secrets or a vault).
- 📊 Use monitoring tools or alerts for failed backup jobs.
- 📅 Keep a retention policy and automate cleanup of old backups.