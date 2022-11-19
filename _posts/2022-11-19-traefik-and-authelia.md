---
title: Securing Traefik with Authelia
---

(Refer to this [post](https://vishnurajeevan.com/traefik/) for a bare setup.

```yaml
version: "3"
services:
  traefik:
    container_name: traefik
    command:
      #- "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=proxy"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=postmaster@example.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    environment:
      - log.filepath=/var/log/traefik.log
      - TZ=America/New_York
    image: traefik:v2.9
    labels:
      traefik.enable: true
      traefik.http.middlewares.redirect-to-https.redirectscheme.scheme: https
      traefik.http.routers.http-catchall.entrypoints: web
      traefik.http.routers.http-catchall.middlewares: redirect-to-https
      traefik.http.routers.http-catchall.rule: HostRegexp(`{host:.+}`)
    ports:
      - 444:443/tcp
      - 81:80/tcp
      - 8081:8080/tcp
    networks:
      - proxy
      - backend
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /mnt/user/appdata/traefik/letsencrypt:/letsencrypt:rw
      - /mnt/user/appdata/traefik/config/traefik.toml:/etc/traefik/traefik.toml:rw
  authelia:
    container_name: authelia
    depends_on:
      - traefik
      - redis
    environment:
      - TZ=America/New_York
    image: authelia/authelia
    labels:
      traefik.enable: true
      traefik.http.middlewares.authelia.forwardauth.address: http://192.168.86.40:9091/api/verify?rd=https://login.example.com//
      traefik.http.middlewares.authelia.forwardauth.authResponseHeaders: 'Remote-User,
        Remote-Groups'
      traefik.http.middlewares.authelia.forwardauth.trustForwardHeader: true
      traefik.http.routers.authelia.entrypoints: websecure
      traefik.http.routers.authelia.rule: Host(`login.example.com`)
      traefik.http.routers.authelia.tls: true
      traefik.http.routers.authelia.tls.certresolver: letsencrypttls
    networks:
      - backend
    ports:
      - 9091:9091/tcp
    volumes:
      - /mnt/user/appdata/authelia/:/var/lib/authelia:rw
      - /mnt/user/configs/authelia/configuration.yml:/config/configuration.yml:ro
      - /mnt/user/configs/authelia/users_database.yml:/etc/authelia/users_database.yml:rw
      - /dev/rtc:/dev/rtc:ro
    working_dir: /app
  redis:
    container_name: redis
    environment:
      - TZ=America/New_York
      - HOST_OS=Unraid
      - GOSU_VERSION=1.12
    image: redis
    networks:
      - backend
    ports:
      - 6379:6379/tcp
    working_dir: /data
networks:
  proxy:
    external: true
  backend:
```
