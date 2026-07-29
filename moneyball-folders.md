# Moneyball NAS Folder Structure
Here is an overview of the folder structure on my NAS 
/volume1
    docker/
        downloads.yml
        code-server.yml
        infrastructure.yml
        network.yml
        appdata/
            adguard/
            code-server/
            glance/
            gluetun/
            jellyfin/
            jellyseerr/
            npm/
            postgres/
            prowlarr/
            qbittorrent/
            radarr/
            radarr-4k/
            sonarr/
            sonarr-4k/
            sonarr-anime/
            tailscale/
        config/ (believe this is just for the code-server)
/volume2
    docker/
        data/
        media/
            movies/
            movies-4k/
            tv/
            tv-4k/
            tv-anime/
        torrents/

Currently all of the docker containers are being ran from volume1 which is on an m.2 
volume2 is where all of the media and torrents are.