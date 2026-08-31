# Steg 1 — Skapa den nya platsen för alla nya filer som kommer
```
mkdir -p /media/miska/Spomini/downloads/complete
mkdir -p /media/miska/Spomini/downloads/incomplete
```

# Steg 2 — Stoppa stacken tillfälligt
## för att tillfärlligt kunna göra ändrigar i docker copouse

```
cd ~/Projects/docker-services/media-stack
docker compose down
```

# Step 3 - Uppdatera docker-compose.yml
## detta är vart alla saker är sparade i docker filen som tar hand om alla saker som man stoppar in
```
nano docker-compose.yml
```

då kommer man att få en sådan fil upp med alla sina konfigurationer

```
version: "3.8"

x-common: &common
  restart: unless-stopped
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=Europe/Stockholm

services:

  qbittorrent:
    <<: *common
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Stockholm
      - WEBUI_PORT=8080
    volumes:
      - /media/miska/Spomini/config/qbittorrent:/config
      - /media/miska/Spomini/jellyfin/movies:/downloads
    ports:
      - 8081:8080
      - 6881:6881
      - 6881:6881/udp

  prowlarr:
    <<: *common
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    volumes:
      - /media/miska/Spomini/config/prowlarr:/config
    ports:
      - 9696:9696

  radarr:
    <<: *common
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    volumes:
      - /media/miska/Spomini/config/radarr:/config
      - /media/miska/Spomini/jellyfin/movies:/movies
      - /media/miska/Spomini/jellyfin/movies:/downloads
    ports:
      - 7878:7878
    depends_on:
      - qbittorrent
      - prowlarr

  bazarr:
    <<: *common
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    volumes:
      - /media/miska/Spomini/config/bazarr:/config
      - /media/miska/Spomini/jellyfin/movies:/movies
    ports:
      - 6767:6767
    depends_on:
      - radarr

  overseerr:
    <<: *common
    image: fallenbagel/jellyseerr:latest
    container_name: overseerr
    volumes:
      - /media/miska/Spomini/config/overseerr:/app/config
    ports:
      - 5055:5055
    depends_on:
      - radarr
```



# Steg 4 — Flytta befintligt innehåll från gamla platsen (om något ligger kvar)
```
mv /media/miska/Spomini/jellyfin/movies/downloaded/* /media/miska/Spomini/downloads/complete/ 2>/dev/null
mv /media/miska/Spomini/jellyfin/movies/pending/* /media/miska/Spomini/downloads/incomplete/ 2>/dev/null
rmdir /media/miska/Spomini/jellyfin/movies/downloaded
rmdir /media/miska/Spomini/jellyfin/movies/pending
```



# Steg 5 — Starta upp stacken igen

```
docker compose up -d
```
