# iPrior_microservices

## Homework 14

* Обновил версию Windows до Pro
* Установил Docker CE
* Сохранил вывод команды `docker images` в файл `docker-1.log`
* Попытался сравнить выводы команды `inspect` для *image* & *container* - не понял что сравнивать надо =\ 
  но отличаются они, как мне кажется, из-за разных сущностей ( **container** & **image** )

## Homework 15

* Настроил новый проект gcloud
* Создал `docker-machine` в GCP
* Настроил Firewall
* С помощью ребят из слака ~и чей то матери~ запустил контейнер

## Homework 16

* Запустил три контейнера для разных компонентов приложения
* Создал `docker volume` для MongoDB


## Homework 17

> Запустим контейнер в сетевом пространстве docker-хоста
> 
> `docker run --network host --rm -d --name net_test joffotron/docker-net-tools -c "sleep 100"`
> 
> Сравните выводы команд:
>
> `docker exec -ti net_test ifconfig`
>
> `docker-machine ssh docker-host ifconfig`

Список сетевых интерфейсов идентичный.
То есть у контейнера и у *docker-machine* одни и теже сетевые интерфейсы

> Запустите несколько раз (2-4)
>
> `docker run --network host -d nginx`
>
> Каков результат? Что выдал docker ps? Как думаете почему? 

Команда `docker ps` отобразила только один контейнер с Nginx:

```text
CONTAINER ID        IMAGE                        COMMAND                  CREATED              STATUS              PORTS                    NAMES
be123681eed4        nginx                        "nginx -g 'daemon of…"   25 seconds ago       Up 20 seconds                                admiring_mcnulty
```

Подозреваю что из-за того, что собсно контейнеры не отличаются сами по себе, поэтому используется один контейнер.

#### Bridge network driver

```bash
otus_ivan_priorov@docker-host:~$ sudo docker network ls
NETWORK ID          NAME                         DRIVER              SCOPE
d06a7a0fb065        back_net                     bridge              local
ec2c2271810d        bridge                       bridge              local
3c39ad1eccf9        dc_default                   bridge              local
e793ab036f98        front_net                    bridge              local
9afb4fb42234        host                         host                local
50f55b2e4a9c        none                         null                local
5cbce81901f8        reddit                       bridge              local
b052777debaf        redditmicroservices_reddit   bridge              local

otus_ivan_priorov@docker-host:~$ ifconfig |grep br
br-3c39ad1eccf9 Link encap:Ethernet  HWaddr 02:42:2e:54:01:56  
br-5cbce81901f8 Link encap:Ethernet  HWaddr 02:42:1b:f9:a6:65  
br-b052777debaf Link encap:Ethernet  HWaddr 02:42:2b:da:b6:2e  
br-d06a7a0fb065 Link encap:Ethernet  HWaddr 02:42:d1:8e:eb:98  
br-e793ab036f98 Link encap:Ethernet  HWaddr 02:42:21:7c:68:c7  

otus_ivan_priorov@docker-host:~$ brctl show br-d06a7a0fb065
bridge name     bridge id               STP enabled     interfaces
br-d06a7a0fb065         8000.0242d18eeb98       no              veth05d2688
                                                        veth89f47e4
                                                        vethcdfefeb
                                                        
otus_ivan_priorov@docker-host:~$ brctl show br-e793ab036f98
bridge name     bridge id               STP enabled     interfaces
br-e793ab036f98         8000.0242217c68c7       no              veth97f8e5d
                                                        vethb3b1618
                                                        vethfcf9ec2
                                                        
otus_ivan_priorov@docker-host:~$ sudo iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  10.0.2.0/24          0.0.0.0/0           
MASQUERADE  all  --  10.0.1.0/24          0.0.0.0/0           
MASQUERADE  all  --  172.20.0.0/16        0.0.0.0/0           
MASQUERADE  all  --  172.19.0.0/16        0.0.0.0/0           
MASQUERADE  all  --  172.18.0.0/16        0.0.0.0/0           
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  10.0.2.4             10.0.2.4             tcp dpt:9292

Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9292 to:10.0.2.4:9292

otus_ivan_priorov@docker-host:~$ ps ax |grep docker-proxy
 8123 ?        Sl     0:00 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 9292 -container-ip 10.0.2.4 -container-port 9292
 9245 pts/0    S+     0:00 grep --color=auto docker-proxy
```

### docker-compose

> Изменить docker-compose под кейс с множеством сетей, сетевых алиасов

Добавил две подсети. Настройки вынес в `.env` файл.

Наткнулся на "ошибку": 
в файле `./comment/Dockerfile` указано `ENV COMMENT_DATABASE_HOST comment_db` но "хоста" **comment_db** не существует - изменил на **post_db**

> Параметризуйте с помощью переменных окружений:
>
> • порт публикации сервиса ui
>
> • версии сервисов
>
> • возможно что-либо еще на ваше усмотрение
>

Помимо обозначенных в задании параметров, параметизировал подсети.

> Узнайте как образуется базовое имя проекта. Можно ли его задать? Если можно то как?

Имя контейнера можно задать параметром в `docker-compose.yml` - [container_name](https://docs.docker.com/compose/compose-file/#container_name)


## Homework 19

Развернул инстанс в GCP через веб интерфейс.

На инстансе развернул GitLab-CI

Добавил *pipeline*


## Homework 20

* Расширил существующий pipeline в Gitlab-CI
* Определил окружение *staging* и *production*
  
  * Установил условия и ограничение - тег в гите
  
* Настроил динамическое окружение

_задания со звездочками не делал_ 

## Homework 21

[Ссылка на докер хаб](https://hub.docker.com/u/iprior/)


## Homework 23

[Ссылка на докер хаб](https://hub.docker.com/u/iprior/)


## Homework 25

Модифицировал файл `src/post-py/Dockerfile` - добавил строку `RUN apk add --update gcc musl-dev`

Создал файл `docker/docker-compose-logging.yml` в котором описал развертывание **Elastic Stack** (EFK)


## Homework 27

> Добавить в кластер еще 1 worker машину

```bash
docker-machine create --driver google \
   --google-project  docker-XXXXXXX  \
   --google-zone europe-west1-b \
   --google-machine-type g1-small \
   --google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
   worker-3

# Running pre-create checks...
# (worker-3) Check that the project exists
# (worker-3) Check if the instance already exists
# Creating machine...
# ...
# Docker is up and running!
# To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env worker-3

docker-machine ssh worker-3
# Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.13.0-1015-gcp x86_64)

sudo docker swarm join --token XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX  000.000.000.000:2377
# This node joined a swarm as a worker
```

> Проследить какие контейнеры запустятся на ней

```bash
docker stack ps DEV
# ID                  NAME                                          IMAGE                        NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
# 0s9nmfdppm6j        DEV_node-exporter.yddrg3mgqhwyp0pqh8gbzj1l0   prom/node-exporter:v0.15.2   worker-3            Running             Running 2 minutes ago
# ...
```

> Проследить какие контейнеры запустятся на новой машине.
> Сравнить с пунктом 2.

**replicas: 3**

```bash
docker stack ps DEV
# ID                  NAME                                          IMAGE                        NODE                DESIRED STATE       CURRENT STATE              ERROR               PORTS
# 0s9nmfdppm6j        DEV_node-exporter.yddrg3mgqhwyp0pqh8gbzj1l0   prom/node-exporter:v0.15.2   worker-3            Running             Running 6 minutes ago                         
# su9siz1vzsjx        DEV_ui.3                                      iprior/ui:latest             worker-3            Running             Assigned 7 seconds ago                         
# 44rku1njqd01        DEV_post.3                                    iprior/post:latest           worker-3            Running             Preparing 9 seconds ago                        
# zwtzq4tpo7c4        DEV_comment.3                                 iprior/comment:latest        worker-3            Running             Preparing 12 seconds ago 
```

**replicas: 5**

```bash
docker stack ps DEV
# ID                  NAME                                          IMAGE                        NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
# yr2ikfedm1fb        DEV_node-exporter.v2317it64mxhdi81zg7tmzvg6   prom/node-exporter:v0.15.2   master-1            Running             Running 19 minutes ago                       
# zh3ds9c82zdk        DEV_prometheus.1                              iprior/prometheus:latest     master-1            Running             Running 19 minutes ago                       
# mcs2ib6en20h        DEV_cadvisor.1                                google/cadvisor:v0.29.0      master-1            Running             Running 19 minutes ago                       
# oty16fmlhmf2        DEV_mongo.1                                   mongo:3.2                    master-1            Running             Running 19 minutes ago                       
# n6wnmp0p5fyz        DEV_alertmanager.1                            iprior/alertmanager:latest   master-1            Running             Running 19 minutes ago
#                        
# uss4r3li4fbp        DEV_node-exporter.lvyuj24iwuvoyohjmvmait4sa   prom/node-exporter:v0.15.2   worker-1            Running             Running 18 minutes ago                       
# dqv3gjyuucdk        DEV_ui.2                                      iprior/ui:latest             worker-1            Running             Running 19 minutes ago                       
# iskq9hiuq79l        DEV_ui.4                                      iprior/ui:latest             worker-1            Running             Running 7 seconds ago                        
# cldnimi0h79t        DEV_post.2                                    iprior/post:latest           worker-1            Running             Running 19 minutes ago                       
# i7v3qp0qye0n        DEV_comment.2                                 iprior/comment:latest        worker-1            Running             Running 19 minutes ago                       
# 3mbfa29xvwqr        DEV_grafana.1                                 grafana/grafana:5.0.0        worker-1            Running             Running 18 minutes ago                       
# 
# eicaw6cmhvam        DEV_node-exporter.p4fwiti2r1c93tupz61trly2s   prom/node-exporter:v0.15.2   worker-2            Running             Running 19 minutes ago                       
# 1qq05r396cyu        DEV_ui.1                                      iprior/ui:latest             worker-2            Running             Running 19 minutes ago                       
# wz8b3l8v61mz        DEV_ui.5                                      iprior/ui:latest             worker-2            Running             Running 7 seconds ago                        
# 7b24kky9qg4j        DEV_post.1                                    iprior/post:latest           worker-2            Running             Running 19 minutes ago                       
# ptf9oaynja8e        DEV_post.4                                    iprior/post:latest           worker-2            Running             Running 9 seconds ago                        
# 8quebb52i11e        DEV_comment.1                                 iprior/comment:latest        worker-2            Running             Running 19 minutes ago                       
# szubsj7n8px4        DEV_comment.5                                 iprior/comment:latest        worker-2            Running             Running 5 seconds ago                        
# 
# 0s9nmfdppm6j        DEV_node-exporter.yddrg3mgqhwyp0pqh8gbzj1l0   prom/node-exporter:v0.15.2   worker-3            Running             Running 13 minutes ago                       
# su9siz1vzsjx        DEV_ui.3                                      iprior/ui:latest             worker-3            Running             Running 5 minutes ago                        
# 44rku1njqd01        DEV_post.3                                    iprior/post:latest           worker-3            Running             Running 5 minutes ago                        
# 2agvtdseo997        DEV_post.5                                    iprior/post:latest           worker-3            Running             Running 9 seconds ago
# zwtzq4tpo7c4        DEV_comment.3                                 iprior/comment:latest        worker-3            Running             Running 5 minutes ago                        
# sy15wszjxe07        DEV_comment.4                                 iprior/comment:latest        worker-3            Running             Running 5 seconds ago 
```

Сервисы равномерно распределяются по свободным нодам.

Интересно размещаются сервисы у которых не определён `placement constraints`, например *grafana*, размещена на `worker-1`, когда остальные сервисы, без указанного `placement constraints`, разместились на `master-1`.

*node-exporter* из-за `global mode` размещается на всех нодах, в том числе и вновь добавленных.

#### Задание

> Помимо сервисов приложения, у вас может быть инфраструктура, описанная в compose-файле (prometheus, nodeexporter, grafana …)
> Нужно выделить ее в отдельный compose-файл. С названием docker-compose.monitoring.yml
> В него выносится все что относится к этим сервисам (volumes, services)

Ноде `worker-3` установлю **label**:

```bash
 docker node update --label-add monitoring=1 worker-3
```

В файл `docker-compose.yml`, сервисам добавлю 

```
        placement:
            constraints:
              - node.role == worker
              - node.labels.monitoring != 1
```

В файл `docker-compose.monitoring.yml` сервисам добавлю

```
    deploy:
        placement:
            constraints:
              - node.labels.monitoring == 1
```

До указания `placement constraints` с лейблем *monitoring* контейнеры были "размазаны" по нодам кластера.
Интересно то, что после указания ограничений, относительно `node.labels.monitoring`, сервисы которые были развернуты на `worker-3`, например **ui**, не были удалены, а просто остановлены:

```
docker stack ps DEV
ID                  NAME                                          IMAGE                        NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
rzrhq19up36s        DEV_node-exporter.p4fwiti2r1c93tupz61trly2s   prom/node-exporter:v0.15.2   worker-2            Running             Running 4 minutes ago                        
5ka1sfex66to        DEV_node-exporter.lvyuj24iwuvoyohjmvmait4sa   prom/node-exporter:v0.15.2   worker-1            Running             Running 4 minutes ago                        
iqb4vi74xxvf        DEV_node-exporter.v2317it64mxhdi81zg7tmzvg6   prom/node-exporter:v0.15.2   master-1            Running             Running 4 minutes ago                        
p3ig2zayfyu9        DEV_node-exporter.yddrg3mgqhwyp0pqh8gbzj1l0   prom/node-exporter:v0.15.2   worker-3            Running             Running 4 minutes ago                        
tv6z5lnhv7jh        DEV_ui.1                                      iprior/ui:latest             worker-1            Running             Running 7 seconds ago                        
qruddp94sud1        DEV_alertmanager.1                            iprior/alertmanager:latest   worker-3            Running             Running 3 minutes ago                        
ft4aedecjica        DEV_grafana.1                                 grafana/grafana:5.0.0        worker-3            Running             Running 3 minutes ago                        
oete56fz3vq5        DEV_mongo.1                                   mongo:3.2                    master-1            Running             Running 4 minutes ago                        
cgdwnbwedchb        DEV_prometheus.1                              iprior/prometheus:latest     worker-3            Running             Running 3 minutes ago                        
nkek3qx34l7n        DEV_post.1                                    iprior/post:latest           worker-1            Running             Running 4 minutes ago                        
ziys46ya6vvm        DEV_comment.1                                 iprior/comment:latest        worker-2            Running             Running 4 minutes ago                        
zamssq6rtubt        DEV_cadvisor.1                                google/cadvisor:v0.29.0      worker-3            Running             Running 4 minutes ago                        
60h0kbg9wfo8        DEV_ui.1                                      iprior/ui:latest             worker-3            Shutdown            Shutdown 8 seconds ago                       
ftsghty9i0e1        DEV_post.2                                    iprior/post:latest           worker-1            Running             Starting 1 second ago                        
tzxbq66hva8b         \_ DEV_post.2                                iprior/post:latest           worker-3            Shutdown            Shutdown 1 second ago                        
miglnyiqexxn        DEV_comment.2                                 iprior/comment:latest        worker-1            Running             Running 4 minutes ago                        
4btezm7a45e4        DEV_ui.2                                      iprior/ui:latest             worker-1            Running             Running 4 minutes ago                        
wpfe1tstdpin        DEV_post.3                                    iprior/post:latest           worker-2            Running             Running 4 minutes ago                        
nxmvypoxw4iv        DEV_comment.3                                 iprior/comment:latest        worker-2            Running             Running 4 minutes ago                        
4k0wsgodju8d        DEV_ui.3                                      iprior/ui:latest             worker-2            Running             Running 4 minutes ago                        
uvcmuc299jck        DEV_ui.4                                      iprior/ui:latest             worker-2            Ready               Ready 2 seconds ago                          
wglbvaoz0jxf        DEV_comment.4                                 iprior/comment:latest        worker-2            Running             Running 3 seconds ago                        
c11bnscetqnh        DEV_post.4                                    iprior/post:latest           worker-1            Running             Running 4 minutes ago                        
js76pjllgu6k        DEV_comment.4                                 iprior/comment:latest        worker-3            Shutdown            Shutdown 4 seconds ago                       
h1c7g7fwuuz1        DEV_ui.4                                      iprior/ui:latest             worker-3            Shutdown            Running 2 seconds ago                        
8rcrnlqxf57r        DEV_post.5                                    iprior/post:latest           worker-2            Running             Starting 1 second ago                        
2054sdui62mf         \_ DEV_post.5                                iprior/post:latest           worker-3            Shutdown            Shutdown 1 second ago                        
atykq3r75a7q        DEV_comment.5                                 iprior/comment:latest        worker-1            Running             Running 4 minutes ago                        
tazxwk8i7b0j        DEV_ui.5                                      iprior/ui:latest             worker-2            Running             Running 4 minutes ago
```
