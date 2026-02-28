# gitea_assigment
This project deploys a Gitea server on an EC2 instance using Docker, with all persistent application data stored on a separate 20 GiB EBS volume mounted at ~/data. The goal is to separate compute (EC2) from state (EBS/S3) so the service can be rebuilt or moved without losing data. Backups are created using a provided script that compresses the Gitea data directory and uploads the archive to an S3 bucket. I also implemented a restore procedure to recover the service state from S3.

EC2 Setup. I launched an Ubuntu EC2 instance and configured the security group to allow:TCP 22 for SSH and TCP 3000 for the Gitea web interface. I connected via SSH to complete the setup.

Create and Attach EBS Volume. I created a 20 GiB EBS volume in the same Availability Zone as my EC2 instance and attached it as /dev/sdf.

Format and Mount the Volume. I identified the device name and formatted it as ext4:
sudo lsblk
sudo mkfs.ext4 /dev/nvme1n1
mkdir -p ~/data
sudo mount /dev/nvme1n1 ~/data
df -

Make the Mount Persistent. I retrieved the UUID and added it to /etc/fstab:
sudo blkid
sudo nano /etc/fstab

I added a line like:
UUID=<MY-UUID> /home/ubuntu/data ext4 defaults,nofail 0 2
Then I tested and rebooted:
sudo mount -a
df -h

After reboot, the volume was still mounted at ~/data.

Running Gitea with Docker. Install Docker
sudo apt update
sudo apt install -y docker.io
docker --version

Run Gitea with EBS-backed Bind Mount. I ran Gitea using a bind mount so all data is stored on the EBS volume:
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -p 2222:22 \
  -v /home/ubuntu/data:/data \
  gitea/gitea:latest
I completed the initial setup in the browser and created a test repository with at least one commit.

Persistence Test (Container Lifecycle). To verify persistence, I tested:
Stop → Start
docker stop gitea
docker start gitea

Remove → Recreate
docker rm -f gitea
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -p 2222:22 \
  -v /home/ubuntu/data:/data \
  gitea/gitea:latest
  
My repository remained intact, confirming that data was stored on the EBS volume.

Backup Instructions (S3). Create S3 Bucket
I created a unique S3 bucket with:
Block Public Access: ON

I used the script exactly as provided:
#!/usr/bin/env bash
set -euo pipefail
TS="$(date -u +%Y%m%dT%H%M%SZ)"
ARCHIVE="/tmp/gitea-backup-${TS}.tar.gz"
sudo tar -czf "${ARCHIVE}" -C "$HOME/data" .
echo "Created backup archive: ${ARCHIVE}"

Run it:
chmod +x backup.sh
./backup.sh

Upload Backup to S3:
BUCKET="s3://<MY-BUCKET>/backups"
ARCHIVE="$(ls -t /tmp/gitea-backup-*.tar.gz | head -n 1)"
aws s3 cp "${ARCHIVE}" "${BUCKET}/"
aws s3 ls "${BUCKET}/"

The backup appeared in S3 successfully.

Restore Instructions
1. Download Backup from S3:
aws s3 cp s3://<MY-BUCKET>/backups/<backup-file>.tar.gz /tmp/
2. Stop and Remove the Container:
docker stop gitea
docker rm gitea
3. Restore the Data:
sudo rm -rf ~/data/*
sudo tar -xzf /tmp/<backup-file>.tar.gz -C ~/data
4. Recreate Gitea:
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -p 2222:22 \
  -v /home/ubuntu/data:/data \
  gitea/gitea:latest

My repository reappeared, confirming the restore process worked.

