# dockercr

Set up a docker container registry using just SSH and docker. No TLS, no CA, no
HTTPS, no PA or other crap required.

If you only want to quickly push stuff to 1 production host, see [the next
section](#pushing-docker-images-to-production---like-a-boss).

If you want to host a docker registry on its own dedicated host, skip the next
section and go straight [to the real
stuff](#setting-up-an-ssh-secured-docker-container-registry-on-a-dedicated-host).

Free [ghcr.io](https://ghcr.io) might be too limiting for some of us as there is
a 10GB download limit per month for downloads from outside of GH actions.

GitHub personal access tokens and stuff can be a pain. TLS certificates and CAs
for a self-hosted registry - we don't even bother with. We already have SSH for
shell access, which is perfectly able to key-authenticate and secure all our
connections.

So instead of following the manual, we set up a local docker registry and handle
the rest via SSH.

## Pushing docker images to production - like a boss

- set up SSH port forwarding of 5000:5000 to PROD in `.ssh/config`
- ssh into the PROD machine to activate the port forwarding
- Start a docker registry image on port 5000 on PROD:
  - see [the docs](https://docs.docker.com/registry/deploying)
  - `docker run -d -p 5000:5000 -e REGISTRY_HTTP_ADDR=127.0.0.1 --restart=always --name registry registry:2`
  - note that we make sure we only expose it to the loopback interface
- locally, tag your image for upload to `localhost:5000`:
  - `docker tag whatever:latest localhost:5000/whatever`
- push it
  - `docker push localhost:5000/whatever`
- on PROD: pull the image:
  - `docker pull localhost:5000/whatever`
- **TA-DAH**! SSH-authenticated docker registry!

If we're really fancy, we add `boss.cr.io 127.0.0.1` into `/etc/hosts` of our
client and PROD - and push to and pull from `boss.cr.io:5000`.

As the registry keeps running, we can just keep pushing and pulling from now on.
The only thing we need to remember is to SSH into PROD before pushing. But since
we're going to pull later anyway, this should not be a big deal.

### Improvements and Variants

#### Redirecting PROD to client

Yes, we could reverse-forward (`-R`) PROD:5000 to our client's :5000 - and start
the registry locally. That would guarantee we always have an SSH tunnel before
we pull. But, who knows, if we host the registry on a 3rd machine - we probably
want to keep it the `-L` way.

## Setting up an SSH-secured Docker Container Registry on a dedicated host

Imagine we host the registry on a third machine called `REGHOST` with IP
`88.88.88.88`. Now, both our client when pushing and the PROD machine when
pulling need to establish an SSH-connection with port forwarding to the REGHOST.

### On the REGHOST

We prepare a special user that runs our docker registry (optional):

```shell
# add a user with home dir, add to docker group
sudo useradd -m dockercr -G docker
sudo passwd dockercr

# become the dockercr user
sudo su - dockercr

# set up a basic .ssh folder
mkdir .ssh
chmod 700 .ssh
touch .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
```

Now, run the docker registry, binding to the loopback interface, on port 5000:

```shell
# we're still the dockercr user
docker run -d -p 5000:5000 -e REGISTRY_HTTP_ADDR=127.0.0.1 \
       --restart=always --name registry registry:2
```

### On all clients (pushing, pulling images)

**Note:** We will not use the `.ssh/config` file this time, as we will use a
script anyway. Distributing all-in-one scripts is easier than doing so with
scripts plus ssh configs. Feel free to separate SSH options out into the config
file.

The basic idea is to establish the SSH tunnel before any docker push or pull
operation. SSH's default port-forwarding behavior is suited well for this. From
the SSH man page:

> The session terminates when the command or shell on the remote machine exits
> and all X11 and TCP connections have been closed.

In addition, SSH won't get in our way when we ask it to forward when the
requested tunnel is already established.

That means:

- SSH will always wait until the last push/pull has finished
- SSH will cause no trouble attempting to establish an already existing tunnel

#### Preparations

For added coolness, we run the following on all involved machines:

```shell
sudo 'echo "cr.boss.org 127.0.0.1" >> /etc/hosts"'
```

This gets us the fictional registry host `cr.boss.org`.

For security reasons, we are going to create a dedicated SSH key pair for
container registry communication and register it with the REGHOST.

```shell
ssh-keygen -t rsa -f .ssh/id_rsa_dockercr
REGHOST=88.88.88.88
ssh-copy-id -i .ssh/id_rsa_dockercr dockercr@$REGHOST
```

It is up to you if you want to provide a passphrase.

#### Pushing and pulling

Our new docker-push command:

```shell
#!/usr/bin/env bash
if [ $# -lt 1 ] ; then
    echo "Please specify a container"
    exit 1
fi

# Config
REGHOST=88.88.88.88             # put ip / name of REGHOST here
REG_ALIAS=cr.boss.org           # our fake repository name
REG_KEY=$HOME/.ssh/id_rsa_boss  # private key used for SSH-auth


ssh -f -L 5000:127.0.0.1:5000 \
       -i $REG_KEY dockercr@$REGHOST \
       -c 'sleep 10' 2> /dev/null &
docker tag $1 $REG_ALIAS:5000/$1
docker push $REG_ALIAS:5000/$1
wait
```

Our new docker-pull command:

```shell
#!/usr/bin/env bash
if [ $# -lt 1 ] ; then
    echo "Please specify a container"
    exit 1
fi

# Config
REGHOST=88.88.88.88             # put ip / name of REGHOST here
REG_ALIAS=cr.boss.org           # our fake repository name
REG_KEY=$HOME/.ssh/id_rsa_boss  # private key used for SSH-auth

ssh -f -L 5000:127.0.0.1:5000 \
       -i $REG_KEY dockercr@$REGHOST \
       -c 'sleep 10' 2> /dev/null &
docker pull $REG_ALIAS:5000/$1
wait
```

Note that we only `sleep 10` no matter how long the docker operation will take.
SSH will take care of keeping the tunnel open as long as a forwarded connection
is active. Even if another push / pull is initiated while the first one is
running, SSH will wait.
