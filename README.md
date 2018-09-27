# Automated Media Server 👻
```
         _               _
    __ _| |__   ___  ___| |_
   / _` | '_ \ / _ \/ __| __/
  | (_| | | | | (_) \__ \ |_
   \__, |_|_|_|\___/|___/\__|
   |___/      /   _ \
          (¯\| o (@) |/¯)
           \_  .___.  _/
            /   !_!   \
           /_.--._.--._\
```
---
# About 
This is an automated media server set up in docker containers via docker-compose. The goal of this project is to automate as much of the installation and configuration as possible.

The end result of this setup is a media server with the following components
- `plex`
- `qBittorrent`
  - with `filebot` binary
  - configured to run `filebot` on torrent completion
- `transmission`
  - with `filebot` binary
  - configured to run `filebot` on torrent completion
- `sonarr`
- `radarr`
- `jackett`

The high-level steps for setup are as follows
1. Install docker and docker-compose
1. `docker-compose up`
1. add an indexer to `jackett`
1. configure `sonarr` / `radarr` to use the `jackett` indexer
1. configure `sonarr` / `radarr` to use `qbittorrent` or `transmission` as their download client
1. add libraries to `plex`

Once these steps have been completed, it is possible to add a TV Show or Movie to `sonarr` / `radarr`, have it automatically download them when available, and then it will be automatically copied into your `plex` libraries. Post-installation, this should be a fully-automated media server (the exception to this is removing completed downloads).

📓 This has only been tested on Ubuntu 18.04 LTS but it should work just fine on other linux distros. MacOS and Windows are unsupported. If you test it on MacOS or Windows and it works, let me know!

# ⚠️ A Note About Torrent Clients
Before we begin, it has been observed in the past few weeks that ratios on new torrents (mostly TV shows) have been much lower than usual since switching to `qbittorrent`. Towards isolating the issue, `transmission` has been added as an option for torrent client. The `.env_sample` has been updated to reflect new config options, and the `docker-compose.yml` file has a new section in it for `transmission`. It is currently commented out, with `qbittorrent` being left as the default for now. Once more testing has been performed, `qbittorrent` may be replaced alltogether with `transmission`.

For now, if you wish to continue using `qbittorrent`, nothing needs to be changed in your setup. If you wish to switch to `transmission`, modify the `docker-compose.yml` file to uncomment the `transmission` service block, and comment out the `qbittorrent` block. You will have to modify your configuration in `sonarr` to point to `transmission` instead of `qbittorrent`, but you should be able to follow the same basic pattern in this README.

# Network
Each service is available on its own ports:

| Service | Port |
| ------- | ---- |
| qbittorrent | 6767 |
| transmission | 5656 |
| sonarr | 8989 |
| radarr | 7878 |
| jackett | 9117 |
| plex | 32400 |

📓To reach `plex`, append `/web` to the address e.g. `192.168.1.11:32400/web`

All of the services except for `plex` are running in the default docker-compose network. From within services, they can access each other via their `<service_name>:<port>` as defined in `docker-compose.yml`.

`transmission:5656`

`qbittorrent:6767`

`sonarr:8989`

`radarr:7878`

`jackett:9117`

# Installation
## Install Docker
  - https://docs.docker.com/install/#supported-platforms

## Install Docker Compose
  - https://docs.docker.com/compose/install/#install-compose

## Clone this repo
```
git clone https://github.com/ghostserverd/mediaserver-docker.git
cd mediaserver-docker
```

## Build your `.env` file
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

- `QBIT_WEBUI_USER` is the Web UI user for `qbittorrent`. The default is `admin`.

- `QBIT_WEBUI_PASS` is the Web UI password for `qbittorrent`. The default is `adminadmin`.

- `QBIT_WEBUI_PORT` is the Web UI port for `qbittorrent`. The default is `6767`. ⚠️ This configuration is required.

- `QBIT_CONNECTION_PORT` is the connection port for `qbittorrent`. The default is `6881`. ⚠️ This configuration is required.

- `TRANS_WEBUI_USER` is the Web UI user for `transmission`. The default is `admin`.

- `TRANS_WEBUI_PASS` is the Web UI password for `transmission`. The default is `adminadmin`.

- `TRANS_WEBUI_PORT` is the Web UI port for `transmission`. The default is `6767`. ⚠️ This configuration is required.

- `TRANS_CONNECTION_PORT` is the connection port for `transmission`. The default is `51413`. ⚠️ This configuration is required.

- `PUID` is the unix `UID` that will be passed to the various services. It can be discovered by running `id $USER` on the host machine.

- `PGID` is the unix `GID` that will be passed to the various services. It can be discovered by running `id $USER` on the host machine.

## Deploy the service
```
docker-compose up
```
Append `-d` to run in detached mode. The first time you run it, it is probably a good idea to not run in dettached mode (i.e. DON'T append `-d`) so you can watch all of the logs for issues.

# Configuration
## Configure Jackett
`<server-ip>:9117`

`jackett` should be configurable the same as any other installation of it. Feel free to skip these steps if you know how to configure `jackett` already.

- Add an Indexer
  - Click `+ Add Indexer`
  - Search for your desired tracker
  - Click on the 🔧icon next to your desired tracker
  - Sign in with your account information
  - Click `Okay`
- You should see `Successfully configured IPTorrents` and a new entry for your tracker
- Copy the `API Key` from the top right corner and save it somewhere
- Click the `Copy Torznab Feed` on the tracker you just added and paste it somewhere to save it

## Configure Sonarr
`<server-ip>:8989`

`sonarr` should be configurable the same as any other installation of it. Feel free to skip these steps if you know how to configure `sonarr` already.

⚠️ Do make sure that your download path is set to `/data/completed/tv` as that is the directory that the container has permissions to.

⚠️ It is also critical that you use `qbittorrent` instead of the IP address when configuring the download client, as well as `jackett` instead of the IP when setting up your indexer. This is because this uses docker-compose networking which means each service is accessible at the name of the service, rather than the host IP address.

Step-by-step for those who need it

- Add an Indexer
  - Click on the `Settings` button at the top
  - Click on the `Indexers` tab
  - Click the big `+` symbol
  - Click the `Custom` button in the `Torznab` section
  - Configure your `Torznab` feed
    - `Name` : the name of this indexer (doesn't matter, just name it the name of your tracker)
    - `URL` : the `Copy Torznab Feed` url from `jackett` that you saved earlier
      - ⚠️ It is necessary to replace the IP with `jackett`
        - `http://jackett:9117/api/v2.0/indexers/<indexer>/results/torznab/` instead of 
        - `http://192.168.1.11:9117/api/v2.0/indexers/<indexer>/results/torznab/`
    - `API Key` : the `API Key` from `jackett` that you saved earlier
  - Click `Test` to verify that it is configured properly
  - Click `Save`
  
- Add a Download Client
  - Click on the `Settings` button at the top
  - Click on the `Download Client` tab
  - Under `Completed Download Handling` toggle `Enable` from Yes to No
    - We'll be using `filebot` to handle our completed downloads
  - Click the `Save` button at the top right
  - Click the large `+` button
  - Click on `qBittorrent`
  - Configure your Download Client
    - `Name` : whatever you want; probably `qBittorrent`
    - `Host` : `qbittorrent`
    - `Port` : `6767` or whatever you have set for `QBIT_WEBUI_PORT` in your `.env` file
    - `Username` : `admin` or whatever you have set for `QBIT_WEBUI_USER` in your `.env` file
    - `Password` : `adminadmin` or whatever you have set for `QBIT_WEBUI_PASS` in your `.env` file
  - Click `Test` to verify that it is configured properly
  - Click `Save`
  
- Add Some TV Shows
  - Click on the `Series` button at the top
  - Click `+ Add Series`
  - Start typing in the search bar. It will search automatically when you stop typing
  - Configure the download path (you should only have to do this on the first show you add)
    - Click the dropdown that says `Select Path`
    - Click `Add a different path`
    - Click the 📁button on the right of the modal
    - Click `tv` # final path should be `/tv/`
    - Click `Ok`
    - Click the green ✔️that is now visible
    - Click the `+` sign
- There are many other configuration options for `sonarr` that are not covered here. `sonarr`'s webpage is [here](https://sonarr.tv/)

## Configure Radarr
`<server-ip>:7878`

`radarr` should be configurable the same as any other installation of it. Feel free to skip these steps if you know how to configure `radarr` already.

⚠️ Do make sure that your download path is set to `/data/completed/movies` as that is the directory that the container has permissions to.

⚠️ It is also critical that you use `qbittorrent` instead of the IP address when configuring the download client, as well as `jackett` instead of the IP when setting up your indexer. This is because this uses docker-compose networking which means each service is accessible at the name of the service, rather than the host IP address.

Step-by-step for those who need it

- Add an Indexer
  - Click on the `Settings` button at the top
  - Click on the `Indexers` tab
  - Click the big `+` symbol
  - Click the `Custom` button in the `Torznab` section
  - Configure your `Torznab` feed
    - `Name` : the name of this indexer (doesn't matter, just name it the name of your tracker)
    - `URL` : the `Copy Torznab Feed` url from `jackett` that you saved earlier
      - ⚠️ It is necessary to replace the IP with `jackett`
        - `http://jackett:9117/api/v2.0/indexers/<indexer>/results/torznab/` instead of 
        - `http://192.168.1.11:9117/api/v2.0/indexers/<indexer>/results/torznab/`
    - `API Key` : the `API Key` from `Jackett` that you saved earlier
  - Click `Test` to verify that it is configured properly
  - Click `Save`
  
- Add a Download Client
  - Click on the `Settings` button at the top
  - Click on the `Download Client` tab
  - Under `Completed Download Handling` toggle `Enable` from Yes to No
    - We'll be using `filebot` to handle our completed downloads
  - Click the `Save` button at the top right
  - Click the large `+` button
  - Click on `qBittorrent`
  - Configure your Download Client
    - `Name` : whatever you want; probably `qBittorrent`
    - `Host` : `qbittorrent`
    - `Port` : `6767` or whatever you have set for `QBIT_WEBUI_PORT` in your `.env` file
    - `Username` : `admin` or whatever you have set for `QBIT_WEBUI_USER` in your `.env` file
    - `Password` : `adminadmin` or whatever you have set for `QBIT_WEBUI_PASS` in your `.env` file
  - Click `Test` to verify that it is configured properly
  - Click `Save`
  
- Add Some Movies
  - Click on the `Add Movies` button at the top
  - Start typing in the search bar. It will search automatically when you stop typing
  - Configure the download path (you should only have to do this on the first movie you add)
    - Click the dropdown that says `Select Path`
    - Click `Add a different path`
    - Click the 📁button on the right of the modal
    - Click `movies` # final path should be `/movies/`
    - Click `Ok`
    - Click the green ✔️that is now visible
    - Click the `+` sign
- There are many other configuration options for `radarr` that are not covered here. `radarr`'s webpage is [here](https://radarr.video/)

## Configure qBittorrent
`<server-ip>:6767`

`qBittorrent` should already be configured. It automatically has configuration for the following:

- `filebot` download completion handling
- the username / password you set in your `.env`
- download directory set to `/downloads/`

If you want other `.env` configurations to be available for `qBittorrent`, open an issue here.

## Configure Plex
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

# Thank You
## Linuxserver
[linuxserverurl]: https://linuxserver.io
[linuxserverforumurl]: https://forum.linuxserver.io
[ircurl]: https://www.linuxserver.io/irc/
[podcasturl]: https://www.linuxserver.io/podcast/
[linuxserverdonate]: https://www.linuxserver.io/donate/

Most of these containers are config wrappers around [LinuxServer.io][linuxserverurl] containers. Without their amazing linuxserver containers, none of this would have been possible. If you find this automated media server useful, go donate to them!

* [forum.linuxserver.io][linuxserverforumurl]
* [IRC][ircurl] on freenode at `#linuxserver.io`
* [Podcast][podcasturl] covers everything to do with getting the most from your Linux Server plus a focus on all things Docker and containerisation!
* [Donate][linuxserverdonate]

## Filebot
[fileboturl]: https://www.filebot.net/
[filebotforumurl]: https://www.filebot.net/forums/
[filebotpurchaseurl]: https://www.filebot.net/purchase.html

This would also not be possible without filebot. This is currently using the free linux version `4.7.7`, but this may change to use the paid `4.8.2` version in the future. If it does so, there will be a separate tag to stay on `4.7.7`, but it will likely become unsupported.

* [Filebot][fileboturl]
* [Forum][filebotforumurl]
* [Purchase][filebotpurchaseurl]

## patorjk
Thanks to patorjk for his [ascii text generator](http://patorjk.com/software/taag/#p=display&f=Ogre&t=ghost)

# Future Plans
* Auto-configuration for linking `radarr` and `sonarr` to `qBittorrent`
* Auto-configuration for `plex` libraries
* Additional containers (`tautulli`, `muximux`, `portainer`)
* Upgrade `filebot` to `4.8.2` and make it easy to license
* Improve documentation (maybe blog post with pictures)
* Support additional `qBittorrent` configurations
