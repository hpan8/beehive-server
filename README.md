# Beehive Server

Waggle cloud software for aggregation, storage and analysis of sensor data from Waggle nodes

## Installation

The recommended installation method for the waggle beehive server is Docker. But it should be easily possible to install everything in a non-virtualized ubuntu environment. In that case we recommend ubuntu trusty (14.04). If you are using Docker, you can use any operating system with a recent Linux kernel that runs Docker. 

### Docker

To get the latest version of Docker in Ubuntu use this (simply copy-paste the whole block):
```
apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
export CODENAME=$(lsb_release --codename | grep -o "[a-z]*$" | tr -d '\n')
echo "deb https://apt.dockerproject.org/repo ubuntu-${CODENAME} main" >> /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install -y  docker-engine
```

### Data directory
While services are running in containers, configuration files, SSL certificates and databases have to be stored persistently on the host. Depending on your system you might want to use a different location to store these files.

```bash
export DATA="/mnt"

# other examples:
# export DATA="${HOME}/waggle"
# export DATA="/media/ephemeral/"
```


### Cassandra:
```bash
docker run -d --name beehive-cassandra -v ${DATA}/cassandra/data/:/var/lib/cassandra/data cassandra:2.2.3
```
You may want to change the -v option (format: "host:container") to point to a host file system location with sufficient space for the Cassandra database files. For simple testing without much data you can omit option "-v" above. Without "-v" Cassandra data is not stored persistently and data is lost when the container is removed. 

Installation instructions for Cassandra without Docker:

http://docs.datastax.com/en/cassandra/2.0/cassandra/install/installDeb_t.html

### RabbitMQ

Download rabbitmq.config
```bash
mkdir -p ${DATA}/rabbitmq/config/
curl https://raw.githubusercontent.com/waggle-sensor/beehive-server/master/SSL/rabbitmq.config > ${DATA}/rabbitmq/config/rabbitmq.config
```

Create server certificates
```bash
docker run -ti --name certs --rm -v ${DATA}/waggle/SSL/:/usr/lib/waggle/SSL/ waggle/beehive-server:latest ./scripts/configure_ssl.sh
```

Start RabbitMQ server
```bash
docker run -d --hostname beehive-rabbit --name beehive-rabbit -v ${DATA}/rabbitmq/config/:/etc/rabbitmq -v ${DATA}/rabbitmq/data/:/var/lib/rabbitmq/mnesia/ rabbitmq:3.5.6
```


Create waggle user:
```bash
# Enter running container
docker exec -ti  beehive-rabbit /bin/bash

# Inside of the container:
if [ $(rabbitmqctl list_users | grep -c waggle) -eq 0 ] ; then 
  echo "add waggle user"  
  rabbitmqctl add_user waggle waggle
  rabbitmqctl set_permissions waggle ".*" ".*" ".*"
fi
```


### Beehive Server
This requires that the cassandra container is already running on the same machine. If cassandra is running remotely, do not use option "--link ...".

```bash
docker run -ti --name beehive-server \
  --link beehive-cassandra:cassandra \
  -v ${DATA}/waggle/SSL:/usr/lib/waggle/SSL/ \
  -p 5671:5671 \
  waggle/beehive-server:latest
```

For developing purposes you can also mount the git repo into the container.
```bash
-v ${HOME}/git/beehive-server:/usr/lib/waggle/beehive-server
```

You should now be inside the container. Run the configure and Server.py scripts:
```bash
cd /beehive-server/
./configure
cd /usr/lib/waggle/beehive-server/
python ./Server.py
```
The beehive server should be running at this point. 

Leave the container and put it in background using key combinations "Ctrl-P" "Ctrl-Q". You can re-attach to the container with
```bash
docker attach beehive-server
```
or enter the container without attaching to the main process (python Server.py) with "docker exec":
```bash
docker exec -ti beehive-server bash
```


## Developer Notes

TODO: Run RabbitMQ in its own container.

### Notes on Cassandra

To directly connect to cassandra:
```bash
docker run -it --link beehive-cassandra:cassandra --rm cassandra:2.2.3 cqlsh cassandra
```
To view database, e.g.:
```bash
use waggle;
SELECT * FROM node_info;
DESCRIBE TABLES;
```
