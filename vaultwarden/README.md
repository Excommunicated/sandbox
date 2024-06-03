# Vaultwarden on Docker With Cloudflare Tunnel and Backups

All of these steps should work no matter where the Docker host, But in this example will be using an LXC container on Proxmox

## Create the LXC container
Using the [Proxmox Helper Script](https://helper-scripts.com/scripts?id=Docker), deploy the container.

## Create the Docker compose 
Using the following `docker-compose.yml`
```yaml
version: '3.4'
services:

  vaultwarden:
    container_name: vaultwarden
    image: vaultwarden/server:latest
    restart: unless-stopped
    environment:
      - DOMAIN=${VW_DOMAIN}
      - SIGNUPS_ALLOWED=false
      - ADMIN_TOKEN=${VW_ADMIN_TOKEN}
      - YUBICO_CLIENT_ID=${VW_YUBICO_CLIENT_ID}
      - YUBICO_SECRET_KEY=${VW_YUBICO_SECRET_KEY}
      - SMTP_HOST=${VW_SMTP_HOST}
      - SMTP_FROM=${VW_SMTP_FROM}
      - SMTP_USERNAME=${VW_SMTP_USERNAME}
      - SMTP_PASSWORD=${VW_SMTP_PASSWORD}
    ports:
      - 8200:80
    volumes:
      - vaultwarden-data:/data/
    networks:
      - vaultwarden-network

  backup:
    container_name: vaultwarden-backup
    image: ttionya/vaultwarden-backup:latest
    restart: always
    environment:
      - BACKUP_KEEP_DAYS=30
      - BACKUP_FILE_SUFFIX=%Y%m%d%H%M
      - MAIL_SMTP_ENABLE=TRUE
      - MAIL_SMTP_VARIABLES=${BACKUP_SMTP_VARIABLES}
      - MAIL_TO=${BACKUP_MAIL_TO}
      - MAIL_WHEN_SUCCESS=FALSE
      - MAIL_WHEN_FAILURE=TRUE
      - ZIP_PASSWORD=${BACKUP_ZIP_PASSWORD}
    volumes:
      - vaultwarden-data:/bitwarden/data/
      - vaultwarden-rclone-data:/config/
    networks:
      - vaultwarden-network

  cloudflared:
    image: cloudflare/cloudflared:2024.5.0
    container_name: vaultwarden-cloudflared
    restart: unless-stopped
    read_only: true
    volumes:
      - ./cloudflared-config:/root/.cloudflared/
    command: [ "tunnel", "run", "${TUNNEL_ID}" ]
    user: root
    depends_on:
      - vaultwarden
    networks:
      - vaultwarden-network

volumes:
  vaultwarden-data:
    name: vaultwarden-data
  vaultwarden-rclone-data:
    external: true
    name: vaultwarden-rclone-data
    
networks:
  vaultwarden-network:
    name: vaultwarden-network
    external: false
```

You then will need to create a `.env` file in the same folder as the `docker-compose.yml`. Inside you will need the following variables:
```ini
# Cloudflare
TUNNEL_ID=aaaa-bbbb-cccc-dddddddddddd

# Vaultwarden
VW_DOMAIN='https://some.domain.com'
VW_ADMIN_TOKEN=''
VW_YUBICO_CLIENT_ID=12345
VW_YUBICO_SECRET_KEY='secret'
VW_SMTP_HOST=smtp.mail.com
VW_SMTP_FROM='admin@domain.com'
VW_SMTP_USERNAME='admin@domain.com'
VW_SMTP_PASSWORD='fake-password'

# Backup
BACKUP_SMTP_VARIABLES='-S smtp-use-starttls -S smtp=smtp://smtp.mail.com:587 -S smtp-auth=login -S smtp-auth-user=admin@domain.com -S smtp-auth-password=fake-password -S from=admin@domain.com'
BACKUP_MAIL_TO='someone@mail.com'
BACKUP_ZIP_PASSWORD='CHANGEME!'
```

## Setup Rclone
See details [here](https://github.com/ttionya/vaultwarden-backup?tab=readme-ov-file#configure-and-check)
```bash
docker run --rm -it \
  --mount type=volume,source=vaultwarden-rclone-data,target=/config/ \
  ttionya/vaultwarden-backup:latest \
  rclone config
```

## Cloudflare Tunnel Setup
Copied from [Vaultwarden wiki - Proxy Examples - Cloudflare Tunnel](https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples)

Contents in `cloudflared-config` folder:
```
config.yml  
aaaaa-bbbb-cccc-dddd-eeeeeeeee.json
```
Please use [this](https://thedxt.ca/2022/10/cloudflare-tunnel-with-docker/) guide to figure out the contents/values below for your cloudflare account.
Note: `aaaaa-bbbb-cccc-dddd-eeeeeeeee` is just a random tunnelID please use a real ID.

`config.yml`:

```yaml
tunnel: aaaaa-bbbb-cccc-dddd-eeeeeeeee
credentials-file: /root/.cloudflared/aaaaa-bbbb-cccc-dddd-eeeeeeeee.json

originRequest:
  noHappyEyeballs: true
  disableChunkedEncoding: true
  noTLSVerify: true

ingress:
  - hostname: vault.example.com # change to your domain
    service: http_status:404
    path: admin
  - hostname: vault.example.com # change to your domain
    service: http://vaultwarden
  - service: http_status:404
```
`aaaaa-bbbb-cccc-dddd-eeeeeeeee.json`:
```json
{
    "AccountTag":"changeme",
    "TunnelSecret":"changeme",
    "TunnelID":"aaaaa-bbbb-cccc-dddd-eeeeeeeee"
}
```

## Setting up the Backup
Using [this](https://github.com/ttionya/vaultwarden-backup?tab=readme-ov-file#configure-and-check), Setup the rclone backup prior to starting the container for the first time.

## Running
`docker compose up -d`

## Restore
See details [here](https://github.com/ttionya/vaultwarden-backup?tab=readme-ov-file#configure-and-check)
```bash
docker run --rm -it \
  --mount type=volume,source=vaultwarden-data,target=/bitwarden/data/ \
  --mount type=bind,source=$(pwd),target=/bitwarden/restore/ \
  ttionya/vaultwarden-backup:latest restore \
  --zip-file backup.zip
```

