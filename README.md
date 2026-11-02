# Shared Storage with Amazon EFS across Multiple EC2 Instances

A hands-on AWS project that shows how a single **Amazon Elastic File System (EFS)** can be mounted on **two different EC2 instances** at the same time, so files created on one server instantly appear on the other.

## Architecture

```
                 +----------------------------+
                 |   Amazon EFS (EFS1)        |
                 |   encrypted, us-east-1     |
                 +-------------+--------------+
                               |  NFS (TCP 2049)
                 Security Group: SGEFS
              +----------------+----------------+
              |                                 |
   +----------+-----------+         +-----------+----------+
   | EC2 Instance A       |         | EC2 Instance B       |
   | Amazon Linux         |         | Ubuntu               |
   | /mnt/efs/fs1         |         | /mnt/efs/fs1         |
   +----------------------+         +----------------------+
```

## Resources Used

| Resource        | Details                                               |
| --------------- | ----------------------------------------------------- |
| Region          | US East (N. Virginia) `us-east-1`                     |
| Security Group  | `SGEFS` (`sg-0bd30d81e383826c9`)                      |
| EFS File System | `EFS1` (`fs-0a0bd72d333478503`), encrypted            |
| EC2 Instance A  | `i-0d111ef1c85498733` (Amazon Linux, user `ec2-user`) |
| EC2 Instance B  | `i-0913e0e1737f2dea6` (Ubuntu, user `ubuntu`)         |
| Mount point     | `/mnt/efs/fs1` on both instances                      |

## Prerequisites

- An AWS account (Free Tier is enough)
- Basic knowledge of EC2 and Linux commands
- Both EC2 instances and the EFS mount targets in the same VPC

## Steps

### 1. Create a Security Group for EFS

EC2 → Security Groups → **Create security group**

- Name: `SGEFS`
- Add an **inbound rule**: Type `NFS`, Port `2049`, Source = the security group of your EC2 instances (or your VPC CIDR)
- Attach `SGEFS` to the EFS mount targets (and make sure the EC2 instances can reach it)

![Security Group](screenshots/01-security-group.png)

### 2. Create the EFS File System

EFS → **Create file system**

- Name: `EFS1`
- Encryption: enabled
- Network: select the VPC and use `SGEFS` on the mount targets

![EFS](screenshots/02-efs-created.png)

### 3. Launch EC2 Instance A (Amazon Linux)

Launch an Amazon Linux instance in the same VPC.

![Instance A](screenshots/03-instance-a-launched.png)

### 4. Mount EFS on Instance A and create a file

```bash
sudo yum install -y amazon-efs-utils
sudo mkdir -p /mnt/efs/fs1
sudo mount -t efs -o tls fs-0a0bd72d333478503:/ /mnt/efs/fs1

sudo -i
cd /mnt/efs/fs1
ls
cat > altaf        # type some text, press Ctrl+D
ls
```

![Instance A terminal](screenshots/04-instance-a-file-created.png)

### 5. Launch EC2 Instance B (Ubuntu)

Launch an Ubuntu instance in the same VPC.

![Instance B](screenshots/05-instance-b-launched.png)

### 6. Connect to Instance B using EC2 Instance Connect

![Instance B connect](screenshots/06-instance-b-connected.png)

### 7. Mount EFS on Instance B and verify the shared file

```bash
sudo apt update
sudo apt install -y nfs-common
sudo mkdir -p /mnt/efs/fs1
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
  fs-0a0bd72d333478503.efs.us-east-1.amazonaws.com:/ /mnt/efs/fs1

sudo -i
cd /mnt/efs/fs1
ls                 # the file "altaf" created on Instance A is visible here
mkdir vcube        # create a new folder from Instance B
ls
```

![Instance B terminal](screenshots/07-instance-b-shared-data.png)

### 8. Verify on Instance A

Run `ls` again on Instance A. The `vcube` directory created on Instance B now appears.

![Instance A verification](screenshots/08-instance-a-sees-vcube.png)

## Result

| Action                 | Performed on | Seen on    |
| ---------------------- | ------------ | ---------- |
| Created file `altaf`   | Instance A   | Instance B |
| Created folder `vcube` | Instance B   | Instance A |

EFS gives both servers one shared, always-in-sync file system.

## Clean Up (avoid charges)

1. Terminate both EC2 instances
2. Delete the EFS file system `EFS1`
3. Delete the security group `SGEFS`

## Key Learnings

- EFS is a managed, elastic NFS file system that scales automatically
- Multiple EC2 instances (even with different operating systems) can mount it at once
- Security groups must allow **NFS (TCP 2049)** between the instances and the mount targets

## Author

**Altaf**
