# Automated Media Server
---
## About 
This is an automated media server set up in docker containers via docker-compose. This goal of this setup is to automate as much of the installation and configuration as possible with sane defaults.

The end result of this setup is a media server with the following components
- `plex`
- `qBittorrent`
  - with `filebot` binary
  - auto-configured to run `filebot` on torrent completion
- `sonarr`
- `radarr`
- `jackett`

The high-level steps for setup are as follows
1. Install docker and docker-compose
1. `docker-compose up`
1. add an indexer to `jackett`
1. configure `sonarr` / `radarr` to use the `jackett` indexer
1. configure `sonarr` / `radarr` to use `qbittorrent` as their download client
1. add libraries to `plex`

Once these steps have been completed, it is possible to add a TV Show or Movie to `sonarr` / `radarr`, have it automatically download them when available, and then it will be automatically copied into your `plex` libraries. Post-installation, this should be a fully-automated media server (the exception to this is removing completed downloads).

## Network
Each service is available on its own ports:
- `qBittorrent` : `8080`
- `sonarr` : `8989`
- `radarr` : `7878`
- `jackett` : `9117`
- `plex` : `32400/web`

The services are all running in `network=host` mode so they can see each other without having to do extra port mapping.

## Installation
#### Install Docker
  - https://docs.docker.com/install/#supported-platforms

#### Install Docker Compose
  - https://docs.docker.com/compose/install/#install-compose

#### Clone this repo
```
git clone https://github.com/ghostserverd/mediaserver-docker.git
cd mediaserver-docker
```

#### Build your `.env` file
```
cp .env_sample .env
id $USER # save the result of this for building your .env file below
```

Modify the `.env` file to specify the following configurations. Note that these directories are for the host machine. They are mapped to various locations inside of their container by the `docker-compose` file.

- `CONFIG_DIR` is where configuration for each of the services will live. You'll end up with multiple directories in here, one for each service. If you move this directory to a different volume with a different instance of the whole media server, it should retain your various configurations.

- `DOWNLOAD_DIR` is where various services will download files to. ⚠️ This should not be on a small partition as it will contain media files.

- `MEDIA_DIR` is where your media will be copied to. ⚠️ This should not be on a small partition as it will contain media files.

- `TV_DIR` is where your TV shows will be placed by `filebot` on download completion. It should be a subdirectory of `MEDIA_DIR`

- `MOVIES_DIR` is where your TV shows will be placed by `filebot` on download completion. It should be a subdirectory of `MEDIA_DIR`

- `QBIT_WEBUI_USER` is the Web UI user for `qBittorrent`. The default is `admin`.

- `QBIT_WEBUI_PASS` is the Web UI password for `qBittorrent`. The default is `adminadmin`.

- `PUID` is the unix `UID` that will be passed to the various services. It can be discovered by running `id $USER` on the host machine as mentioned above.

- `PGID` is the unix `GID` that will be passed to the various services. It can be discovered by running `id $USER` on the host machine as mentioned above.

#### Deploy the service
```
docker-compose up
```

## Configuration
#### Configure Jackett
`<server-ip>:9117`

#### Configure Sonarr
`<server-ip>:8989`

#### Configure Radarr
`<server-ip>:7878`

#### Configure qBittorrent
`<server-ip>:8080`
- This is mostly already configured. It automatically has configuration for `filebot` download completion handling, the username / password you set in your `.env`, and sets your download directory to `/downloads/`. If you want other `.env` configurations to be available for `qBittorrent`, open an issue here.

#### Configure Plex
`<server-ip>:32400/web`
- Add some libraries
  - TV Shows will be at `/data/TV Shows` assuming you followed the `/media/TV Shows` convention for `TV_DIR`
  - Movies will be at `/data/Movies` assuming you followed the `/media/Movies` convention for `MOVIES_DIR`
- ⚠️ Set up your media agents to not use local files (hopefully this will be fixed in the future)
  - There is a problem that I have not been able to fix yet where local TV Series art is not available. It appears the artwork downloaded by `filebot` is not readable by `plex` for an unkown reason (I don't believe it's permissions related, but if you have ideas, please open an issue).
  - To get around this, uncheck `Local Media Assets` for all Agents under `Settings` > `Server` > `Agents`. Artwork will be downloaded by plex and accessible.
- Kill and restart the containers after logging in to Plex if you have Plexpass
```
docker-compose down
docker-compose up
```
