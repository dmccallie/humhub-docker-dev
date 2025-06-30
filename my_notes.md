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

## how to restore - coming soon

