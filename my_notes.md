## notes on getting humhub running on lightsail

### Create a ligthsail instance via AWS web console
1. us-east-2a (Ohio) zone
1. Ubuntu 22.04 LTS
1. default ssh key
1. dual ipv4 and ipv6
1. processor 2vcu + 1GB + 40GB at $7/mo
1. instance mccallie-humhub

### Create and assign static ip
1. mccallie-humhub-ip = 3.149.191.221
1. open port 443 for https (both ip4 and ip6)

### Setup ssh
1. download regional default pem
1. `mv ~/Downloads/LightsailDefaultPrivateKey-us-east-2.pem ~/.ssh/lightsail.pem`
1. `chmod 400 ~/.ssh/lightsail.pem`
1. add to `.ssh/config`
    ```
    Host lightsail
    HostName <your-instance-ip>
    User ubuntu
    IdentityFile ~/.ssh/lightsail.pem
    IdentitiesOnly yes
    ```
1. then connect with simple `$ssh`


### Install docker - using GPT script below:
```
#!/bin/bash

set -e

echo "🧱 Updating package index..."
sudo apt update
sudo apt upgrade -y

echo "📦 Installing required dependencies..."
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

echo "🔑 Adding Docker GPG key..."
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "📦 Adding Docker APT repository..."
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

echo "📦 Installing Docker Engine and Compose plugin..."
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

echo "👤 Adding $USER to docker group (you may need to log out and back in)..."
sudo usermod -aG docker $USER

echo "✅ Installation complete!"
docker version
docker compose version

echo "ℹ️ You may need to log out and back in or run 'newgrp docker' to use Docker without sudo."
```
### Reboot if necessary and test docker
```
$> reboot
$> docker ps
```
### repoint noip to the new IP address
1. Change `*.mccalliefamilystories.com` A record to point to -> 3.149.191.221

### Deploy Humhub test
1. git clone https from humhub/docker-dev into local directory (REVIST)
1. `scp ./.env lightsail:~/humhub-projects/docker-dev/`
1. `scp ./humhub.env lightsail:~/humhub-projects/docker-dev/`
1. name `network mccalliefamilynetwork`
1. login with david / ************ (set in .env)

## backup and restore

For backups I am using `offen/docker-volume-backup` added to the main `compose.yaml` file. I am keeping a local archive copy as well as copying the whole backup to S3. Be sure to set the proper S3 bucket name in the backup.env file

The gist of the extra compose code is this:
```
  # dpm add offen/docker-volume-backup sidecar for testing
  # must use version v2 to support mariadb-dump
  backup:
    image: offen/docker-volume-backup:v2
    restart: always
    env_file:
      - ./backup.env
    volumes:
      # bind-mount the HumHub data dir read-only into /backup/humhub-data
      - ./humhub-data:/backup/humhub-data:ro
      # volume mount the MySQL temp dump read-only into /backup/db-dump
      - db-dump:/backup/db-dump:ro
      # make a local copy for testing
      - ./backups-local:/archive
      # - ./caddy-data:/data/caddy-data
      # allow the backup container to stop & start the HumHub container
      - /var/run/docker.sock:/var/run/docker.sock:ro      
    
    # optional: label humhub so it’s stopped during backup for consistency
    # labels:
    #   - docker-volume-backup.stop-during-backup=true
    
    environment:
      # every day at 2am
      BACKUP_CRON_EXPRESSION: 0 2 * * *
      BACKUP_FILENAME: backup-humhubdata-%Y-%m-%dT%H-%M-%S.tar.gz
      BACKUP_PRUNING_PREFIX: backup-
      BACKUP_RETENTION_DAYS: 7
```

Then add the following to the "db" instructions.

The "labels" tag tells offen to dump the SQL file into a temporary named volume called `db-dump` which is then volume-mounted to /backup/db-dump by the offen instructions.  Offen backs up anything under /backup/*

Don't forget to add db-dump as a named volume at the end of the compose file

```
    # tell mariadb-dump to add a drop database and drop table statement so that 
    #  the restore use a clean database
    # note the hard-coded database name humhub (FIXME?)
    labels:
      - "docker-volume-backup.archive-pre=/bin/sh -c 'mariadb-dump \
        --databases humhub \
        --add-drop-database \
        --add-drop-table \
        --default-character-set=utf8mb4 \
        -u${HUMHUB_DOCKER_DB_USER} \
        -p${HUMHUB_DOCKER_DB_PASSWORD} \
        > /tmp/dumps/dump.sql'"
    volumes:
      - ./mysql-data:/var/lib/mysql
      - db-dump:/tmp/dumps
```

The `backup.env` file should contain this:
```
AWS_ACCESS_KEY_ID='key-goes-here'
AWS_SECRET_ACCESS_KEY='longer-key-goes-here'
AWS_S3_FILE_OVERWRITE=True
AWS_S3_REGION_NAME='us-east-1'
AWS_S3_BUCKET_NAME='test-humhub-backup'
```

## How to restore - rough notes

### from static backup (offen)
1. Get or clear a VPS and install docker, etc.  Or clone from existing one.
1. Clone down the customized repo from github into `/home/ubuntu/humhub-projects\humhub-docker-dev` (probably should be using /opt/humhub but doesn't seem to matter?)
1. `scp` all the .env files (there are at least three) from secure home to the vps - verify the env values
1. Copy the backup tar from s3 to the vps. Use s3 'temporary signed url' plus wget or curl. Will look similar to this:
```
wget --progress=bar:force -O ~/humhub-projects/backups/humhub-data-2025-06-30.tar.gz \
"https://test-humhub-backup.s3.us-east-1.amazonaws.com/backup-humhubdata-2025-06-29T19-00-00.tar.gz?response-content-disposition=inline&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&> X-Amz-Security-Token=IQoJb3JpZ2l ... 4"
```
5. extract backup to `/tmp` using tar with `--strip-components=1` to give you humhub-data and dump.sql files. like this (assuming you are in the `humhub-docker-dev` directory)
```
tar -xzvf humhub-data-2025-06-30.tar.gz --strip_components=2 -C .
```
6. docker compose -f xxxx.yaml up humbub -> let installation run, but do not log in
1. docker compose -f xxxx.yaml down -> stop humhub container
1. remove the humhub-data directory (not necessary?)
1. mv the /tmp/humhub-data to working directory (replacing humhub-data)
1. some suggest chown all files in humhub-data, but it did not need it
1. `docker compose -f xxxx.yaml up db` -> bring up mariadb
1. use `./helper.sh import-db path/to/sql/dump.sql` to restore mariadb
1. `docker compose -f xxxx.yaml up -d` -> should be running after a minute or two

### Clone from a running system - might help?
1. assuming vps1 and vps2 - bring up vps2 as clone of vps1 (gets docker etc for free)
1. docker compose down on vps1
1. use tar to backup all of humhub-data to /tmp like this (from within `humhub-docker-dev`)
```
sudo tar -czvf /tmp/humhub-data.tar.gz humhub-data
``` 
4. `docker compose -f xxx.yaml up db -d` (bring the db up)
1. use `./helper.sh db-export path/to/export/location` to extract the db to an sql file

####

6. set up ~/.ssh/config on the home machine to point to the two vps addresses, like this:
```
Host vps1
  HostName x.x.x.x
  User ubuntu
  IdentityFile ~/.ssh/xxxxxxx.pem
  IdentitiesOnly yes

Host vps2
  Hostname y.y.y.y
  User ubuntu
  IdentityFile ~/.ssh/xxxxxxx.pem
  IdentitiesOnly yes
```
7. use scp "third party" mode (-3) to copy the backup files from vps1 to vps2 backups folder
```
scp -3 vps1:/tmp/dump.sql vps2:/home/ubuntu//humhub-projects/humhub-docker-dev/backups/
scp -3 vps1:/tmp/humhub-data.tar.gz vps2:/home/ubuntu//humhub-projects/humhub-docker-dev/backups/

```
8. Bring down the containers `docker compose -f xxx.yaml down`
1. replace the old humhub-data with the new data from backup
```
sudo rm -rf humhub-data # (get rid of all old humhub data)
tar -xzvf ./backups/humhub-data.tar.gz -C backups # (I think the directory must exist?)
mv backups/humhub-data .

```
10. Bring db up and use helper script to import the sql data
```
docker compose -f xxx.yaml up db -d
./helper.sh import-db backups/testdbbu.sql
```
1. restart containers



## Other notes
1. If you point an existing site to a new VPS, and remap the static IP, you still have to ensure that the caddy files are correctly pointing to the inbound expected DNS name.  There won't be a proper cert on the new VPS even if the caddyfile is correct.  I had to take the container down, brink up caddy alone to let it allocate a new cert, then bring everything back up.  Also true of any SSH connections that pointed to the old instance.  Even though the IP address moved to a new instance, SSH will refuse to connect since the host keys won't match that IP address.  Some useful commands:
```
docker compose up -d caddy
docker compose logs -f caddy
and
ssh-keygen -f "/home/david/.ssh/known_hosts" -R "ip.that.points.differentlynow"
```
2. Difference between an `A` record and `CNAME`.  The `A` record is part of my registered domain, but I can point it directly to any IP address, so `test.mfs.com` can point to any vps address I want.  A `CNAME` on the other hand points is just an alias to any `A` record that I have created, so `humhub.mfs.com` points to `mfs.com` and requires a double DNS lookup to be resolved.  All of them are part of my paid-for domain registration.
