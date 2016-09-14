# Vibrobox Docker Nginx Proxy

Данный репозиторий предназначен для установки на сервера, которые должны включать в себя
множество сервисов. Посколько все они не могут занять 80 порт, а вручную вести настройку
nginx хостов слишком утомительно, и был создан этот репозиторий.

Пригодно для использования на dev машинах.

## Установка на сервер

_Стоит понимать, что настройка выполняется один раз и работает в рамках всего сервера._

Сперва нужно создать сеть (network):

```sh
docker network create nginx-proxy
```

Затем клонируем этот репозиторий в любое удобное место и переходим в его папку:

```sh
git clone git@github.com:VibroBox/nginx-proxy.git
cd nginx-proxy
```

После просто поднимаем контейнеры:

```sh
docker-compose up -d
```

На этом настройка звершена. Контейнеры скачаются\построятся и запустятся. Если сервер будет
перезагружен, то контейнеры автоматически запустятся вместе с системой.

## Конфигурация проектов для взаимодействия

* [Документация по nginx-proxy](https://github.com/jwilder/nginx-proxy#usage).

nginx-proxy автоматически подхватывает лишь те контейнеры, у которых задана environment
variable `VIRTUAL_HOST`. Если этого не сделать, то nginx-proxy не сгенерирует конфигурации
и ничего не будет работать. Кроме того, контейнер должен экспозить (expose) хотя бы один
порт (nginx и apache автоматически экспозят 80 и 443 порты).

Кроме того, необходимо обеспечить работу всех контейнеров, что должны выходить во внешнюю
сеть (интернет) внутри одной внутренней сети. Командами выше мы создали сеть `nginx-proxy`.
Следовательно, те контейнеры, что должны проксироваться, также должны находиться в этой
сети.

Ниже приведён пример конфигурации docker-compose.yml для проекта, которых хочет быть
проксирован под именем `example.com`:

```yml
version: '2'
services:
    web:
        from: nginx
        environment:
            - VIRTUAL_HOST=example.com
        networks:
            - default
            - nginx-proxy

# Этим самым мы связываем внутреннюю сеть контейнера с глобальной, которую мы создали ранее
networks:
    nginx-proxy:
        external:
            name: nginx-proxy
```

## SSL сертификаты

* [Документация по nginx-proxy (SSL support)](https://github.com/jwilder/nginx-proxy#ssl-support).

В compose файле этого репозитория уже прописаны необходимые volume. Следуя инструкции
по ссылке выше, сертификаты необходимо размещать в папке `certs`.

#### Let's Encrypt

* [Документация по letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion).

Также возможна автоматическая выписка и дальнейшее автообновление сертификатов
Let's Encrypt. Чтобы включить эту функцию, необходимо в environment параметрах
контейнера наряду с `VIRTUAL_HOST` указать два дополнительных:

* `LETSENCRYPT_HOST` - задаёт имя хоста, на которое выписывается сертификат.
  Должен совпадать с `VIRTUAL_HOST` и, по сути, является просто триггером к тому,
  чтобы контейнер-компаньон сгенерировал сертификат.
 
* `LETSENCRYPT_EMAIL` - задаёт E-mail, к которому будет привязан сертификат.
  Никакой проверки не производится, можно писать что угодно, но стоит помнить о тому,
  что на этот E-mail будут приходить уведомления от Let's Encrypt в случае чего.

Сертификаты генерируются в течение пары минут. Если этого так и не произошло, то можно
просмотреть логи командой:

```sh
docker-compose logs -f --tail 30 letsencrypt-nginx-proxy-companion
# Выход совершается комбинацией клавиш Ctrl+C
```

Обращу также внимание на то, что сертификаты будут успешно выписаны только в том
случае, если сервер реально доступен извне по указанному имени хоста. Выписать таким
образом сертификат на .local или иную другую несуществующую и недоступную доменную
зону/домен не выйдет.

## Basic Authentication

* [Документация по nginx-proxy (Basic Authentication)](https://github.com/jwilder/nginx-proxy#basic-authentication-support).

* [Пример работы с утилитой htpasswd](http://www.cyberciti.biz/faq/create-update-user-authentication-files/).

В compose файле этого репозитория уже прописаны необходимые volume. Следуя инструкции
по ссылке выше, файлы с логинами и паролями необходимо размещать в папке `htpasswd`.

Как пример, чтобы установить Basic Authentication на хост `example.com`, необходимо
выполнить следующие действия (предполагается, что консоль открыта в папке htpasswd):

```sh
htpasswd -c example.com my-username
# Далее будет запрошен пароль для пользователя
```

Если необходимо добавить ещё одного пользователя, то необходимо выполнить почти
такую же команду, только без флага `-c`:

```sh
htpasswd example.com another-user
# Далее будет запрошен пароль для нового пользователя
```

**Важно**: при создании файла для хоста не происходит автоматической перезагрузки
конфигурации nginx-proxy, так что после того, как вы создали файл, необходимо
выполнить перезагрузку контейнера nginx-proxy:

```sh
docker-compose restart nginx-proxy
```
