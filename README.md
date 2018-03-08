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

