# nextcloud-synapse

At Customs bridge we spent some time linking together great pieces of software in order to build our daily working tools. It is called nextcloud-synapse as it mainly relies on Nextcloud and Synapse (which is a Matrix server implementation).

By reading through this README and acting when asked for, you should have a working solution in 30 minutes or less.
PRs are welcome if they simplify the project and make it less complicated to setup. For example a setup script.

If you are looking for more adavanced options regarding synapse setup, head over this [nice Ansible playbook](https://github.com/spantaleev/matrix-docker-ansible-deploy)

# Setup

First ensure you have git, docker, and docker-compose installed on your server.
We take the asumption you own a server/cloud server with at least 4GB memory. It may work with less though.
You must also possess a domain name (`domain.name` in the bellow config).  You must be able to create subdomains and own the DNS in order to redirect via `A` and `AAAA` records the trafic to your server.

In our configuration, we have mounted a disk under ```/mnt/disk``` to store all the data. So you might want to do the same or change the path everywhere

## clone repo
run `git clone https://github.com/CustomsBridge/nextcloud-synapse.git`

## Understanding `docker-compose.yml`

We have no less than 7 services in deveral docker-compose files (starting with "dc_").

Lets quickly go through them and learn how they interact with each others.

The first two ones `nextcloud` and `nextcloud_db` are, as you can imagine nextcloud and it's related database. Nextcloud will provide us with the functionalities of file sync, cloud drive, collaborative document edition, light project management. It could even provide some remote working capabilities as chat and video calls.
However for the moment (2022) we estimate these chat and video calls feature would be better handled by matrix.
The onlyoffice container will enable online edition of documents from inside nextcloud

Here comes the second bundle of our setup. `synapse_db`, `turn`, `synapse`, `synapse_admin`:

- `synapse` is an implementation of the matrix protocol and is linked with it's database: `synapse_db`.\
- `turn` is a VoIP media traffic NAT traversal server and gateway. In other words, it helps software talk to each other when behind NAT. It's needed to have working Voice calls in matrix.\
- `synapse_admin` is not really necessary but helps with initial setup of your synapse server. Can be safely removed afterwards.

Lastely, `caddy` is a rising webserver that handles https by himself, saving us from having to define a certbot service to handle our SSL certificates.
For us it also acts as reverse proxy for nextcloud that is in it's `fpm` variant(More performant, but requires a reverse proxy).

Now that the landscape is set, let's see interresting lines of these files.
<br/>

```
- /mnt/disk/nextcloud_db:/var/lib/mysql
```

This is the volume mount of the nextcloud DB. the left side is where you want to mount this persisted database outside your docker.

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
As you can see, the port 8080 will redirect the request to the administration container `synapse_admin`.

<br/>

The last one as explained in the comments is solely to get a SSL cert for `turn.domain.name`

```
# the bellow reverse proxy shouldn't be necessary, but it enable caddy to know it must request certificates for this subdomain as well, as this is needed by coturn
turn.domain.name {
        reverse_proxy localhost
}
```
You can see in `dc-synapse.yml` how we link these certificates to the turn docker: 
`- "/mnt/disk/caddy_data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/turn.domain.name:/etc/certs/"`


<br/><br/><br/>
# First run and initialisation

First, you need to init the synapse configuration
```
docker run -it --rm \
    -v /mnt/disk/synapse/synapse_data:/data \
    -e SYNAPSE_SERVER_NAME=matrix.domain.name \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest migrate_config
```

Then you can edit the following file ```/mnt/disk/synapse/synapse_data/homeserver.yaml```:
Please comment (add a "#")the following lines:
```
tls_certificate_path
tls_private_key_path
```
You can also comment all these lines (make sure the port is 8448, we can to keep the "8008 bloc"):
```
port: 8448
bind_addresses: ['::']
type: http
tls: true
x_forwarded: false
resources:
  - names: [client]
    compress: true
  - names: [federation]  # Federation APIs
    compress: false
```
Indeed we do not need to enable TLS because the caddy webserver is already acting as a reverse proxy.<br/>
HTTP communication will be done between synapse and caddy, but since it is locally, it should not be much of a problem.<br/>
This could be achievable though by mounting a new volume in the synapse docker to aim the caddy certificates

Next, we are going to remove federation because we want our server to be private:
in the "8008 bloc", remove those two lines:
```
- names: [federation]
  compress: false
```
and replace ```- names: [client]``` by ```- names: [client,openid]```. The openid part allows the integration manager to connect. [reference](https://github.com/vector-im/element-web/issues/3329#issuecomment-485822392)

Just bellow, jump a few lines and add:
```
## Federation. If no whitelist is defined, every server can connect ##
federation_domain_whitelist:
  - matrix.domain.name

```

Next, we are going to configure Synapse to use the postgresql database instead of sqlite:<br/>
change the ```database``` record as follow:
```
database:
  name: psycopg2
  args:
    user: synapse
    password: ZZZ
    database: synapse
    host: synapse_db
    cp_min: 5
    cp_max: 10
```

Next, lets configure TURN (The PPPPP key is the same you defined in the conturn.conf file):
```
turn_uris: ["turn:turn.domain.name?transport=udp", "turn:turn.domain.name?transport=tcp"]
turn_shared_secret: "PPPPPP"
turn_user_lifetime: "1h"
turn_allow_guests: False
```

Finally run ```docker exec -it synapse register_new_matrix_user http://localhost:8008 -c data/homeserver.yaml``` to create an admin user, and then edit the config file to mirror those lines:
```
enable_registration: False
# registration_shared_secret: "xxxxxxxxxxx"
```

You can find an example of the correct ```homeserver.yaml``` file to have in the example folder.


WIP

run ```docker-compose -f dc-nextcloud.yml -f dc-synapse.yml -f dc-caddy.yml up -d``` to launch everything in detached mode
You can remove the ```-d``` option if you want to see logs and can also totally merge the three ```.yml``` files if you want.
Here they have been separated so that in case of maintenance, you do not need to bring down all the infra.
This way you can decide to bring down only the nextcloud part for example with ```docker-compose -f dc-nextcloud.yml down```, do your maintenance, and relaunch this part with ```docker-compose -f dc.yml up -d```. This way the synapse and Caddy part wont be affected


# Updates

change the images versions in the ```docker-compose``` files and ```Dockerfile-nextcloud```
for each images, verify the updates requirements.
and run ```docker-compose -f dc-nextcloud.yml -f dc-synapse.yml -f dc-caddy.yml up --force-recreate --build -d``` to start with new images
followed by ```docker image prune -f``` to clean unused images
