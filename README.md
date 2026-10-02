# Home Media Server

A simple self-hosted home server built with **Ubuntu Server + Docker**,
running:

-   **Jellyfin** - movies, TV and music
-   **Immich** - photo and video backup
-   **Nextcloud** - private cloud file storage
-   **Homarr** - a dashboard for accessing the services

This repository accompanies my home media server build. The hardware I
used was a **GMKtec NucBox G5 with an Intel N97**, an internal SSD for
Ubuntu/application data, and an external SSD for media, photos and
files.

You do not need to use the same hardware. The guide is intentionally written
so that hardware-specific values are checked on your own server rather
than copied from mine.

> \[!IMPORTANT\] This guide assumes an x86-64 Ubuntu Server machine and
> uses Intel Quick Sync for the optional Jellyfin hardware-transcoding
> section. Jellyfin itself will run without Intel hardware acceleration,
> but users of AMD/NVIDIA hardware will need to adapt that section.

## Contents

1.  [Architecture](#architecture)
2.  [Before You Start](#before-you-start)
3.  [Install Ubuntu Server](#1-install-ubuntu-server)
4.  [Prepare the Storage Drive](#2-prepare-the-storage-drive)
5.  [Install Docker](#3-install-docker)
6.  [Install Jellyfin](#4-install-jellyfin)
7.  [Install Immich](#5-install-immich)
8.  [Install Nextcloud](#6-install-nextcloud)
9.  [Install Homarr](#7-install-homarr)
10. [Final Checks](#8-final-checks)
11. [Useful Docker Commands](#useful-docker-commands)

------------------------------------------------------------------------

## Architecture

The build separates application/configuration data from the larger user
data:

``` text
Internal system SSD
├── Ubuntu Server
└── ~/docker/
    ├── jellyfin/
    ├── immich/
    ├── nextcloud/
    └── homarr/

External storage SSD
└── /mnt/storage/
    ├── media/
    │   ├── movies/
    │   ├── tv/
    │   └── music/
    ├── immich/
    ├── nextcloud/
    └── backups/
```

This is only one sensible layout. Change the paths to suit your storage
configuration.

### Services and ports

  Service     Purpose              Default URL
  ----------- -------------------- -------------------------
  Jellyfin    Media server         `http://SERVER-IP:8096`
  Immich      Photo/video backup   `http://SERVER-IP:2283`
  Nextcloud   File storage         `http://SERVER-IP:8080`
  Homarr      Dashboard            `http://SERVER-IP:7575`

> \[!NOTE\] These services are initially exposed only over normal HTTP
> on your local network. Do not simply port-forward them to the public
> internet. If you need remote access, use an appropriately secured
> solution such as a VPN/reverse-proxy setup and follow the current
> documentation for each service.

------------------------------------------------------------------------

# 1. Install Ubuntu Server

Install a current Ubuntu Server LTS release on the server's internal
drive.

During installation:

-   Create your own username and password.
-   Enable **OpenSSH Server** if you want to administer the machine
    remotely.
-   Install Ubuntu to the internal system drive, not the external drive
    that will hold your media.

After logging in:

``` bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

After the reboot, reconnect to the server.

You can find its IP address with:

``` bash
hostname -I
```

Make a note of the LAN IP you will use to access the services.

------------------------------------------------------------------------

# 2. Prepare the Storage Drive

I used a separate SSD mounted at:

``` text
/mnt/storage
```

The commands below assume the drive is already partitioned and formatted
as **ext4**.

> \[!WARNING\] Do not blindly copy a device name such as `/dev/sda1`.
> Identify your own storage device first. Formatting or modifying the
> wrong disk can destroy data.

## 2.1 Identify the drive

Run:

``` bash
lsblk -f
```

Look for the partition you want to use for server storage.

Example only:

``` text
sda
└─sda1 ext4
```

For the rest of this section, replace `/dev/sdX1` with your actual
partition.

Create the mount point:

``` bash
sudo mkdir -p /mnt/storage
```

Test mount the drive:

``` bash
sudo mount /dev/sdX1 /mnt/storage
```

Check it:

``` bash
df -hT /mnt/storage
```

You should see the drive mounted as `ext4`.

## 2.2 Find the UUID

Run:

``` bash
sudo blkid /dev/sdX1
```

Copy the value shown after `UUID=`.

Open `/etc/fstab`:

``` bash
sudo nano /etc/fstab
```

Add this line, replacing `YOUR-DRIVE-UUID`:

``` text
UUID=YOUR-DRIVE-UUID /mnt/storage ext4 defaults,nofail 0 2
```

Save with `Ctrl+O`, press `Enter`, then exit with `Ctrl+X`.

Test the entry **before rebooting**:

``` bash
sudo umount /mnt/storage
sudo mount -a
```

If `mount -a` produces no errors, check:

``` bash
df -hT /mnt/storage
```

## 2.3 Set ownership

Make your current user the owner:

``` bash
sudo chown "$USER":"$USER" /mnt/storage
```

Check:

``` bash
ls -ld /mnt/storage
```

Test that you can write to the drive:

``` bash
touch /mnt/storage/testfile
ls -l /mnt/storage/testfile
rm /mnt/storage/testfile
```

## 2.4 Create the storage folders

``` bash
mkdir -p /mnt/storage/media/{movies,tv,music}
mkdir -p /mnt/storage/{immich,nextcloud,backups}
```

Install `tree` and check the structure:

``` bash
sudo apt install tree -y
tree /mnt/storage
```

You should have roughly:

``` text
/mnt/storage
├── backups
├── immich
├── media
│   ├── movies
│   ├── music
│   └── tv
└── nextcloud
```

------------------------------------------------------------------------

# 3. Install Docker

This uses Docker's official Ubuntu repository rather than Ubuntu's
`docker.io` package.

## 3.1 Remove conflicting packages

``` bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
    sudo apt-get remove -y "$pkg"
done
```

It is fine if some packages were not installed.

## 3.2 Add Docker's repository

``` bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's signing key:

``` bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

``` bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the repository:

``` bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Then:

``` bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 3.3 Check Docker

``` bash
sudo systemctl status docker --no-pager
```

Look for:

``` text
Active: active (running)
```

Test Docker:

``` bash
sudo docker run hello-world
```

You should see:

``` text
Hello from Docker!
```

## 3.4 Run Docker without sudo

``` bash
sudo usermod -aG docker "$USER"
```

Log out:

``` bash
exit
```

Reconnect to the server, then test:

``` bash
docker run hello-world
docker compose version
docker version
```

------------------------------------------------------------------------

# 4. Install Jellyfin

Jellyfin will keep its configuration/cache on the system drive and read
media from `/mnt/storage/media`.

## 4.1 Check the Intel GPU

If you have a compatible Intel CPU/iGPU and want hardware transcoding:

``` bash
ls -l /dev/dri
```

A typical Intel system will show entries including:

``` text
card0
renderD128
```

Check the render device:

``` bash
ls -l /dev/dri/renderD128
```

Install the Intel GPU utilities:

``` bash
sudo apt install -y intel-gpu-tools vainfo
```

Then:

``` bash
vainfo
```

> \[!NOTE\] `vainfo` can produce display-related warnings on a headless
> Ubuntu Server. The important checks for this build are that
> `/dev/dri/renderD128` exists and that the Jellyfin container can
> access it.

## 4.2 Get the IDs required by the Compose file

Get your user's UID and GID:

``` bash
id -u
id -g
```

On a typical first Ubuntu user these are both `1000`, but **check rather
than assume**.

Now check the Intel GPU groups:

``` bash
getent group render
getent group video
```

Example:

``` text
render:x:991:
video:x:44:
```

The numeric values can vary between installations.

## 4.3 Create the Jellyfin directory

``` bash
mkdir -p ~/docker/jellyfin/{config,cache}
cd ~/docker/jellyfin
```

Copy `jellyfin/compose.yml` and `jellyfin/.env.example` from this
repository into this directory.

Create your local `.env`:

``` bash
cp .env.example .env
```

Edit it:

``` bash
nano .env
```

Set these four values using the commands from the previous section:

``` text
PUID=1000
PGID=1000
RENDER_GID=991
VIDEO_GID=44
```

Save and exit.

> \[!TIP\] The supplied Compose file maps `/mnt/storage/media/movies`,
> `/tv`, and `/music`. If your media lives somewhere else, edit the
> volume paths in `compose.yml`.

## 4.4 Validate and start Jellyfin

``` bash
docker compose config
```

If there are no errors:

``` bash
docker compose up -d
```

Check:

``` bash
docker compose ps
```

And:

``` bash
docker logs jellyfin --tail 100
```

The container should remain `Up` rather than repeatedly restarting.

## 4.5 Open Jellyfin

Find the server IP if required:

``` bash
hostname -I
```

Open:

``` text
http://YOUR-SERVER-IP:8096
```

During Jellyfin's setup:

1.  Choose your language.
2.  Create an administrator account.
3.  Add a **Movies** library using `/media/movies`.
4.  Add a **Shows** library using `/media/tv`.
5.  Optionally add a **Music** library using `/media/music`.
6.  Select your preferred metadata language/country.
7.  Finish setup.

## 4.6 Enable Intel Quick Sync transcoding

This is optional and applies to compatible Intel hardware.

In Jellyfin go to:

**Dashboard → Playback → Transcoding**

Select:

``` text
Intel QuickSync (QSV)
```

Use:

``` text
/dev/dri/renderD128
```

Enable hardware decoding only for codecs supported by your hardware.

> \[!IMPORTANT\] Transcoding capabilities vary by CPU generation and
> GPU. The Intel N97 used in my build supports the configuration I
> demonstrated, but do not assume every Intel CPU supports the same
> codecs.

Check that the container can see the GPU:

``` bash
docker exec jellyfin ls -l /dev/dri/renderD128
```

Check its group memberships:

``` bash
docker exec jellyfin id
```

------------------------------------------------------------------------

# 5. Install Immich

For Immich, this guide deliberately downloads the **current official
Compose deployment** rather than keeping a potentially outdated Immich
Compose file in this repository.

The layout is:

``` text
~/docker/immich/           configuration + PostgreSQL
/mnt/storage/immich/       photo/video library
```

## 5.1 Create the directories

``` bash
mkdir -p ~/docker/immich/postgres
mkdir -p /mnt/storage/immich
cd ~/docker/immich
```

## 5.2 Download Immich's current deployment files

``` bash
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
```

``` bash
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

Check:

``` bash
ls -lah
```

You should see at least:

``` text
.env
docker-compose.yml
postgres/
```

## 5.3 Generate the database password

``` bash
openssl rand -hex 16
```

Copy the generated value.

Do **not** commit this password to GitHub.

## 5.4 Configure Immich

First get the full path to your home directory:

``` bash
echo "$HOME"
```

Open:

``` bash
nano .env
```

Set:

``` text
UPLOAD_LOCATION=/mnt/storage/immich
```

Set `DB_DATA_LOCATION` to the full path to your Immich PostgreSQL
directory.

For example, if `echo "$HOME"` returned `/home/michael`:

``` text
DB_DATA_LOCATION=/home/michael/docker/immich/postgres
```

Set your timezone. My server used:

``` text
TZ=Australia/Perth
```

Change that to your own timezone if required.

Set:

``` text
DB_PASSWORD=YOUR_GENERATED_PASSWORD
```

Leave Immich's supplied values for `IMMICH_VERSION`, `DB_USERNAME` and
`DB_DATABASE_NAME` at their current defaults unless the current Immich
documentation instructs otherwise.

Save and exit.

Check the non-secret values:

``` bash
grep -E '^(UPLOAD_LOCATION|DB_DATA_LOCATION|TZ|IMMICH_VERSION|DB_USERNAME|DB_DATABASE_NAME)=' .env
```

Then check the services:

``` bash
docker compose config --services
```

## 5.5 Start Immich

``` bash
docker compose up -d
```

Check:

``` bash
docker compose ps
```

Give the services a little time on the first start, then run it again if
necessary.

Check the Immich server log:

``` bash
docker compose logs --tail=50 immich-server
```

Check the database:

``` bash
docker compose logs --tail=30 database
```

Check that Immich is creating data on the storage SSD:

``` bash
tree -L 2 /mnt/storage/immich
```

## 5.6 Open Immich

Browse to:

``` text
http://YOUR-SERVER-IP:2283
```

Create the first account. The first account becomes the administrator.

You can then install the Immich mobile app and connect it to:

``` text
http://YOUR-SERVER-IP:2283
```

For an initial test, back up a small album or a few photos rather than
immediately uploading your entire library.

Confirm storage usage:

``` bash
du -sh /mnt/storage/immich
```

------------------------------------------------------------------------

# 6. Install Nextcloud

Nextcloud will use:

``` text
~/docker/nextcloud/         application + MariaDB
/mnt/storage/nextcloud/     user files
```

## 6.1 Create the directories

``` bash
mkdir -p ~/docker/nextcloud/{db,html}
mkdir -p /mnt/storage/nextcloud
cd ~/docker/nextcloud
```

Copy `nextcloud/compose.yml` and `nextcloud/.env.example` from this
repository into this directory.

## 6.2 Create the environment file

``` bash
cp .env.example .env
```

Generate the MariaDB root password:

``` bash
openssl rand -hex 16
```

Generate a second password for Nextcloud's database user:

``` bash
openssl rand -hex 16
```

Edit:

``` bash
nano .env
```

Replace the two password placeholders:

``` text
MYSQL_ROOT_PASSWORD=YOUR_FIRST_GENERATED_PASSWORD
MYSQL_PASSWORD=YOUR_SECOND_GENERATED_PASSWORD
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
```

Save and exit.

Never commit `.env`.

## 6.3 Set storage permissions

The Nextcloud Apache container needs write access to its data directory:

``` bash
sudo chown -R 33:33 /mnt/storage/nextcloud
```

Check:

``` bash
ls -ld /mnt/storage/nextcloud
```

## 6.4 Validate and start Nextcloud

``` bash
docker compose config
docker compose config --services
```

You should see:

``` text
db
redis
nextcloud
```

Start it:

``` bash
docker compose up -d
```

Check:

``` bash
docker compose ps
```

Then:

``` bash
docker compose logs --tail=50 nextcloud
docker compose logs --tail=30 db
```

## 6.5 Open Nextcloud

Browse to:

``` text
http://YOUR-SERVER-IP:8080
```

Create your administrator account.

If Nextcloud asks for database settings, use:

  Setting             Value
  ------------------- -----------------------------
  Database user       `nextcloud`
  Database password   Your `MYSQL_PASSWORD` value
  Database name       `nextcloud`
  Database host       `db`

Do **not** use the server's LAN IP as the database host. Docker resolves
the service name `db` internally.

After setup, create a test folder and upload a small file.

Confirm that data is being written to the external drive:

``` bash
sudo du -sh /mnt/storage/nextcloud
```

``` bash
sudo find /mnt/storage/nextcloud -maxdepth 3 -type d | head -30
```

------------------------------------------------------------------------

# 7. Install Homarr

Homarr provides a single dashboard linking the services together.

Its data is small, so this build keeps it on the system drive.

## 7.1 Create the directory

``` bash
mkdir -p ~/docker/homarr/appdata
cd ~/docker/homarr
```

Copy `homarr/compose.yml` and `homarr/.env.example` from this repository
into this directory.

## 7.2 Create the encryption key

``` bash
cp .env.example .env
```

Generate a key:

``` bash
openssl rand -hex 32
```

Edit:

``` bash
nano .env
```

Replace:

``` text
SECRET_ENCRYPTION_KEY=PASTE_YOUR_GENERATED_KEY_HERE
```

with the generated value.

Save and exit.

Do not commit `.env`.

## 7.3 Start Homarr

Validate:

``` bash
docker compose config
```

Start:

``` bash
docker compose up -d
```

Check:

``` bash
docker compose ps
```

And:

``` bash
docker logs homarr --tail 50
```

## 7.4 Open Homarr

Browse to:

``` text
http://YOUR-SERVER-IP:7575
```

Create the administrator account and complete onboarding.

Add these applications:

  Name        URL
  ----------- ------------------------------
  Jellyfin    `http://YOUR-SERVER-IP:8096`
  Immich      `http://YOUR-SERVER-IP:2283`
  Nextcloud   `http://YOUR-SERVER-IP:8080`

### Optional Jellyfin integration

Homarr can integrate with Jellyfin rather than simply linking to it.

In Jellyfin go to:

**Dashboard → API Keys**

Create a dedicated API key named something like:

``` text
Homarr
```

Enter that key into Homarr's Jellyfin integration settings.

Never publish the API key.

> \[!NOTE\] This basic build intentionally does not mount
> `/var/run/docker.sock` into Homarr. Direct Docker socket access gives
> a container extensive control over the Docker host. If you later want
> container-management integration, review Homarr's current security
> guidance and consider a restricted socket proxy.

------------------------------------------------------------------------

# 8. Final Checks

The Compose files use:

``` text
restart: unless-stopped
```

so the containers should automatically return after a server reboot.

Restart the server:

``` bash
sudo reboot
```

After it comes back online, reconnect over SSH and run:

``` bash
docker ps
```

You should see containers for Jellyfin, Immich, Nextcloud and Homarr.

Then check each service:

``` text
Jellyfin   http://YOUR-SERVER-IP:8096
Immich     http://YOUR-SERVER-IP:2283
Nextcloud  http://YOUR-SERVER-IP:8080
Homarr     http://YOUR-SERVER-IP:7575
```

Also confirm the storage drive remounted correctly:

``` bash
df -hT /mnt/storage
```

If everything is running after a reboot, the basic server setup is
complete.

------------------------------------------------------------------------

# Useful Docker Commands

## See running containers

``` bash
docker ps
```

## See all containers

``` bash
docker ps -a
```

## Start a Compose stack

From its directory:

``` bash
docker compose up -d
```

## Stop a Compose stack

``` bash
docker compose down
```

## Restart a Compose stack

``` bash
docker compose restart
```

## Check stack status

``` bash
docker compose ps
```

## Validate a Compose file

``` bash
docker compose config
```

## Follow logs

For example:

``` bash
docker logs -f jellyfin
```

or for a Compose service:

``` bash
docker compose logs -f
```

## Update a stack

From the relevant Compose directory:

``` bash
docker compose pull
docker compose up -d
```

> \[!IMPORTANT\] Before major application upgrades, read the project's
> release notes and make sure you have backups. Some applications can
> require migration steps between versions.

------------------------------------------------------------------------

# Repository Files

``` text
.
├── README.md
├── .gitignore
├── jellyfin/
│   ├── compose.yml
│   └── .env.example
├── nextcloud/
│   ├── compose.yml
│   └── .env.example
└── homarr/
    ├── compose.yml
    └── .env.example
```

Immich is intentionally downloaded from its latest official release
during installation rather than storing a static copy here.

------------------------------------------------------------------------

# Notes

This repository documents the configuration used for my YouTube
home-server project, adapted to make the installation portable across
similar Ubuntu systems.

Paths, user/group IDs, storage devices, GPU capabilities, IP addresses
and timezones may differ on your system. Check each of these rather than
copying example values blindly.

If you change the storage layout, make the corresponding changes to the
Compose volume mappings before starting the containers.
