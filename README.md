# InfluxDBv3 Speedtest-Tracker v2 (SQL)

## Set up InfluxDB3.

Create a docker container with the following information:
(Suggest you use Alpine - if you use Proxmox just create a new Alpine container and add docker)

```yaml
name: influxdb3

services:

  influxdb3:
    image: influxdb:3-enterprise
    container_name: influxdb3
    network_mode: host
    #stdin_open: true
    #tty: true
    environment:
      INFLUXDB3_ENTERPRISE_LICENSE_EMAIL: <your email address goes here>
      INFLUXDB3_NODE_IDENTIFIER_PREFIX: influxdb3
      INFLUXDB3_ENTERPRISE_CLUSTER_ID: cluster01
      INFLUXDB3_OBJECT_STORE: file
      INFLUXDB3_DB_DIR: /var/lib/influxdb3
      PLUGIN_DIR: /plugins
    volumes:
      # the location /opt/influxdb3/data will need A LOT!! of data
      - /opt/influxdb3/data:/var/lib/influxdb3
      - /opt/influxdb3/config:/etc/influxdb3
      - /opt/influxdb3/plugins:/plugins
    restart: unless-stopped
    command: >
      influxdb3
      serve
      --license-email <your email address goes here>
      --node-id influxdb3
      --cluster-id cluster01
      --object-store file
      --data-dir /var/lib/influxdb3
      --plugin-dir /plugins
      --disable-authz health,ping,metrics
      --num-cores 2

  explorer:
    image: influxdata/influxdb3-ui:latest
    container_name: influxdb3-ui
    hostname: influxdb3-explorer
    network_mode: host
    #ports:
    #  - 8888:80
    command: '--mode=admin'
    depends_on:
      - influxdb3
    restart: unless-stopped
    environment:
      SESSION_SECRET_KEY: <generate some secret key for access, can be random>
      DATABASE_URL: /db/sqlite.db
    volumes:
      - /opt/influxdb3/ui/db:/db
      - /opt/influxdb3/ui/config:/app-root/config
```
Adjust volumes / paths to your linking. This is not a docker tutorial, if you need help with this, please look elsewhere. Sorry.

That's it, start it up.
`docker compose up -d`

### Create an admin token

Enter the following command in the shell you're in.
```
docker exec -it influxdb3 influxdb3 create token --admin
```

Write that token down in a safe place, you'll need it a lot. A password manager might be a good option.

### Make life easier in the shall with influx CLI

In order to execute commands easily via the CLI, we define an alias in bash / .bashrc:

`nano ~/.bashrc`

```
alias influxdb3='docker exec -e INFLUXDB3_AUTH_TOKEN=<<OUR ADMIN TOKEN>> -it influxdb3 influxdb3 $@'
```

then we need to make it available in the current shell session:

`source ~/.bashrc`

Now we can just use the influxdb 3 CLI just like that, it will call the container runtime.


### Create speedtest database

Create a new database with

`influxdb3 create database --retention-period 365d speedtest`

Change the retention to what you like.

## Set up Speedtest

Refer to [Speedtest-Tracker](https://docs.speedtest-tracker.dev/getting-started/installation/using-docker-compose) for more information.
I run speedtest in another docker container. Set it up with the following compose file:

```yaml
name: speedtest

services:
    speedtest:
        container_name: speedtest
        # I se host network mode but you can also comment out that line and use ports instaed
        network_mode: host
        #ports:
        #    - 80:80
        #    - 443:443
        environment:
            # - APP_DEBUG=true
            - PUID=1000
            - PGID=1000
            - APP_KEY=base64:bdCZIgrKdT2eI+T1X5F4WvUqmnnDzfBEXjZq1EbaG2s=
            - DB_CONNECTION=sqlite
              # We run this every 2 hrs
            - SPEEDTEST_SCHEDULE="0 */2 * * *"
            - SPEEDTEST_SERVERS=38177
            - PRUNE_RESULTS_OLDER_THAN=365
            - CHART_DATETIME_FORMAT= 
            - DATETIME_FORMAT=
              # Adjust your timezone
            - APP_TIMEZONE="Pacific/Auckland"
            - DISPLAY_TIMEZONE="Pacific/Auckland"
            - PUBLIC_DASHBOARD=true
              # Delete the next 2 lines or change to your IP
            - APP_URL="http://192.168.100.16"
            - ASSET_URL="http://192.168.100.16"
            # Mail setup
            - MAIL_MAILER=smtp
            - MAIL_HOST=<your mailserver address>
            - MAIL_PORT=465 # choose your port
            - MAIL_USERNAME=<your mail username here>
            - MAIL_PASSWORD=<obviously your mail password here>
            - MAIL_FROM_ADDRESS=<pick your sender address>
            - MAIL_FROM_NAME="Speedtest Tracker"
            - MAIL_SCHEME=smtp # smtp/smtps
            - MAIL_ENCRYPTION=tls # tls/ssl
        volumes:
            - /opt/speedtest/config:/config
            - /opt/speedtest/keys:/config/keys
        image: lscr.io/linuxserver/speedtest-tracker:latest
        restart: unless-stopped
```

### Start the Container

You can now start the container accordingly the platform you are on.

`docker compose up -d`

## Grafana Dashboard

A dashboard to display data exported by Speedtest Tracker v2 using Influxdb v3 SQL as data source

This dashboard shows data collected by Speedtest Tracker v2 https://github.com/alexjustesen/speedtest-tracker and exported in an InfluxDB v3 database.

Dashboard based on the excellent work by [@masterwishx](https://github.com/masterwishx/Speedtest-Tracker-v2-InfluxDBv2).

Screenshot
![Screenshot](Screenshot.png)
