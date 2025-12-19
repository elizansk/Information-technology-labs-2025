# Отчет по лабораторной работе

## Цель работы
**1. Развернуть GitLab и Nexus с помощью Docker Compose.**

**2.Создать простое Python-приложение и покрыть его модульными тестами.**

**3. Настроить GitLab CI/CD:**
- автоматический запуск тестов;
- сборку Docker-образа;
- публикацию образа в Docker-репозиторий Nexus.

## Задача 1. Подготовка окружения (GitLab + Nexus)

### Выполняемые действия 

1. Мной был создан и настроен `docker-compose.yaml` файл со следующим содержанием 
***(рис - 1. [docker-compose1](screen/docker-compose1.png))***:


```yaml
version: "3.9"

networks:                           
  devops-lab:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.0.0/16

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    ports:
      - "8081:80"
      - "2222:22"
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/logs:/var/log/gitlab
      - ./gitlab/data:/var/opt/gitlab
    restart: always
    networks:
      devops-lab:
        ipv4_address: 172.21.0.2

  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    ports:
      - "8082:8081"
      - "5000:5000"
    volumes:
      - ./nexus:/nexus-data
    restart: always
    networks:
      devops-lab:
        ipv4_address: 172.21.0.5
```
**version: "3.9"** - версия файла docker-compose.

**networks:** — раздел для описания пользовательских Docker-сетей.

**devops-lab:** — имя пользовательской сети, в которой работают все контейнеры.

**driver: bridge** — тип сети Docker, стандартная изолированная bridge-сеть.

**ipam:** — настройка управления IP-адресами внутри сети.

**config:** — конфигурация IP-адресного пространства.

**subnet: 172.21.0.0/16** — диапазон IP-адресов, используемый в сети devops-lab.

---

**services:** — раздел с описанием всех контейнеров.

***GitLab***

**gitlab:** — сервис GitLab.

**image: gitlab/gitlab-ce:latest** — Docker-образ GitLab Community Edition.

**container_name: gitlab** — имя контейнера GitLab.

**ports:** — проброс портов контейнера на хост.

**"8081:80"** — доступ к веб-интерфейсу GitLab через порт 8081.

**"2222:22"** — проброс SSH-порта для работы с Git по SSH.

**volumes:** — подключение директорий хоста для хранения данных.

**./gitlab/config:/etc/gitlab** — конфигурационные файлы GitLab.

**./gitlab/logs:/var/log/gitlab** — логи GitLab.

**./gitlab/data:/var/opt/gitlab** — основные данные GitLab.

**restart: always** — автоматический перезапуск контейнера.

**networks:** — подключение контейнера к сети.

**devops-lab:** — использование сети devops-lab.

**ipv4_address: 172.21.0.2** — фиксированный IP-адрес контейнера GitLab.

---

***Nexus***

**nexus:** — сервис Nexus Repository Manager.

**image: sonatype/nexus3:latest** — Docker-образ Nexus Repository Manager 3.

**container_name: nexus** — имя контейнера Nexus.

**ports:** — проброс портов Nexus.

**"8082:8081"** — доступ к веб-интерфейсу Nexus через порт 8082.

**"5000:5000"** — порт Docker-реестра Nexus.

**volumes:** — хранение данных Nexus.

**./nexus:/nexus-data** — директория с данными Nexus.

**restart: always** — автоматический перезапуск контейнера.

**networks:** — подключение контейнера к сети.

**devops-lab:** — использование сети devops-lab.

**ipv4_address: 172.21.0.5** — фиксированный IP-адрес контейнера Nexus.

2. Далее было выполнено поднятие всех контейнеров командой:

```shell
docker compose up -d
```

3. Со своей личной машины подключилась к gitlab - 192.168.1.142:8081 ***(рис - 2. [ GitlabLoginPage](screen/GitlabLoginPage.png))***
и к nexus - 192.168.1.142:8082 ***(рис - 3. [ NexusLoginPage](screen/NexusLoginPage.png))***.
192.168.1.142 - внешний IP моей ВМ. 8081, 8082 - порт на котором развернут Gitlab и Nexus.

### Возникшие проблемы

Изначально оперативная память на ВМ - 2ГБ. Из-за этого ВМ просто зависала и время отклика было 10-15 минут.

**Решение** - увеличила память до 8ГБ

## Задача 2. Создание Python-приложения

### Выполняемые действия 

#### Вход на Gitlab

1. Зашла на страницу регистрации ввела все необходимые данные;
2. После успешной регистрации необходимо подтверждения от администратора (`root).
Для этого получила пароль от root командой:

```shell
cat gitlab/config/initial_root_password
```
3. Далее произвела вход в Gitlab от root. Зашла в управление пользователями и допустила вход зарегесттрированному мной пользователю.
4. Создала Project и по склонировала репозиторий себе на машину коммандой:

```shell
git clone http://192.168.1.142:8081/Eliza/devops-lab-python.git
```

#### Создание приложения

1. Создала файл `text_tool.py` с следующим кодом:

```python
def count_words(text):
    """Подсчитывает количество слов в тексте."""
    return len(text.split())


def longest_word(text):
    """Находит самое длинное слово в тексте."""
    words = text.split()
    return max(words, key=len) if words else None
```

2. Написала модульные тесты в файле `test_text_tool.py`:

```python
import pytest
from text_tool import count_words, longest_word


def test_count_words():
    assert count_words("Hello world") == 2
    assert count_words("") == 0


def test_longest_word():
    assert longest_word("Hello world") == "Hello"
    assert longest_word("") is None
```

3. Создайте файл `requirements.txt`:

```text
pytest
```

4. Сделала commit и push на удаленный репозиторий ***(рис - 4. [ RepositoryStructure](screen/RepositoryStructure.png))***:

```shell
git commit -m "Добавлены файлы test_text_tool text_tool requirements"
git push origin main
```

## Задача 3. GitLab CI: тесты + сборка и push Docker-образа в Nexus. Пункт -  3.1. Этап тестирования

### Создание .gitlab-ci.yml

1. Создала файл `.gitlab-ci.yml` в корне репозитория с этапами `test` и `build`:
```yaml
stages:
  - test
  - build

test:
  stage: test
  image: python:3.10
  script:
    - pip install -r requirements.txt
    - pytest --junitxml=report.xml
  artifacts:
    paths:
      - report.xml
```

### Создание и регистрация gitlab-runner

#### Выполняемые действия
1. Первоначально удалила все контейнеры

```shell
docker compose down
```

2. Для создания gitlab-runner мне было необходимо отредактировать `docker-compose.yaml` ***(рис - 5.[ docker-compose](screen/docker-compose.png))***:

```yaml
version: "3.9"

networks:
  devops-lab:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.0.0/16

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    ports:
      - "8081:80"
      - "2222:22"
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/logs:/var/log/gitlab
      - ./gitlab/data:/var/opt/gitlab
    restart: always
    networks:
      devops-lab:
        ipv4_address: 172.21.0.2

  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    ports:
      - "8082:8081"
      - "5000:5000"
    volumes:
      - ./nexus:/nexus-data
    restart: always
    networks:
      devops-lab:
        ipv4_address: 172.21.0.5

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    volumes:
      - ./gitlab-runner-config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      devops-lab:
        ipv4_address: 172.21.0.3
```
### GitLab Runner

**gitlab-runner:** — сервис GitLab Runner, отвечающий за выполнение CI/CD job.

**image: gitlab/gitlab-runner:latest** — Docker-образ GitLab Runner последней версии.

**container_name: gitlab-runner** — имя контейнера GitLab Runner.

**restart: always** — автоматический перезапуск контейнера при сбоях или перезагрузке системы.

**volumes:** — подключение директорий хоста внутрь контейнера.

**./gitlab-runner-config:/etc/gitlab-runner** — хранение конфигурации GitLab Runner (включая файл `config.toml`) на хосте.

**/var/run/docker.sock:/var/run/docker.sock** — доступ контейнера GitLab Runner к Docker API хоста для запуска job-контейнеров.

**networks:** — подключение контейнера GitLab Runner к сети Docker.

**devops-lab:** — использование пользовательской сети devops-lab.

**ipv4_address: 172.21.0.3** — фиксированный IP-адрес контейнера GitLab Runner внутри сети.

3. Запустила все контейнеры снова (вместе с runner)

```shell
docker compose up -d
```

4. Далее необходимо было зарегестрировать runner и связать его с gitlab.
5. Зашла на gitlab -> setting -> CI/CD -> Runners и нажала на кнопку `create project runner`. 
6. Установила флаг на `"Run untagged jobs"` для того чтобы не прописывать теги внутри файла `.gitlab-ci.yml`.
Нажала `create runner`
7. Вернулась в терминал. Начала регистрацию ранера с помощью комманды:

```shell
docker exec -it gitlab-runner gitlab-runner register
```
8. Указала URL, токен, описание и способ запуска
   1. URL - http://gitlab:80 (имя контейнера)
   2. token - для получения токена вернулась на страницу gitlab и там был представлен токен для связывания runner и gitlab.
   3. Описание - оставлено по умолчанию (имя контейнера)
   4. Способ запуска - docker
   5. image - docker:24
9. Вернулась на главный экран и увидела, что раннер успешно зарегестрирован ***(рис 6 - [RunnerSuccessfullyRegistered](screen/RunnerSuccessfullyRegistered.png))***.

#### Возникшие проблемы

Несмотря на проделанные ранее действия `test` все равно не проходил успешно.

**Решение** - отредактировать файл config.toml у gilab-runner и добавить в конфигурацию строку `network_mode = "devops-lab_devops-lab"`, где `devops-lab_devops-lab` - название сети контейнеров Docker.

1. Для этого были воплнена комманда

```shell
nano gitlab-runner-config/config.toml 
```

2. Добавлена строка `network_mode = "devops-lab_devops-lab"`
3. Перезапустила контейнер gitlab-runner:

```shell
docker restart gitlab-runner
```
4. Для повторных тестов были внсены изменения на удаленном репозитории.


## Задача 3. GitLab CI: тесты + сборка и push Docker-образа в Nexus. Пункт - 3.2 Этап сборки и публикации Docker-образа в Nexus

### Создание регистрация Nexus и создание variables gitlab

1. Зашла в nexus адресу 192.168.1.142:8082;
2. Вернулась в терминал для получения пароля от учетной записи admin:

```shell
cat /nexus/admin.password
```

3. Зашла в Nexus логин - admin, пароль - полученный ранее пароль. Сменила пароль на новый;
4. Зашла в gitlab адресу 192.168.1.142:8081;
5. Далее Setting -> CI/CD -> Variables
6. Добавила 2 переменные с key NEXUS_PASSWORD и NEXUS_USER. Value такие же как и у логина и пароля Nexus ***(рис 7 - [GitLabVariables](screen/GitLabVariables.png))***;
7. Вернулась в Nexus. setting -> repository -> create repository -> docker hosted;
8. Создала репозиторий ***(рис 8 - [NexusRepository](screen/NexusRepository.png))***

### Редактирование .gitlab-ci.yml, добавление Dockerfile и редактирование /etc/docker/daemon.json

1. Создала `Dockerfile`:

```dockerfile
FROM python:3.10
WORKDIR /app
COPY text_tool.py /app/
CMD ["python"]
```

2. В файле /etc/docker/daemon.json добавила адрес `nexus` в insecure (в файле все попытки):

```json
{
  "insecure-registries": [
    "localhost:5000",
    "nexus:5000",
    "192.168.1.142:5000",
    "172.21.0.5:5000"
  ]
}
```

3. Перезапустила docker на ВМ:

```shell
systemctl restart docker
```

4. Далее на удаленном репозитории привела файл .gitlab-ci.yml к такому виду 
(P.S. Это не первая версия файла, данная версия получилась в результате многих ошибок до этого):

```yaml
stages:
  - test
  - build

test:
  stage: test
  image: python:3.10
  script:
    - pip install -r requirements.txt
    - pytest --junitxml=report.xml
  artifacts:
    paths:
      - report.xml

build_and_push:
  stage: build
  image: docker:24.0.1
  variables:
    DOCKER_HOST: unix:///var/run/docker.sock
    DOCKER_REGISTRY: "172.21.0.5:5000"
    DOCKER_IMAGE: "$DOCKER_REGISTRY/python-lab"
    DOCKER_API_VERSION: "1.44"
  script:
    - docker info
    - docker login "$DOCKER_REGISTRY" -u "$NEXUS_USER" -p "$NEXUS_PASSWORD"
    - docker build --platform linux/arm64 -t "$DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA" .
    - docker push "$DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA"
  needs:
    - test
  only:
    - main
```

#### Возникшие проблемы

##### 1. Не работающая конфигурация config.toml

Несмотря на проделанные ранее действия `build` все равно не проходил успешно.

**Решение** - отредактировать файл config.toml у gilab-runner и привести к виду строку `volumes = ["/var/run/docker.sock:/var/run/docker.sock","/cache"]` тем самым добавив values docker.sock.
В результате docker.sock ВМ -> docker.sock gitlab-runner -> docker.sock внутри исполняющих контейнеров gitlab-runner.

Итоговый вид config.toml:

```toml
concurrent = 1
  check_interval = 0
  shutdown_timeout = 0

  [session_server]
  session_timeout = 1800

  [[runners]]
  name = "aef38a64b8ed"
  url = "http://gitlab:80"
  id = 3
  token = "glrt-3Z0_a4lvlzrqdPQZnKkoum86MQpwOjEKdDozCnU6Mg8.01.170e2a1wv"
  token_obtained_at = 2025-12-18T06:28:01Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
  MaxUploadedArchiveSize = 0
  [runners.cache.s3]
  [runners.cache.gcs]
  [runners.cache.azure]
  [runners.docker]
  tls_verify = false
  image = "docker:24.0"
  privileged = false
  disable_entrypoint_overwrite = false
  oom_kill_disable = false
  disable_cache = false
  volumes = ["/var/run/docker.sock:/var/run/docker.sock","/cache"]
  shm_size = 0
  network_mtu = 0
  network_mode = "devops-lab_devops-lab"
```

##### 2. Не удача подключения DOCKER_REGISTRY внутри .gitlab-ci.yml по наименованию контейнера

Первоначально была задача, чтобы внутри `.gitlab-ci.yml` `DOCKER_REGISTRY` имел следующий вид:

```yaml
    DOCKER_REGISTRY: "nexus.local:5000"
```

Данный вариант не сработал. В дальнейшем была попытка подключения по IP ВМ:

```yaml
    DOCKER_REGISTRY: "192.168.1.142:5000"
```

Данный вариант конфигурации сработал, но из-за динамических IP при смене сети данный вариант пришлось заменить.

**Решение** - установить сеть внутри docker, прописать заранее известные IP для каждого контейнера и установить DOCKER_REGISTRY по IP адресу контейнера:

```yaml
    DOCKER_REGISTRY: "172.21.0.5:5000"
```

Также появилась необходимость установить общую сеть и заранее известные IP адреса каждому контейнеру.
В изначальном варианте `docker-compose.yaml` не было информации об общей сети внутри контейнера (выше представлена с сетью тк изначальные файлы не сохранились)
и все контейнеры были в общей сети `devops-lab_default`.

### Получение результата в Gitlab и Nexus

1. Зашла в Gitlab в свой проект. Зафиксировала успешность пайплана в Gitlab ***(рис 9 - [SuccessfulPipeline](screen/SuccessfulPipeline.png))***;
2. Зашла в Nexus. Browse -> docker. Зафиксировала успешность загрузки Docker-образа (`python-lab`) ***(рис 10 - [SuccessfulDockerNexus](screen/SuccessfulDockerNexus.png))***.
3. Итоговое содержание репозиория на Gitlad ***(рис 11 - [ SuccessfulGitLab](screen/SuccessfulGitLab.png))***;

## Заключение

В лабораторной работе я развернула GitLab, GitLab Runner и Nexus с помощью Docker Compose, настроив их для совместной работы в одной сети.

Создано простое Python-приложение с функциями подсчёта слов и поиска самого длинного слова, написаны модульные тесты и загружены в репозиторий GitLab.

Настроен GitLab CI/CD: при пуше в репозиторий автоматически запускаются тесты, собирается Docker-образ и публикуется в Nexus. GitLab Runner был зарегистрирован и подключён к Docker на хосте через `docker.sock`.

Все возникшие проблемы (ограниченные ресурсы ВМ, настройка сети и Runner) были успешно решены. В итоге DevOps-цикл был полностью реализован, а цели лабораторной работы достигнуты.
