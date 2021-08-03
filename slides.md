---
theme: 'default'
download: true
colorSchema: 'dark'
aspectRatio: '16/9'
highlighter: 'prism'
lineNumbers: true
fonts:
  sans: 'Roboto'
  serif: 'Roboto Slab'
  mono: 'Fira Code'

layout: cover
background: 'wp.jpg'
---

# Automate everything with MicroOS (and DevOps)
##### Or how I should never name anything

<img src="/osc.png" class="m-20 h-40 rounded shadow" align="right"/>
<img src="/os.png" class="m-20 h-40 rounded shadow" align="left" />

---
layout: intro
---
# Whoami
<img src="/adathor.png" class="m-40 h-70 rounded shadow" align="right"/>
<br>

- Attila Pinter aka Adathor 
- Father, husband, DevOps eng. 
- FOSS advocate
- openSUSE docs
- openSUSE Telegram
- Tumbleweed user
- CTO and founder at OpenStorage.io

For MicroOS Desktop talks:
- [Dario's collection](https://dariofaggioli.wordpress.com/2021/06/18/microos-as-your-desktop-prime-time/) and [talk]()
- [Richard's talk](https://www.youtube.com/watch?v=cZLckDUDYjw)

---
layout: intro
---
# OpenStorage
<img src="/osc.png" class="m-40 h-40 rounded shadow" align="right"/>

- Est. 2014 in Indonesia
- FOSS company
- Hybrid cloud storage
- . . . 
- Everything runs on openSUSE Tubleweed and MicroOS

--- 
layout: center
---
# Why MicroOS?

I am super lazy... (mostly)

---
layout: center
---
## No (yes), but really..
<br>

- Tumbleweed base
- Simple, minimal, and sleek
- Container host by design
- Transactional-updates, health-checker, rebootmgr 
- Not your typical server os

---
layout: center
---
# OMG iS ThAt A rOllInG DiStrO iN ProD?!

<style>
h1 {
  color: red
}
</style>

---
layout: center
---
### *Heavy sigh* 
## Just hear me out... 

---
layout: center
---
# openSUSE rolling != everyone else

- Feauters now Vs. years from here Vs. backport craze?
- _OpenQA_
- _snapper_
- __health-checker__

---
layout: center
---

## Create a health-checker plugin

---
layout: two-cols
---

```bash
#!/bin/bash

run_checks() {
    # Check first if it is installed:
    rpm -q --quiet cri-o
    test $? -ne 0 && return

    # ignore if crio is not enabled.
    systemctl is-enabled -q crio
    test $? -ne 0 && return

    systemctl is-failed -q crio
    test $? -ne 1 && exit 1
}

stop_services() {
    systemctl stop crio
}
```
::right::

```bash

case "$1" in
    check)
	run_checks
	;;
    stop)
	stop_services
	;;
    *)
	echo "Usage: $0 {check|stop}"
	exit 1
	;;
esac

exit 0
```
### src: [Health-checker Github](https://github.com/kubic-project/health-checker/blob/master/plugins/crio.sh)
---
layout: center
---
## Let's talk security

- What is your update strategy?
- ro `/` less attack surface 
- AppArmor vs SELinux

---
layout: center
---
## Set a maintenance window
### Avoid multiple nodes to restart at the same time

---
layout: center
---

`vim /etc/rebootmgr.conf`

```bash
[rebootmgr]
window-start=03:30
window-duration=1h30m
strategy=best-effort
lock-group=default
```    
<br>

#### [rebootmgrd man](https://kubic.opensuse.org/documentation/man-pages/org.opensuse.RebootMgr.conf.8.html)

---
layout: center
---

## Changing `t-u` update schedule
#### it's a hack, but hey...
---

`systemctl edit --full transactional-update.timer`
```bash
[Unit]
Description=Daily update of the system
Documentation=man:transactional-update(8)
After=network.target local-fs.target

[Timer]
OnCalendar=daily
AccuracySec=1m
RandomizedDelaySec=2h
#Persistent=true

[Install]
WantedBy=timers.target
```

---
layout: center
---

## Quick sum/ what happened so far>
- Automated updates
- Automated health-checks
- Automated rollbacks

---
layout: center
---
## What's next?
### Applications deployment

---
layout: center
---

## MicroOS should do one thing and no more
### OSC is using it for hosting containers

---
layout: center
---

### Podman

---
layout: two-cols
---
## Why Podman
<br>
<br>

- Fully OCI compliant
- Can use Docker hub images as those are OCI compliant
- Same/similar commands as Docker
- Uses `buildah` to build container images
- Implementation of the `pod` concept (export a pod from `podman` and import in k8s)
- Systemd integration 
- If you really must you can run systemd inside the containers
- Works better with `firewalld` (a lot better...)
- __Rootless and daemonless container runtime__
- __Auto update images__

---

### Deploy `nginx` with ro access to certs and content
```bash
$ podman run -d --name routing --label io.containers.autoupdate=image \
-p 80:80 \
-p 443:443 \
-v nginx:/etc/nginx:ro \
-v osc-www:/srv/osc-www:ro \
-v /etc/letsencrypt:/etc/letsencrypt:ro nginx:mainline`
```

---
<h1 align="center"> Thank you</h1>
<head>
  <link href="/fontawesome/css/all.css" rel="stylesheet">
</head>
<body>
<br>   <i class="fas fa-envelope"> adathor@opensuse.org</i>
<br>   <i class="fas fa-envelope"> attila@openstorage.io</i>
<br>   <i class="fab fa-github"> adathor@opensuse.org</i>
<br>   <i class="fab fa-twitter"> @adathorium</i>
<br>

<img src="/osc.png" class="m-20 h-40 rounded shadow" align="right"/>
<img src="/os.png" class="m-20 h-40 rounded shadow" align="left" />
</body>

