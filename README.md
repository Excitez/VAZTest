# VAZTest
# Описание выполненной работы
## CI/CD пайплайн для микросервисного приложения Online Boutique

---

## 1. Общее описание

В рамках задания выполнено:

- Развёртывание GitLab CE на виртуальной машине в Yandex Cloud
- Подготовка инфраструктуры для GitLab Runner и Kubernetes-кластера
- Разработка полного CI/CD конвейера для мультисервисного приложения

Исходный код: https://github.com/GoogleCloudPlatform/microservices-demo — перенесён в собственный GitLab-экземпляр.

Приложение состоит из 11 микросервисов на разных языках: Go, Python, Java, C#, Node.js.

---

## 2. Инфраструктура Yandex Cloud

### 2.1. Сетевая топология

Создана виртуальная сеть `prod-network` с тремя подсетями:

| Подсеть | Назначение |
|---|---|
| `subnet-gitlab` | VM с GitLab CE |
| `subnet-cicd` | VM с GitLab Runner |
| `subnet-k8s` | Узлы Kubernetes-кластера |

### 2.2. Группы безопасности

**sg-gitlab**

Входящий трафик:

| Протокол | Порты | Источник | Описание |
|---|---|---|---|
| TCP | 80 | 0.0.0.0/0 | HTTP |
| TCP | 443 | 0.0.0.0/0 | HTTPS |
| TCP | 22 | 80.234.77.163/32 | SSH |
| Any | — | Self | Внутренний трафик |

Исходящий трафик:

| Протокол | Порты | Назначение |
|---|---|---|
| Any | — | 0.0.0.0/0 |

---

**sg-k8s**

Входящий трафик:

| Протокол | Порты | Источник | Описание |
|---|---|---|---|
| TCP | 80 | 0.0.0.0/0 | HTTP |
| TCP | 443 | 0.0.0.0/0 | HTTPS |
| TCP | 6443 | 80.234.77.163/32 | Kubernetes API |
| Any | — | Self | Внутренний трафик кластера |
| Any | — | 10.10.0.0/16 | Трафик из VPC |

Исходящий трафик:

| Протокол | Порты | Назначение |
|---|---|---|
| Any | — | 0.0.0.0/0 |

---

**sg-cicd**

Входящий трафик:

| Протокол | Порты | Источник | Описание |
|---|---|---|---|
| TCP | 22 | 80.234.77.163/32 | SSH |
| TCP | 2375–2376 | 10.10.1.13/32 | Docker API от GitLab |
| TCP | 22 | 10.10.1.13/32 | SSH от GitLab |
| ICMP | — | 10.10.1.13/32 | Ping от GitLab |

Исходящий трафик:

| Протокол | Порты | Назначение | Описание |
|---|---|---|---|
| TCP | 443 | 0.0.0.0/0 | HTTPS (registry, интернет) |
| TCP | 80 | 0.0.0.0/0 | HTTP |
| Any | — | 10.10.1.13/32 | Трафик к GitLab VM |

### 2.3. Сервисный аккаунт

Создан сервисный аккаунт `gitlab-ci-sa`. Сгенерирован авторизационный ключ в формате JSON. Аккаунту назначены права на запись и чтение в Yandex Container Registry. Ключ используется в CI/CD для аутентификации `docker login` при публикации образов.

### 2.4. Container Registry

В Yandex Container Registry создан реестр для хранения Docker-образов микросервисов. Каждый образ публикуется с двумя тегами:

- `CI_COMMIT_SHORT_SHA` — для однозначной идентификации версии
- `latest` — для удобства ручного использования

### 2.5. Kubernetes-кластер

Развёрнут управляемый Kubernetes-кластер (Yandex Managed Service for Kubernetes). Для доступа из CI/CD создан статический `ServiceAccount` с токеном без срока действия. Kubeconfig в формате base64 хранится в GitLab как защищённая переменная `KUBECONFIG_DATA`.

---

## 3. Развёртывание GitLab CE

GitLab Community Edition развёрнут на VM в подсети `subnet-gitlab`. В конфигурации `/etc/gitlab/gitlab.rb` задан `external_url`, после чего выполнена реконфигурация:

```bash
gitlab-ctl reconfigure
```

### 3.1. Структура GitLab

Создана группа `microservices` с двумя репозиториями:

| Репозиторий | Описание |
|---|---|
| `microservices-demo` | Зеркало репозитория Google, содержит код приложения и `.gitlab-ci.yml` |
| `cicd` | Шаблоны пайплайна, доступен только DevOps-инженеру |

Созданы два пользователя:

| Пользователь | Роль в microservices-demo | Доступ к cicd |
|---|---|---|
| devops | Maintainer | Есть |
| developer | Developer | Нет |

### 3.2. Защита ветки main

- Прямой push в `main` запрещён
- Изменения принимаются только через Merge Request
- Для слияния требуется апрув от двух участников
- Пайплайн запускается автоматически после слияния MR

### 3.3. Переменные CI/CD

Переменные заданы на уровне группы `microservices` — разработчики их не видят:

| Переменная | Описание |
|---|---|
| `YC_REGISTRY_ID` | Идентификатор Container Registry в Yandex Cloud |
| `YC_SA_JSON_KEY` | JSON-ключ сервисного аккаунта для `docker login` |
| `KUBECONFIG_DATA` | Kubeconfig в base64 для доступа к Kubernetes-кластеру |

---

## 4. GitLab Runner

На отдельной VM в подсети `subnet-cicd` установлен GitLab Runner. Создан системный пользователь `gitlab-runner`. Раннер зарегистрирован на уровне группы `microservices`:

| Параметр | Значение |
|---|---|
| Executor | docker |
| Теги | `docker`, `ci` |
| Docker image по умолчанию | `docker:24.0.5` |
| Уровень регистрации | Группа microservices |

Регистрация на уровне группы обеспечивает доступность раннера для всех проектов группы.

---

## 5. Структура CI/CD пайплайна

Файл `.gitlab-ci.yml` в `microservices-demo` содержит только директиву `include`, которая подключает пайплайн из репозитория `cicd`. Это разделяет ответственность: разработчики работают с кодом, DevOps управляет пайплайном независимо.

Пайплайн состоит из трёх стадий:

```
lint  →  build  →  deploy
```

---

## 6. Листинги пайплайна

### 6.1. microservices-demo — .gitlab-ci.yml

```yaml
include:
  - project: 'microservices/cicd'
    file: '/.gitlab-ci.yml'
    ref: main
```

### 6.2. cicd — .gitlab-ci.yml

```yaml
stages:
  - lint
  - build
  - deploy

include:
  - local: 'templates/rules.yml'
  - local: 'templates/lint.yml'
  - local: 'templates/build.yml'
  - local: 'templates/deploy.yml'
```

### 6.3. templates/rules.yml

```yaml
.default_rules:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

.mr_rules:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

.protected_rules:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: on_success

```

### 6.4. templates/lint.yml

Статическое тестирование охватывает все языки приложения: Go, Python, Java, C#, а также Dockerfile и security-сканирование. Все джобы имеют `allow_failure: true`.

```yaml
lint-go:
  stage: lint
  image: golangci/golangci-lint:v2.11.3
  variables:
    GOLANGCI_LINT_CACHE: "$CI_PROJECT_DIR/.cache/golangci-lint"
  script:
    - |
      find . -name go.mod -execdir \
        sh -c 'echo "Linting in $(pwd)" && golangci-lint run --timeout 5m ./...' \;
  cache:
    key: golangci-lint-cache
    paths:
      - .cache/golangci-lint
  allow_failure: true
  extends: .default_rules
  tags:
    - docker

lint-python:
  stage: lint
  image: python:3.11-slim
  script:
    - pip install flake8 black --quiet
    - flake8 ./src --exclude=*_pb2.py,*_pb2_grpc.py --max-line-length=120 || true
    - black --check ./src || true
  allow_failure: true
  extends: .default_rules
  tags:
    - docker

lint-dockerfiles:
  stage: lint
  image: hadolint/hadolint:latest-alpine
  script:
    - find . -name "Dockerfile" | xargs -I{} hadolint {} || true
  allow_failure: true
  extends: .default_rules
  tags:
    - docker

lint-go-security:
  stage: lint
  image: golang:1.26-alpine
  script:
    - go install github.com/securego/gosec/v2/cmd/gosec@latest
    - go install golang.org/x/vuln/cmd/govulncheck@latest
    - |
      find . -name go.mod -exec sh -c '
        echo "Security scan in $(pwd)"
        gosec ./... || true
        govulncheck ./... || true
      ' \;
  allow_failure: true
  extends: .default_rules
  tags:
    - docker

lint-python-security:
  stage: lint
  image: python:3.11-slim
  script:
    - pip install bandit pip-audit --quiet
    - bandit -r ./src || true
    - pip-audit || true
  allow_failure: true
  extends: .default_rules
  tags:
    - docker

lint-java:
  stage: lint
  image: maven:3.9.1-eclipse-temurin-20
  script:
    - find ./src -name "pom.xml" -execdir sh -c \
        'echo "Linting Java in $(pwd)" && mvn checkstyle:check spotbugs:check || true' \;
  allow_failure: true
  extends: .default_rules
  tags:
    - docker

lint-csharp:
  stage: lint
  image: mcr.microsoft.com/dotnet/sdk:10.0
  script:
    - find ./src -name "*.csproj" -execdir sh -c \
        'echo "Linting C# in $(pwd)" && dotnet restore && dotnet format --verify-no-changes || true' \;
  allow_failure: true
  extends: .default_rules
  tags:
    - docker
```

### 6.5. templates/build.yml

Все 11 сервисов собираются параллельно через `parallel matrix`. Образы публикуются в YC Registry с тегом коммита и `latest`.

```yaml
.build_template:
  stage: build
  image: docker:24.0.5
  services:
    - name: docker:24.0.5-dind
      alias: docker
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
    DOCKER_DRIVER: overlay2
    REGISTRY: "cr.yandex/${YC_REGISTRY_ID}"
    DOCKERFILE: "src/${SERVICE_NAME}/Dockerfile"
    CONTEXT: "src/${SERVICE_NAME}/"
  before_script:
    - docker info
    - echo "$YC_SA_JSON_KEY" > /tmp/sa-key.json
    - cat /tmp/sa-key.json | docker login cr.yandex -u json_key --password-stdin
  after_script:
    - rm -f /tmp/sa-key.json
  script:
    - echo "Building $SERVICE_NAME"
    - |
      docker build \
        -f "${DOCKERFILE}" \
        -t "${REGISTRY}/${SERVICE_NAME}:${CI_COMMIT_SHORT_SHA}" \
        -t "${REGISTRY}/${SERVICE_NAME}:latest" \
        "${CONTEXT}"
      docker push "${REGISTRY}/${SERVICE_NAME}:${CI_COMMIT_SHORT_SHA}"
      docker push "${REGISTRY}/${SERVICE_NAME}:latest"
  extends: .default_rules
  tags:
    - docker

build:
  extends: .build_template
  parallel:
    matrix:
      - SERVICE_NAME: adservice
      - SERVICE_NAME: checkoutservice
      - SERVICE_NAME: currencyservice
      - SERVICE_NAME: emailservice
      - SERVICE_NAME: frontend
      - SERVICE_NAME: paymentservice
      - SERVICE_NAME: productcatalogservice
      - SERVICE_NAME: recommendationservice
      - SERVICE_NAME: loadgenerator
      - SERVICE_NAME: shippingservice
      - SERVICE_NAME: cartservice
        DOCKERFILE: "src/cartservice/src/Dockerfile"
        CONTEXT: "src/cartservice/src/"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### 6.6. templates/deploy.yml

Два метода деплоя на выбор, оба запускаются вручную (`when: manual`). Важно не использовать их одновременно на одном кластере — kubectl и helm хранят состояние по-разному и могут конфликтовать.

```yaml
.kube_setup:
  before_script:
    - echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
  after_script:
    - rm -f /tmp/kubeconfig

deploy:kubectl:
  stage: deploy
  image: alpine/k8s:1.32.13
  extends:
    - .protected_rules
    - .kube_setup
  needs:
    - job: build
      artifacts: false
  when: manual
  script:
    - |
      echo "=== Namespace ==="
      kubectl create namespace microservices --dry-run=client -o yaml \
        | kubectl apply -f -

      echo "=== Deploy ==="
      for manifest in kubernetes-manifests/*.yaml; do
        [[ "$manifest" == *"kustomization"* ]] && continue
        sed -E \
          "s|image: ([a-z]+)$|image: cr.yandex/${YC_REGISTRY_ID}/\1:${CI_COMMIT_SHORT_SHA}|g" \
          "$manifest" | kubectl apply -f - -n microservices
      done

      echo "=== Rollout ==="
      kubectl get deploy -n microservices -o name \
        | xargs -I{} kubectl rollout status {} \
            -n microservices --timeout=300s

      echo "=== Поды ==="
      kubectl get pods -n microservices -o wide

      echo "=== Внешний IP ==="
      kubectl get svc frontend-external -n microservices
  environment:
    name: production/kubectl
  tags:
    - docker

deploy:helm:
  stage: deploy
  image: alpine/k8s:1.32.13
  extends:
    - .protected_rules
    - .kube_setup
  needs:
    - job: build
      artifacts: false
  when: manual
  script:
    - |
      echo "=== Helm deploy ==="
      helm upgrade --install online-boutique ./helm-chart \
        --namespace microservices \
        --create-namespace \
        --set images.repository="cr.yandex/${YC_REGISTRY_ID}" \
        --set images.tag="${CI_COMMIT_SHORT_SHA}" \
        --set cartDatabase.inClusterRedis.publicRepository=true \
        --timeout 5m \
        --wait \
        --atomic

      echo "=== Rollout ==="
      kubectl get deploy -n microservices -o name \
        | xargs -I{} kubectl rollout status {} \
            -n microservices --timeout=300s

      echo "=== Поды ==="
      kubectl get pods -n microservices -o wide

      echo "=== Внешний IP ==="
      kubectl get svc frontend-external -n microservices
  environment:
    name: production/helm
  tags:
    - docker
```

---

## 7. Процесс доставки изменений

| Шаг | Действие |
|---|---|
| 1. Разработчик | Создаёт ветку, вносит изменения, открывает Merge Request в `main` |
| 2. Ревью | MR требует апрув от двух участников, прямой push в `main` запрещён |
| 3. Слияние MR | После апрува MR сливается в `main`, GitLab автоматически запускает пайплайн |
| 4. Стадия lint | Параллельно запускаются 7 линтеров: Go, Python, Java, C#, Dockerfile, security-сканы |
| 5. Стадия build | 11 сервисов собираются параллельно, образы пушатся в YC Registry с тегом коммита |
| 6. Стадия deploy | DevOps вручную выбирает метод деплоя: `kubectl` или `helm` |
| 7. Верификация | Пайплайн проверяет rollout status всех деплойментов, выводит список подов и внешний IP |

---

## 8. Технические решения

### 8.1. Разделение репозиториев

Пайплайн вынесен в отдельный репозиторий `cicd`, недоступный разработчикам. Это исключает возможность разработчика изменить пайплайн или получить доступ к секретам через CI. Переменные с ключами и kubeconfig хранятся на уровне группы.

### 8.2. Параллельная сборка

Использование `parallel matrix` позволяет собирать все 11 сервисов одновременно, что значительно сокращает время пайплайна по сравнению с последовательной сборкой.

### 8.3. Два метода деплоя

- **kubectl** — прозрачен, не требует дополнительных инструментов, подходит для отладки и первичного развёртывания
- **helm** — обеспечивает историю релизов и возможность отката командой `helm rollback`

### 8.4. Статический kubeconfig

Для доступа к кластеру из CI используется `ServiceAccount` с бессрочным токеном вместо exec-провайдера `yc` CLI. Это исключает зависимость от установленного `yc` на раннере и проблемы с истечением токена.

### 8.5. Подстановка образов без изменения репозитория

Оригинальные манифесты Google из `kubernetes-manifests/` не изменяются. В процессе деплоя `sed` заменяет короткие имена образов (`image: frontend`) на полные пути в YC Registry с тегом текущего коммита. Файл `kustomization.yaml` пропускается — он не является k8s-манифестом.

При деплое через Helm параметры `images.repository` и `images.tag` передаются через `--set` — файлы в репозитории не изменяются.