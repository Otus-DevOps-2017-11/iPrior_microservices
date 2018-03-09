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
