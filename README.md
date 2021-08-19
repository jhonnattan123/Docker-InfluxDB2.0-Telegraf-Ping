# Install Docker + InfluxDB2.0 + Telegraf + Ping

We’ll create a new network using the following command:

```console
docker network create --driver bridge influxdb-telegraf-net
```

We’ll create a new folder for persistent storage
```console
cd
sudo mkdir testdata/
sudo mkdir testdata/influx                  # for influx
sudo mkdir testdata/telegraf                # for telegraf
sudo mkdir testdata/telegraf-config-clon    # for telegraf config clon
```
We’ll create a Docker with influxdb

```console
docker run -d --name=influxdb -p 8086:8086 -v  /testdata/influx:/root/.influxdbv2 --net=influxdb-telegraf-net quay.io/influxdb/influxdb:v2.0.3
```

init configuration (can use same bucket and organization name ej: backbone).

```console
docker exec -it influxdb influx setup
```

now, we need the bucket id:
```console
docker exec -it influxdb influx bucket list

ID                      Name            Retention       Organization ID
3d40b727a0b3bed6        _monitoring     168h0m0s        ee7a4d9eaebc2754
6128ba4b99ad1b47        _tasks          72h0m0s         ee7a4d9eaebc2754
4d6a72e1d14b6d88        MY_PROYECT        0s              ee7a4d9eaebc2754
```

in it example the id is **4d6a72e1d14b6d88**

We’ll create a Docker with telegraf

```console
docker run -d --name=telegraf -v /testdata/telegraf:/var/lib/influxdb --net=influxdb-telegraf-net telegraf
```

now we will get the configuration file from docker and store it for influxdb telegraf creator
```console
docker cp telegraf:\etc\telegraf\telegraf.conf testdata\telegraf-config-clon\telegraf.conf
```
now copy into influxdb docker
```console
docker cp testdata\telegraf-config-clon\telegraf.conf influxdb:/telegraf.conf
```

create telegraf id in influxdb docker:
```console
docker exec -it influxdb influx telegrafs create -f /telegraf.conf
```

to see telegrafs id:
```console
docker exec -it influxdb influx telegrafs
ID                      OrgID                   Name                    Description
0802bdb04b324000        ee7a4d9eaebc2754
```

copy id, in this example is:  **0802bdb04b324000**

now, using the id of the bucket and the id of the telegraf, we will create a tocken

```console

docker exec -it influxdb influx auth create -o backbone --read-buckets 4d6a72e1d14b6d88 --read-dashboards --read-tasks --read-telegrafs 0802bdb04b324000 --read-user

ID                      Description     Token                                                                       User Name        User ID                 Permissions
0802c0a7baf99000                        _0dtBNqTVudEV0E1b5tzN8TZuvilsOST1ZNZA6_8W7mbEgmB7yPFFjxg1kNpg4hiOin5N3dbns1xXUNekiOGDA==     user            0802b3ea86799000        [write:orgs/ee7a4d9eaebc2754/buckets read:orgs/ee7a4d9eaebc2754/dashboards read:orgs/ee7a4d9eaebc2754/tasks read:orgs/ee7a4d9eaebc2754/telegrafs read:orgs/ee7a4d9eaebc2754/user
```

in this example the token is **_0dtBNqTVudEV0E1b5tzN8TZuvilsOST1ZNZA6_8W7mbEgmB7yPFFjxg1kNpg4hiOin5N3dbns1xXUNekiOGDA==**


now we will edit the configuration file that we copy in our folder

```console
sudo vim testdata/telegraf-config-clon/telegraf.conf
```

now we will add all the urls to which we want to ping replace HERE-ADD-URL-TARGET-TO-PING with urls.
add :
```console
[[inputs.ping]]
  ## Hosts to send ping packets to.
  urls = ["HERE-ADD-URL-TARGET-TO-PING"]
```

and add (or replace) in OUTPUT PLUGINS section with this data
```console
[[outputs.influxdb_v2]]
 urls = ["http://influxdb:8086"]

 token = "_0dtBNqTVudEV0E1b5tzN8TZuvilsOST1ZNZA6_8W7mbEgmB7yPFFjxg1kNpg4hiOin5N3dbns1xXUNekiOGDA=="
 
 ## Organization is the name of the organization you wish to write to; must exist.
 organization = "MY_PROYECT"
 
 ## Destination bucket to write into.
 bucket = "MY_PROYECT"
```

now we will copy this configuration to the telegraf docker

```console
docker cp testdata\telegraf-config-clon\telegraf.conf telegraf:\etc\telegraf\telegraf.conf 
```
now we restart telegraf
```console
docker exec -ti telegraf telegraf restart bash
```
to exit without stop agent: ctrl + q

now entering influxdb in localhost: 8086 with the username and password that we created at the beginning, we could go to the data section, filtering by the bucket that we created and the ping option, we should be able to see the data arriving

## References (APA):
- influx cli: no puede crear auth: Error: Id debe tener una longitud de 16 bytes. (2021, August 18). github.com
https://github.com/influxdata/influx-cli/issues/69

- Running InfluxDB 2.0 and Telegraf Using Docker (2021, August 18). influxdata.com
https://www.influxdata.com/blog/running-influxdb-2-0-and-telegraf-using-docker/