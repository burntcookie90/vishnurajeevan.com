---
title: Self Hosting Bluesky PDS with Docker-Compose and Traefik
---

**Update 11/26/2024**: Added `PDS_DATA_DIRECTORY` environment variable to compose.yml. The `installer.sh` script in the pds repo unusually aliases `PDS_DATADIR` to this for actually binding the volume.

(This assumes you have a running traefik instance, and working domain/dns)


`docker-compose.yml`

```yaml
services:
  pds:
    container_name: pds
    image: ghcr.io/bluesky-social/pds:0.4
    restart: unless-stopped
    volumes:
      - ./bluesky/pds:/pds
    networks:
      - proxy
    labels:
      traefik.enable: true
      traefik.http.routers.bluesky-pds.rule: Host(`example.com`)
      traefik.http.routers.bluesky-pds.entrypoints: websecure
      traefik.http.routers.bluesky-pds.tls.certresolver: letsencrypttls
      traefik.http.services.bluesky-pds.loadbalancer.server.port: 3000
    environment:
      - PDS_DATADIR=/pds
      - PDS_DATA_DIRECTORY=/pds
      - PDS_BLOBSTORE_DISK_LOCATION=/pds/blocks
      - PDS_HOSTNAME= #example.com
      - PDS_JWT_SECRET= #openssl rand --hex 16
      - PDS_ADMIN_PASSWORD= #openssl rand --hex 16
      - PDS_PLC_ROTATION_KEY_K256_PRIVATE_KEY_HEX= #openssl ecparam --name secp256k1 --genkey --noout --outform DER | tail --bytes=+8 | head --bytes=32 | xxd --plain --cols 32
      - PDS_EMAIL_SMTP_URL= #smtp://username@gmail.com:password@smtp.gmail.com:587
      - PDS_EMAIL_FROM_ADDRESS= #admin@domain.com
      - PDS_MODERATION_EMAIL_SMTP_URL= #smtp://username@gmail.com:password@smtp.gmail.com:587
      - PDS_MODERATION_EMAIL_ADDRESS= #admin@domain.com
      - PDS_DID_PLC_URL=https://plc.directory
      - PDS_BSKY_APP_VIEW_URL=https://api.bsky.app
      - PDS_BSKY_APP_VIEW_DID=did:web:api.bsky.app
      - PDS_REPORT_SERVICE_URL=https://mod.bsky.app
      - PDS_REPORT_SERVICE_DID=did:plc:ar7c4by46qjdydhdevvrndac
      - PDS_CRAWLERS=https://bsky.network
      - LOG_ENABLED=true
networks:
  proxy:
    external: true
```

next run the following to generate an invite:

```bash
curl --request POST \
  --url https://example.com/xrpc/com.atproto.server.createInviteCode \
  --header 'Authorization: Basic <token>' \
  --header 'Content-Type: application/json' \
  --data '{
  "useCount": 1
}'
```

Then you should be able to sign up on https://bsky.app with your $PDS_HOSTNAME as the hosting provider.

If you're looking for mastodon check [here]({% post_url 2022-11-03-self-hosting-mastodon-with-docker-compose-and-traefik %}).

If you're looking for setting up traefik check [here]({% post_url 2022-11-19-traefik %}).


Used [https://therobbiedavis.com/selfhosting-bluesky-with-docker-and-swag/](https://therobbiedavis.com/selfhosting-bluesky-with-docker-and-swag/) as a resource for this.

<style>
  :root {
    color-scheme: dark;
  }
  bsky-comments {
    --background-color: none;
    --text-color: rgba(255,255,255,0.8);
    --link-color: #0077cc;
    --link-hover-color: #005fa3;
    --comment-meta-color: rgba(255,255,255,0.5);
    --error-color: #ff4d4d;
    --reply-border-color: rgba(255,255,255,0.1);
    --button-background-color: rgba(255,255,255,0.1);
    --button-hover-background-color: rgba(255,255,255,0.2);
    --author-avatar-border-radius: 50%;
  }
</style>
<script type="module" src="https://esm.sh/gh/loueed/bsky@v1.0.0/comments"></script>
<bsky-comments post="at://did:plc:utqsoejrgbeaczfmiepczpax/app.bsky.feed.post/3lbulpxnenc2t"></bsky-comments>


Comments via: [https://github.com/LoueeD/bsky](https://github.com/LoueeD/bsky)