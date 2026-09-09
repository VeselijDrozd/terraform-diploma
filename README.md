# Диплом / отчёт: Terraform на homelab (Proxmox)

Адаптация учебного задания по Yandex Cloud под домашнюю лабораторию на **Proxmox**.  
Скриншоты приложите самостоятельно в соответствующие места (помечены как *«скрин»*).

## Связанные репозитории

| Путь / проект | Роль |
|---------------|------|
| [`homelab`](https://github.com/VeselijDrozd/homelab) / GitLab `homelab/homelab` | Инфраструктура: Terraform, Ansible, манифесты K8s, CI инфры |
| [`terraform-proxmox-vm-module`](https://github.com/VeselijDrozd/terraform-proxmox-vm-module) | Переиспользуемый модуль создания ВМ в Proxmox |
| `surfhouse` (GitLab) | Учебное web-приложение + CI/CD в кластер |
| этот каталог (`terraform-diploma`) | Текст отчёта |

---

## Соответствие: Yandex Cloud → Homelab

| Пункт задания (YC) | Реализация в homelab |
|--------------------|----------------------|
| VPC | Сеть гипервизора Proxmox, bridge `vmbr0` |
| Подсети | Адресное пространство LAN `10.10.10.0/24`, статические IP через cloud-init |
| VM + security groups (22, 80, 443) | ВМ через Terraform; аналог SG — UFW в примере user-data; снаружи HTTP через Ingress |
| Managed MySQL | MariaDB в Kubernetes (`website`), пароль из GitLab Variable |
| Container Registry | GitLab Container Registry (`:5050`) |
| user-data → Docker/Compose | Пример `cloud-init-docker-compose.yaml`; на k8s-нодах — containerd (Ansible) |
| Dockerfile → Registry | Multi-stage Dockerfile + push из GitLab CI |
| Приложение ↔ БД | Статика + Flask API (`/api/hits`) → MariaDB |
| LockBox | Аналог: GitLab CI Variable `DB_PASSWORD` → K8s Secret (не путать с Managed State) |

---

## Задание 1. Инфраструктура

### Что было изначально

- Terraform (`homelab/terraform`) создаёт ВМ в Proxmox через модуль `terraform-proxmox-vm-module`.
- cloud-init на ВМ: пользователь, SSH-ключ, IPv4.
- Сеть: bridge `vmbr0`, адреса вида `10.10.10.x` (inventory Ansible).
- Кластер Kubernetes (kubeadm) поднимается Ansible-плейбуком; runtime — **containerd**, не Docker.
- Приложение и мониторинг — манифесты/Helm в `homelab/k8s/`.
- State Terraform хранился **локально** (`backend "local"`).
- Отдельных «security groups» как в YC не было.

*Скрин: список ВМ в Proxmox / вывод `terraform state list`.*

### VPC и подсети (адаптация)

В Proxmox нет сущности VPC как в YC. Эквивалент:

- **VPC** — L2/L3 сегмент лаборатории (bridge `vmbr0` на узле PVE).
- **Подсеть** — `10.10.10.0/24`, адреса ВМ задаются в Terraform/`tfvars` и применяются cloud-init.

*Скрин: схема сети / bridge в Proxmox / фрагмент inventory.*

### Виртуальные машины

ВМ описываются в `vm_groups` / `vms`, разворачиваются модулем. После apply генерируется `ansible/inventory.ini`.

Группы в лаборатории (по inventory): `k8s_master`, `k8s_worker`, `gitlab_server`, `gitlab_runner`.

*Скрин: фрагмент `terraform.tfvars.example` или UI Proxmox с ВМ.*

### Группы безопасности (порты 22, 80, 443)

**Изначально:** доступ по SSH ключом; веб-трафик к приложению — через Ingress NGINX (хост `surfhouse.local`), снаружи при необходимости NodePort **30080**.

**Добавлено для соответствия заданию:** пример user-data  
`homelab/terraform/examples/cloud-init-docker-compose.yaml` — правила UFW:

- 22/tcp (SSH)
- 80/tcp (HTTP)
- 443/tcp (HTTPS)

На уже работающих k8s-нодах этот фрагмент **намеренно не накатывался** (чтобы не ломать containerd/кластер). Для отчёта — демонстрационный аналог security group; для отдельной demo-ВМ его можно применить как snippets/user-data.

*Скрин: содержимое YAML с `ufw allow` / статус `ufw status` на demo-ВМ (если поднимали).*

### База данных MySQL

**Было:** отдельной СУБД не было (мониторинг — Prometheus TSDB / SQLite Grafana, не для приложения).

**Стало:** MariaDB 11 в namespace `website` (`surfhouse/k8s/mariadb.yml` и копия в `homelab/k8s/01-website/`).  
База/пользователь `surfhouse`, данные на PVC `mariadb-data`. Пароль — только из Secret `surfhouse-db` (создаётся из CI Variable `DB_PASSWORD`).

*Скрин: `kubectl -n website get deploy,svc,pvc | grep -E 'mariadb|NAME'`.*

### Container Registry

**Изначально уже было:** GitLab Container Registry  
`gitlab.homelab.local:5050/homelab/surfhouse:…`  
Сборка и push — job `build-image` в `surfhouse/.gitlab-ci.yml`.  
В кластере — `imagePullSecrets: gitlab-registry`.

Аналог **Yandex Container Registry**.

*Скрин: Packages/Registry в GitLab с тегами образа.*

### Remote state (важное усиление к заданию 1)

**Было:** `backend "local"`.

**Стало:**

- GitLab Managed Terraform State (`backend "http"`).
- Проект GitLab: `homelab/homelab` (Project ID `3`), имя state: `homelab`.
- Локальные креды: `gitlab_http_backend_cred.sh` (gitignore).
- CI инфры: `.gitlab-ci.yml` — `fmt` → `validate` → `plan` → ручной `apply`.
- Модуль ВМ подключается по git-ref (`v1.1.2`), чтобы runner не зависел от sibling-каталога.
- Код залит и на GitLab, и на GitHub.

*Скрин: Operate → Terraform states в GitLab; зелёный pipeline plan.*

---

## Задание 2. user-data (cloud-init): Docker и Docker Compose

### Что было изначально

В модуле ВМ cloud-init настраивал только учётную запись и сеть.  
Docker Compose на k8s-нодах не ставился: для Kubernetes используется **containerd** (роль Ansible `k8s_node`).

### Что добавлено

Файл-пример:

`homelab/terraform/examples/cloud-init-docker-compose.yaml`

Он:

1. Ставит `docker.io` и `docker-compose-v2`.
2. Включает сервис Docker.
3. Открывает порты 22/80/443 через UFW (см. задание 1).

Это прямой ответ на формулировку задания про user-data.  
В «боевом» контуре лаборатории эквивалент подготовки runtime — **Ansible**, а запуск приложения — **Kubernetes** (сильнее, чем один Compose на ВМ).

Дополнительно для локальной демонстрации Compose: `surfhouse/docker-compose.yml`.

*Скрин: фрагмент YAML; при желании — `docker compose version` на demo-ВМ.*

---

## Задание 3. Dockerfile и сохранение образа в Registry

### Что было изначально

Простой Dockerfile:

```dockerfile
FROM nginx:latest
COPY . /usr/share/nginx/html
...
```

Образ уже пушился в GitLab Registry пайплайном `surfhouse`.

### Что изменено

1. **Multi-stage Dockerfile** в `surfhouse/`:
   - stage `build` (alpine) — подготовка статики;
   - stage runtime — `nginx:1.27-alpine`, копирование артефакта.
2. **`docker-compose.yml`** — локальный запуск `docker compose up --build`.
3. Пайплайн без смены логики: `docker build` → `docker push` в Registry.

*Скрин: Dockerfile; образ в Registry; `docker compose up` или поды в namespace `website`.*

---

## Задание 4. Приложение ↔ БД

### Что было изначально

Только статический nginx; к БД приложение не обращалось.

### Что сделано

1. Мини-API на Flask (`surfhouse/api/`): `GET /api/hits` увеличивает счётчик в таблице `hits` и возвращает JSON.
2. Лендинг вызывает `/api/hits` (JS в `index.html`) и показывает «visitors: N».
3. Ingress: `/api` → `surfhouse-api`, `/` → статика.
4. Локально: `docker compose up --build` (сайт + API + MariaDB).

*Скрин: ответ `curl …/api/hits`; блок visitors на сайте; поды `mariadb` и `surfhouse-api`.*

---

## Задание 5* (опционально). Секреты / аналог LockBox

**Yandex LockBox** — отдельный secrets manager. Полный Vault/LockBox не ставился.

### Прямой аналог для пароля БД

```text
GitLab CI Variable DB_PASSWORD (Masked)
        → deploy job
        → kubectl create secret generic surfhouse-db
        → MariaDB и API читают password через secretKeyRef
```

Пароль **не хранится в git**. Managed Terraform State — это remote backend для state, **не** замена LockBox; в отчёте он описан отдельно (задание про инфраструктуру).

Дополнительно: gitignore для `tfvars` / cred-скриптов; CI File variables для kubeconfig и Terraform.

*Скрин: Variable `DB_PASSWORD` (имя, masked); фрагмент `.gitlab-ci.yml` с `create secret`; `kubectl describe secret surfhouse-db` без values.*

---

## Docker Compose vs Kubernetes (пояснение)

Задание ориентировано на Compose. В проекте:

| Учебный минимум | Фактический прод лаборатории |
|-----------------|------------------------------|
| `docker-compose.yml` + cloud-init пример | Deployment `surfhouse`, **3 реплики**, Service, Ingress, GitLab CI deploy |

Compose оставлен как упрощённый стенд; оркестрация — Kubernetes.

---

## Итого: что изменили под задание (чеклист)

- [x] Remote state → GitLab Managed Terraform State + CI plan/manual apply  
- [x] Пример security group (UFW 22/80/443) в user-data  
- [x] Пример cloud-init с Docker + Compose  
- [x] Multi-stage Dockerfile + compose-файл приложения  
- [x] Container Registry — уже был, описан в отчёте  
- [x] MariaDB + API + привязка лендинга (`/api/hits`)  
- [x] Аналог LockBox — GitLab Variable `DB_PASSWORD` → K8s Secret  

---

## Куда вставлять скриншоты

1. Proxmox / Terraform ВМ  
2. Сеть / inventory `10.10.10.x`  
3. GitLab → Terraform states  
4. GitLab → Registry (surfhouse + api)  
5. Pipeline `homelab` (plan) и/или `surfhouse` (build+deploy)  
6. Фрагменты Dockerfile / cloud-init YAML  
7. Поды website + `curl /api/hits` / visitors на сайте  
8. CI Variable `DB_PASSWORD` (без значения) + Secret в кластере  
