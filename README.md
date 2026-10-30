# Дипломный проект в Yandex Cloud

Дипломная работа по профессии DevOps-инженер. Облачная инфраструктура, Kubernetes-кластер, мониторинг,
тестовое приложение и CI/CD, развёрнутые в Yandex Cloud.

## Репозитории

| Репозиторий | Назначение |
|---|---|
| [diploma-infra](https://github.com/JefiJFire/diploma-infra) | Terraform (bootstrap и инфраструктура), Ansible и Kubespray, pipeline Terraform |
| [diploma-app](https://github.com/JefiJFire/diploma-app) | Тестовое приложение (nginx), Dockerfile, pipeline сборки и деплоя |
| [diploma-k8s](https://github.com/JefiJFire/diploma-k8s) | Манифесты и Helm-значения: ingress, kube-prometheus-stack, приложение, RBAC для CI |

## Результат

| Что | Адрес |
|---|---|
| Тестовое приложение | http://178.154.210.22/ |
| Grafana | http://178.154.210.22/grafana |
| Образы приложения | `cr.yandex/crppb6ltiiiam8vg56g8/diploma-app` (Yandex Container Registry) |

Приложение и Grafana работают на одном адресе и порту 80, их разделяют пути `/` и `/grafana`.
Адрес указан на момент сдачи: после `terraform destroy` и `apply` он меняется.

## Архитектура

```
GitHub Actions ──► Yandex Container Registry
      │                      │ pull
      │ terraform / kubectl  ▼
      ▼             ┌───────────────────── VPC (подсети в зонах a и d) ─────────────────────┐
 Yandex Cloud API   │ master (a)           monitoring (a, прерываемая)     worker1 (d, прерываемая) │
                    │ control plane, etcd  Prometheus, Grafana,            ingress-nginx :80        │
                    │ статический IP       Alertmanager                    приложение (2 реплики)   │
                    └───────────────────────────────────────────────────────────────────────┘
```

| Узел | Зона | Роль | vCPU / ОЗУ / диск | Прерываемая |
|---|---|---|---|---|
| master | ru-central1-a | control plane, etcd | 2 (20%) / 4 ГБ / 30 ГБ HDD | нет |
| monitoring | ru-central1-a | рабочий узел с меткой `role=monitoring` | 2 (20%) / 6 ГБ / 30 ГБ HDD | да |
| worker1 | ru-central1-d | рабочий узел, ingress (hostPort 80) | 2 (20%) / 4 ГБ / 30 ГБ HDD | да |

## Что и как сделано

### 1. Облачная инфраструктура (Terraform)

Репозиторий `diploma-infra/terraform`. Состоит из двух частей.

- **`bootstrap`** создаётся один раз с локальным state. Сервисный аккаунт `terraform-sa` с минимально необходимыми ролями
  (compute.editor, vpc.admin, storage.admin), бакет Object Storage для state с версионированием, Container Registry и
  отдельный сервисный аккаунт CI, который может только отправлять образы (`images.pusher`).
- **`infra`** хранит state в S3-бакете. VPC, подсети в зонах `ru-central1-a` и `ru-central1-d`, группа безопасности, ВМ
  (две прерываемые) и статические адреса. По списку узлов в `compute.tf` Terraform сам генерирует inventory для Kubespray
  (`ansible/inventory/hosts.yaml`), переносить IP вручную не нужно.

`terraform destroy` и `terraform apply` в каталоге `infra` работают без ручных действий.

### 2. Kubernetes-кластер (Kubespray)

Репозиторий `diploma-infra/ansible`. Версия Kubespray v2.32.0 (Kubernetes 1.36, containerd, Calico). Кластер из трёх
узлов (1 control plane, 2 worker). Команда `make up` выполняет всё: подготовку окружения, генерацию inventory, установку
кластера и слияние kubeconfig с текущим.

### 3. Тестовое приложение

Репозиторий `diploma-app`: статическая страница на nginx, `Dockerfile`, проверка `/healthz`, версия образа подставляется при
сборке. Образ публикуется в Yandex Container Registry.

### 4. Мониторинг и доступ по HTTP

Репозиторий `diploma-k8s`:
- **ingress-nginx** на `worker1` через hostPort 80.
- **kube-prometheus-stack** (Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, правила и дашборды)
  размещён на узле `monitoring` через nodeSelector. Пароль Grafana передаётся в секрет Kubernetes и в репозитории не хранится.
- Приложение (2 реплики, Service, Ingress) разворачивается через kustomize.

### 5. CI/CD (GitHub Actions)

**diploma-infra** (`.github/workflows/terraform.yml`):
- pull request: `fmt`, `validate`, `plan`; результат плана публикуется комментарием в PR;
- коммит в `main`: `apply` по сохранённому плану.

**diploma-app** (`.github/workflows/ci-cd.yml`):
- любой коммит: сборка образа, проверка (`/healthz` и версия), push в реестр с тегом из короткого SHA;
- git-тег `v*`: сборка, push образа с этим тегом и деплой в кластер (`kubectl set image`, ожидание `rollout status`).

Для деплоя используется ServiceAccount `ci-deployer` с ограниченными правами: изменять только Deployment `diploma-app`
в namespace `diploma`. Ключ реестра имеет только право отправки образов. Секреты хранятся в GitHub Secrets.

## Как воспроизвести

```bash
# 1. Один раз: сервисные аккаунты, бакет state, реестр
cd diploma-infra/terraform/bootstrap && terraform init && terraform apply

# 2. Инфраструктура
cd ../infra
cp backend.hcl.example backend.hcl && cp terraform.tfvars.example terraform.tfvars   # заполнить значения
terraform init -backend-config=backend.hcl && terraform apply

# 3. Кластер
cd ../../ansible && make up

# 4. Образ приложения
cd ../../diploma-app && make push REGISTRY_ID=<registry_id>

# 5. Ingress, мониторинг, приложение
cd ../diploma-k8s && export GRAFANA_PASSWORD='...' && make all

# 6. Доступ для CI
./ci/make-kubeconfig.sh <IP мастера>      # результат в секрет KUBE_CONFIG
```
Секреты репозиториев перечислены в README каждого репозитория.

## Скриншоты

Все скриншоты лежат в каталоге [`docs/screenshots`](docs/screenshots).

| № | Что показано | Файл |
|---|---|---|
| 1 | Сервисные аккаунты и ролевая модель в консоли Yandex Cloud | `01-service-accounts.png` |
| 2 | Бакет с state и Container Registry | `02-bucket-registry.png` |
| 3 | `terraform apply` в `infra` | `03-terraform-apply.png` |
| 4 | VPC, подсети в разных зонах, список ВМ (прерываемые) | `04-vpc-vms.png` |
| 5 | `kubectl get nodes` и `kubectl get pods -A` | `05-kubectl.png` |
| 6 | Образ приложения в реестре | `06-registry-image.png` |
| 7 | Страница приложения по HTTP (порт 80) | `07-app.png` |
| 8 | Grafana: дашборды кластера и узлов | `08-grafana.png` |
| 9 | Prometheus/Alertmanager (или `kubectl get pods -n monitoring`) | `09-monitoring-pods.png` |
| 10 | PR в `diploma-infra` с комментарием `terraform plan` | `10-ci-infra-pr-plan.png` |
| 11 | Прогон `terraform apply` на `main` | `11-ci-infra-apply.png` |
| 12 | Прогон `ci-cd` на коммит: сборка и push образа | `12-ci-app-build.png` |
| 13 | Прогон `ci-cd` на тег `v*` с деплоем | `13-ci-app-deploy.png` |
| 14 | Страница приложения после деплоя новой версии | `14-app-new-version.png` |

## Особенности

- **Прерываемые ВМ.** Yandex Cloud останавливает их через 24 часа:
  `yc compute instance start diploma-monitoring diploma-worker1`. Данные Prometheus хранятся на `emptyDir` и при остановке
  узла теряются.
- **ingress-nginx** с марта 2026 года не получает исправлений безопасности (проект закрыт сообществом). Для стенда
  оставлен как наиболее распространённый вариант; замена, например, на Traefik потребует изменить только values и
  `ingressClassName`.
- **Адреса после пересоздания.** После `destroy` и `apply` публичные IP меняются, поэтому в манифестах и документации
  адреса не зашиты. Секрет `KUBE_CONFIG` после пересоздания нужно обновить скриптом `ci/make-kubeconfig.sh`.
- **GitLab.** Pipeline написаны под GitHub Actions; шаги (init/plan/apply, build/push, `kubectl set image`) переносятся в
  `.gitlab-ci.yml` без изменения логики.
