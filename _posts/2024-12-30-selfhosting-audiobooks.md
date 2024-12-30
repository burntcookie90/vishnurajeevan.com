---
title: Self Hosting Your Audiobooks
---

4 years ago I made the switch from Audible to Libro.fm. Libro.fm serves DRM free audiobooks, shares revenue with a local bookshop you select and works on a similar monthly subscription/credit system to Audible. 

However, one issue arises: how I listen to the books? The official mobile app is good, but part of owning DRM free files is storing/archiving them locally. To start I had setup a Plex audiobook library but ran into a lot of the edges of running a "non-standard" library with Plex. Things like metadata, cover images and official app support are either hacky or manually managed. I was willing to do deal with it because the third party [app](https://prologue.audio/) I was using was so good. Eventually, the manual work with Plex became overwhelming and I felt the need to switch to something more purpose built.

[Audiobookshelf](https://www.audiobookshelf.org/) ended up being the choice I went with. Solid metadata and management features as well as OpenID Connect authentication. Plus its an easy service to run:

```yaml
services:
  audiobookshelf:
    image: ghcr.io/advplyr/audiobookshelf:latest
    volumes:
      - /mnt/user/media/audiobooks:/audiobooks
      - /mnt/user/media/books:/books
      - /mnt/user/media/comics:/comics
      - /mnt/runtime/appdata/audiobookshelf/config:/config
      - /mnt/runtime/appdata/audiobookshelf/metadata:/metadata
    labels:
      traefik.enable: true
      traefik.http.routers.abs.entrypoints: websecure
      traefik.http.routers.abs.rule: Host(`<>`)
    networks:
      - proxy
    environment:
      - user=99:100
networks:
  proxy:
    external: true
```

I've been using a third party [app](https://github.com/LeoKlaus/plappa) because the official iOS app has a full testflight. This setup has worked _really well_ but there was still one bit of friction with the workflow:

1. Purchase book from libro.fm
2. Download files via browser
3. Unzip files
4. Upload files to the server. Used to be via samba, but with audiobookshelf I was able to upload via the webapp.

I have almost 200 audiobooks, and this year I'd added 15+ so this got annoying very quickly. 

I saw [this issue](https://github.com/advplyr/audiobookshelf/issues/2112) and agreed that the enhancement is out of scope for ABS, but [this comment](https://github.com/advplyr/audiobookshelf/issues/2112#issuecomment-1866724546) made me realize I could just write my own service to handle the responsibility. So, [I did](https://github.com/burntcookie90/librofm-downloader).

```yaml
services:
  librofm-downloader:
    image: ghcr.io/burntcookie90/librofm-downloader:latest
    volumes:
      - /mnt/runtime/appdata/librofm-downloader:/data
      - /mnt/user/media/audiobooks:/media
    ports:
      # optional if you want to use the /update webhook
      - 8080:8080 
    environment:
      - LIBRO_FM_USERNAME=<>
      - LIBRO_FM_PASSWORD=<>
      # extra optional: setting these enables them, dont add them if you dont want them.
      - DRY_RUN=true 
      - VERBOSE=true
      - RENAME_CHAPTERS=true
      - WRITE_TITLE_TAG=true #this one requires RENAME_CHAPTERS to be true as well
```

Give it a shot! 