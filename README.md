# Systemctl hack for wsl 2

**Auto-start/services** (`systemctl` for wsl)

**1. Install dependencies**\
Ubuntu:
```
$ sudo apt update
$ sudo apt install dbus policykit-1 daemonize
```
Arch Linux (yay recommended):
```
$ yay -Syyu
$ yay -S dbus polkit daemonize
```
**2. Create fake-bash**
```
$ sudo touch /usr/bin/fbash
$ sudo chmod +x /usr/bin/fbash
$ sudo editor /usr/bin/fbash
```
Add the following, be sure to replace `<YOURUSER>` with your WSL2 Linux username
```
#!/bin/bash
# your WSL2 username
UNAME="<YOURUSER>"

UUID=$(id -u "${UNAME}")
UGID=$(id -g "${UNAME}")
UHOME=$(getent passwd "${UNAME}" | cut -d: -f6)
USHELL=$(getent passwd "${UNAME}" | cut -d: -f7)

if [[ -p /dev/stdin || "${BASH_ARGC}" > 0 && "${BASH_ARGV[1]}" != "-c" ]]; then
    USHELL=/bin/bash
fi

if [[ "${PWD}" = "/root" ]]; then
    cd "${UHOME}"
fi

# get pid of systemd
SYSTEMD_PID=$(pgrep -xo systemd)

# if we're already in the systemd environment
if [[ "${SYSTEMD_PID}" -eq "1" ]]; then
    exec "${USHELL}" "$@"
fi

# start systemd if not started
/usr/sbin/daemonize -l "${HOME}/.systemd.lock" /usr/bin/unshare -fp --mount-proc /lib/systemd/systemd --system-unit=basic.target 2>/dev/null
# wait for systemd to start
while [[ "${SYSTEMD_PID}" = "" ]]; do
    sleep 0.05
    SYSTEMD_PID=$(pgrep -xo systemd)
done

# enter systemd namespace
exec /usr/bin/nsenter -t "${SYSTEMD_PID}" -m -p --wd="${PWD}" /sbin/runuser -s "${USHELL}" "${UNAME}" -- "${@}"
```
**3. Set the fake-bash as our root user's shell**\
Edit the /etc/passwd file:
```
$ sudo editor /etc/passwd
```
Find the line starting with `root:`, it should be the first line. Change it to:
```
root:x:0:0:root:/root:/usr/bin/fbash
```
*Note the `/usr/bin/fbash` here, slight difference*\
Save and close this file.\
**4. Restart wsl**\
Exit out of wsl shell\
Ubuntu:
```
> wsl --shutdown
> ubuntu1804.exe config --default-user root
```
Arch Linux:
```
> wsl --shutdown
> arch.exe config --default-user root
```
Done. Now test systemctl with:
```
$ systemctl is-active dbus
active
```
---
# Install Docker and config docker
**1. Install docker**\
Ubuntu:\
*Follow this official guide: https://docs.docker.com/engine/install/ubuntu/* \
Arch Linux: (yay recommended)
```
$ yay -S docker
```
**2. Start docker service**
```
$ sudo systemctl start docker
$ sudo systemctl enable docker
```
**3. Add user to docker group to run without sudo**
```
$ sudo groupadd docker
$ sudo usermod -aG docker <Your username>
```
---
# Config wsl environment (Optional)
**Add wsl.conf to Wsl**\
Save the following file as `/etc/wsl.conf` on your Wsl host:
```
[automount]
root = /
options = "metadata"
```
This will mount all of your mapped drives to a corresponding letter in the root of your Wsl filesystem. This is what will allow you the "native" experience of editing file on Windows and running them from your windows or wsl terminals. \
**Config docker to run from Windows**\
Create the following file in Wsl `/etc/docker/daemon.json`
```
{
    "hosts": ["unix://", "tcp://0.0.0.0:2375"],
    "experimental": true
}
```
Set docker env var in Windows
| Env Var | Value | Required/Optional |
| --------| ----- | ----------------- |
| `DOCKER_HOST` | `tcp://localhost:2375` | Required |
| `DOCKER_CLI_EXPERIMENTAL` | `enabled` | Optional |
| `DOCKER_API_VERSION` | `1.40` | Optional |

Install docker client in Windows from 
```
https://dockermsft.blob.core.windows.net/dockercontainer/docker-19-03-1.zip
```
Test docker in Windows with
```
> docker version
```
You should see output like this
```
Client: Docker Engine - Community
 Version:           19.03.4
 API version:       1.40
 Go version:        go1.12.10
 Git commit:        9013bf583a
 Built:             Fri Oct 18 15:54:09 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.4
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.10
  Git commit:       9013bf583a
  Built:            Fri Oct 18 15:52:40 2019
  OS/Arch:          linux/amd64
  Experimental:     true
 containerd:
  Version:          1.2.10
  GitCommit:        b34a5c8af56e510852c35414db4c1f4fa6172339
 runc:
  Version:          1.0.0-rc8+dev
  GitCommit:        3e425f80a8c931f88e6d94a8c831b9d5aa481657
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```
