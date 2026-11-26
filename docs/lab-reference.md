# Lab and deployment reference

This is a record of a Universidad del Quindío lab, not an automated installer. Perform storage work only in a disposable VM after identifying every device; formatting or reassigning a block device can destroy its data. The lab is not a demonstration of failover between hosts.

## VM and disks

The VirtualBox exercise used an x86-64 Linux VM with 6 GB of RAM, five vCPUs, a 50 GB system disk, and nine 5 GB VDI data disks. The original record cites differing Ubuntu LTS versions in its text and screenshot caption; do not infer a tested release requirement from it. The extra disks were allocated as three three-way RAID 1 mirrors, providing 5 GB of nominal usable space per array before filesystem overhead.

| RAID device in the lab | Virtual disks | LVM group / logical volume | Mount | Container path |
| --- | --- | --- | --- | --- |
| `/dev/md0` | `sdb`, `sdc`, `sdd` | `vg_apache/lv_apache` | `/srv/apache_data` | `/usr/local/apache2/htdocs` |
| `/dev/md1` | `sde`, `sdf`, `sdg` | `vg_mysql/lv_mysql` | `/srv/mysql_data` | `/var/lib/mysql` |
| `/dev/md2` | `sdh`, `sdi`, `sdj` | `vg_nginx/lv_nginx` | `/srv/nginx_data` | `/usr/share/nginx/html` |

The sequence was to check kernel device discovery, assemble and verify the software mirrors, persist their definitions for early boot, and create one LVM physical volume, group, and logical volume per array. Each logical volume used its group's available space and was formatted as ext4. The mounts were persisted in `/etc/fstab`. The [original array status](../images/2.3.png) and [final storage layout](../images/3.5.png) illustrate the stages. Verify the actual mounts before attaching container bind mounts; do not assume the device names above match your VM.

## Service definitions

- [Nginx](../dockerfiles/Dockerfile_nginx.txt) uses `nginx:stable` and bakes a sample page into the image. A bind mount hides the baked web root, so supply the page in the host mount when using the lab layout.
- [Apache](../dockerfiles/Dockerfile_apache.txt) uses `httpd:2.4` and expects `html/index.html` in its build context. That file is not included.
- [MySQL](../dockerfiles/Dockerfile_mysql.txt) uses `mysql:8.0` and expects `init.sql` to initialize a lab database. That file is not included. Use private credentials and supply your own fixtures; the checked-in password is for the historical lab only.
- The [Compose file](docker-compose.yml) documents their ports, persistence, and restart policy, plus Netdata and its named state and cache volumes. Its named build directories are absent and must be corrected in any runnable adaptation.

The original persistence walkthrough edited the host web roots, restarted Apache, replaced Nginx while retaining its bind mount, and checked the resulting HTTP responses. For MySQL it queried the lab database, wrote test rows, restarted the container, and queried again. The fixture table in the historical lab was `prueba` in `proyecto_db`; it is not part of the checked-in sources. Do not substitute a real database or its credentials for that fixture.

## Monitoring

The documented Netdata host deployment uses host process and network access in its manual setup, elevated capabilities, and host mounts including the Docker socket. The Compose reference uses port `19999`, `SYS_PTRACE`, `SYS_ADMIN`, an unconfined AppArmor profile, read-only host mounts, and Docker socket access. Access to the socket is privileged even if mounted read-only. Review the collection design before exposing it; neither the lab nor prior dashboard screenshots establish that it is production-hardened.

The host dashboard shows CPU, RAM and swap, disk I/O, interfaces, and processes. The container views add per-service CPU, memory, network, and disk activity. The [case study](https://diegnghtmr.me/en/work/enterprise-monitoring-stack/) distinguishes the original host collection from the local screenshot run, which used service collectors and named Docker volumes without Docker socket access.

## Further reading

- [Docker Engine overview](https://docs.docker.com/engine/)
- [Red Hat LVM administration guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/index)
- [Netdata documentation](https://learn.netdata.cloud/docs/)
- [Ubuntu Server documentation](https://ubuntu.com/server/docs)
- [Omakub](https://omakub.org/), the original lab's development setup reference
