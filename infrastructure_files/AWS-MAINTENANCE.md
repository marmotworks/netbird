# NetBird AWS Infrastructure Maintenance Guide

This document describes operational and maintenance procedures for the self-hosted NetBird VPN stack on AWS deployed via CloudFormation (`netbird-aws-cfn.yaml`).

---

## 1. Architecture Overview

* **Compute**: AWS Graviton (`t4g.small` ARM64, Ubuntu 24.04 LTS).
* **Network**: Dedicated Elastic IP attached directly to the instance, mapped to Route 53 A-record (`netbird.<domain>`).
* **Ingress Ports**:
  * TCP 80: HTTP redirect and Let's Encrypt ACME verification.
  * TCP 443: HTTPS (Dashboard UI, REST API, gRPC HTTP/2, WebSocket relay).
  * UDP 3478: STUN NAT discovery.
  * UDP 49152–65535: Coturn TURN relay port range.
  * Port 22: Closed. All shell access is managed via AWS Systems Manager (SSM).
* **Storage & Persistence**:
  * Root Volume: 20 GB gp3 (`DeleteOnTermination: false`).
  * Data Volume (`NetBirdDataVolume`): Dedicated 20 GB gp3 persistent volume mounted to `/mnt/netbird-data` (`LABEL=netbird-data` in `/etc/fstab`, `DeletionPolicy: Retain`).
  * SQLite DB (`store.db`, `idp.db`, `events.db`, `access_logs.db`) and Let's Encrypt certificates (`acme.json`) live on the persistent data volume and are bind-mounted into Docker containers.
* **Auto-Healing**:
  * CloudWatch Alarm (`NetBirdRecoveryAlarm` on `StatusCheckFailed_System`) triggers `arn:aws:automate:<region>:ec2:recover` if underlying host hardware degrades.

---

## 2. Administration & Access

Admin access is conducted via AWS SSM Session Manager (no open SSH port or bastion needed).

### Connect to Instance Shell
```bash
aws ssm start-session --target <INSTANCE_ID>
```

Switch to root or ubuntu user:
```bash
sudo su - ubuntu
```

---

## 3. Service Management (Docker Compose)

All NetBird services run via Docker Compose in `/home/ubuntu/netbird/infrastructure_files`.

### Inspect Container Status
```bash
cd /home/ubuntu/netbird/infrastructure_files
docker compose ps
```

Expected containers:
* `netbird-dashboard`
* `netbird-traefik`
* `netbird-server`

### View Logs
```bash
# All services
docker compose logs -f --tail=100

# Specific service
docker compose logs -f netbird-server
docker compose logs -f netbird-traefik
```

### Restart Services
```bash
cd /home/ubuntu/netbird/infrastructure_files
docker compose restart
```

---

## 4. Upgrading NetBird Containers

To update NetBird container images to the latest release:

1. Connect to the instance via SSM.
2. Navigate to Compose directory:
   ```bash
   cd /home/ubuntu/netbird/infrastructure_files
   ```
3. Pull updated container images:
   ```bash
   docker compose pull
   ```
4. Recreate containers with minimal interruption (~5-10s):
   ```bash
   docker compose up -d
   ```
5. Verify health:
   ```bash
   docker compose ps
   curl -I https://localhost/ -H "Host: <DOMAIN_NAME>"
   ```

---

## 5. OS & Security Patching

Operating system updates (kernel, systemd, libraries) should be installed in-place via SSM rather than replacing the EC2 instance in CloudFormation.

```bash
# Update package repositories and upgrade packages
sudo DEBIAN_FRONTEND=noninteractive apt-get update -y
sudo DEBIAN_FRONTEND=noninteractive apt-get dist-upgrade -y
sudo apt-get autoremove -y

# Check if a kernel reboot is required
if [ -f /var/run/reboot-required ]; then
  sudo reboot
fi
```

Docker and NetBird services will restart automatically after boot (`restart: unless-stopped`).

---

## 6. Backup & Disaster Recovery

### Automated Backups
A daily cron job runs `/etc/cron.daily/netbird-sqlite-backup` to produce non-blocking SQLite database backups:
* Backup location: `/mnt/netbird-data/backups/`
* Retention: 7 days (older files automatically purged).

### Manual On-Demand Backup
```bash
DATE=$(date +%Y%m%d_%H%M%S)
sudo sqlite3 /mnt/netbird-data/data/store.db ".backup /mnt/netbird-data/backups/manual_store_$DATE.db"
sudo sqlite3 /mnt/netbird-data/data/idp.db ".backup /mnt/netbird-data/backups/manual_idp_$DATE.db"
```

### EBS Snapshots
To snapshot the persistent data volume before maintenance:
```bash
aws ec2 create-snapshot \
  --volume-id <DATA_VOLUME_ID> \
  --description "Pre-maintenance NetBird data volume snapshot"
```

### Disaster Recovery (Re-attaching Data Volume)
Because `NetBirdDataVolume` has `DeletionPolicy: Retain`, the EBS volume survives stack deletion or instance failure:
1. If launching a new instance, attach the existing volume ID as `/dev/xvdf`.
2. Mount the volume:
   ```bash
   mkdir -p /mnt/netbird-data
   mount /dev/disk/by-label/netbird-data /mnt/netbird-data
   ```
3. Start Docker Compose pointing to `/mnt/netbird-data`. All users, keys, and network routes are immediately restored.

---

## 7. CloudFormation Updates

When making changes to `netbird-aws-cfn.yaml`:

1. **Always use Change Sets**:
   ```bash
   aws cloudformation create-change-set \
     --stack-name netbird-server \
     --change-set-name cfn-update \
     --template-body file://netbird-aws-cfn.yaml \
     --capabilities CAPABILITY_IAM \
     --change-set-type UPDATE
   ```
2. **Review the Change Set**:
   ```bash
   aws cloudformation describe-change-set \
     --stack-name netbird-server \
     --change-set-name cfn-update
   ```
   **Verify that `NetBirdInstance` does NOT show `Replacement: True`.** If `ImageId` or `UserData` changed, CloudFormation will attempt instance replacement. Pin `AmiId` to preserve the running host.
3. **Execute**:
   ```bash
   aws cloudformation execute-change-set \
     --stack-name netbird-server \
     --change-set-name cfn-update
   ```
