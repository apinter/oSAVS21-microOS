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
- No YaST, but there is Cockpit ;))
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
# podman run -d --name routing --label io.containers.autoupdate=image \
-p 80:80 \
-p 443:443 \
-v nginx:/etc/nginx:ro \
-v osc-www:/srv/osc-www:ro \
-v /etc/letsencrypt:/etc/letsencrypt:ro docker.io/nginx:mainline`
```
### Deploy a rootless `mariadb` server

```bash
$ podman run -d --name sx5-db --label io.containers.autoupdate=image \
-v /home/podman_user/var-lib-mysql:/var/lib/mysql:Z \
docker.io/mariadb:latest
```
### Discuss
- `--label io.containers.autoupdate=image`
- `:Z`
- `:ro`
---
layout: two-cols
---
### Building your own containers

::right:: 

### make file

---
layout: center
---
## Should you run this manually all the time?
Probably not...
### CI/CD

---
layout: center
---
## Gitlab CI, configuration at OSC
### Discuss

---
layout: center
--- 
### Gitlab CI

- Daily builds of custom containers
- Automated testing
- Automated deployment to staging and prod env
- Actions per MR/Push

---
layout: center
---
### Ansible congifuration management


- YAML
- Easy to extend with custom modules
- Agentless
- Ansible-vault for storing and managing secrets
- Centralized inventory
- AWX and ARA

---
layout: two-cols
---
### Example pipeline

```bash
image: opensuse/tumblweed

stages:     
  - build
  - test # there will be one for sure at some point 

build-job:      
  stage: build
  before_script:
    - zypper ref
    - zypper install -y npm-default nodejs-common python38 python38-pip openssh-clients
    - pip install ansible
    - echo "mywww ansible_host=myhost.com ansible_user=ansible" > /inventory.ini
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh; chmod 700 ~/.ssh
    - mv /builds/host/portal/ansible_deploy.yml /ansible_deploy.yml
  script:
    - cd /builds/host/portal/wp-content/themes/host
    - npm install
    - npm run build
    - ansible-playbook -i /inventory.ini /ansible_deploy.yml
```
::right::
### Build output

```bash
dist/menu-arrow-purple.86f417ef.svg                                      246 B     690ms
dist/mobile-menu-arrow-gray.fbf6b7c2.svg                                 246 B     695ms
dist/single-page-header-mobile.f2e730d5.svg                              243 B     711ms
dist/partnerships-customer-lifecycle-mobile-arrow.2a60f611.svg           242 B     1.40s
dist/partnerships-customer-lifecycle-arrow.6474d63b.svg                  236 B     1.38s
dist/down-arrow-menu.ea22cf62.svg                                        231 B     580ms
dist/releases-quote-mobile-bg.d74f87e6.svg                               227 B     806ms
dist/gray-down-arrow-menu.f81a83b6.svg                                   224 B     580ms
dist/industry-ebook-mobile-bg.a19cb7d7.svg                               223 B     1.31s
dist/footer-arrow.e4d01d0f.svg                                           223 B     487ms
dist/industry-case-study-foodbeverage-bg-mobile.24fa8d71.svg             222 B     1.32s
dist/industry-case-study-mobile-bg.b389a54b.svg                          222 B     1.30s
dist/back-submenu.f4afcb2a.svg                                           216 B     711ms
dist/coaltion-gray-bg.01f7d70f.svg                                       171 B     1.68s
dist/gray-bg.face3d8d.svg                                                171 B     1.67s
$ touch -a /builds/host/portal/wp-content/themes/host/dist/test
$ ansible-playbook -i /inventory.ini /ansible_deploy.yml
PLAY [host.com deployment] ***************************************************
TASK [Deploy the updated WP] ***************************************************
Cleaning up file based variables
00:01
Job succeeded
```
---
layout: center
---
## Conclusion

- Rolling in production is a good thing
- There are no more negativity then running "stable"
- Use automtaion wherever it is possible
- Use configuration management 
- Keep learning

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

