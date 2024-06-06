---
header: 
  overlay_image: /assets/posts/2024-06-06-making-a-production-kubernetes-cluster/overlay.png
---

# Building a production-ready Kubernetes Cluster

## The situation

So, our club had been in a pinch for a while. We had:
- 5 rarely-used [beasts of machines](https://www.asus.com/us/site/gaming/rog/gaming-desktops/strix-g35.html) idling on campus
- An overzealous IT department that blocks all outgoing traffic except on port 80 and 443

With these restrictions in mind, it would seem kinda insane to want to host anything on these PCs, but we couldn't just leave them lying around when cloud hosting is so insanely expensive these days. So, I set about to make a production-ready cluster that would be stable enough to host a large event like Hack@AC.

## Chapter 1: Choosing a Linux Distribution

Right off the start, we had a couple of candidates we wanted to try:
- Ubuntu Server ("default" distribution, industry standard, most commonly used)
- Rocky Linux (RHEL, but community driven. Also an industry standard)
- NixOS (Relatively new, used for immutable, reproducible and declarative production environments)

Straight off the bat, Ubuntu server was kicked to the back of the list of distros I wanted to try. 
The main reason for this was the presence of [Snap](https://snapcraft.io/). Snap is [Canonical](https://canonical.com/)(company that made ubuntu)'s software packaging and sandboxing solution.
- Snaps are a giant waste of disk space.[^1]
- Snaps are slow.[^1]
- The Snap backend is proprietary, which goes against my stance of using fully-free and open source software.[^1]

While Snap may be a good thing for a normal end user, it clutters up a production environment and is considered bloat.

NixOS was the next obvious contender to me. I'd always wanted to see what the hype was about, and reproducible build environments sounded epic, so I downloaded it on one of the machines to try it out. However, I was immediately faced with a couple of pitfalls that Nix had.
- It's only as reproducible as the end user configures it to be.
- It took me a couple of hours to figure out the language, as the documentation is pretty bad.
- For packages like k3s, the configuration is not centralised and cannot be configured like usual, and the documentation for individual packages is horrible.
- NixOS does not follow the Unix Filesystem Hierarchy Standard, resulting in confusion amongst system administrators about the locations of things like configuration files.

All these things made me change my mind about running NixOS, and as such, I turned to my last contender, Rocky Linux.

It was **great**.

It had:
- Great documentation
- Great community support
- Stability

I would recommend it to anyone looking to setup a production server in a pinch. The setup was really easy and fast, however, I was faced with a couple of problems after installing.

The kernel log would be spammed with PCIe Bus error messages. After searching around, I found that I had to set the `pcie_aspm=off` kernel parameter. "Easy", I thought, and I edited /etc/default/grub to and ran `grub-mkconfig -o /boot/grub2/grub.cfg`, rebooted, and to my despair, I saw that the issue had not been fixed. Poking around in the grub editor, I didn't see the kernel parameter set. This was on the day of the DuckDuckGo outage, and as such, it took me 30 minutes to find this [post](https://forums.rockylinux.org/t/grub2-mkconfig-rl9-3-fresh-install-bug/12118/8) on the Rocky Linux forum, and fixed the issue with the `--update-bls-cmdline` parameter in the `grub-mkconfig` command. This quirk, paired with the unusual path to the grub configuration, really frustrated me, but it was a one time thing and a small nitpick, so I do still think it's a good distribution.

## Chapter 2: Remote Access

> "Remote access is a pain in the ass" - Unknown

In our case, the above quote really holds true. Our PCs were in a surveilled network with heavily restricted inbound and outbound traffic. My goto was [frp](https://github.com/fatedier/frp), being a favourite of my seniors and software I had experience with, it was easy and quick to setup. However, this could not last for long. FRP has problems, mainly being notoriously slow and compute-intensive. This was a temporary solution though, and I wanted to pivot to a VPN based solution sometime in the future.

For VPNs, there were 2 obvious contenders: Wireguard and OpenVPN.

I went for wireguard first. I will not detail the setup process here, but it was absolutely HORRIBLE for a first-time user. I would NOT recommend setting up a wireguard server to an inexperienced sysadmin. Debugging the server and the client configuration took no less than a week, but I finally got it done. I set it up and IT WORKED. I got up and went to eat dinner after a long day of doing nothing but configuring wireguard, and when I came back, I was crestfallen to find out that I had lost connection. 

This is one of the major pitfalls of wireguard: It uses UDP and has no support for TCP, making it unable to masquerade as normal network traffic. The IT Department had shut it down so quickly for how long the configuration took.

So, I set off configuring OpenVPN. As compared to wireguard, it only took me around 2 hours of configuring and debugging TLS issues before getting it up and running, and the setup was REALLY easy. The only thing a junior sysadmin should know before trying to set it up is knowledge of iptables, which has to be manually configured. As of writing this post, it is working like a charm.

## Chapter 3: K3s

Thankfully, the setup process for `k3s` was a breeze. 

```
# Master node
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -

# Agent node
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent" sh -s
```

For the agent nodes, getting the token from `/var/lib/rancher/k3s/server/token` in the master node and putting it in `/etc/rancher/k3s/config.yaml` was all it took.

After this, I setup DHCP reservations for all of the nodes, and put them in the `/etc/hosts` files of all the nodes.

### Setting up an image registry

In order to setup custom images, one has to setup a private image registry.
Installing docker and docker-compose with the following compose file allows us to setup an **insecure** registry on the master node.

Make the directories `/var/lib/registry` and `/etc/registry/auth`

```yaml
services:
  registry:
    image: docker.io/registry:2
    restart: always
    ports:
      - 5000:5000
    volumes:
      - /var/lib/registry:/var/lib/registry
      # - /etc/registry/certs:/certs # Uncomment for secure registry
      - /etc/registry/auth:/auth
    environment:
      - REGISTRY_AUTH=htpasswd
      - REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd
      - REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm

```

Running the following will setup the auth/htpasswd file for you.
```sh
set -h
read -p "Password: " password
docker run --entrypoint htpasswd httpd:2 -Bbn <username> $password > auth/htpasswd
set +h
```

Put this into `/etc/rancher/k3s/registries.yaml` to configure the insecure registry:
```
mirrors:
  "<hostname>:5000":
    endpoint:
      - http://rgb1:5000/v2

configs:
  "<hostname>:5000":
    auth:
      username: "<username>"
      password: "<password>"
    tls:
      insecure_skip_verify: true
```

and voila! I had a registry that I could push images to.

## Conclusion

How well does it run? A friend proposed deploying `distcc` to all of the nodes to benchmark how fast it could compile the linux kernel.

I built the distcc image with the following Dockerfile:
```Dockerfile
FROM debian:latest

RUN DEBIAN_FRONTEND="noninteractive" apt-get -q update &&\
    DEBIAN_FRONTEND="noninteractive" apt-get -y -q --no-install-recommends upgrade &&\
    DEBIAN_FRONTEND="noninteractive" apt-get install -y -q g++ gcc clang distcc bc binutils bison dwarves flex gcc git gnupg2 gzip libelf-dev libncurses5-dev libssl-dev make openssl pahole perl-base rsync tar xz-utils build-essential curl &&\
    DEBIAN_FRONTEND="noninteractive" apt-get -y -q autoremove &&\
    DEBIAN_FRONTEND="noninteractive" apt-get -y -q clean

RUN update-distcc-symlinks
RUN ln -sf /usr/bin/distcc /usr/lib/distcc/x86_64-pc-linux-gnu-gcc 
ENV ALLOW 192.168.0.0/16
# RUN useradd distcc
# USER distcc
EXPOSE 3632

CMD distccd --jobs $(nproc) --log-stderr --no-detach --daemon --allow ${ALLOW} --log-level debug
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: distcc
---
apiVersion: v1
kind: Pod
metadata:
  name: distcc-rgb1
  namespace: distcc
  labels:
    app: distcc
    node: rgb1
spec:
  nodeSelector:
    kubernetes.io/hostname: rgb1
  containers:
    - name: distcc
      image: rgb1:5000/distcc
      ports:
        - containerPort: 3632
          protocol: TCP
      env:
        - name: ALLOW
          value: 10.0.0.0/8
---
apiVersion: v1
kind: Pod
metadata:
  name: distcc-rgb2
  namespace: distcc
  labels:
    app: distcc
    node: rgb2
spec:
  nodeSelector:
    kubernetes.io/hostname: rgb2
  containers:
    - name: distcc
      image: rgb1:5000/distcc
      ports:
        - containerPort: 3632
          protocol: TCP
      env:
        - name: ALLOW
          value: 10.0.0.0/8
.
.
.
---
apiVersion: v1
kind: Service
metadata:
  name: distcc-rgb1
  namespace: distcc
spec:
  type: NodePort
  ports:
  - name: distcc-rgb1
    port: 8001
    nodePort: 8001
    targetPort: 3632
  selector:
    node: rgb1

---
apiVersion: v1
kind: Service
metadata:
  name: distcc-rgb2
  namespace: distcc
spec:
  type: NodePort
  ports:
  - name: distcc-rgb2
    port: 8002
    nodePort: 8002
    targetPort: 3632
  selector:
    node: rgb2

```

This deploys distcc to all of the nodes on my network (if you wish to run this, adapt the number of pods and the hostnames of the registry and the pods.)

Running the same image locally and mounting a volume with the source code of the Linux kernel, you can export the following environment variables:
```sh
export DISTCC_HOSTS="host:8001/<cores> host:8002/<cores> host:8003/<cores> host:8004/<cores>"
export PATH="/usr/lib/distcc/bin:$PATH"
```
and in the source directory run:
```sh
make defconfig
time make -j<total number of cores> CC=distcc CXX=distcc
```

Timing this process, I got a number of roughly 13 minutes and 20 seconds, and I can proudly say our kubernetes cluster can do 4.5 linux-6.9.3 compiles per hour!

[^1]: https://hackaday.com/2020/06/24/whats-the-deal-with-snap-packages/


