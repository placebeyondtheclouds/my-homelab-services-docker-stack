# my-homelab-services-docker-stack

This is a general framework which I can use to add services (that require a GPU to run) to my homelab. These are my notes and not a refined guide.

> [!WARNING]
> work in progress

### the end result:

docker containers with the following services:

- homepage
- hashcat
- jellyfin
- juice (to run CUDA code or rendering tasks on the server's GPU from other machines on the network)
- scuda (another GPU over IP solution, running code that uses CUDA, Linux server and Linux client)

### to do

- [ ] clean up the docker-compose file

### prerequisites

1. [Proxmox with NVIDIA drivers](https://github.com/placebeyondtheclouds/gpu-home-server?tab=readme-ov-file#software-setup-process)
2. [GPU-enabled LXC](https://github.com/placebeyondtheclouds/gpu-home-server?tab=readme-ov-file#common-setup-for-all-lxcs)
3. [docker in a GPU-enabled LXC](https://github.com/placebeyondtheclouds/gpu-home-server?tab=readme-ov-file#continue-setting-up-the-debian-lxc-with-gpu-enabled-docker)

### add ssh key

```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/"$USER"_homelabservices -C "$USER"@homelabservices
ssh-copy-id -o IdentitiesOnly=yes -i ~/.ssh/"$USER"\_homelabservices $USER@192.168.19.234
tee -a ~/.ssh/config <<EOF

Host 192.168.19.234
  HostName 192.168.19.234
  PreferredAuthentications publickey
  Port 22
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
CADVISOR_USERNAME=admin
CADVISOR_PASSWORD_HASH=$$.....
```

password hashes can be generated with:

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

- hashcat

```
-------------------------------------------------------------
* Hash-Mode 22000 (WPA-PBKDF2-PMKID+EAPOL) [Iterations: 4095]
-------------------------------------------------------------

Speed.#1.........:   570.3 kH/s (50.78ms) @ Accel:8 Loops:1024 Thr:512 Vec:1
```

> approximately 2.92 minutes to brute-force an 8-digit numeric password

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

- install metadata provider plugins from the Jellyfin UI

- copy new media files to the bind mount on the host, keeping the same directory structure:

```bash
scp -r ./temp/_video/1 root@192.168.19.237:/mnt/media/video/
```

### juice (GPU over IP)

to run inference or rendering tasks on the server's GPU from other machines on the network.

server that is used in this stack: https://hub.docker.com/layers/juicelabs/server/11.8-2023.08.10-2103.0633b794/images/sha256-adfd824014a2425cefe9412f9bae30c25151facff158fcced5d2bb2f8c911a3b

- rendering tasks are available for windows client only https://github.com/Juice-Labs/Juice-Labs/releases/download/2023.08.10-2103.0633b794/JuiceClient-windows.zip

Edit juice.cfg

```
{
  "servers": ["192.168.19.234:43210"],
  "logGroup": "info",
  "logFile": "juice.log",
  "forceSoftwareDecode": false,
  "disableCache": true,
  "disableCompression": false,
  "headless": false,
  "allowTearing": false,
  "frameRateLimit": 0,
  "framesQueueAhead": 3,
  "swapChainBuffers": 3,
  "waitForDebugger": false,
  "acccessToken": ""
}
```

- how to run: https://github.com/Juice-Labs/Juice-Labs/wiki/Run-Juice

- test with `juicify vkcube` for rendering tasks

- running pytorch code on the server's GPU from a client machine, Linux example:

```bash
wget https://github.com/Juice-Labs/Juice-Labs/releases/download/2023.08.10-2103.0633b794/JuiceClient-linux.tar.gz
tar -xvf JuiceClient-linux.tar.gz -C ~/JuiceClient-linux
cd ~/JuiceClient-linux
sed -i 's/"servers": \["127.0.0.1:43210"\]/"servers": \["192.168.19.234:43210"\]/' juice.cfg
conda create --name pytorch_env python=3.10 -y
conda activate pytorch_env
conda install pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia -y
./juicify python -c "import torch; print(torch.__version__); print(torch.version.cuda) ; print(torch.backends.cudnn.version()); print(torch.cuda.get_arch_list())"
```

the firewall must be set up accordingly to limit access to the server, there is no authentication in Juice server. The client supports setting an access token, but I could not find how to set the token on the server.

> [!WARNING]
> work in progress

### SCUDA

https://github.com/kevmo314/scuda

another GPU over IP solution, mainly for running code that uses CUDA, Linux server and Linux client.

- copy the client to the client machine running Ubuntu 24.04.

```bash
cd ~
ssh $USER@192.168.19.234 'docker cp scuda:/scuda/libscuda_12.6.so ~/libscuda_12.6.so'
scp $USER@192.168.19.234:~/libscuda_12.6.so ~/libscuda_12.6.so

export SCUDA_SERVER=192.168.19.234 SCUDA_PORT=14833
LD_PRELOAD=~/libscuda_12.6.so nvidia-smi
```

<br><img src="./pictures/Screenshot%20from%202025-01-22 08-45-48.png" alt="screenshot" width="50%"><br>

I yet to make it work with pytorch

> [!WARNING]
> work in progress

### cadvisor

navigate to http://192.168.19.234:8080/

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
- https://github.com/google/cadvisor
