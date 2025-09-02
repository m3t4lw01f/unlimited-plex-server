# Unlimited Plex Server
A minimal docker compose configuration for an unlimited plex server using debrid services and usenet

## Assumptions
- You're running this on a local Linux machine
- You have a GUI on the same machine or another pc on the network that can access the server
  - Below, where you see localhost, replace this with the hostname or IP of your server if not working on the same machine
- You already have a working docker compose environment with your user able to run docker (your user is in the docker group)
- You are familiar with the linux command line, environment variables, etc
- You have a premium debrid service
- You have a premium usenet account (unlimited preferred)
- You have a [Plex](https://plex.tv) account

## Step one
Clone this repo somewhere handy `git clone https://github.com/m3t4lw01f/unlimited-plex-server`

## Required configuration
- rclone uses FUSE to do its thing, so install it from your package manager
  - On Ubuntu: `sudo apt install fuse`
  - other distros [Google it](https://www.google.com/search?q=how+to+install+fuse+on+linux)
- Find your [timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) and set `TZ` in `.env`
- Get your local user ID with `id`
  - This will output something like `uid=1000(username) gid=1000(groupname)`
  - Set `USER_ID` and `GROUP_ID` in `.env` to the numbers seen in the output (in this case `1000` for both)
  - Use these IDs in the `chown` command in the next step
- Some required folders
  - `sudo mkdir /mnt/tordav && sudo chown 1000:1000 /mnt/tordav` where decypharr's built in rclone will mount its remotes
  - `sudo mkdir /mnt/nzbdav && sudo chown 1000:1000 /mnt/nzbdav` where nzbdav-rclone will mount nzbdav's webdav server
  - `sudo mkdir /mnt/library && sudo chown 1000:1000 /mnt/library` where Plex will look for media, and the Sonarr/Radarr root folder
    - `mkdir /mnt/library/movies`
    - `mkdir /mnt/library/tv`
- For Plex
  - Ensure `PORT_PLEX` in `.env` is open on your router/firewall so that Plex is externally accessible
  - Get a [Plex claim token](https://account.plex.tv/claim) and set `PLEX_CLAIM_TOKEN` in `.env`
    - This token is only good for a few minutes, so be quick with the next step
  - The above is required so that when Plex starts, it automatically signs into your account and is available via Plex's [web app](https://app.plex.tv).

## Get things running
In the folder you cloned the repo to run `docker compose up -d`

## Configure Plex
- Head over to Plex's [web app](https://app.plex.tv). 
- Configure two libraries:
  - Movies at `/mnt/library/movies`
  - TV Shows at `/mnt/library/tv`
- In Plex settings for the server, on the transcode menu item, set 'Transcoder temporary directory' to `/transcode`
- If you have plex pass, enable hardware acceleration
- Special notes:
  - Plex thumbnail generation, chapter image generation, ad markers, intro markers, credit markers and voice activity can cause Plex to pull a LOT of data from real-debrid. It is recommended to not enable these if you are going to add a lot of content all at once.

# Configure rclone.conf
- in the `rclone.conf` file, set the user and password following the instructions in the comments

# Configure NzbDAV
- Head to localhost:3000
- Click Settings
  - Fill in your Usenet host details and hit save
  - Click SabNZBD
    - For categories, enter `nzbdav-tv,nzbdav-movie`
    - For mount path, enter `/mnt/nzbdav`
    - Make a note of API Key, this will be your `SABNZBD_API_KEY`
    - Click save
  - Click WebDAV
    - enter the username that matches what you entered in `rclone.conf`
    - enter the _original_ password, NOT the obfuscated one, from `rclone.conf`
    - click save

# Configure Decypharr
- head to localhost:8282 and click settings
- On the debrid tab, add a debrid service, such as real-debrid, and fill in the following:
  - API Key: `Your real-debrid API key`
  - Mount/rclone folder: `/mnt/tordav/realdebrid/__all__`
  - Check the `Enable webdav` checkbox
  - RC Refresh Directories: `__all__`
  - Save Configuration
- On the download tab
  - set the download path to `/mnt/tordav`
  - Save Configuration
- On the rclone tab, configure the following:
  - Global mount path: `/mnt/tordav`
  - User ID and Group ID: `1000` (Or same as above from the id command)
  - VFS cache settings as desired. I've had the best luck with
    - VFS Cache Mode: `writes`
    - Read Chunk Size: `16M`
    - Read Chunk Size Limit: `256M`
    - VFS Read Ahead: `256k`
    - Directory Cache Time: `5m`
    - VFS Read Chunk Streams: `4`
    - Save Configuration

## Configure Sonarr and Radarr
- Head to localhost:8989 (Sonarr) or localhost:7878 (Radarr)
  - These steps are the same for both tools, so set them up the same way
- In the media management settings make sure `use hardlinks instead of copy` is enabled
- In the settings, click on 'indexers'
  - Configure as many indexers as you want, for both torrents and usenet
  - You can use an indexer proxy like Prowlarr to make this easier
- In the settings, click on Download Clients
  - Click the + and pick SABnzbd and fill out the fields:
    - Host: `nzbdav`
    - Port: `3000`
    - Use SSL: `unchecked`
    - API Key: `SABNZBD_API_KEY` from above
    - Category: `nzbdav-tv` for Sonarr or `nzbdav-movie` for Radarr
    - Scroll down and hit test. Then save.
  - Click the + again, and pick qBittorrent
    - Host: `tordav`
    - Port: `8282`
    - Use SSL: `unchecked`
    - Username: `http://sonarr:8989` in Sonarr or `http://radarr:7878` in Radarr
    - Password: The API Key from the General Settings in either Sonarr or Radarr
      - This information is passed to Decypharr to set up automated repairs
    - Category: `tordav-tv` for Sonarr or `tordav-movie` for Radarr
    - Scroll down and hit test. Then save.
   
## Add a movie, or a show
- Add a movie to Radarr or a show to Sonarr and set the root path to `/mnt/library/movies` or `/mnt/library/tv`.
