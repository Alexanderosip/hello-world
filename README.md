# hello-world

Hi Humans!

Hubot here? I like alphabet
Hello from 2021
Second hello

# Развёртка, настройка и работа с контейнером klowner
Проект, обеспечивающий зеркалирование представляет из себя докер, в который завёрнут основной сервис: https://github.com/Klowner/docker-gitlab-mirrors
Инструкция по развёртке присутствует в проекте. Ниже расписана её интерпретация с корректировками на имеющиеся условия.
Создать на хосте папку /srv/klowner/config:
`sudo mkdir -p /srv/klowner/config`
В ней создать папку .ssh:
`sudo mkdir /srv/klowner/config/.ssh`
Создать файл:
`sudo vim /srv/klowner/config/.ssh/config`
```
Host gitlab.example.com
Hostname 192.168.134.128
User alexander.osipov
ForwardAgent yes
StrictHostKeyChecking no
IdentityFile ~/.ssh/osip_rsa
```
Положить в папку /srv/klowner/config/.ssh/ закрытый ключ пользователя для зеркалирования Gitlab, открытый ключ которого прописан в web-интерфейсе Gitlab:
`sudo cp  ~/.ssh/osip_rsa /srv/klowner/config/.ssh/id_rsa`
Установить на ключ права:
`chmod 600 /srv/klowner/config/.ssh/id_rsa`
Завести в интерфейсе Gitlab пользователя (специально для зеркалирования) и дать ему права Maintainer на группу, которая будет содержать зеркала.
Зайти в систему под созданным пользователем.
Прописать пользователю в его настройках ssh ключ.
В интерфейсе Gitlab сгенерировать приватный токен для API в настройках пользователя.
Токен прописать в файле:
`sudo vim /srv/klowner/config/private_token`
```
9aUMLFsWSCxZVkaBMmm1
```
Скачать контейнер klowner:
docker pull quay.io/klowner/gitlab-mirrors:latest
Все команды доступные для контейнера уже перечислены выше. Основные:
- add - добавить зеркалируемый репозиторий
- ls - перечислить заведённые репозитории
- delete -d pugixml - удалить зеркалируемый репозиторий (описана в Readme не верно, без -d)
#### Запуск контейнера командой или скриптом:
Запустить для проверки контейнер klowner из командной строки:
```
docker run --rm -it \
-v ${PWD}/config:/config \
quay.io/klowner/gitlab-mirrors:latest \
run ssh gitlab_test
```
Запустить контейнер c добавлением зеркала:
```
docker run --rm -it \
-v ${PWD}/config:/config \
-v ${PWD}/data:/data/" \
-e GITLAB_MIRROR_GITLAB_USER=alexander.osipov \
-e GITLAB_MIRROR_GITLAB_NAMESPACE=Mirrors \
-e GITLAB_MIRROR_GITLAB_URL=https://gitlab.streamlabs.cxm \
quay.io/klowner/gitlab-mirrors:latest \
add --git --project-name pugi-mirrored --mirror https://github.com/zeux/pugixml.git
```
Либо создать в любом месте скрипт скрипт:
`vim klowner_run.sh`
```
docker run --rm -i \
  -v $(dirname $SSH_AUTH_SOCK):$(dirname $SSH_AUTH_SOCK) \
  -v "${PWD}/config:/config" \
  -v "/data:/data" \
  -e SSH_AUTH_SOCK=$SSH_AUTH_SOCK \
  -e GITLAB_MIRROR_UID=$(id -u git) \
  -e GITLAB_MIRROR_GITLAB_USER=mirror.user \
  -e GITLAB_MIRROR_GITLAB_NAMESPACE=third-party \
  -e GITLAB_MIRROR_GITLAB_URL=https://gitlab.streamlabs.cxm \
  -e GITLAB_MIRROR_SSL_VERIFY=false \
  quay.io/klowner/gitlab-mirrors:latest ${@:1}
```
Запустить скрипт с пробросом действия в качестве параметра:
`./klowner_run.sh add --git --project-name pugi-mirrored --mirror https://github.com/zeux/pugixml.git`

Чтобы прописать команду добавления репозитория прямо в скрипт, изменяем скрипт, прописав в него любую команду:
```
docker run --rm -i \
  -v $(dirname $SSH_AUTH_SOCK):$(dirname $SSH_AUTH_SOCK) \
  -v "${PWD}/config:/config" \
  -v "/data:/data" \
  -e SSH_AUTH_SOCK=$SSH_AUTH_SOCK \
  -e GITLAB_MIRROR_UID=$(id -u git) \
  -e GITLAB_MIRROR_GITLAB_USER=mirror.user \
  -e GITLAB_MIRROR_GITLAB_NAMESPACE=third-party \
  -e GITLAB_MIRROR_GITLAB_URL=https://gitlab.streamlabs.cxm \
  -e GITLAB_MIRROR_SSL_VERIFY=false \
    quay.io/klowner/gitlab-mirrors:latest add --git ${REPO_NAME} -- url ${REPO_URL}
```

Контейнер сохраняет конфигурацию добавленных зеркалируемых репозиториев в своё хранилище на вольюме /data, который необходимо прокинуть в контейнер из раннера, в котором будет запускаться проект. (Проброс вольюма в Readme не верно, необходимо пробрасывать папку /data на папку /data, в ней и будут храниться конфигурации, а не в папке /data/mirror, как сказано в Readme)
Хранилище в папке /data должно сохраняться, чтобы не изменяться между запусками докера.

Для автоматизации некоторых вышеописанных действий, а также для решения некоторых проблем, связанных с правами и пользователями на раннере, в основной скрипт запуска докера ./klowner-run.sh были добавлены несколько блоков кода:
```
chmod 644 ./config/.ssh/config
chmod 600 ./config/.ssh/id_rsa

echo "pwd: $(pwd)"
echo REOPO_NAME=$REPO_NAME
echo REPO_URL=$REPO_URL
echo "command: ""${@:1}"""

if ! id "git" &>/dev/null; then
  echo "User git does not exists and will be created."
  yes | adduser -s /bin/bash -u 1010 git
  if id "git" &>/dev/null; then
    echo "User git was created successfully."
  fi
else
  echo "User git exists in system"
fi
chown git:git ./config/*
chown git:git ./config/.ssh/*
```
Для запуска настроенного докера и всех конфигурационных файлов на Gitlab необходимо в корневой папке создать файл:
./.gitlab-ci.yml. В нём стейдж mirrors запускается только для пайпов, запущенных по расписанию. Остальные стейджи запускаются только вручную.
Содержимое файла см. в проекте [gitlab_mirror_handler](https://gitlab.streamlabs.cxm/third-party/gitlab_mirror_handler)

После внесения всех изменений, все изменения необходимо в той же группе third-party на Gitlab создать проект в который войдут созданные выше файлы.
Закоммитить и отправить все файлы в данный проект.
Настроить для этого проекта запуск Pipeline по расписанию, чтобы зеркала периодически обновлялись.
Добавить проекты посредством запуска стейджа add вручную, добавляя переменные:
REPO_NAME = <Любое локальное имя зеркалируемого репозитория>
REPU_URL = <URL удалённого репозитория>
Только проекты добавленные через add будут обновляться в дальнейшем. Обо всех остальных проектах в группе third-party сервис обновления не будет осведомлён.
Удаление проекта и просмотр списка зеркал описаны в первой части данного документа.
