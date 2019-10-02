# interlock-routing
L7 Routing with Docker EE and Interlock for  Swarm Based Applications

## Label Interlock proxy nodes 

```
docker node update --label-add role=interlock-infra  interlock-node-0
```

Apply the same role label for kubectl management 
```
kubectl label node  interlock-node-0 role=interlock-infra
```


## Interlock Service configuration

### Proxy Constraints

```
    ProxyConstraints = ["node.labels.role==interlock-infra","node.labels.com.docker.ucp.orchestrator.swarm==true"]
```

### Listen Ports
### Mode ingress/host

## Setup 

### Create the interlock configuration 
```
docker config create interlock.conf service.interlock.conf
$ docker config  inspect  interlock.conf --pretty
$ docker network create -d overlay interlock
```

### Create the interlock service 
```
$ docker service create \
     --name interlock \
     --mount src=/var/run/docker.sock,dst=/var/run/docker.sock,type=bind \
     --network interlock \
     --constraint node.role==manager \
     --config src=interlock.conf,target=/config.toml \
     docker/ucp-interlock:3.2.1 -D run -c /config.toml
t85x3cgjmed8qto4t46ncc052
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

##Check service list
You should have at least these three services 
interlock, interloc-proxy and interlock-extension
![Service List ](./ServiceList.png)
```
[enono@ucp-node-0 ~]$ docker service ps brave_roentgen
ID                  NAME                   IMAGE                              NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
t6j11xoyxvxc        brave_roentgen.1       docker/ucp-interlock-proxy:3.2.1   interlock-node-0    Running             Running 5 minutes ago                        *:443->443/tcp,*:80->80/tcp
l9wxkxhpchxj         \_ brave_roentgen.1   docker/ucp-interlock-proxy:3.2.1   interlock-node-0    Shutdown            Shutdown 5 minutes ago
```

## Create a wildcard cert
to serve all the applications *.swarmapps.dev01.dockernetes.org, we created a wildcard cert like this

```
openssl req -new -newkey rsa:4096   -days 3650   -nodes   -x509  -subj "/O=Dockernetes/CN=hello.swarmapps.dev01.dockernetes.org" -keyout wildcard.key -out wildcard.cert
```
Then create secrets from cert and key file 
```
$ docker secret create wildcardcert ./wildcard.cert
$ docker secret create wildcardkey ./wildcard.key
```



## Deploy the hello stack 

```
docker stack deploy -c hello-svc-compose.yaml hello
Creating network hello_app_ovl
Creating service hello_app
```
The stack create an application with two running containers

```
 docker service  ps hello_app
ID                  NAME                IMAGE                                 NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
kzg4plbz84ux        hello_app.1         gcr.io/google-samples/hello-app:1.0   interlock-node-0    Running             Running about a minute ago
0z02pksdjejw        hello_app.2         gcr.io/google-samples/hello-app:1.0   worker-node-1       Running             Running about a minute ago
```


## Test the application 
```
curl -k  https://hello.swarmapps.dev01.dockernetes.org  --resolve hello.swarmapps.dev01.dockernetes.org:35.246.203.59
Hello, world!
Version: 1.0.0
Hostname: 2f2f03ddddce
```

35.246.203.59 refers to Public IP address of one Interlock Node

When using a loadbalancer, create an A record to resolve *.swarmapps.dev01.dockernetes.org -> 35.246.203.59

```
curl -k  https://hello.swarmapps.dev01.dockernetes.org
Hello, world!
Version: 1.0.0
Hostname: cb941a7e5c7d
```


