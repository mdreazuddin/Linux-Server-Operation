# MinIO high-performance object storage Site-replication in production.
**What is MinIO?**
- MinIO is a high-performance, open-source object storage system for managing large volumes of unstructured data. It is known for its high performance and its full compatibility with the Amazon S3 API, allowing it to serve as a self-hosted alternative to AWS S3

**Consedering node & IP**

**Version**: minio RELEASE.2024-02-16T11-05-48Z.
- SiteA (source): 10.10.67.162
- SiteB: 192.168.100.162
- SiteC: 10.9.0.162

**Install & run MinIO on Ubuntu 24 (each host)**

**- Create service user + data dir:**

`sudo apt update -y`

`sudo timedatectl set-timezone Asia/Dhaka`

`sudo useradd -M -r -g minio-user minio-user`

`sudo mkdir -p /minio-data`

`sudo chown -R minio-user:minio-user /minio-data`

`sudo chmod -R 750 /minio-data`

**- Install MinIO server binary:**

`sudo curl -fsSLo /usr/local/bin/minio "https://dl.min.io/server/minio/release/linux-amd64/archive/minio.RELEASE.2024-07-10T18-41-49Z"`

`sudo chmod +x /usr/local/bin/minio`

**Systemd unit configuration:** sudo vim /etc/systemd/system/minio.service
```
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
Type=notify

WorkingDirectory=/usr/local

User=minio-user
Group=minio-user
ProtectProc=invisible

EnvironmentFile=-/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# Let systemd restart this service always
Restart=always

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=1048576

# Turn-off memory accounting by systemd, which is buggy.
MemoryAccounting=no

# Specifies the maximum number of threads this process can create
TasksMax=infinity

# Disable timeout logic and wait until process is stopped
TimeoutSec=infinity

SendSIGKILL=no

[Install]
WantedBy=multi-user.target

# Built for ${project.name}-${project.version} (${project.name})
```

**Create the Service Environment File on Each Node:** sudo vim /etc/default/minio
```
# Set the hosts and volumes MinIO uses at startup
# The command uses MinIO expansion notation {x...y} to denote a
# sequential series.
#
# The following example covers four MinIO hosts
# with 4 drives each at the specified hostname and drive locations.
# The command includes the port that each MinIO server listens on
# (default 9000)

#MINIO_VOLUMES="https://minio{1...4}.example.net:9000/mnt/disk{1...4}/minio"

MINIO_VOLUMES="/minio-data/minio"


# Set all MinIO server options
#
# The following explicitly sets the MinIO Console listen address to
# port 9001 on all network interfaces. The default behavior is dynamic
# port selection.

MINIO_OPTS="--console-address :9001"

# Set the root username. This user has unrestricted permissions to
# perform S3 and administrative API operations on any resource in the
# deployment.
#
# Defer to your organizations requirements for superadmin user name.

MINIO_ROOT_USER=admin

# Set the root password
#
# Use a long, random, unique string that meets your organizations
# requirements for passwords.

MINIO_ROOT_PASSWORD=Admin123
```

`sudo systemctl daemon-reload`

`sudo systemctl enable --now minio`

`sudo systemctl status minio`


**Install MinIO Client(mc) on admin box (or any host):**

`sudo curl -L https://dl.min.io/client/mc/release/linux-amd64/mc -o mc`

`sudo chmod +x mc`

`sudo mv mc /usr/local/bin/`

`mc --version`

**Create aliases** here site A,B,C all are same user and Password (user: admin Pass: admin123)

`mc alias set siteA http://10.10.67.162:9000 <ADMIN_A> '<PASS_A>'`

`mc alias set siteB http://192.168.100.162:9000 <ADMIN_B> '<PASS_B>'`

`mc alias set siteC http://10.9.0.162:9000 <ADMIN_C> '<PASS_C>'`

- Verify

`mc admin info siteA`

`mc admin info siteB`

`mc admin info siteC`


**Now rules of Site Replication (SR) — auto-mirror bucket create/delete, objects, IAM**

`mc admin replicate add siteA siteB`

`mc admin replicate add siteA siteC`

`mc admin replicate update siteA --mode sync`

- Create bucket & Enable versioning:
```
# Create bucket on each site (no-op if exists)
mc mb siteA/my-bucket
mc mb siteB/my-bucket
mc mb siteC/my-bucket

# Enable versioning (required)
mc version enable siteA/my-bucket
mc version enable siteB/my-bucket
mc version enable siteC/my-bucket
```

- Rules bucket:
```
# A → B (priority 1) replace with admin user & pass
mc replicate add siteA/my-bucket \
  --remote-bucket 'http://<ADMIN_B>:<URL_ENC_PASS_B>@192.168.100.162:9000/my-bucket' \
  --replicate 'existing-objects,delete,delete-marker' \
  --priority 1

# A → C (priority 2) replace with admin user & pass
mc replicate add siteA/my-bucket \
  --remote-bucket 'http://<ADMIN_C>:<URL_ENC_PASS_C>@10.9.0.162:9000/my-bucket' \
  --replicate 'existing-objects,delete,delete-marker' \
  --priority 2
```

- reacablity check of all nodes:

`curl -s -o /dev/null -w "%{http_code}\n" http://10.10.67.162:9000/minio/health/ready`

`curl -s -o /dev/null -w "%{http_code}\n" http://192.168.100.162:9000/minio/health/ready`

`curl -s -o /dev/null -w "%{http_code}\n" http://10.9.0.162:9000/minio/health/ready`


**Note: Initial replication may time require.**


**From GUI also can manage Site Replication:** 
- login: http://10.9.0.162:9001

- here the example of successfull replication
<img width="1853" height="862" alt="image" src="https://github.com/user-attachments/assets/0e550e7a-7cc6-405c-8a69-ef66ea90338c" />
