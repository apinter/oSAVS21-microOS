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
layout: center
---

## This is a server focused talk
### For MicroOS desktop check:
- [Dario's collection](https://dariofaggioli.wordpress.com/2021/06/18/microos-as-your-desktop-prime-time/) `https://dariofaggioli.wordpress.com/2021/06/18/microos-as-your-desktop-prime-time/` and [talk](https://www.youtube.com/watch?v=6F7iCntjWB8) `https://www.youtube.com/watch?v=6F7iCntjWB8`
- [Richard's talk](https://www.youtube.com/watch?v=cZLckDUDYjw) `https://www.youtube.com/watch?v=cZLckDUDYjw`

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
- Combustion and Ignition support
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
### src: [Health-checker Github](https://github.com/kubic-project/health-checker/blob/master/plugins/crio.sh) `https://github.com/kubic-project/health-checker/blob/master/plugins/crio.sh`
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
`https://kubic.opensuse.org/documentation/man-pages/org.opensuse.RebootMgr.conf.8.html`

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
```bash
$ podman build -t [tag] .
```
::right:: 
### make file
```bash
FROM registry.opensuse.org/opensuse/tumbleweed
RUN zypper ref; zypper dup -y; \
    zypper in -y coturn sudo 
COPY coturn /etc/coturn
RUN chown -R coturn:coturn /etc/coturn
EXPOSE 5349 5440
CMD turnserver -c /etc/coturn/turnserver.conf
```
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
### Example container build

```bash
coturn-sx5-build-master:
  variables:
    OSC_REGISTRY: registry.openstorage.io
    OSC_CONTAINER_NAME: coturn-sx5
  before_script:
    - podman info
  script:
    - podman pull registry.opensuse.org/opensuse/tumbleweed
    - podman build -t $OSC_REGISTRY/$OSC_CONTAINER_NAME .
    - podman push $OSC_REGISTRY/$OSC_CONTAINER_NAME --authfile $OSC_REGISTRY_AUTHFILE
    - podman rmi $OSC_REGISTRY/$OSC_CONTAINER_NAME
  only:
    - master
  retry: 2
```
::right::
### Build output

```bash
$ podman push $OSC_REGISTRY/$OSC_CONTAINER_NAME --authfile $OSC_REGISTRY_AUTHFILE
Getting image source signatures
Copying blob sha256:1ff05b98b1b9982471103ab81f0c16112cabf1d93792d16c5522d416c1d09bb6
Copying blob sha256:dbc5f6b930257df4f84d3f385e4c530106105f58e61761f4c9a67c2eb34052af
Copying blob sha256:2366142f151499d3461849cae503d599c81eaaece4a4becdbafb6d444964c5eb
Copying blob sha256:9d7a5c4b0dd37014713a0175d1fb97b690f7cc610b5a9d86b06bdfeeb059031b
Copying config sha256:39d8eeecb72633d9e9409ddcf056cac6f5e32e2bc418aa63f02e47f45a8185ca
Writing manifest to image destination
Storing signatures
$ podman rmi $OSC_REGISTRY/$OSC_CONTAINER_NAME
Untagged: registry.openstorage.io/coturn-sx5:latest
Deleted: 39d8eeecb72633d9e9409ddcf056cac6f5e32e2bc418aa63f02e47f45a8185ca
Cleaning up file based variables
00:00
Job succeeded
```

---
layout: two-cols
---
### Something a little more complex

```bash
image: opensuse/leap

stages:     
  - build
  - deploy
  - test # there will be one for sure at some point 

build-job:
  variables:
    GIT_STRATEGY: fetch
    GIT_CHECKOUT: "false" 
  stage: build
  before_script:
    - zypper ref
    - zypper in npm-default
  script:
    - cd wp-content/themes/host
    - npm install
    - npm run build
  after_script:
    - tar czvf portal.tar.gz .
```
::right::
### Deploy Stage
```bash
deploy-job:
  variables:
    GIT_STRATEGY: fetch
    GIT_CHECKOUT: "false"
  stage: deploy
  before_script:
    - zypper install -y python38 python38-pip openssh-clients
    - pip install ansible
    - echo "hostwww ansible_host=host.com ansible_user=ansible" > /inventory.ini
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh; chmod 700 ~/.ssh
    - mv /builds/host/portal/ansible_deploy.yml /ansible_deploy.yml
  script:
    - ansible-playbook -i /inventory.ini /ansible_deploy.yml
```
---
layout: center
---
### Build output

```bash
./wp-includes/widgets/class-wp-widget-tag-cloud.php
./wp-includes/widgets/class-wp-widget-text.php
./wp-includes/wlwmanifest.xml
./wp-includes/wp-db.php
./wp-includes/wp-diff.php
./wp-links-opml.php
./wp-load.php
./wp-login.php
./wp-mail.php
./wp-settings.php
./wp-signup.php
./wp-trackback.php
./xmlrpc.php
./ansible_deploy.yml
./.gitlab-ci.yml
./bltest
./portal.tar.gz
tar: ./portal.tar.gz: file changed as we read it
Cleaning up file based variables
00:02
Job succeeded
```

---
layout: center
---
### The play
```bash
---
 - name: Deployment
   gather_facts: false
   become: true
   hosts: hostxyz
   tasks:
   - name: Deploy
     ansible.builtin.copy:
       src: /builds/host/portal/
       dest: /var/lib/test_run/
       owner: 33
       group: 33
       backup: yes
   - name: Delete .git
     ansible.builtin.file:
       path: /var/lib/test_run/.git
       state: absent
```

---
layout: center
---
## Conclusion

- Rolling in production is a good thing
- There is no more negative then running "stable"
- Use automtaion wherever it is possible
- Use configuration management 
- Implement monitoring (maybe next time)

---
<h1 align="center">Thank you</h1>
<head>
  <link href="/fontawesome/css/all.css" rel="stylesheet">
</head>
<body>
<br>   <i class="fas fa-envelope"> adathor@opensuse.org</i>
<br>   <i class="fas fa-envelope"> attila@openstorage.io</i>
<br>   <i class="fab fa-github"> adathor@opensuse.org</i>
<br>   <i class="fab fa-twitter"> @adathorium</i>
<br>

### The presentation source is available at https://github.com/apinter/oSAVS21-microOS

<img src="/osc.png" class="m-20 h-40 rounded shadow" align="right"/>
<img src="/os.png" class="m-20 h-40 rounded shadow" align="left" />
</body>

