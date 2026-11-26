# Concept: локальний Kubernetes для PoC «AsciiArtify»

## 1. Вступ

AsciiArtify потрібне локальне середовище Kubernetes для розробки та PoC.
Розглянуто три інструменти, які запускають кластер у контейнерах на одній машині:

- **minikube** — офіційний інструмент спільноти Kubernetes для локального кластера. Підтримує різні драйвери (Docker, Podman, VirtualBox, Hyper-V, KVM тощо) та має систему аддонів.
- **kind** (Kubernetes IN Docker) — запускає вузли Kubernetes як Docker-контейнери. Створений для тестування самого Kubernetes і для CI.
- **k3d** — легка утиліта командного рядка для запуску k3s (мінімального дистрибутива Kubernetes від Rancher) всередині контейнерів Docker.


## 2. Характеристики

| Характеристика | minikube | kind | k3d |
|---|---|---|---|
| ОС | Linux, macOS, Windows | Linux, macOS, Windows | Linux, macOS, Windows |
| Архітектури | amd64, arm64 | amd64, arm64 | amd64, arm64 |
| Середовище запуску | Docker, Podman, VM (VirtualBox, Hyper-V, KVM…) | Docker, Podman (експериментально) | Docker, Podman (через сокет, з обмеженнями) |
| Дистрибутив K8s | upstream (kubeadm) | upstream (kubeadm) | k3s (один бінарник, SQLite замість etcd) |
| Версія K8s (тест) | v1.37.0 | v1.37.0 | v1.35.5+k3s1 |
| Автоматизація | CLI-прапорці, профілі | YAML-конфіг кластера | YAML-конфіг кластера, CLI |
| Кілька вузлів | `--nodes N` | через YAML-конфіг | `--servers N --agents N` |
| Моніторинг | addon `metrics-server` | немає, ставиться вручну | metrics-server вбудований |
| Dashboard / UI | addon `dashboard` | немає | немає (ставиться окремо) |
| Ingress / LoadBalancer | addons `ingress`, `minikube tunnel` | немає | Traefik + ServiceLB вбудовані |
| Локальні образи | `minikube image load` | `kind load docker-image` | `k3d image import`, `k3d registry create` |

### Результати практичних тестів (WSL2, Docker 29.9)

| Метрика | minikube | kind | k3d |
|---|---|---|---|
| Розмір образів (завантаження) | 532 MB | 388 MB | **125 MB** |
| Розмір на диску | 1.9 GB | 1.34 GB | **478 MB** |
| Холодний старт (з завантаженням) | 3:39 | 1:36 | **0:39** |
| Старт, 1 вузол | 44 с | 21 с* | **21 с** |
| Старт, 2 вузли (до Ready) | 1:27 | 52 с | **24 с** |
| RAM control-plane | **~516 MiB** | ~545 MiB | ~720 MiB** |
| CPU у простої | 483m | ~15–20 % | **141m** |

\* без очікування стану Ready  
\** разом із вбудованими Traefik, metrics-server та ServiceLB

## 3. Переваги та недоліки

### minikube
**Переваги:**
- найбільше можливостей «з коробки»: аддони (dashboard, metrics-server, ingress, registry);
- найближчий до upstream Kubernetes;
- найкраща документація та велика спільнота;
- підтримка VM-драйверів, якщо Docker недоступний.

**Недоліки:**
- найповільніший старт і найбільші образи;
- кілька вузлів створюються послідовно та повільно;
- розрахований на один кластер одночасно (профілі допомагають, але важко).

### kind
**Переваги:**
- швидкий старт, простий інструмент;
- upstream Kubernetes, вибір версії через образ вузла;
- декларативний YAML-конфіг, зручно для CI;
- використовується для тестування самого Kubernetes, тож стабільний.

**Недоліки:**
- немає моніторингу, ingress та LoadBalancer «з коробки»;
- `kind create cluster` без `--wait` повертає керування до готовності вузлів;
- для доступу ззовні потрібні додаткові налаштування (extraPortMappings, MetalLB).

### k3d
**Переваги:**
- найшвидший старт і найменші образи;
- кілька вузлів створюються паралельно (2 вузли за ~24 с);
- вбудовані Traefik, ServiceLB, metrics-server, local-path storage;
- проброс портів через вбудований LoadBalancer (`-p "8080:80@loadbalancer"`);
- вбудований локальний реєстр образів.

**Недоліки:**
- k3s — не повний upstream: інші компоненти за замовчуванням (SQLite, Traefik), версія K8s відстає;
- більше споживання RAM у простої через вбудовані компоненти;
- менша спільнота, ніж у minikube та kind.

## 4. Ризики ліцензування Docker та Podman

- **Docker Desktop** безкоштовний лише для компаній до 250 співробітників і з доходом до $10 млн на рік. Далі потрібна платна підписка. Зараз AsciiArtify під умови підпадає, але при зростанні це стане ризиком.
- **Docker Engine** (Linux) має ліцензію Apache 2.0 і безкоштовний завжди.
- **Podman** — open source (Apache 2.0) альтернатива від Red Hat: без демона, з rootless-режимом і CLI, сумісним із Docker. Є безкоштовний Podman Desktop. Підтримка в minikube, kind та k3d поки експериментальна.

## 5. Демонстрація

Розгортання «Hello World» на k3d: кластер з 2 вузлів, 2 репліки, доступ через вбудований Traefik ingress, балансування між подами, вбудований моніторинг.

![k3d demo](demo.gif)

Команди демо:
```bash
# 1. кластер из двух нод, порт 8080 проброшен на встроенный LoadBalancer
k3d cluster create demo --agents 1 -p "8080:80@loadbalancer"
# 2. приложение Hello World на две реплики
kubectl create deployment hello --image=us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0 --replicas=2
kubectl expose deployment hello --port=80 --target-port=8080
kubectl create ingress hello --rule="/*=hello:80"
# 3. проверка через Traefik ingress: Hostname меняется, значит идёт балансировка
curl localhost:8080
# 4. встроенный мониторинг
kubectl top nodes
# 5. удаление
k3d cluster delete demo
```

## 6. Висновки

| Інструмент | Коли використовувати |
|---|---|
| **k3d** ✅ | **Рекомендовано для PoC**: найшвидший, легкий, кілька вузлів і ingress/LB «з коробки» |
| kind | CI-пайплайни та тести на конкретній upstream-версії Kubernetes |
| minikube | навчання, вивчення Kubernetes, коли потрібні dashboard та аддони або VM-драйвер |


Для локального Kubernetes-кластера в PoC AsciiArtify рекомендовано **k3d**. Він найшвидший і найлегший
за розміром образів, підтримує кілька вузлів і має ingress, LoadBalancer та моніторинг «з коробки».
Розробники без DevOps-досвіду отримують робоче середовище, близьке до реального, однією командою.


Як контейнерне середовище, щоб уникнути ліцензійних ризиків Docker Desktop: Docker Engine
(Linux/WSL2) або Rancher Desktop, а Podman — як резервна альтернатива.


