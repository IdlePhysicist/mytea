# MyTea

This repo is heavily based on High Tea ([repo here](https://git.habd.as/comfusion/high-tea.git)), compiled by Josh Habdas ([homepage](https://habd.as/))

This installation user Postgres, and Traefik to compliment Gitea.

## Setting it up
By far the trickiest part is allowing SSH passthrough to the container, if you can live without SSH Git access then this will be a breeze. But it was something I could not give up, for this “guide” I will assume you want SSH Git access.

You’ll need docker-ce and docker-compose installed on the server.

Before you run `docker-compose up -d` you’ll need to have a `git` user on your host machine (the Lightsail instance), you can create it as follows.

```
adduser \
   --system \
   --shell /bin/bash \
   --gecos 'Git Version Control' \
   --group \
   --disabled-password \
   --home /home/git \
   git
```

Then you need to create the following directory structure.

```
mkdir -p /var/lib/gitea/{custom,data,log}
chown -R git:git /var/lib/gitea/
chmod -R 750 /var/lib/gitea/
mkdir /etc/gitea
chown root:git /etc/gitea
chmod 770 /etc/gitea
```

Clone the [mytea repo](https://github.com/IdlePhysicist/mytea), create your own `env.sh` from the example provided. Then create a docker network for the backend database, and one for the "frontend" network.

```
docker network create --internal back
docker network create front
```

Now we can source the env.sh file, and fire up the docker-compose file. Assuming that your DNS records have populated you can navigate to your FQDN and complete the web based installation of Gitea.

### The following strictly pertains to the SSH access

Following this create the executable file `/app/gitea/gitea` and give it the following contents

```
#!/bin/sh
ssh -p 2222 -o StrictHostKeyChecking=no git@127.0.0.1 "SSH_ORIGINAL_COMMAND=\"$SSH_ORIGINAL_COMMAND\" $0 $@"
```

Now create an SSH key for the `git` user

```
sudo -u git ssh-keygen -t rsa -b 4096 -C "Gitea Host Key"
```

Link the authorised keys file from the container to the same for the `git` user on the host

```
ln -s /var/lib/gitea/git/.ssh/authorized_keys /home/git/.ssh/authorized_keys
```

Now we need to put the "Gitea Host Key" into the authorised keys file for the git user, this allows the SSH access to the container.

```
sudo -u git bash -c 'echo "no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty $(cat /home/git/.ssh/id_rsa.pub)" >> /var/lib/gitea/git/.ssh/authorized_keys'
```

Now do a `docker-compose restart` for good measure, and after you upload an SSH key to your user in Gitea, test with `ssh -T git@code.speleo.dev`.

