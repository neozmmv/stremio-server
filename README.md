# The Stremio streaming Server Docker image
[![Docker Image Version (latest semver)](https://img.shields.io/docker/v/stremio/server?label=stremio%2Fserver%3Alatest)](https://hub.docker.com/r/stremio/server)

## Step by step guide to setup Stremio Server on your VPS
Stremio Server is a powerful tool to watch your favorite movies or series, but there is a catch: the Stremio app is available on mobile devices, but lacks core functionalities.
For instance, the App Store version of Stremio does not allow streaming content via Torrent, which is probably the most common way to use Stremio (besides paid services like Real Debrid). 

This is unfortunate, but we can work around it.

If you have a VPS on services like AWS, Oracle Cloud (very good free tier, use with caution to avoid 2k dollars on your credit card without you knowing),
you can host your own Stremio server, which can be used to stream content directly to your device, avoiding the Torrent issue.

### Creating your Tailscale account
1. First of all, create an account on [Tailscale](https://tailscale.com/).
2. Download the Tailscale app and login with the device you want to use Stremio.

### VPS Setup
On the VPS terminal, run:
```bash
sudo apt update && sudo apt upgrade -y
```

```bash
sudo apt install -y curl wget gnupg git docker.io -y
```

```bash
#Clone the repository
git clone https://github.com/Stremio/server-docker
```

```bash
sudo systemctl enable docker && sudo systemctl start docker
```

```bash
#Install Tailscale on VPS and login on the same Tailnet
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

After that, run the docker image with the command:
```bash
cd server-docker
docker run --rm -d -p 11470:11470 -p 12470:12470 stremio/server:latest
```
And check if it's running with
```bash
curl http://localhost:11470
```

If it's done correctly and running, you now can access your admin panel on Tailscale, get the VPS's IPv4 and put the HTTP endpoint on the mobile Stremio app settings.
Something like: `http://100.x.x.x:11470`, on port 11470 for HTTP and 12470 for HTTPS, but that requires a little more setup.


## Run the image

`sudo docker run --rm -d -p 11470:11470 -p 12470:12470 stremio/server:latest`

If you're running `stremio-web` locally then you should disable CORS on the server by passing `NO_CORS=1` to the env. variables:

`docker run --rm -d -p 11470:11470 -p 12470:12470 -e NO_CORS=1 stremio/server:latest`

Available ports:
- 11470 - http
- 12470 - https

Env. variables:

`FFMPEG_BIN` - full path to the ffmpeg binary, on platforms where it cannot be reliably determined by the ffmpeg-static package (e.g. darwin aarch64)

`FFPROBE_BIN` - full path to the ffprobe binary

`APP_PATH` - custom application path for storing server settings, certificates, etc

`NO_CORS` - if set to any value it will disable the CORS checks on the server.

## Build image

Docker image can be easily built using the included [`Dockerfile`](./Dockerfile).

By default, the image has `ffmpeg` installed with a specific version of `ffmpeg-jellyfin`,
for more information check the `Dockerfile`.

For the **desktop build** (currently, the only supported platform) do not pass the `BUILD` argument.

### Example: Build a Docker image with Server v4.20.1

**NB:** On a new Ubuntu release you must update the [`setup_jellyfin_repo.sh`](./setup_jellyfin_repo.sh) shell script for `jellyfin-ffmpeg`.

If you're cross-building the image from x86 to arm, you need to either use a [QEMU binary or `multiarch/qemu-user-static` (see below)](#cross-building)

- Platform: `linux/amd64` (also used for `linux/x86_64`):

`docker buildx build --platform linux/amd64 --build-arg VERSION=v4.20.1 -t stremio/server:latest .`

- Platform: `linux/arm64` (alias of `linux/arm64/v8`):

`docker buildx build --platform linux/arm64 --build-arg VERSION=v4.20.1 -t stremio/server:latest .`

- Platform `linux/arm/v7`:

`docker buildx build --platform linux/arm/v7 --build-arg VERSION=v4.20.1 -t stremio/server:latest .`

### Cross building
Cross building the image from an `x86` to `arm` architecture, you need to either use QEMU emulation binary or the `multiarch/qemu-user-static` docker image.

#### Using QEMU

Setup `binfmt` if you're not on Docker Desktop, if you are you can skip this step.
See https://docs.docker.com/build/building/multi-platform/#qemu for more details.

`docker run --privileged --rm tonistiigi/binfmt --install all`

- arm/v7
  `docker buildx build --platform linux/arm/v7 --build-arg VERSION=v4.20.1 -t stremio/server:latest .`

- arm64 / arm64/v8
  `docker buildx build --platform linux/arm64 --build-arg VERSION=v4.20.1 -t stremio/server:latest .`

#### Using `multiarch/qemu-user-static` image

For more details check https://github.com/multiarch/qemu-user-static.

`docker run --rm --privileged multiarch/qemu-user-static --reset -p yes`

- Build a Docker image with a local `server.js` found in the root of the folder:
**Note:** By passing an empty `VERSION` argument you will skip downloading the `server.js` from AWS before overriding it with your local one.

`docker buildx build --build-arg VERSION= -t stremio/server:latest .`

#### Arguments

- `VERSION` - specify which version of the `server.js` you'd like to be downloaded for the docker image.
- `BUILD` - For which platform you'd like to download the `server.js`.

Other arguments:

- `NODE_VERSION` - the version which will be included in the image and `server.js` will be ran with.
- `JELLYFIN_VERSION` - `jellyfin-ffmpeg` version, we currently require version **<= 4.4.1**.

## Publishing on Docker Hub

New releases of Stremio Server are automatically released in Docker Hub using the [publish.yml](.github/workflows/publish.yml) using a custom event type called "new-release" and client payload containing the release tag version.

You can also manually trigger this action with curl and a Personal Access Token generated from Github:
```
curl -L
   -X POST
   -H "Accept: application/vnd.github+json"
   -H "Authorization: Bearer PERSONAL_ACCESS_TOKEN"
   -H "X-GitHub-Api-Version: 2022-11-28"
   https://api.github.com/repos/stremio/server-docker/dispatches
   -d '{"event_type":"new-release","client_payload":{"tag": "v4.20.3"}}'
```

### Build images manually

1. Update version tag:

`docker buildx build --push --platform linux/arm64,linux/arm/v7,linux/amd64 --build-arg VERSION=v4.20.1 -t stremio/server:4.20.1 .`

2. Update latest tag:


`docker buildx build --push --platform linux/arm64,linux/arm/v7,linux/amd64 --build-arg VERSION=v4.20.1 -t stremio/server:latest .`
