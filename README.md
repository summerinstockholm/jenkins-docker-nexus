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