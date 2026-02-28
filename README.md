# gitea_assigment
This project deploys a Gitea server on an EC2 instance using Docker, with all persistent application data stored on a separate 20 GiB EBS volume mounted at ~/data. The goal is to separate compute (EC2) from state (EBS/S3) so the service can be rebuilt or moved without losing data. Backups are created using a provided script that compresses the Gitea data directory and uploads the archive to an S3 bucket. I also implemented a restore procedure to recover the service state from S3.

EC2 Setup. I launched an Ubuntu EC2 instance and configured the security group to allow:TCP 22 for SSH and TCP 3000 for the Gitea web interface. I connected via SSH to complete the setup.

Create and Attach EBS Volume. I created a 20 GiB EBS volume in the same Availability Zone as my EC2 instance and attached it as /dev/sdf.

Format and Mount the Volume
I identified the device name and formatted it as ext4:
sudo lsblk
sudo mkfs.ext4 /dev/nvme1n1
mkdir -p ~/data
sudo mount /dev/nvme1n1 ~/data
df -
