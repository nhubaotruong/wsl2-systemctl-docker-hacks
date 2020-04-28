# wsl2
wsl2 systemctl hack and docker config

**Auto-start/services** (`systemctl` for wsl)

**1. Install dependencies**
Ubuntu and Debian:
```
$ sudo apt update
$ sudo apt install dbus policykit-1 daemonize
```
Arch Linux (yay recommended):
```
$ sudo yay -Syyu
$ sudo yay -S dbus polkit daemonize
```
**2. Create fake-bash**
```
$ sudo touch /usr/bin/bash
$ sudo chmod +x /usr/bin/bash
$ sudo editor /usr/bin/bash
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
**3. Set the fake-bash as our root user's shell**
Edit the /etc/passwd file:
```$ sudo editor /etc/passwd```
Find the line starting with `root:`, it should be the first line. Change it to:
`root:x:0:0:root:/root:/usr/bin/bash`
*Note the `/usr/bin/bash` here, slight difference*
Save and close this file.
