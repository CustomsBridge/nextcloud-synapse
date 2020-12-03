# nextcloud-synapse

At Customs bridge we spent some time linking together great pieces of software in order to build our daily working tools. It is called nextcloud-synaype as it mainly relies on Nextcloud and Synapse (which is a Matrix server implementation).

By reading through this README and acting when asked for, you should have a working solution in 30 minutes or less.
PRs are welcome if they simplify the project and make it less complicated to setup. For example a setup script.
If you are looking for more adavanced options regarding synapse setup, head over this [nice Ansible playbook](https://github.com/spantaleev/matrix-docker-ansible-deploy)

# Setup

First ensure you have git, docker, and docker-compose installed on your server.
We take the asumption you own a server/cloud server with at least 4GB memory. It may work with less though.
You must also possess a domain name (`domain.name`) in the bellow config.  You must be able to create subdomains and own the DNS in order to redirect via `A` and `AAAA` records the trafic to your server.

## clone repo
run `git clone https://github.com/CustomsBridge/nextcloud-synapse.git`

## Understanding `docker-compose.yml`

We have no less than 7 services in our docker-compose.

Lets quickly go through them and learn how they interact with each others.

The first two ones `nextcloud` and `nextcloud_db` are, as you can imagine nextcloud and it's related database. Nextcloud will provide us with the functionalities of file sync, cloud drive, collaborative document edition, light project management. It could even provide some remote working capabilities as chat and video calls.
However for the moment (2020) we estimate these chat and video calls feature would be better handled by matrix.

Here come the second bundle of our setup. `synapse_db`, `turn`, `synapse`, `synapse_admin`:

`synapse` is an implementation of the matrix protocol and is linked with it's database: `synapse_db`
`turn` is a VoIP media traffic NAT traversal server and gateway. In other words, it helps software talk to each other when behind NAT. It's needed to have working Voice calls in matrix.
`synapse_admin` is not really necessary but helps with initial setup of your synapse server. Can be safely removed afterwards.

Lastely, `caddy` is a rising webserver that handles https by himself, saving us from having to define a certbot service to handle our SSL certificates.
For us it also acts as reverse proxy for nextcloud that is in it's `fpm` variant. More performant, but requires a reverse proxy.

Now that the landscape is set, let's see interresting lines of this file.
<br/>

```
- /mnt/disk/nextcloud_db:/var/lib/mysql
```

This is the volume mount of the nextcloud DB. the left side is where you want to mount this persisted database outside your docker. For us, we attached a disk to our cloud instance, hence the `/mnt/disk`. but you can replace this to any place you want. Just make sure to replace `/mnt/disk` everywhere.

<br/>

```
      - MYSQL_ROOT_PASSWORD=XXXX
      - MYSQL_PASSWORD=YYYY
      - MYSQL_DATABASE=nextclouddb
      - MYSQL_USER=nextclouduser
```

You will need these informations on the first startup of your nextcloud instance to link it with the database.
<br/>
```
SYNAPSE_SERVER_NAME: "matrix.domain.name"
```

Here you must adjust domain.name to your domain name. You can also change the subdomain "matrix" to anything of your liking.
<br/>
```- "/mnt/disk/caddy_data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/turn.domain.name:/etc/certs/"```
Here you must also adjust domain.name to your domain name.

## Understanding `www2.conf`

This file enable us to adjust php-fpm settings in the nextcloud docker

```
[www]
pm = dynamic
pm.max_children = 120
pm.start_servers = 12
pm.min_spare_servers = 6
pm.max_spare_servers = 18
```

These values are defined as described [here](https://docs.nextcloud.com/server/20/admin_manual/installation/server_tuning.html#tune-php-fpm). You can adjust them to higher settings if you have a more powerful server. Other than that you should not touch this file

## Understanding `Caddyfile`

This file defines what our caddy webserver should listen and reply to.
In this file, make sure to replace each `domain.name` with your domain name.
The interesting bits in this file are the last two blocks:

```
matrix.domain.name:8080 {
        reverse_proxy synapse_admin:80
}
```
As you can see, the port 8080 will redirect the request to the administration docker `synapse_admin` in our `docker-compose.yml`

<br/>

The last one as explained in the comments is solely  to get a SSL cert for `turn.domain.name`

```
# the bellow reverse proxy shouldn't be necessary, but it enable caddy to know it must request certificates for this subdomain as well, as this is needed by coturn
turn.domain.name {
        reverse_proxy localhost
}
```
You can see in `docker-compose.yml` how we link these certificates to the turn docker: 
`- "/mnt/disk/caddy_data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/turn.domain.name:/etc/certs/"`



# First run and initialisation

WIP