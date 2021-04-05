# Automated Media Server 👻

- [Automated Media Server 👻](#automated-media-server-)
  - [About](#about)
  - [⚠️ A Note About Torrent Clients](#️-a-note-about-torrent-clients)
  - [Network](#network)
  - [Installation](#installation)
    - [Install Docker](#install-docker)
    - [Install Docker Compose](#install-docker-compose)
    - [Clone this repo](#clone-this-repo)
    - [Build your `.env` file](#build-your-env-file)
      - [Directories](#directories)
      - [PUID and PGID](#puid-and-pgid)
    - [Deploy the mediaserver](#deploy-the-mediaserver)
  - [Configuration](#configuration)
    - [Configure Jackett](#configure-jackett)
    - [Configure Sonarr](#configure-sonarr)
    - [Configure Radarr](#configure-radarr)
    - [Configure Bazarr](#configure-bazarr)
    - [Configure filebot](#configure-filebot)
    - [Configure transmission](#configure-transmission)
    - [Configure nzbget](#configure-nzbget)
    - [Configure Plex](#configure-plex)
    - [Reverse proxy / letsencrypt](#reverse-proxy--letsencrypt)
    - [VPN networks for transmission, nzbget, and others](#vpn-networks-for-transmission-nzbget-and-others)
      - [Why do you need to mount `docker.sock` for the wireguard container](#why-do-you-need-to-mount-dockersock-for-the-wireguard-container)
        - [Update: mounting `docker.sock` is no longer necessary](#update-mounting-dockersock-is-no-longer-necessary)
  - [Thank You](#thank-you)
    - [Linuxserver](#linuxserver)
    - [Filebot](#filebot)
    - [ASCII word generation](#ascii-word-generation)
    - [combustion UI for transmission](#combustion-ui-for-transmission)
    - [Donate](#donate)
  - [Future Plans](#future-plans)

```
         _               _
    __ _| |__   ___  ___| |_
   / _` | '_ \ / _ \/ __| __/
  | (_| | | | | (_) \__ \ |_
   \__, |_| |_|\___/|___/\__|
   |___/      /   _ \
          (¯\| o (@) |/¯)
           \_  .___.  _/
            /   !_!   \
           /_.--._.--._\
```

---

## About

This is an automated media server set up in docker containers via docker-compose. The goal of this project is to automate as much of the installation and configuration as possible while maintaining simplicity of the setup (no ansible playbooks or fancy bash scripts) to avoid making it a complete black box.

The end result of this setup is a media server with the following components

- `plex`
- `filebot`
  - with a web interface for triggering `amc` script
- `transmission`
  - configured to call `filebot` container on torrent completion
  - with `combustion` web UI
- `nzbget`
  - configured to call `filebot` container on nzb completion
- `sonarr`
- `radarr`
- `jackett`
- `bazarr`
- `tautulli`
  - configured to look at plex server logs
- `portainer`
- `watchtower`
- `nginx + letsencrypt`
  - for reverse proxying to reach your services at \*.domain.tld
- `heimdall`

The high-level steps for setup are as follows

1. Install docker and docker-compose
2. `docker-compose up`
3. add an indexer to `jackett`
4. configure `sonarr` / `radarr` to use the `jackett` indexer
5. configure `sonarr` / `radarr` to use an `nzb` indexer
6. configure `sonarr` / `radarr` to use `transmission` as their download client
7. configure `sonarr` / `radarr` to use `nzbget` as their nzb client
8. configure `nzbget` with your newsgroup provider
9. add libraries to `plex`

Once these steps have been completed, it is possible to add a TV Show or Movie to `sonarr` / `radarr`, have it automatically download them when available, and then it will be automatically copied into your `plex` libraries. Post-installation, this should be a fully-automated media server. The `transmission` container will automatically clean any files older than 30 days (configurable).

📓 This has only been tested on Ubuntu 18.04 LTS but it should work just fine on other linux distros. MacOS and Windows are unsupported. If you test it on MacOS or Windows and it works, let me know!

## ⚠️ A Note About Torrent Clients

I've removed the cofiguration for `qbittorrent`. If you want to keep using it, feel free to search through git history to recover the configuration, but it's no longer supported and has not been updated to use the new `filebot` container.

## Network

Each service is available on its own ports:

| Service      | Port  |
| ------------ | ----- |
| transmission | 5656  |
| nzbget       | 6789  |
| filebot      | 7676  |
| sonarr       | 8989  |
| radarr       | 7878  |
| bazarr       | 6767  |
| jackett      | 9117  |
| plex         | 32400 |
| portainer    | 9000  |
| heimdall     | 8888  |
| netdata      | 19999 |

📓To reach `plex`, append `/web` to the address e.g. `192.168.1.11:32400/web`

All of the services are running in the default docker-compose network. From within services, they can access each other via their `<service_name>:<port>` as defined in `docker-compose.yml`. These are _NOT_ the ports you've configured in your `.env` file. Those ports are mapped to the host device's network, but the services are operating within the network that docker-compose sets up, so they will not respect the rules forwarding ports to the host network.

| Service      | Port  |
| ------------ | ----- |
| transmission | 5656  |
| nzbget       | 6789  |
| filebot      | 7676  |
| sonarr       | 8989  |
| radarr       | 7878  |
| bazarr       | 6767  |
| jackett      | 9117  |
| plex         | 32400 |
| tautulli     | 8181  |
| heimdall     | 80    |
| netdata      | 19999 |

## Installation

### Install Docker

- <https://docs.docker.com/install/#supported-platforms>

### Install Docker Compose

- <https://docs.docker.com/compose/install/#install-compose>

### Clone this repo

```sh
git clone https://github.com/ghostserverd/mediaserver-docker.git
cd mediaserver-docker
```

### Build your `.env` file

```sh
cp .env_sample .env
id $USER # save the result of this for building your .env file below
```

Modify the `.env` file to specify the directory configurations. Note that these directories are for the host machine. They are mapped to various locations inside of their container by the `docker-compose` file.

The `.env_sample` file has some notes about the various configuration options. There are additional descriptions of each service's configuration below.

#### Directories

A good directory setup looks something like this

```sh
CONFIG_DIR=/some/directory/media_server/config
DOWNLOAD_DIR=/some/directory/media_server/downloads
MEDIA_DIR=/some/directory/media_server/media
TV_DIR=/some/directory/media_server/media/TV Shows
MOVIES_DIR=/some/directory/media_server/media/Movies
BASE_DIR=/some/directory/media_server
```

Note that there is a base directory that holds both `media` and `downloads`

```sh
/some/directory/media_server
```

Configuring a base directory with `downloads` and `media` present allows `filebot` to take advantage of `hardlinks` when renaming and moving files. Hardlinks are much quicker than actually copying the file, but they require that the source and destination be on the same device which is what the base directory accomplishes.

| variable     | description                                                                                                                                                                                                                                                                        |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CONFIG_DIR   | where configuration for each of the services will live. You'll end up with multiple directories in here, one for each service. If you move this directory to a different volume with a different instance of the whole media server, it should retain your various configurations. |
| DOWNLOAD_DIR | where various services will download files to. ⚠️ This should not be on a small partition as it will contain media files.                                                                                                                                                          |
| MEDIA_DIR    | where your media will be copied to. ⚠️ This should not be on a small partition as it will contain media files.                                                                                                                                                                     |
| TV_DIR       | where your TV shows will be placed by `filebot` on download completion. It should be a subdirectory of `MEDIA_DIR`                                                                                                                                                                 |
| MOVIES_DIR   | where your TV shows will be placed by `filebot` on download completion. It should be a subdirectory of `MEDIA_DIR`                                                                                                                                                                 |
| BASE_DIR     | a shared directory that houses `media` and `downloads` directories to be used for hardlinks                                                                                                                                                                                        |

#### PUID and PGID

| variable | description                                                                                                                    |
| -------- | ------------------------------------------------------------------------------------------------------------------------------ |
| PUID     | the unix `UID` that will be passed to the various services. It can be discovered by running `id $USER` on the host machine.    |
| PGID     | is the unix `GID` that will be passed to the various services. It can be discovered by running `id $USER` on the host machine. |

### Deploy the mediaserver

Fill out the rest of the configuration options before deploying. See each individual service's section for configuration details.

```sh
docker-compose up
```

Append `-d` to run in detached mode. The first time you run a service, it is probably a good idea to not run in dettached mode (i.e. DON'T append `-d`) so you can watch all of the logs for issues.

## Configuration

### Configure Jackett

| variable     | description                         |
| ------------ | ----------------------------------- |
| JACKETT_PORT | the port that jacket will listen on |

`<server-ip>:9117`

`jackett` should be configurable the same as any other installation of it. Feel free to skip these steps if you know how to configure `jackett` already.

- Add an Indexer
  - Click `+ Add Indexer`
  - Search for your desired tracker
  - Click on the 🔧icon next to your desired tracker
  - Sign in with your account information
  - Click `Okay`
- You should see `Successfully configured <tracker>` and a new entry for your tracker
- Copy the `API Key` from the top right corner and save it somewhere
- Click the `Copy Torznab Feed` on the tracker you just added and paste it somewhere to save it

### Configure Sonarr

| variable    | description                         |
| ----------- | ----------------------------------- |
| SONARR_PORT | the port that sonarr will listen on |

`<server-ip>:8989`

`sonarr` should be configurable the same as any other installation of it. Feel free to skip these steps if you know how to configure `sonarr` already.

⚠️ It is critical that you use `transmission` instead of the IP address when configuring the download client, as well as `jackett` instead of the IP when setting up your indexer. This is because this uses docker-compose networking which means each service is accessible at the name of the service, rather than the host or even local IP address.

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
  - Click on `transmission`
  - Configure your Download Client
    - `Name` : whatever you want; probably `transmission`
    - `Host` : `transmission`
    - `Port` : `5656` or whatever you have set for `TRANS_WEBUI_PORT` in your `.env` file
    - `Username` : `admin` or whatever you have set for `TRANS_WEBUI_USER` in your `.env` file
    - `Password` : `adminadmin` or whatever you have set for `TRANS_WEBUI_PASS` in your `.env` file
    - `Category`: `sonarr` - this allows transmission GC to properly remove torrents after seed limits exceeds
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

### Configure Radarr

| variable    | description                         |
| ----------- | ----------------------------------- |
| RADARR_PORT | the port that radarr will listen on |

`<server-ip>:7878`

`radarr` should be configurable the same as any other installation of it. Feel free to skip these steps if you know how to configure `radarr` already.

⚠️ It is critical that you use `transmission` instead of the IP address when configuring the download client, as well as `jackett` instead of the IP when setting up your indexer. This is because this uses docker-compose networking which means each service is accessible at the name of the service, rather than the host or even local IP address.

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
  - Click on `transmission`
  - Configure your Download Client
    - `Name` : whatever you want; probably `transmission`
    - `Host` : `transmission`
    - `Port` : `5656` or whatever you have set for `TRANS_WEBUI_PORT` in your `.env` file
    - `Username` : `admin` or whatever you have set for `TRANS_WEBUI_USER` in your `.env` file
    - `Password` : `adminadmin` or whatever you have set for `TRANS_WEBUI_PASS` in your `.env` file
    - `Category`: `radarr` - this allows transmission GC to properly remove torrents after seed limits exceeds
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

### Configure Bazarr

| variable    | description                        |
| ----------- | ---------------------------------- |
| BAZARR_PORT | the port that bazar will listen on |

`<server-ip>:6767`

`bazarr` should be configurable the same as any other installation of it. Feel free to skip these steps if you know how to configure `bazarr` already.

Step-by-step for those who need it

- Configure connection settings for Sonarr:

  - `Hostname or IP Address`: `sonarr`
  - `Listening Port`: `8989` or whatever you have set for `SONARR_PORT` in your `.env` file
  - `API Key`: the `API Key` from `Sonarr`

- Configure connection settings for Radarr:
  - `Hostname or IP Address`: `radarr`
  - `Listening Port`: `7878` or whatever you have set for `RADARR_PORT` in your `.env` file
  - `API Key`: the `API Key` from `Radarr`

### Configure filebot

`<server-ip>:7676`

| variable          | description                                                                                               |
| ----------------- | --------------------------------------------------------------------------------------------------------- |
| FILEBOT_PORT      | the web interface port for the filebot container                                                          |
| FILEBOT_FORMAT    | the filebot [format expression](https://www.filebot.net/naming.html) to use                               |
| FILEBOT_ACTION    | the [action](https://www.filebot.net/forums/viewtopic.php?t=4915) for filebot to take when renaming files |
| FILEBOT_CONFLICT  | what filebot does when it sees a conflicting filename                                                     |
| FILEBOT_SERIES_DB | the database to use for TV metadata lookup                                                                |
| FILEBOT_ANIME_DB  | the database to use for anime metadata lookup                                                             |
| FILEBOT_MOVIE_DB  | the database to use for movie metadata lookup                                                             |
| FILEBOT_MUSIC_DB  | the database to use for music metadata lookup                                                             |
| OPEN_SUB_USER     | the opensubtitles username for filebot to use when downloading subtitles [NOT FUNCTIONAL]                 |
| OPEN_SUB_PASS     | the opensubtitles password for filebot to use when downloading subtitles [NOT FUNCTIONAL]                 |

The opensubtitles configuration is called by the container startup script, but opensubtitles still fails when filebot is actually run. Until I figure out how to make this work, stick to `bazarr`.

You can navigate to `<server-ip>:7676` to see the filebot command that will be run when a download is completed. Do NOT expose this port to the internet as it is not password protected.

These are the `filebot` CLI options: <https://www.filebot.net/cli.html>. If there are additional options you want to be able to configure, open an issue.

⚠️ This has been updated to use the `4.9.x` version of `filebot` by default. In order to properly register after you have purchased a license, copy your `license.psm` file to the filebot config directory on your host machine. The container will automatically register `filebot` with that license.

### Configure transmission

| variable              | description                                                   |
| --------------------- | ------------------------------------------------------------- |
| TRANS_WEBUI_USER      | the username with which to log into transmission              |
| TRANS_WEBUI_PASS      | the password with which to log into transmission              |
| TRANS_WEBUI_PORT      | the port for transmission's web interface                     |
| TRANS_CONNECTION_PORT | the connection port for tranmission to use                    |
| TRANS_MAX_RETENTION   | the time in seconds before a torrent is automatically removed |
| TRANS_MAX_RATIO       | the ratio at which a torrent is automatically removed         |

`<server-ip>:5656`

It should not be necessary to configure transmission beyond the default configuration. The container writes a config with reasonable defaults. If you need access to additional transmission settings, feel free to open an issue.

The container is already automatically configured to call `filebot` to post-process a download.

### Configure nzbget

`<server-ip:7890`

| variable    | description                       |
| ----------- | --------------------------------- |
| NZBGET_PORT | the web interface port for nzbget |

The container is already automatically configured to call `filebot` to post-process a download.

These are the automatic configurations in `nzbget` now:

- The `MainDir` will automatically be set to `/downloads/nzb`
- The `ScriptDir` will automatically be set to `/usr/local/bin`
- The `Extensions` will automatically be set to `nzbget-postprocess.sh`
- The `ControlUsername` will automatically be set to whatever is configured in `NZBGET_WEB_USER` _iff_ it is set
- The `ControlPassword` will automatically be set to whatever is configured in `NZBGET_WEB_PASS` _iff_ it is set

The paths are configured that way to support calling `filebot` with the post-process script.

See [the documentation](https://nzbget.net/documentation) for instructions on setting up the rest of `nzbget`.

### Configure Plex

The first time that you set up your plex server, you will need to claim the server to associate it with your plex account. You need to access the server via `localhost` or `127.0.0.1` in order to claim it. The easiest way to accomplish this is to create an SSH tunnel to your server so you can access plex on port 32400 on localhost.

```
ssh <server-ip> -L 32400:localhost:32400
```

Once you do this, you can now log into plex via `localhost:32400/web` or `127.0.0.1:32400/web` and claim the server. Once the server has been claimed, you can log into it directly via the server's IP address.

| variable      | description                                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------------------------ |
| PLUGIN_LIST   | a list of plugins to install. supported plugins are `trakt` and `subzero`. leave empty to install no plugins |
| PLEX_WEB_PORT | the port for the plex web interface                                                                          |

`<server-ip>:32400/web`

- Add some libraries
  - TV Shows will be at `/data/TV Shows` assuming you followed the `/media/TV Shows` convention for `TV_DIR`
  - Movies will be at `/data/Movies` assuming you followed the `/media/Movies` convention for `MOVIES_DIR`
- ⚠️ If you don't use `Bazarr` set up your media agents to not use local files (hopefully this will be fixed in the future)
  - There is a problem that I have not been able to fix yet where local TV Series art is not available. It appears the artwork downloaded by `filebot` is not readable by `plex` for an unkown reason (I don't believe it's permissions related, but if you have ideas, please open an issue).
  - To get around this, uncheck `Local Media Assets` for all Agents under `Settings` > `Server` > `Agents`. Artwork will be downloaded by plex and accessible.
- Kill and restart the containers after logging in to Plex if you have Plexpass

```sh
docker stop plex
docker-compose up -d plex
```

### Reverse proxy / letsencrypt

The compose file includes a letsencrypt container which you can use to set up a reverse proxy to various services which will automatically provision an SSL certificate for you to use. If you do not want to use a reverse proxy, simply delete that entry from the compose file.

I recommend that you launch the full mediaserver once without the reverse proxy enabled so you can set up authentication for `sonarr`, `radarr`, `bazarr`, `nzbget`, `tautulli`, `netdata`, and `heimdall`

The reverse proxy is configured to use subdomain routing by default. It copies in appropriate configurations for each service with these names

| service      | config name      |
| ------------ | ---------------- |
| sonarr       | s.subdomain.conf |
| radarr       | r.subdomain.conf |
| transmission | t.subdomain.conf |
| nzbget       | n.subdomain.conf |
| plex         | p.subdomain.conf |
| tautulli     | u.subdomain.conf |
| netdata      | m.subdomain.conf |
| heimdall     | h.subdomain.conf |
| bazarr       | b.subdomain.conf |

Config files for each subdomain will be present in the `config` directory on your host machine if you wish to change the configurations. The directory is `config/letsencrypt/nginx/proxy-confs`. If you wish to use different subdomains (e.g. `plex.domain.tld` instead of `p.domain.tld`) you need to change the configuration for the subdomain in that directory, and update the `LE_SUBDOMAINS` to include the new subdomain.

Note that the first time you run the letsencrypt container, it can take some time for it to register an SSL cert.

| variable      | description                                                      |
| ------------- | ---------------------------------------------------------------- |
| LE_HOSTNAME   | the hostname you are using to host the reverse proxy             |
| LE_EMAIL      | the email for which your letsencrypt SSL cert will be registered |
| LE_SUBDOMAINS | the subdomains for which to register the SSL cert                |

### VPN networks for transmission, nzbget, and others

I have built a wireguard container following the principles described [here](https://nbsoftsolutions.com/blog/routing-select-docker-containers-through-wireguard-vpn) and have successfully tested using docker-compose to force all traffic for `nzbget` through the wireguard VPN using a config from <http://mullvad.net/>.

Download or build a wireguard config file from your VPN provider. For example, mullvad has a wireguard config generator at <https://mullvad.net/en/download/wireguard-config/>. Once you have the config file (e.g. mine is called `wgnet0.conf`, copy that file into your config/wireguard directory. The container will automatically find the config file and build the VPN network for you.

⚠️ Do NOT use your VPN's killswitch in the `wireguard` config file. This will disallow traffic from the docker network which will make it inaccessible from the reverse proxy container. There is a rudimentary killswitch implemented in the `wireguard` container already. I may improve this over time.

`docker-compose.yml` has been updated with a commented out wireguard service. There is also a commented out `nzbget` service configured to use the wireguard container's network as an example. This will force `nzbget` to proxy all traffic through the `wireguard` container, and thus through the VPN. You can follow the same pattern with the `transmission` container, or even `sonarr` and `radarr` if you want all queries to torrent trackers to go through the VPN.

To proxy an additional container through the `wireguard` container's network, add the service name (e.g. `transmission`) to the list of network aliases defined in the `wireguard` service.

```yaml
networks:
  default:
    aliases:
      - nzbget
      - transmission
```

Then update the new service's definition (e.g. `transmission`) to remove the port mapping list, and add

```yaml
network_mode: "service:wireguard"
depends_on:
  - wireguard
```

Because the `wireguard` has an alias to `transmission`, it is now accessible on `transmission:5656` just like it was before. You will also want to add the port mapping that were originally in the `transmission` service definition to the `wireguard` service definition.

The `wireguard` container now has a proper route back to the local network over `eth0` so services that are using the `wireguard` network should now be accessible via your local subnet. You need to specify this subnet in the `LOCAL_NETWORK` environment variable. Thanks to [htilly](https://github.com/htilly/wireguard-docker) and [cmulk](https://github.com/cmulk/wireguard-docker) who's containers I shamelessly copied and modified. Once you have set up the route back to the local subnet, you can properly port foward through the VPN. See <https://mullvad.net/en/help/port-forwarding-and-mullvad/> for details on port forwarding with `mullvad`.

#### Why do you need to mount `docker.sock` for the wireguard container

Mounting `docker.sock` is normally an anti-pattern for containers. However, when the `wireguard` container is started, `wg-quick` rewrites `/etc/resolv.conf` with the DNS address specified in the interface's `.conf` file. This breaks DNS resolution for docker services from within any container that is using the `wireguard` container as its network. In practice, this means that from within e.g. the `transmission` or `nzbget` containers, `curl filebot:7676` will fail DNS resolution. This breaks the auto-config between downlaoders and `filebot`.

To work around this issue, the `wireguard` container now inspects the network (`docker network inspect mediaserver-docker_default`) and writes `/etc/hosts` entries for each service it finds in the network. This means that if one of your service IPs changes for any reason, you'll need to restart the `wireguard` container in order to be able to properly resolve DNS for docker services again.

I believe it is possible to fix this issue using `dnsmasq` instead of writing `/etc/hosts` which would remove the requirement to read from `docker.sock`, but I have not been able to get it to work, so I'm leaving this hack in place for now.

If you do not mount `docker.sock`, the `wireguard` container will still run, but any containers in the `wireguard` network will be unable to resolve DNS for other docker services. If that's fine with you, you don't need to mount `docker.sock`.

##### Update: mounting `docker.sock` is no longer necessary

The `wireguard` container has been updated to use `dnsmasq` to conditionally route DNS requests.

The container also now scrapes the wireguard interface file for its DNS server and write /etc/dnsmasq.conf with that as the final fallback DNS address.

There are now three options for enabling docker DNS resolution from within the wireguard container:

1. If `docker.sock` is mounted, write `/etc/hosts` with any service in the network (this is really just for backwards comptability and is the same behavior as described above.)

2. If `LOCAL_TLD` is set (e.g. to local) write `/etc/dnsmasq.conf` to use `127.0.0.11` for that TLD. Also, write `/etc/resolv.conf` to search `LOCAL_TLD` so that containers can access the addresses of the services without having to know to append `.local` to match the rule in `/etc/dnsmasq.conf`. This will require aliases with the TLD in each of the containers that need to be accessible from within the wireguard network. An alias should look something like this (e.g. for `LOCAL_TLD=ghost`).

    ```yaml
    filebot:
      image: ghostserverd/filebot:4.9.x
        container_name: filebot
        restart: always
        networks:
          default:
            aliases:
              - filebot.ghost
    ```

    which will result in `filebot` being accessible from containers within the `wireguard` network.

3. If `SERVICE_NAMES` is set (a list of services to make available from within the wireguard network), write each service name individually to `/etc/dnsmasq.conf` to force `127.0.0.11` as the DNS server for each service address. This is nice because you don't have to write an alias for each service to make available, but you do need to list out all of the services. A sample `SERVICE_NAMES` variable is set in the `docker-compose.yml` file. It shouldn't need to be modified, but if there is a reason to, open an issue and I can make it pull from the `.env` file.

The last three settings are mutually exclusive. Only one of the mechanisms should be used. Of the last two, I'm not really sure which one I prefer, but they're both better than mounting docker.sock in my opinion.

## Thank You

### Linuxserver

[linuxserverurl]: https://linuxserver.io
[linuxserverforumurl]: https://forum.linuxserver.io
[ircurl]: https://www.linuxserver.io/irc/
[podcasturl]: https://www.linuxserver.io/podcast/
[linuxserverdonate]: https://www.linuxserver.io/donate/

Most of these containers are config wrappers around [LinuxServer.io][linuxserverurl] containers. Without their amazing linuxserver containers, none of this would have been possible. If you find this automated media server useful, go donate to them! They probably deserve it more than I do.

- [forum.linuxserver.io][linuxserverforumurl]
- [IRC][ircurl] on freenode at `#linuxserver.io`
- [Podcast][podcasturl] covers everything to do with getting the most from your Linux Server plus a focus on all things Docker and containerisation!
- [Donate][linuxserverdonate]

### Filebot

[fileboturl]: https://www.filebot.net/
[filebotforumurl]: https://www.filebot.net/forums/
[filebotpurchaseurl]: https://www.filebot.net/purchase.html

This would also not be possible without filebot. This has been updated to use the latest `4.9.x` version of filebot. It supports automatic registration as long as you provide a license file.

- [Filebot][fileboturl]
- [Forum][filebotforumurl]
- [Purchase][filebotpurchaseurl]

### ASCII word generation

Thanks to patorjk for his [ascii text generator](http://patorjk.com/software/taag/#p=display&f=Ogre&t=ghost)

[combustionurl]: https://github.com/Secretmapper/combustion

### combustion UI for transmission

Thanks to `secretmapper` for the `combustion` UI for `transmission`

- [Combustion][combustionurl]

### Donate

If you really want to donate to me, you can do that here

[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=Y8B6ES2LDJ2N8&currency_code=USD&source=url)

## Future Plans

- Consider switching from `nginx` to `traefik`
- Auto-configuration for linking `radarr` and `sonarr` to `transmission`
- Auto-configuration for linking `radarr` and `sonarr` to `nzbget`
- Auto-configuration for `plex` libraries
- Improve documentation (maybe blog post with pictures)
- ~~Better configuration options for `nzbget`~~
- ~~Wireguard as a network container for downloaders~~
- ~~Additional containers (`tautulli`, `muximux`, `portainer`)~~
- ~~Add reverse proxy support (`traefik`?)~~
- ~~Upgrade `filebot` to `4.8.2` and make it easy to license~~
