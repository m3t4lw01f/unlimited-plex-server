# Unlimited Plex Server
A Docker Compose configuration for an unlimited plex server with decent defaults.

## Assumptions
- You're running this on a local Linux machine with a GUI
- You already have a working docker compose environment
- You are familiar with the linux command line, environment variables, etc
- You have a premium [real-debrid](https://real-debrid.com) subscription
- You have a [Plex](https://plex.tv) account

## Software and services this project uses
- [real-debrid](https://real-debrid.com) - A debrid service
- [TorrentIO](https://torrentio.strem.fun/configure) - A torrent index service
- [rclone](https://rclone.org) - command-line program to manage files on cloud storage
- [Riven](https://github.com/rivenmedia/riven) - A media library management tool
- [Zilean](https://github.com/iPromKnight/zilean) - A scraper of [Debrid Media Manager](https://debridmediamanager.com/start) shared releases
- [Zurg](https://github.com/debridmediamanager/zurg-testing) - A webdav server that serves up releases from your real-debrid account
- [Overseerr](https://overseerr.dev/) - A powerful request system
- [Plex](https://plex.tv) - a full featured media server
- Postgres - A relational database server
- FUSE - Filesystem in User Space

## Step one
Clone this repo somewhere handy `git clone https://github.com/m3t4lw01f/unlimited-plex-server`

## Step two
Zurg's docker image is in a private repository, so it requires some legwork to be able to obtain.\
Requirements:
- A [github](https://github.com) account
- A [Personal Classic Access Token](https://github.com/settings/tokens) with `read:packages` permission we'll call `PERSONAL_ACCESS_TOKEN`
- Login to ghcr using the token: `docker login ghcr.io -u username -p PERSONAL_ACCESS_TOKEN`
  
## Required configuration
- rclone uses FUSE to do its thing, so install it from your package manager
  - On Ubuntu: `sudo apt install fuse`
  - other distros [Google it](https://www.google.com/search?q=how+to+install+fuse+on+linux)
- Find your [timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) and set `TZ` in `.env`
- Get your local user ID with `id`
  - This will output something like `uid=1000(username) gid=1000(groupname)`
  - Set `USER_ID` and `GROUP_ID` in `.env` to the numbers seen in the output (in this case `1000` for both)
  - Use these IDs in the `chown` command in the next step
- create required folders:
  - rclone: `sudo mkdir /mnt/zurg` and `sudo chown 1000:1000 /mnt/zurg`
  - library: `sudo mkdir /mnt/library` and `sudo chown 1000:1000 /mnt/library`
  - These match `RCLONE_MNT` and `LIBRARY_MNT` in `.env` so if you create them in a different path don't forget to change them in `.env`
- Get your [real-debrid API private token](https://real-debrid.com/devices) and set `REAL_DEBRID` in `.env` and `token: ` at the top of `zurg-config.yml`
- Get a [Plex claim token](https://account.plex.tv/claim) and set `PLEX_CLAIM_TOKEN` in `.env`
  - This token is only good for a few minutes, so be quick with the next step

## Get things running
In the folder you cloned the repo to run `docker compose up -d`

## Configure Plex
The Plex server should automatically appear in your account in their [web app](https://app.plex.tv). Configure two (or four) libraries:
- Movies at `/mnt/library/movies`
- TV Shows at `/mnt/library/shows`
- `#OPTIONAL` Anime Movies at `/mnt/library/anime_movies`
- `#OPTIONAL` Anime TV at `/mnt/library/anime_shows`
- In Plex settings for the server, on the transcode menu item, set 'Transcoder temporary directory' to `/transcode`
- If you have plex pass, enable hardware acceleration
- Special notes:
  - Plex thumbnail generation, chapter image generation, ad markers, intro markers, credit markers and voice activity can cause Plex to pull a LOT of data from real-debrid. It is recommended to not enable these if you are going to add a lot of content all at once.

## Configure Overseerr
- Get your `RIVEN_API_KEY':
  - run `docker exec ups-riven-backend sh -c "grep -m 1 '\"api_key\"' /riven/data/settings.json | cut -d '\"' -f 4"`
  - The output will be a long string of characters
- In your browser head over to [Overseerr](http://localhost:5055), log in using your Plex account and head to the settings
- Connect to your Plex account on the "Plex" tab
- Navigate to the Notifications tab and select `webhooks`
  - Set the Webhook URL to `http://ups-riven-backend:8080/api/v1/webhook/overseerr`
  - Set Authorization Header to `Bearer RIVEN_API_KEY`
  - Check the boxes for `Request Automatically Approved` and `Request Approved`
  - Scroll down and hit `Test`. If successfull, hit `Save Changes`.
- Now, back on the General tab in the settings, click the copy button next to `API Key`. This is your `OVERSEERR_API_KEY`

## Configure Riven
- In your browser head over to [Riven](http://localhost:3000)
  - Enter `http://ups-riven-backend:8080` for the Backend URL
  - Enter the same `RIVEN_API_KEY` as above in the API Key box and hit save and connect.
- Click on settings and then General
  - Click the checkbox next to Separate Anime Dirs if you created the `#OPTIONAL` Plex libraries for anime above
  - Hit save
- Navigate to the Content tab and
  - Click the checkbox next to Overseerr
  - For the URL enter `http://ups-overseerr:5055` and paste `OVERSEERR_API_KEY` into the Overseerr API Key box
  - Check the checkbox next to Overseerr Use Webhook
  - Hit save

## Make a request
Back in [Overseerr](http://localhost:5055), search for something and when your results come back, click the little "Request" button on the thumbnail. It will use the webhook to send the request to Riven, Riven will use Zilean or TorrentIO as the source and if it finds an available release, it will add it to your real-debrid account. Once added to your real-debrid account, it will show up in the rclone webdav mount at `/mnt/zurg/__all__`, Riven will symlink that to `/mnt/library` in the appropriate subfolder and Plex will pick it up and scan it into your library.

## Troubleshooting
### Zilean
Zilean may require you to manually create the database. You can check Zilean's logs `docker logs zilean` and if it has failed to start run the following:\
`docker exec -it ups-database createdb -U postgres -W zilean`\
Make sure ups-database matches the container name (it will by default if you didn't change it in the docker-compose.yml) and the user (after -U) and database (after -W) match `PG_USER` and `ZILEAN_DB` in the `/env`. It will ask for the password. Use the same one set in `.env` as `PG_PASSWORD`

### I need more releases
To add more scrapers, simply head to the [Riven Settings for Scrapers](http://localhost:3000/settings/scrapers). This is useful if Zilean and TorrentIO are not providing what you want.
