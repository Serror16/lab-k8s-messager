# Лабораторная работа: Запуск микросервисного приложения в Kubernetes

## Выполнил
Белкин Сергей Викторович

М8О-106БВ-25


### Используемые технологии:
- **Kubernetes** — оркестрация контейнеров.
- **Kustomize** — шаблонизация и управление конфигурациями для разных окружений (`base`, `dev`, `prod`).
- **Argo CD** — реализация GitOps-подхода для автоматического деплоя из репозитория.
- **S3 CSI** — монтирование S3-бакета как локальной директории для хранения файлов `message-service`.

## Цель работы
Подготовить инфраструктурную конфигурацию для запуска приложения в Kubernetes и продемонстрировать:
- Развертывание всех микросервисов приложения.
- Применение миграций базы данных через Kubernetes Jobs.
- Подключение файлового хранилища через S3 CSI (MinIO).
- Настройку правил планировщика `nodeAffinity` для распределения нагрузки.
- Поддержку изолированных окружений `dev` и `prod` через Kustomize.
- Автоматическую GitOps-синхронизацию через Argo CD.

## Состав приложения
- `frontend` — SPA интерфейс пользователя.
- `bff` — Backend For Frontend, API-шлюз.
- `user-service` — сервис управления пользователями.
- `message-service` — сервис отправки сообщений и загрузки файлов.
- `postgres` — база данных.
- `minio` — локальное S3-совместимое хранилище.
- `migrate-users` / `migrate-messages` — Jobs задачи для SQL-миграций.

## Структура репозитория
```text
lab-k8s-messager/
├── argocd/
│   └── application_prod.yaml # Манифест Argo CD для деплоя prod-окружения
├── k8s/
│   ├── base/                 # Базовые манифесты (Deployments, Services, ConfigMaps, Jobs)
│   └── overlays/
│       ├── dev/              # Конфигурация для тестирования (1 реплика)
│       └── prod/             # Конфигурация для продакшена (увеличенное число реплик)
└── README.md
```

## Используемые образы
Для работы используются готовые контейнерные образы из Docker Hub:
- `mablinov2704/frontend:latest`
- `mablinov2704/bff:latest`
- `mablinov2704/user-service:latest`
- `mablinov2704/message-service:latest`
- `postgres:16-alpine` (база данных)
- `ghcr.io/kukymbr/goose-docker:latest` (утилита для миграций)
- `minio/minio:latest` (S3 хранилище)

## Архитектурные особенности реализации

### 1. S3 (MinIO + CSI)
Для сервиса `message-service` реализована прозрачная загрузка файлов в S3. Приложение пишет файлы в локальную директорию `/uploads`, которая физически является вмонтированным томом через провайдер CSI-S3. Все файлы фактически сохраняются в бакет MinIO.

### 2. Умный планировщик (Node Affinity)
Настроено распределение подов по узлам кластера в зависимости от их роли:
- **Системные** (`postgres`, `minio`) размещаются строго на узлах с меткой `workload=system`.
- **Прикладные** (`frontend`, `bff`, `user-service`) размещаются строго на узлах с меткой `workload=app`.
- **Специфичные** (`message-service`) требуют `workload=app`, но при планировании отдают приоритет узлам с меткой `disk=fast` (используя `preferredDuringSchedulingIgnoredDuringExecution`).


## Инструкция по запуску

### 1. Подготовка кластера Minikube
Необходимо запустить Minikube с несколькими узлами и назначить им нужные метки для работы `nodeAffinity`:

```bash
minikube -p messager start --nodes 3
kubectl label nodes messager workload=system
kubectl label nodes messager-m02 workload=app
kubectl label nodes messager-m03 workload=app disk=fast
```

### 2. Настройка Argo CD
Устанавливаем Argo CD в кластер:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
```

Пробрасываем порт для доступа к веб-интерфейсу Argo CD:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Получаем пароль администратора (пользователь `admin`):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
```

### 3. Деплой приложения
Применяем конфигурацию Argo CD, которая автоматически подтянет манифесты из ветки `HEAD` репозитория `Serror16/lab-k8s-messager` и развернет окружение `prod`:

```bash
kubectl apply -f argocd/application_prod.yaml
```

### 4. Доступ к сервисам
После того как в интерфейсе Argo CD статус приложения сменится на `Synced` и `Healthy`, можно пробросить порты к необходимым сервисам.

**Для доступа к интерфейсу мессенджера:**

```bash
minikube -p messager service frontend -n messenger-prod --url
```

**Для проверки S3-хранилища MinIO:**
Проброс порта к консоли управления:

```bash
kubectl port-forward svc/minio 9001:9001 -n messenger-prod
```

*Авторизация: логин `minioadmin`, пароль `minioadmin123`.*