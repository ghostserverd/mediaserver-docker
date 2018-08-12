# Automated Media Server
---
### About 
This is an automated media server set up in docker containers via docker-compose. This goal of this setup is to automate as much of the installation and configuration as possible with sane defaults.

The end result of this setup is a media server with the following components
: `Plex`
: `qBittorrent`
  : with `filebot` binary
  : auto-configured to run `filebot` on torrent completion
: `sonarr`
: `radarr`
: `jackett`

The high-level steps for setup are as follows
1. Install docker and docker-compose
1. `docker-compose up`
1. add an indexer to `jackett`
1. configure `sonarr` / `radarr` to use the `jackett` indexer
1. configure `sonarr` / `radarr` to recognize `qbittorrent` as their download client
1. add libraries to `plex`

Once these steps have been completed, it is possible to add a TV Show or Movie to `sonarr` / `radarr`, have it automatically download them when available, and then it will be automatically copied into your `plex` libraries. Post-installation, this should be a fully-automated media server (the exception to this is removing completed downloads).

### Network
Each service is available on its own ports:
- `qBittorrent` : `8080`
- `sonarr` : `8989`
- `radarr` : `7878`
- `jackett` : `9117`
- `plex` : `32400/web`

The services are all running in `network=host` mode so they can see each other without having to do extra port mapping.

### Installation
##### Install Docker
  - https://docs.docker.com/install/#supported-platforms
- Install Docker Compose
  - https://docs.docker.com/compose/install/#install-compose

##### Clone this repo
```
git clone https://github.com/ghostserverd/mediaserver-docker.git
cd mediaserver-docker
```

##### Build your `.env` file
  - TBD - see example

##### Deploy the service
```
docker-compose up
```

##### Configure Jackett
`<server-ip>:9117`

##### Configure Sonarr
`<server-ip>:8989`

##### Configure Radarr
`<server-ip>:7878`

##### Configure qBittorrent
`<server-ip>:8080`
- Just kidding, this is already configured for you

##### Configure Plex
`<server-ip>:32400/web`
- Add some libraries
- Set up your media agents to not use local files (hopefully fixable in the future)
- Kill and restart the containers after logging in to Plex if you have Plexpass
