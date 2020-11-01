# Docker 101
[![N|Solid](https://cdn.iconscout.com/icon/free/png-256/docker-1-282655.png)](https://www.docker.com/)

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

Este repositorio se ha realizado para la documentacion y demostracion del uso de contenedores con dos aplicaciones diferentes.
-  Webserver con NodeJS (Authjs) by LintangWisesa
-  MongoDB ReplicaSet

Estas dos aplicaciones seran deployadas sobre Docker containers de la manera mas sencilla posible sobre Maquinas virtuales de Azure. 

MongoDB sera deployado utilizando un ReplicaSet (mas informacion aqui: https://docs.mongodb.com/manual/replication/)

NodeJs sera el Frontend que utilizaremos para demostrar fisicamente como funcionan los docker containers como servicio y con replicacion de servicios con un load balancer en Azure. 

### Installacion de MongoDB

**VM's:**
Deployar 5 maquinas virtuales en Azure
- 3 Maquinas seran para MongoDB (mongodb1, mongodb2, mongodb3)
- 2 Maquinas seran para Authjs (authjs1, authjs2)

**Redes:**
- Crear una red 10.0.x.x/28
- Crear un LoadBalancer SKU: Standard

Crear usuario:
```shell
# useradd deployer
# passwd deployer
```
Agregar Hosts:
```console
# echo " * 10.0.3.4 mongodb1
10.0.3.5  mongodb2
10.0.3.6  mongodb3">> /etc/hosts
```
Crear Paths:
```shell
# mkdir -p /mongodata/log/mongodb
# mkdir -p /mongodata/mongodb/db
```
Copiar los archivos de mongo de este repositorio al Path de MongoDB

```shell
git clone https://github.com/mariovelasco6/docker-101.git
cd docker-101
cp mongodb/* /mongodata/mongodb/
```

Cambiar de Owner al Mongo path:
```shell
# chown -R deployer:deployer /mongodata/
```
Instalacion de Docker:

```console
$ sudo apt-get install apt-transport-https ca-certificates  curl gnupg-agent software-properties-common 
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo apt-key fingerprint 0EBFCD88
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu  $(lsb_release -cs) stable"
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io -y
$ sudo systemctl enable docker
$ sudo usermod -aG docker deployer 
```
Iniciar el Docker Swarm:
```shell
docker swarm init --advertise-addr 10.0.3.4
```
Automáticamente generará una salida en la que mostrará el comando a ejecutar en los worker nodes, para que puedan unirse al docker Swarm.

> docker swarm join --token SWMTKN-1-pftln2us40xpasrlqr5 10.0.3.4:2377

Una vez que los worker nodes, se hayan unido, recibirá un mensaje como el siguiente:
> This node joined a swarm as a worker.

Crear una red de Docker:
```shell
$ docker network create --driver overlay --attachable mongoshard 
```
Iniciar el Docker Stack
```shell
$ docker stack deploy -c /mongodata/mongodb/mongodb.yml mongoswarm
```

Hacer login en la Base de Datos:

```sh
$ mongo
``` 
Ejecutar los siguientes comandos para crear el primer usuario y collection:
```sh
  $ use signin
  $ db.createUser({user: 'mongouser', pwd: 'anypass', roles: ['readWrite', 'dbAdmin']})
  $ db.createCollection('users')
```
Establecer nodos de prioridad en la ReplicaSet:
```shell
cfg = rs.conf();
cfg.members[0].priority = 3;
cfg.members[1].priority = 2;
cfg.members[2].priority = 1;
rs.reconfig(cfg);
 ```
 
 

### Installacion de Authjs

Autenticarse en el Container Registry:
```sh
$ docker login mongodbworkshop.azurecr.io
```

Ejecutar el contenedor en standalone:
```sh
$ docker run -d --name authjs -p 3210:3210  mongodbworkshop.azurecr.io/authjs:latest 
```

Crear un Docker service:
```shell 
docker service create --with-registry-auth --detach=false --name authjs --replicas=2 --restart-condition any --publish published=3210,target=3210,mode=host mongodbworkshop.azurecr.io/authjs:latest
```


**Free Software, Hell Yeah!**
