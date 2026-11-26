# Lab and deployment reference

This runbook records the Universidad del Quindío Computational Infrastructure lab completed in November 2025. The team consolidated Apache, Nginx, and MySQL from separate hosts into containers on one Linux VM, with mirrored storage for the services and Netdata for monitoring. The commands and Compose example below are the historical lab record, in execution order. Screenshots show the original lab, not a fresh validation of this checkout.

> **Use a disposable VM.** The storage steps write device metadata, format filesystems, and edit boot-time configuration. Confirm the identity of every disk before proceeding; the device names in the examples are not portable. The Docker examples use throwaway fixture credentials and expose ports. Do not use those credentials, mounts, or privileges on a shared or production host. The Dockerfile names and build contexts in the historical commands do not match the checkout; the gaps are called out at the relevant steps.

## 1. Prepare the server and attach disks

### 1.1 Install the base system

Create an x86-64 VM in Oracle VirtualBox with 6 GB of RAM and five vCPUs. The lab used Ubuntu LTS with DHH's Omakub development setup. The original write-up specifies Ubuntu 24.04.3 LTS, while the screenshot caption says Ubuntu 22.04.5 LTS; this record does not establish a minimum or verified Ubuntu version.

[Figure 1: VirtualBox guest with Ubuntu and Omakub](../images/0.png)

### 1.2 Create the virtual disks

In VirtualBox, create and attach one 50 GB VDI for the operating system and nine 5 GB data VDIs. Leave the data disks unpartitioned for the RAID steps. Three three-way mirrors provide 5 GB of nominal capacity per array, for 15 GB total before filesystem overhead.

[Figure 2: System VDI and the nine data VDIs](../images/1.1.png)

### 1.3 Verify Linux recognizes the disks

```bash
lsblk
```

The lab showed `sda` as the 50 GB system disk and `sdb` through `sdj` as nine unpartitioned 5 GB disks. Verify the system disk and all data disks before creating arrays. Stop if anything differs from the intended layout.

[Figure 3: Block devices reported by `lsblk`](../images/1.2.png)

## 2. Configure RAID and LVM

### 2.1 Install the storage tools

`mdadm` manages Linux software RAID. `lvm2` manages the physical volumes, volume groups, and logical volumes above it.

```bash
sudo apt update
sudo apt install mdadm lvm2 -y
```

[Figure 4: Installing `mdadm` and `lvm2`](../images/2.1.png)

### 2.2 Create three RAID 1 arrays

Each array mirrors data across three verified data disks. `--create` makes a new array, `--level=1` selects mirroring, and `--raid-devices=3` specifies three members. These commands write RAID metadata: never point them at the system disk or a disk whose contents you need to retain.

```bash
# Create the first RAID 1 array
sudo mdadm --create /dev/md0 --level=1 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd

# Create the second RAID 1 array
sudo mdadm --create /dev/md1 --level=1 --raid-devices=3 /dev/sde /dev/sdf /dev/sdg

# Create the third RAID 1 array
sudo mdadm --create /dev/md2 --level=1 --raid-devices=3 /dev/sdh /dev/sdi /dev/sdj
```

[Figure 5: Creating the three RAID arrays](../images/2.2.png)

### 2.3 Verify array state and synchronization

```bash
cat /proc/mdstat
```

Check the active devices, RAID level, synchronization state, and rebuild speed (if a rebuild is in progress).

[Figure 6: Array synchronization in `/proc/mdstat`](../images/2.3.png)

### 2.4 Persist the RAID configuration

Record the array definitions in `mdadm.conf` and update the initramfs so the guest can assemble them early in boot. The original command appends to the file. Check for existing definitions before using it; do not rerun it blindly.

```bash
# Save the array configuration
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf

# Update the initramfs to include the RAID configuration
sudo update-initramfs -u
```

[Figure 7: Recording the arrays and updating the initramfs](../images/2.4.png)

### 2.5 Create the LVM physical volumes

Initialize each RAID device as a physical volume. `pvcreate` writes LVM metadata to the selected block device.

```bash
sudo pvcreate /dev/md0
sudo pvcreate /dev/md1
sudo pvcreate /dev/md2
```

[Figure 8: Initializing the three LVM physical volumes](../images/3.1.png)

### 2.6 Create the volume groups

Create one volume group for each service. Each group pools the capacity on its respective physical volume.

```bash
sudo vgcreate vg_apache /dev/md0
sudo vgcreate vg_mysql /dev/md1
sudo vgcreate vg_nginx /dev/md2
```

[Figure 9: Creating the service volume groups](../images/3.2.png)

### 2.7 Create the logical volumes

Create one logical volume in each group. `-n` names it and `-l 100%FREE` allocates all currently available space in the group.

```bash
sudo lvcreate -n lv_apache -l 100%FREE vg_apache
sudo lvcreate -n lv_mysql -l 100%FREE vg_mysql
sudo lvcreate -n lv_nginx -l 100%FREE vg_nginx
```

[Figure 10: Allocating the logical volumes](../images/3.3.png)

### 2.8 Format the logical volumes

Format the new logical volumes as ext4, a journaling filesystem. This step overwrites existing filesystem data on the named devices.

```bash
sudo mkfs.ext4 /dev/vg_apache/lv_apache
sudo mkfs.ext4 /dev/vg_mysql/lv_mysql
sudo mkfs.ext4 /dev/vg_nginx/lv_nginx
```

[Figure 11: Formatting the logical volumes as ext4](../images/3.4.png)

### 2.9 Mount the volumes and make the mounts persistent

Create mount directories, mount each logical volume, and record the mounts in `/etc/fstab` for subsequent boots. The original commands append entries, so check the existing file first and avoid duplicate rows. Confirm each directory is backed by the expected logical volume before starting a container.

```bash
# Create the mount directories
sudo mkdir -p /srv/apache_data
sudo mkdir -p /srv/mysql_data
sudo mkdir -p /srv/nginx_data

# Mount the volumes
sudo mount /dev/vg_apache/lv_apache /srv/apache_data
sudo mount /dev/vg_mysql/lv_mysql /srv/mysql_data
sudo mount /dev/vg_nginx/lv_nginx /srv/nginx_data

# Add entries to /etc/fstab for automatic mounting
echo "/dev/vg_apache/lv_apache /srv/apache_data ext4 defaults 0 2" | sudo tee -a /etc/fstab
echo "/dev/vg_mysql/lv_mysql /srv/mysql_data ext4 defaults 0 2" | sudo tee -a /etc/fstab
echo "/dev/vg_nginx/lv_nginx /srv/nginx_data ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

| Array | Data disks in the lab | Volume group / logical volume | Host mount | Container path |
| --- | --- | --- | --- | --- |
| `/dev/md0` | `sdb`, `sdc`, `sdd` | `vg_apache/lv_apache` | `/srv/apache_data` | `/usr/local/apache2/htdocs` |
| `/dev/md1` | `sde`, `sdf`, `sdg` | `vg_mysql/lv_mysql` | `/srv/mysql_data` | `/var/lib/mysql` |
| `/dev/md2` | `sdh`, `sdi`, `sdj` | `vg_nginx/lv_nginx` | `/srv/nginx_data` | `/usr/share/nginx/html` |

[Figure 12: The final device and volume layout](../images/3.5.png)

## 3. Build and deploy the service containers

The manual procedure uses Docker on the lab VM and runs image builds from the project directory. The historical commands refer to lower-case Dockerfile names without the `.txt` suffix. The repository currently tracks the capitalized `.txt` files linked below. To reproduce the lab, prepare matching build contexts and align the names or `-f` arguments before running the recorded commands. Do not start the corresponding Compose services at the same time: both versions use the same names and ports.

### 3.1 Prepare the Nginx image

The [Nginx Dockerfile](../dockerfiles/Dockerfile_nginx.txt) is based on `nginx:stable`. It declares `/usr/share/nginx/html` as a volume, labels the image, and generates a sample `index.html` at build time. A bind mount over that directory hides the baked page; supply a page in the host web root when using the lab mount.

[Figure 13: Nginx Dockerfile](../images/4.png)

### 3.2 Build, run, and check Nginx

```bash
# Build the image from the project directory
sudo docker build -t mi_nginx:1.0 -f dockerfiles/dockerfile_nginx .

# Run the container with persistent storage
sudo docker run -d \
  --name nginx \
  -p 8081:80 \
  -v /srv/nginx_data:/usr/share/nginx/html \
  mi_nginx:1.0

# Check the running container
sudo docker ps

# Check connectivity
curl http://localhost:8081
```

`-d` runs the container in the background, `--name` identifies it, `-p 8081:80` maps the host port to the container's HTTP port, and `-v` binds the LVM-backed web root. Check that the expected page is served on port 8081.

- [Figure 14: Nginx container and command output](../images/4.1.png)
- [Figure 15: Nginx page in a browser](../images/4.2.png)

### 3.3 Prepare the Apache image

The [Apache Dockerfile](../dockerfiles/Dockerfile_apache.txt) uses `httpd:2.4`, labels the image, declares `/usr/local/apache2/htdocs` as a volume, and copies `html/index.html` into the image. Supply that fixture in the build context; it is not included. The host web root needs its own page when mounted over the image's web root.

[Figure 16: Apache Dockerfile](../images/5.png)

### 3.4 Build, run, and check Apache

```bash
# Build the Apache image from the project directory
sudo docker build -t mi_apache:1.0 -f dockerfiles/dockerfile_apache .

# Run the Apache container
sudo docker run -d \
  --name apache \
  -p 8080:80 \
  -v /srv/apache_data:/usr/local/apache2/htdocs \
  mi_apache:1.0

# Verify the service
sudo docker ps
curl http://localhost:8080
```

Check that the container is running and that the expected Apache page appears on port 8080.

- [Figure 17: Apache running with its persistent volume](../images/5.1.png)
- [Figure 18: Apache page in a browser](../images/5.2.png)

### 3.5 Prepare the MySQL image

The [MySQL Dockerfile](../dockerfiles/Dockerfile_mysql.txt) uses `mysql:8.0`, labels the image, declares `/var/lib/mysql` as a volume, and copies `init.sql` into `/docker-entrypoint-initdb.d/`. The lab used an initial database named `proyecto_db` and a seed script to create tables and insert sample data. The script is not included; supply a disposable fixture that creates the `prueba` table with `nombre` and `descripcion` fields for the later checks. The historical password values below are examples only, not credentials to reuse. Replace them consistently in any reproduction and avoid passing real credentials on the command line.

[Figure 19: MySQL Dockerfile](../images/6.png)

### 3.6 Build and run MySQL, then open the SQL client

```bash
# Build the MySQL image from the project directory
sudo docker build -t mi_mysql:1.0 -f dockerfiles/dockerfile_mysql .

# Run the MySQL container
sudo docker run -d \
  --name mysql \
  -p 3306:3306 \
  -v /srv/mysql_data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  mi_mysql:1.0

# Verify the database
sudo docker exec -it mysql mysql -uroot -prootpass
```

[Figure 20: MySQL container with its persistent data volume](../images/6.1.png)

### 3.7 Verify the database and seed records

Run the following in the MySQL client. Check that the database and table exist and that the seed rows are visible.

```sql
SHOW DATABASES;
USE proyecto_db;
SHOW TABLES;
SELECT * FROM prueba;
```

[Figure 21: SQL output showing initialized schemas and records](../images/6.2.png)

## 4. Verify persistence

### 4.1 Apache: edit the host web root and restart

Edit the mounted `index.html`, restart the Apache container, and check that the changed text appears in the HTTP response. The quoted HTML is the original fixture edit.

```bash
# Edit the file on the mounted volume
sudo nano /srv/apache_data/index.html

# Add the following content:
# <h2>Modificación realizada directamente en el volumen - Prueba de persistencia</h2>

# Restart the container
sudo docker restart apache

# Verify that the changes persist
curl http://localhost:8080
```

- [Figure 22: Editing the Apache content on its LVM mount](../images/7.1.png)
- [Figure 23: Apache serves the changed page after restart](../images/7.2.png)

### 4.2 Nginx: edit the host web root and replace the container

Edit the mounted `index.html`, remove the disposable Nginx container, and recreate it with the same bind mount. Its writable layer is removed; the host web root is not. Verify the new container serves the edited page.

```bash
# Edit the content on the Nginx volume
sudo nano /srv/nginx_data/index.html

# Remove and recreate the container
sudo docker rm -f nginx
sudo docker run -d \
  --name nginx \
  -p 8081:80 \
  -v /srv/nginx_data:/usr/share/nginx/html \
  mi_nginx:1.0

# Verify persistence
curl http://localhost:8081
```

- [Figure 24: Nginx replacement with the original mount](../images/7.3.png)
- [Figure 25: Nginx serves the changed content after replacement](../images/7.4.png)

### 4.3 MySQL: insert rows and query again after restart

In the original transcript below, enter the SQL statements and `exit` in the MySQL client; run the Docker commands in the host shell. The last query should return the rows just inserted. It checks persistence across a container restart, not recovery from loss of a disk or host.

```bash
# Connect to the MySQL container
sudo docker exec -it mysql mysql -uroot -prootpass

# In the MySQL client:
USE proyecto_db;
INSERT INTO prueba (nombre, descripcion) VALUES 
('Test Persistencia', 'Registro agregado para validar persistencia'),
('Registro Final', 'Última prueba de persistencia de datos');

SELECT * FROM prueba;

# Exit the client and restart the container
exit
sudo docker restart mysql

# Verify the data after restart
sudo docker exec -it mysql mysql -uroot -prootpass -e "USE proyecto_db; SELECT * FROM prueba;"
```

[Figure 26: Inserted MySQL rows still visible after restart](../images/7.5.png)

## 5. Deploy and check Netdata monitoring

### 5.1 Start the Netdata container

The original VM setup gave Netdata host PID and network access, elevated capabilities, host filesystem reads, and Docker socket access. `SYS_PTRACE` allows tracing host processes. Socket access is privileged even when mounted read-only, and the unconfined AppArmor profile removes a protection. These permissions are part of the historical lab, not a least-privilege deployment recommendation. With host networking, Docker ignores the published-port flag; the Netdata listener is reached on the host's port.

```bash
# Run Netdata with the permissions used in the lab
sudo docker run -d \
  --name netdata \
  --pid host \
  --network host \
  --cap-add SYS_PTRACE \
  --cap-add SYS_ADMIN \
  --security-opt apparmor=unconfined \
  -v /etc/passwd:/host/etc/passwd:ro \
  -v /etc/group:/host/etc/group:ro \
  -v /etc/localtime:/etc/localtime:ro \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /etc/os-release:/host/etc/os-release:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v netdata_lib:/var/lib/netdata \
  -v netdata_cache:/var/cache/netdata \
  -p 19999:19999 \
  netdata/netdata

# Verify deployment
sudo docker ps | grep netdata
```

[Figure 27: Netdata running in the lab VM](../images/8.1.png)

### 5.2 Open the dashboard

Replace the placeholder with the lab host's IP address and open the dashboard from the lab network:

```
http://[IP_DEL_HOST]:19999
```

### 5.3 Check host monitoring

Check the host view for per-core CPU activity, frequency and temperature where available, RAM and swap use, network traffic by interface, disk throughput, IOPS and latency, and process activity.

[Figure 28: Netdata host dashboard](../images/8.3.png)

### 5.4 Check container monitoring

Locate Apache, Nginx, and MySQL by container name. Inspect each container's CPU, memory, network traffic, and disk reads and writes. This verifies the dashboard sees those services; use the HTTP and SQL checks above to verify what each service returns.

[Figure 29: Container metrics for Apache, Nginx, and MySQL](../images/8.2.png)

## Docker Compose configuration

The original write-up also included this alternative to the manual Docker commands. It records the intended build contexts, ports, bind mounts, restart policy, and the named Netdata state and cache volumes. Unlike the manual Netdata command above, this example publishes a port and does not request host PID or host networking. The checked-in [Compose reference](docker-compose.yml) differs in path casing; neither version includes the required build directories or the Apache and MySQL fixture files. Prepare those inputs and replace the example MySQL password before adapting either file for a lab run.

```yaml
version: '3.8'

services:
  apache:
    build:
      context: ./dockerfiles/apache
    container_name: apache
    ports:
      - "8080:80"
    volumes:
      - /srv/apache_data:/usr/local/apache2/htdocs
    restart: unless-stopped

  nginx:
    build:
      context: ./dockerfiles/nginx
    container_name: nginx
    ports:
      - "8081:80"
    volumes:
      - /srv/nginx_data:/usr/share/nginx/html
    restart: unless-stopped

  mysql:
    build:
      context: ./dockerfiles/mysql
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: proyecto_db
    volumes:
      - /srv/mysql_data:/var/lib/mysql
    restart: unless-stopped

  netdata:
    image: netdata/netdata
    container_name: netdata
    hostname: monitor.local
    ports:
      - "19999:19999"
    cap_add:
      - SYS_PTRACE
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    volumes:
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - netdata_lib:/var/lib/netdata
      - netdata_cache:/var/cache/netdata
    restart: unless-stopped

volumes:
  netdata_lib:
  netdata_cache:
```

## What the lab demonstrated

The exercise combined virtual disks, RAID 1 mirrors, LVM, ext4 mounts, isolated service containers, and one monitoring dashboard. The team checked both web servers over HTTP, inspected the MySQL seed data, verified file or row persistence after restarting or replacing containers, and viewed host and container activity in Netdata. The LVM layout provides separate storage for each service and can be extended or managed with snapshots; a snapshot is not an independent backup. A three-way mirror can tolerate two failed members if the remaining member is healthy, but this lab did not demonstrate disk failover, a host outage, hot-swapping, load impact, or production alerting. RAID is not a backup and the shared VM is still a single point of failure.

The [portfolio case study](https://diegnghtmr.me/en/work/enterprise-monitoring-stack/) distinguishes the original lab from its later capture run. That run used service collectors over a Docker network, supplied fixtures, and used named Docker volumes rather than RAID/LVM or host-mounted Netdata collection.

## Further reading

- [Docker Engine overview](https://docs.docker.com/engine/)
- [Red Hat LVM administration guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/index)
- [Netdata documentation](https://learn.netdata.cloud/docs/)
- [Ubuntu Server documentation](https://ubuntu.com/server/docs)
- [Omakub](https://omakub.org/), the original lab's development setup reference

## Appendix: Historical repository layout

This is the original layout diagram. The comments are translated; the historical names are unchanged. See the linked `.txt` Dockerfiles in section 3 for their current paths.

```
enterprise-monitoring-stack/
│
├── README.md                   # Original lab guide
├── docs/                       # Additional documentation
│   └── docker-compose.yml      # Container orchestration
│
├── images/                     # Screenshots
│   ├── 0.png
│   ├── 1.1.png
│   ├── 1.2.png
│   ├── 2.1.png
│   ├── 2.2.png
│   ├── 2.3.png
│   ├── 2.4.png
│   ├── 3.1.png
│   ├── 3.2.png
│   ├── 3.3.png
│   ├── 3.4.png
│   ├── 3.5.png
│   ├── 4.png
│   ├── 4.1.png
│   ├── 4.2.png
│   ├── 5.png
│   ├── 5.1.png
│   ├── 5.2.png
│   ├── 6.png
│   ├── 6.1.png
│   ├── 6.2.png
│   ├── 7.1.png
│   ├── 7.2.png
│   ├── 7.3.png
│   ├── 7.4.png
│   ├── 7.5.png
│   ├── 8.1.png
│   ├── 8.2.png
│   └── 8.3.png
│
└── dockerfiles/                # Project Dockerfiles
    ├── dockerfile_apache       # Apache container configuration
    ├── dockerfile_nginx        # Nginx container configuration
    └── dockerfile_mysql        # MySQL container configuration

```
