# my-homelab-services-docker-stack

This is a general framework which I can use to add services (that require a GPU to run) to my homelab.

### to do

- [ ] clean up the docker-compose file
- [ ] rootless docker

### prerequisites

- [docker engine with nvidia runtime](https://github.com/placebeyondtheclouds/gpu-home-server?tab=readme-ov-file#common-setup-for-all-lxcs)

### add ssh key

```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/"$USER"_homelabservices -C "$USER"@homelabservices
ssh-copy-id -o IdentitiesOnly=yes -i ~/.ssh/"$USER"\_homelabservices $USER@192.168.19.234
tee -a ~/.ssh/config <<EOF

Host 192.168.19.234
  HostName 192.168.19.234
  User $USER
  IdentityFile ~/.ssh/$(echo $USER)_homelabservices
EOF

ssh 192.168.19.234
```

## main stack deployment

### homepage

- create proxmox api key in PMVE interface for the widget https://gethomepage.dev/widgets/services/proxmox/

### set up the `.env` file

`nano .env`

```bash
PORTAINER_PASSWORD_HASH=$$.....
PROXMOX_API_TOKEN_ID=api@pam!.....
PROXMOX_API_TOKEN_SECRET=
PROXMOX_IP_ADDRESS=192.168.19.237
LXC_IP_ADDRESS=192.168.19.234
OPENWRT_IP_ADDRESS=192.168.19.223
```

password hash can be generated with:

```bash
docker run --rm httpd:2.4-alpine htpasswd -nbB admin 'secure_password' | cut -d ":" -f 2  | sed -e s/\\$/\\$\\$/g
```

### deploy

```bash
DOCKER_HOST="ssh://$USER@192.168.19.234" docker compose up -d --build --force-recreate
DOCKER_HOST="ssh://$USER@192.168.19.234" docker cp homepage/config homepage:/app
```

use the last line to update homepage config after making changes to the config files

### hashcat

- https://github.com/dizcza/docker-hashcat

its ready to go, test and use like that:

```bash
DOCKER_HOST="ssh://$USER@192.168.19.234" docker exec -it hashcat bash
hashcat -I
hashcat -b
hashcat -m 22000 /mnt/hashcat/hash.hc22000 -a 3 ?d?d?d?d?d?d?d?d  -o /mnt/hashcat/hashcat_output.txt
```

### jellyfin

- https://jellyfin.org/docs/general/installation/container/
- https://github.com/gma1n/LXC-JellyFin-GPU?tab=readme-ov-file
- https://github.com/CodePlayer/webfont-noto

setup is needed.

- add proxmox bind mount for /mnt/media to LXC ID 104 (use the actual ID) by running on the host:

```bash
pct set 104 -mp0 /mnt/media,mp=/mnt/media
```

- Jellyfin initial setup http://192.168.19.234:8096/web/index.html#/wizardstart.html

- file naming: https://jellyfin.org/docs/general/server/media/movies/

- LXC 里面安装中文字体:

```bash
apt install fonts-noto-cjk
fc-cache -f -v
```

- copy fallback [fonts](https://github.com/CodePlayer/webfont-noto/tree/master) to the bind mount on the host:

```bash
wget https://github.com/CodePlayer/webfont-noto/raw/refs/heads/master/dist/NotoSans/NotoSansCJKsc-hinted/subset/NotoSansCJKsc-hinted-standard/NotoSansCJKsc-Light.woff2 -O /tmp/NotoSansCJKsc-Light.woff2
ssh $USER@192.168.19.234 mkdir -p ~/fonts
scp /tmp/NotoSansCJKsc-Light.woff2 $USER@192.168.19.234:~/fonts/NotoSansCJKsc-Light.woff2
```

- go to settings -> subtitles -> Burn subtitles -> set to `all complex formats` [issue](https://github.com/jellyfin/jellyfin-web/issues/5198)

- go to dashboard/playback/transcoding -> Hardware acceleration set to `nvidia nvenc`

- remove the metadata `DOCKER_HOST="ssh://$USER@192.168.19.234" docker exec -it jellyfin rm -rf config/metadata/library/*` and refresh each library's metadata (`refresh metadata`) if doing it after creating the libraries.

- copy new media files to the bind mount on the host, keeping the same directory structure:

```bash
scp -r ./temp/_video/1 root@192.168.19.237:/mnt/media/video/
```

### openrgb

```bash
lsusb | grep ARGB
ls -l /dev/bus/usb/003/004
udevadm info --query=all --name=/dev/bus/usb/003/004
nano /etc/pve/lxc/104.conf
lxc.cgroup2.devices.allow = c 189:259 rwm

```

```bash
DOCKER_HOST="ssh://$USER@192.168.19.234" docker exec -it openrgb sh
```

> [!WARNING]
> work in progress

### other

- nextcloud

- https://runtipi.io/docs/apps-available
- https://docs.techdox.nz/paperless/
- https://www.linuxserver.io/
- https://fleet.linuxserver.io/image?name=linuxserver/code-server
- https://fleet.linuxserver.io/image?name=linuxserver/transmission
- https://fleet.linuxserver.io/image?name=linuxserver/jellyfin

> [!WARNING]
> work in progress

## checkmate stack

https://github.com/bluewave-labs/checkmate?tab=readme-ov-file

https://bluewavelabs.gitbook.io/checkmate/users-guide/quickstart

```bash
cd checkmate
DOCKER_HOST="ssh://$USER@192.168.19.234" docker compose up -d --build --force-recreate
```

> [!WARNING]
> work in progress

## checkmate agent (capture)

edit .env file

```
API_SECRET=secret
```

on a server: add monitoring agent https://github.com/bluewave-labs/capture

deploy

```bash
DOCKER_HOST="ssh://$USER@192.168.19.234" docker compose up -d --build --force-recreate
```

add the agent in the checkmate webui:
http://192.168.19.234:3333/api/v1/metrics
secret

> [!WARNING]
> work in progress

## references

- https://technotim.live/posts/homepage-dashboard/
- https://github.com/jellyfin/jellyfin/issues/5636#issuecomment-814781468
- https://gethomepage.dev/widgets/services/proxmox/
- https://gethomepage.dev/configs/services/#icons
