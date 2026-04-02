Enterprise Disaster Recovery & Backup Architecture
1. EC2 Setup
GUI Steps
• Go to Amazon EC2 → Launch Instance
• Create 2 instances with the following config:
• - AMI: Ubuntu
• - Instance type: t3.micro
• - VPC: Default
• - Same AZ (important for EBS Multi-Attach)
• Security Group:
• - SSH → Port 22 → Your IP
• - NFS → Port 2049 → Same Security Group
![preview](images/01.png)

CLI
• ssh -i key.pem ubuntu@<ip>
2. EBS Multi-Attach
GUI Steps
• Go to Amazon EBS → Volumes → Create Volume
• Settings:
• - Type: io2
• - Size: 100GB
• - Same AZ as EC2
• - Enable Multi-Attach
• Attach to both instances via Actions → Attach
![preview](images/02-v.png)

CLI
# Instance 1
lsblk
sudo mkfs.ext4 /dev/nvme1n1
sudo mkdir /data
sudo mount /dev/nvme1n1 /data
![preview](images/03-v2.png)

# Instance 2
sudo mkdir /data
sudo mount /dev/nvme1n1 /data
![preview](images/04-v2.png)
# Test
echo "EBS test" > /data/file.txt  # on Instance 1
cat /data/file.txt  # on Instance 2
# Resize
sudo resize2fs /dev/nvme1n1
![preview](images/05-file.png)

3. EFS Setup
GUI Steps
• Go to Amazon EFS → Create file system
• Select VPC and create mount targets (at least 1)
• Enable lifecycle policies:
• - IA after 30 days
• - Archive after 90 days
• Security Group Inbound Rules:
• - Type: NFS
• - Port: 2049
• - Source: EC2 Security Group
![preview](images/06- file2.png)


CLI
sudo apt update
sudo apt install -y nfs-common
sudo mkdir -p /efs
sudo mount -t nfs4 -o nfsvers=4.1 fs-XXXX.efs.us-east-1.amazonaws.com:/ /efs
sudo chown -R ubuntu:ubuntu /efs
# Test
echo "EFS test" > /efs/file.txt  # on Instance 1
cat /efs/file.txt  # on Instance 2
![preview](images/07-vm.png)
![preview](images/08-vm.png)
![preview](images/09-efs.png)
![preview](images/10-efs.png)
![preview](images/11-efsworking.png)


4. AWS Backup
GUI Steps
• Go to AWS Backup → Backup vaults → Create
• - Name: prod-vault
• Create Backup Plan → Build new
• - Rule 1: Hourly-EBS (1 hour, 1 day retention)
• - Rule 2: Daily-EFS-AMI (daily, 30 days retention)
• - Cross Region Copy: us-west-2, 1 year retention
• Assign resources: EC2, EBS, EFS
• Trigger backup: Protected resources → Backup now
![preview](images/12-key.png)
CLI
# Trigger backup via CLI (example)
aws backup start-backup-job --backup-vault-name prod-vault --resource-arn <arn> --iam-role-arn <role-arn> --start-window-minutes 60 --complete-window-minutes 180
![preview](images/13-back.png)

# Verify
aws backup list-backup-jobs
![preview](images/14-ps.png)

5. S3 Cross-Region Replication
GUI Steps
• Create Primary Bucket in us-east-1
• Create DR Bucket in us-west-2
• Enable Versioning on both
• Go to Management → Replication → Create rule
• Set destination to DR bucket
![preview](images/16-s3p.png)
![preview](images/16-s3s.png)
![preview](images/17-cors.png)
CLI
echo "crr test" > test.txt
aws s3 cp test.txt s3://<primary-bucket>
![preview](images/18.png)
6. Final Validation
Checklist
• ✅ EBS: echo "test" > /data/test.txt → check on other instance
• ✅ EFS: echo "test" > /efs/test.txt → check on other instance
• ✅ Backup Restore: Delete file → Restore from AWS Backup
• ✅ S3 Replication: Upload file → verify in DR bucket
