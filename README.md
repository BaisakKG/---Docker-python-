# Debian9 + Docker + Python3 + Postgresql > DigitalOcean
## Скачиваем образ питона
```
docker ps #работающие докеры
docker images #готовые контейнеры в докер хабе
docker pull python:3 #скачиваем с докер хаба
docker run -it python:3 python
```
***-it** -- говорит что будем работать c запущенным контейнером в интерактивном режиме\
**python:3** -- указываем имя контейнера\
**python** -- указываем ту программу которую хотим запустить внутри контейнера*

## Ставим дополнительные зависимости
```
docker run -it python:3 pip install django #--proxy <proxy> --если через прокси
docker ps -a #смотрим историю

#после взять ID и посмотреть все файлы и зависимости
docker diff  <ID>

#после можно закомитить этот контейнер и создать образ
docker commit  <ID>  mydjango

#если теперь посмотреть появился новый образ mydjango
docker images | grep mydjango

#можем его запустить и работать сним
docker run -it mydjango django-admin.py --help
docker ps -a

#после каждого запуска, копятся новые слои изменений на файловой системе,поэтому не нужные надо удалять
docker rm <ID CONTAINER> <ID CONTAINER> <ID CONTAINER>

#можно запустить контейнер с очисткой --rm (чтобы Docker автоматически очищал контейнер и удалял файловую систему при выходе из контейнера )
docker run --rm -it mydjango django-admin.py --help
```

## Пробуем другой более удобный вариант
```
mkdir /opt/docker-workshop
cd /opt/docker-workshop

nano Dockerfile
#здесь последовательно описываем что мы хотим сделать
FROM python:3
RUN pip install django #--proxy <proxy>

#теперь можем собрать контейнер
docker build -t mydjango .
```
***-t** -- указываем имя контейнера\
**.** -- путь до дериктории где находится докер файл*

Если запустить через *docker run* он запуститься внутри контейнера, но мы хотим чтобы изменения сбрасывались в файловую систему хоста, для этого есть система томов(для постгреса или для разработки, когда меняеш код локально она изменялась и в контейнере, для работы с томами используется ключик **-v**

```
#для начало посмотрим что представляет из себя этот контейнер
docker run --rm -it mydjango bash

ls
#мы видим что находимся в корне образа, и если мы выполним какую нибудь команду нр: start project он создатся в корне проекта, это неудобно, поэтому создаем спец директорию
exit

nano Dockerfile
....
RUN mkdir /data
WORKDIR /data #программа которая будет запускаться, будет брать это в качестве корневой директории

#снова собираем
docker build -t mydjano .

#ели теперь проверить
docker run --rm -it mydjango bash 

#мы оказываемся в /data
ls
exit

#мы с пом-ю -v подбросим наш текущий директорий в /data в контейнере
docker run -v `pwd`:/data --rm -it mydjango django-admin.py startproject mynewproject
ls

#создался проект и видно что он создался и внашей директории хост системы
ls -la

#видно что права на проект рутовский, потому что внутри контейнера только один юззер рут
#можно поменять так: chown user:user ./ -R

#запускаем проект
docker run -v `pwd`:/data --rm -it mydjango ./mynewproject/manage.py runserver

#но чтобы иметь доступ к ниму, надо запустить след. образом
docker run -p 8000:8000 -v `pwd`:/data --rm -it mydjango ./mynewproject/manage.py runserver 0.0.0.0:8000
```
***0.0.0.0:8000** -- чтобы веб сервер слушал все интерфейсы\
**-р 8000:8000** -- подбрасываем порты локальный:контейнер*

## Для управление целой инфраструктурой и несколькими контейнерами
```
nano Dockerfile
FROM python:3
RUN mkdir /data
WORKDIR /data
ADD requirements.txt /data
RUN pip install -r requirements.txt
ADD . /data

mv Dockerfile mynewproject

nano mynewproject/requirements.txt
django

nano docker-compose.yml 
version: '2'
services:
 web:
  build: ./mynewproject
  command: python ./manage.py runserver 0.0.0.0:8000
  ports:
  - 8000:8000
  volumes:
  - ./mynewproject:/data 

docker-compose build
#если нету: apt install docker-compose

#и запускаем
docker-compose up
```

## Добавляем PSQL
```
nano docker-compose.yml 
version: '2'
services:
 web:
  build: ./mynewproject
  command: python ./manage.py runserver 0.0.0.0:8000
  ports:
  - 8000:8000
  volumes:
  - ./mynewproject:/data 
  depends_on:
  - db    #указываем зависимость от контейнера db

 db:
  image: postgres:9.4

#запускаем
docker-compose up

#теперь надо подключить к db через переменное окружение
nano docker-compose.yml 
version: '2'
services:
 web:
  build: ./mynewproject
  command: python ./manage.py runserver 0.0.0.0:8000
  ports:
  - 8000:8000
  volumes:
  - ./mynewproject:/data 
  depends_on:
  - db
  environment:
    DATABASE_URL: postgres(указываем это постгрес)://postgres(пользователь)@db(находится)/postgres(базаданных)

 db:
  image: postgres:9.4

#поменять настройки
nano mynewproject/mynewproject/settings.py

import dj_database_url
DATABASES = {
 'default': dj_database_url.config()
}

#теперь пересобираем наш контейнер
docker-compose build

#запускаем
docker-compose up

#запускаем migrate
docker-compose run --rm web python ./manage.py migrate

#создаем юзера
docker-compose run --rm web python ./manage.py createsuperuser
```
## Подключение PSQL
```
#для просмотра проброшенных портов
docker ps
docker port  <ID>

#заходим в shell
docker-compose run --rm db psql -h db -U postgres postgres(это БД)
\dt
#и видим табл. джанго
\q

#делаем dump
docker-compose run --rm db pg_dump -h db -U postgres postgres > database.sql

#смотрим заголовки
head database.sql
```

## Deploy on DigitalOcean
```
docker-machine #позволяет управлять виртуальными серверами на которых крутится докеры 

docker-machine create --driver digitalocean --digitalocean-access-token $DIGITAL_OCEAN_TOKEN mytestserver(назовем)

docker-machine env mytestserver

eval $(docker-machine env mytestserver)

docker-compose build

docker-compose up

docker-compose run --rm web bash

ls

exit

docker-machine ssh mytestserver

ls /home/user/workshop/docker-workshop/mynewproject

touch ls /home/user/workshop/docker-workshop/mynewproject/somefile

exit

docker-compose run --rm web bash

ls

exit

#создаем compose для девелопера и для продакшна
cp docker-compose.yml docker-compose.staging.yml
cp docker-compose.yml docker-compose.develop.yml

vim docker-compose.staging.yml
version: '2'
services:
 web:
   ports:
   - 80:8000

vim docker-compose.develop.yml
version: '2'
services:
 web:
  ports:
  - 8000:8000
  volumes:
  - ./mynewproject:/data

#запускаем по разному от нужды
docker-compose -f docker-compose.yml -f docker.compose.staging.yml up -d
# -d -- бекграунд режим

#зальем дамп который делали ранее на продакшн
#docker-machine ip mytestserver

docker-compose run --rm db psql -h db -U postgres postgres < database.sql
```

**и все !!!**

Ссылка на оригинал: https://www.youtube.com/watch?v=5LuHkG3fiFY


