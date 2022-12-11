# jenkins-docker-nexus
CI/CD с помощью Jenkins, Docker и Nexus.
Итак стоит задача допустим-то cоздать 2 задачи Jenkins: Первая задача должна собирать Docker
образ с заданным тегом (например v1.0), на сборочной ноде, и
пушить его в registry.
Вторая Job'а должна пулить (подтягивать) Docker образ с требуемым тегом из registry и запускать контейнер на продовой ноде.
Итого нужно 4 хоста. 
1ый для Jenkins
2ой для сборки докер образа
3ий Nexus для хранения артефактов
4ый для запуска полученного артефакта

Пререквесты:
Создать 4 вм:
yc-jenkins-host
yc-build-host
yc-nexus
yc-deploy-host
на трех последних задать пароль для рута через passwd root, затем сделать авторизацию по руту с паролем отредактировав файл /etc/ssh/sshd_config а точнее
выставить:
PermitRootLogin yes
PasswordAuthentication yes
ChallengeResponseAuthentication yes
а затем рестартануть демон sshd
service sshd restart
На yc-jenkins-host установить Jenkins в виде докер-контейнера предварительно установив сам докер.
На yc-build-host и yc-deploy-host установить docker. 
На yc-nexus установить wget, докер и докер-компоуз. Докер-компоуз установить через wget в домашний каталог, а затем переместить mv в директорию /bin/docker-compose
и дать бит на исполнение бинарнику docker-compose.
Форкнуть в свой личный гитхаб проект https://github.com/boxfuse/boxfuse-sample-java-war-hello и добавить в него Dockerfile: https://github.com/summerinstockholm/dockerfile_tomcat_boxfuse_sample_java/blob/main/Dockerfile
В результате получится репозиторий c докерфайлом:
https://github.com/summerinstockholm/boxfuse-sample-java-war-hello


1. Установка nexus и настройка нексус.
На yc-nexus выполнить установку nexus через docker-compose. Спуллить на хост yaml отсюда
https://github.com/summerinstockholm/docker-compose-nexus
В полученной директории создать папку data и применить к ней: 
mkdir -p /data
chown -R 200 /data
После из директории с yaml файлом выполнить
docker-compose up -d
Нексус запустится через некоторое время. Первичный пароль лежит в папке /data. Затем необходимо создать докер-репозиторий в /#admin/repository/repositories
Указав к нему порт http 8123. В Security выбрать Realms. Docker Bearer Token переместить из левой таблицы в правую. Сохранить.

2. Настройка Jenkins.
Добавить в Jenkins все креды от всех хостов, в том числе отдельно добавить Nexus пользователя (admin). Затем в настройках дженкинса добавть SSH хосты с кредами добавлеными ранее.
Создать фристайл Build Java App параметризированный $version проект где по умолчанию указать latest. В шагах сборки выбрать Execute shell script on remote host using ssh (указав вставив yc-build-host) туда скрипт:
git clone https://github.com/summerinstockholm/boxfuse-sample-java-war-hello
cd boxfuse-sample-java-war-hello
git pull origin master
docker build --no-cache -t tomcat_boxfuse_sample_java:$version . 
docker tag tomcat_boxfuse_sample_java:$version 84.201.177.137:8123/tomcat_boxfuse_sample_java:$version
docker login 84.201.177.137:8123 -u admin -p *пароль от пользователя нексуса*
docker push 84.201.177.137:8123/tomcat_boxfuse_sample_java:$version
Перейти по ssh на yc-build-host и yc-deploy-host, где необходимо в /etc/docker/daemon.json согласно статье:
https://docs.docker.com/registry/insecure/ указать инсекьюрные http конекты к репозиториям нексуса.
{
  "insecure-registries" : ["84.201.177.137:8123"]
}
Создать фристайл Deploy Java App параметризированный $version проект где по умолчанию указать latest. В шагах сборки выбрать Execute shell script on remote host using ssh (указав yc-deploy-host) вставив туда скрипт:
docker login 84.201.177.137:8123 -u admin -p *пароль от пользователя нексуса*
docker kill $(docker ps -q)
docker pull 84.201.177.137:8123/tomcat_boxfuse_sample_java:$version
docker run -d -p 8080:8080 $(docker images --format "{{.ID}}")

В случае успеха первая джоба соберет проект и запушит его в nexus, а вторая джоба задеплоит его на деплой-хосте.