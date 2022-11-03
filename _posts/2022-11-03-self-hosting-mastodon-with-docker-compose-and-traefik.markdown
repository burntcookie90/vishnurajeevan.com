---
layout: post
title: Self Hosting Mastodon with Docker-Compose and Traefik
date: November 03, 2022
---

(This assumes you have a running traefik instance)

Use `.env.production` from [github](https://github.com/mastodon/mastodon/blob/main/.env.production.sample).

Things to edit:
```bash
LOCAL_DOMAIN=example.com
WEB_DOMAIN=mastodon.example.com #This is optional and only needed if your instance is hosted on a subdomain, but you want the instance to be known by its domain.
SINGLE_USER_MODE=true #Optional, only needed if you're hosting for just yourself

# Sending mail
# ------------
SMTP_SERVER=<smtp host>
SMTP_PORT=587
SMTP_LOGIN=<username>
SMTP_PASSWORD=<password>
SMTP_FROM_ADDRESS=<address the mastodon instance will email from>
```

Move `PostgreSQL` block to a `db.env` file so you can share with the postgres container.

`db.env`
```bash
# PostgreSQL
# There might be some fancy way to de-dupe this, i just sent it.
# ----------
DB_HOST=db
DB_USER=mastodon
DB_NAME=mastodon_production
DB_PASS=<generated password>
DB_PORT=5432

# Needed so that the pgsql container creates the expected roles and dbs for you
POSTGRES_DB=mastodon_production
POSTGRES_USER=mastodon
POSTGRES_PASSWORD=<generated password>
```

`docker-compose.yml` (sourced from [github](https://github.com/mastodon/mastodon/blob/main/docker-compose.yml) and edited)
```yaml
version: '3'
services:
  db:
    restart: always
    image: postgres:14-alpine
    shm_size: 256mb
    networks:
      - internal_network
    volumes:
      - /mnt/user/appdata/mastodon/postgres14:/var/lib/postgresql/data
    # added from the default, otherwise you'll need to manually create a `mastodon` db and user/role
    env_file: db.env  
    environment:
      - 'POSTGRES_HOST_AUTH_METHOD=trust'
  redis:
    restart: always
    image: redis:7-alpine
    networks:
      - internal_network
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
    volumes:
      - /mnt/user/appdata/mastodon/redis:/data

  web:
    image: tootsuite/mastodon
    restart: always
    env_file:
      - .env.production
      - db.env
    command: bash -c "rm -f /mastodon/tmp/pids/server.pid; bundle exec rails s -p 3000"
    networks:
      - proxy
      - internal_network
    healthcheck:
      # prettier-ignore
      test: ['CMD-SHELL', 'wget -q --spider --proxy=off localhost:3000/health || exit 1']
    depends_on:
      - db
      - redis
    volumes:
      - /mnt/user/appdata/mastodon/public/system:/mastodon/public/system
    labels:
      - traefik.enable=true
      - traefik.http.routers.mastodonweb.rule=Host(`example.com`)
      - traefik.http.routers.mastodonweb.entrypoints=<https-entry-point>
      - traefik.http.routers.mastodonweb.tls.certresolver=letsencrypttls
      - traefik.http.services.mastodonweb.loadbalancer.server.port=3000

  streaming:
    image: tootsuite/mastodon
    restart: always
    env_file:
      - .env.production
      - db.env
    command: node ./streaming
    networks:
      - proxy
      - internal_network
    healthcheck:
      # prettier-ignore
      test: ['CMD-SHELL', 'wget -q --spider --proxy=off localhost:4000/api/v1/streaming/health || exit 1']
    depends_on:
      - db
      - redis
    labels:
      - traefik.enable=true
      - traefik.http.routers.mastodonstreaming.rule=(Host(`example.com`) && PathPrefix(/api/v1/streaming))
      - traefik.http.routers.mastodonstreaming.entrypoints=<https-entry-point>
      - traefik.http.routers.mastodonstreaming.tls.certresolver=letsencrypttls
      - traefik.http.services.mastodonstreaming.loadbalancer.server.port=4000


  sidekiq:
    image: tootsuite/mastodon
    restart: always
    env_file:
      - .env.production
      - db.env
    command: bundle exec sidekiq
    depends_on:
      - db
      - redis
    networks:
      - proxy
      - internal_network
    volumes:
      - /mnt/user/appdata/mastodon/public/system:/mastodon/public/system
    healthcheck:
      test: ['CMD-SHELL', "ps aux | grep '[s]idekiq\ 6' || false"]


networks:
  #the network that your traefik instance has access to
  proxy:
    external: true
  internal_network:
    internal: true
```

Run the following:

```bash
docker-compose run --rm web rake secret  #copy to SECRET_KEY_BASE in .env.production
docker-compose run --rm web rake secret  #copy to OTP_SECRET in .env.production
docker-compose run --rm web bundle exec rake mastodon:webpush:generate_vapid_key #copy to VAPID_* keys
docker-compose -f run --rm web bundle exec rake db:migrate
docker-compose run --rm web bin/tootctl accounts create <username> --email=<email>
docker-compose run --rm web bin/tootctl accounts modify <username> --confirm --approve --enable --role=admin
```

### __draw the owl section__:
Mastodon needs a [webfinger](https://docs.joinmastodon.org/spec/webfinger/) to resolve links across the fediverse.
Setting this up will be different based on your infrastructure. I ended up using netlify to host my jekyll website and setup redirects via that.

`_redirects`
```
/.well-known/webfinger	https://$(WEB_DOMAIN)/.well-known/webfinger
```

`_config.yml`
```
include:
  - '_redirects'
```

On Netlify make sure your bare domain is the primary domain, and that Netlify is serving your SSL certs via Let's Encrypt.

On my domain registrar, I've setup an `A` Record pointing my `WEB_DOMAIN` subdomain to my instance's IP address.


### Test webfinger

Test that your instance's web finger is setup right.

```bash
curl -vL https://example.com/.well-known/webfinger?resource=acct:@username@example.com
```

Will return json with redirects.

```
* Connection #1 to host mastodon.example.com left intact
{"subject":"acct:username@example.com","aliases":["https://mastodon.example.com/@username","https://mastodon.example.com/users/username"],"links":[{"rel":"http://webfinger.net/rel/profile-page","type":"text/html","href":"https://mastodon.example.com/@username"},{"rel":"self","type":"application/activity+json","href":"https://mastodon.example.com/users/username"},{"rel":"http://ostatus.org/schema/1.0/subscribe","template":"https://mastodon.example.com/authorize_interaction?uri={uri}"}]}%
```