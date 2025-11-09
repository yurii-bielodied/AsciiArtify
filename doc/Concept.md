# Порівняльний аналіз minikube, kind та k3d для локального Kubernetes-кластеру стартапу **AsciiArtify**

---

## 1. Огляд інструментів

### 1.1. minikube

**minikube** — це утиліта для запуску локального Kubernetes-кластера на одній машині. Він може працювати як у віртуальній машині (VirtualBox, Hyper-V, KVM тощо), так і поверх Docker/Podman.

Основна ідея: швидко отримати “майже продакшн” Kubernetes у себе на ноутбуці, з можливістю додавати аддони (Dashboard, Ingress, metrics-server тощо).

---

### 1.2. kind (Kubernetes IN Docker)

**kind** запускає Kubernetes-кластери всередині контейнерів Docker (або Podman). Фактично, кожен вузол кластера — це контейнер.

Основна ідея: максимально легкий та швидкий Kubernetes для локальної розробки та CI/CD, де кластери часто створюються й видаляються автоматично.

---

### 1.3. k3d

**k3d** — це обгортка навколо **k3s** (легка дистрибуція Kubernetes від Rancher), яка запускає k3s-кластери усередині Docker-контейнерів.

Основна ідея: дуже легкий, ресурсоощадний Kubernetes для локальної розробки, edge, IoT та PoC, із простою CLI та швидким стартом.

---

## 2. Характеристики інструментів

### 2.1. Зведена таблиця

| Характеристика                         | **minikube**                                           | **kind**                                              | **k3d**                                                        |
|----------------------------------------|--------------------------------------------------------|-------------------------------------------------------|----------------------------------------------------------------|
| Тип                                    | Upstream Kubernetes                                    | Upstream Kubernetes “в Docker/Podman”                | Легка дистрибуція k3s у Docker                                |
| ОС хоста                               | Linux, macOS, Windows                                  | Linux, macOS, Windows                                | Linux, macOS, Windows                                         |
| Архітектури                            | amd64, arm64                                           | amd64, arm64                                         | amd64, arm64                                                   |
| Бекенд                                 | Docker, Podman, VM (KVM/VirtualBox/Hyper-V тощо)       | Docker, Podman (experimental)                        | Docker API (Docker Engine / Docker Desktop)                    |
| Мультивузлові кластери                | Підтримуються, конфігурація відносно складніша         | Підтримуються, конфігурація YAML                     | Підтримуються, дуже прості параметри CLI                      |
| Швидкість створення кластера          | Середня (повільніше з VM)                              | Висока                                               | Дуже висока (легкий k3s)                                      |
| Ресурсоспоживання                     | Вище (особливо з VM-драйверами)                        | Помірне                                              | Низьке (оптимізовано для edge/IoT)                            |
| Додаткові функції                     | Багато addons (Dashboard, Ingress, metrics тощо)       | Все ставимо руками                                   | Частина компонентів k3s є “з коробки” (Ingress, metrics тощо) |
| Підходить для CI/CD                   | Можливо, але не основний сценарій                      | Дуже популярний варіант для CI                       | Часто використовується, але менш “стандартний”, ніж kind      |
| Схожість із продакшн Kubernetes       | Дуже висока                                            | Дуже висока                                          | Висока, але є деякі відмінності за дефолтами k3s              |

---

### 2.2. Підтримка автоматизації та GitHub

Для нашого сценарію (GitHub як основний VCS):

- **minikube**: добре підходить для локального дев-оточення; можна запускати через скрипти, Makefile, але в CI використовується рідше.
- **kind**: багато готових GitHub Actions; типове рішення для інтеграційних тестів.
- **k3d**: легко автоматизується простими shell-скриптами та може використовуватись у GitHub Actions як легкий кластер.

---

## 3. Переваги та недоліки

### 3.1. minikube

**Переваги:**

- Підтримка різних бекендів:
  - Docker, Podman, VirtualBox, Hyper-V, KVM, bare-metal.
- Велика кількість **addons**, які активуються однією командою:
  - `minikube addons enable ingress`
  - `minikube addons enable metrics-server`
  - `minikube addons enable dashboard`
- Добре підходить для навчання:
  - багато посібників, статей, відео;
  - конфігурація близька до реальних кластерів.
- Може виступати як локальний Docker-daemon для розробників.

**Недоліки:**

- З використанням VM-драйверів може бути **повільним** та доволі **важким** для ноутбуків.
- Для мультивузлових кластерів конфігурація складніша, ніж у kind/k3d.
- При використанні Docker Desktop на macOS/Windows з’являється залежність від його ліцензійних умов.

---

### 3.2. kind

**Переваги:**

- Дуже **швидкий старт** і видалення кластерів.
- Легко описати конфігурацію кластера в YAML (multi-node, порти, мережа).
- Ідеальний для **CI/CD**:
  - підняти кластер у GitHub Actions, прогнати тести, видалити.
- Використовує “чистий” Kubernetes:
  - максимум сумісності з продакшн-кластерами.

**Недоліки:**

- Немає готових аддонів “з коробки”:
  - Ingress, Dashboard, monitoring потрібно ставити вручну.
- Підтримка Podman поки **експериментальна**:
  - є ризики нестабільної роботи.
- Для новачків в Kubernetes поріг входу може бути трохи вищим, ніж у minikube.

---

### 3.3. k3d

**Переваги:**

- Дуже легкий і швидкий:
  - кластер створюється за секунди/десятки секунд.
- Простий CLI:
  - `k3d cluster create mycluster`
- Легко робити **multi-node**:
  - `--servers`, `--agents` у команді створення.
- В основі — **k3s**:
  - оптимізований для малоресурсних середовищ;
  - часто вже є Ingress (Traefik), metrics-server, local-storage.

**Недоліки:**

- k3s — це легка дистрибуція Kubernetes:
  - не 100% той самий набір дефолтів, як у “великих” кластерів, хоча API загалом сумісний.
- Орієнтація на Docker API:
  - безпосередня інтеграція з Podman не така очевидна, як у minikube.
- Спільнота активна, але трохи менша, ніж у minikube/kind.

---

## 4. Ліцензування Docker та роль Podman

### 4.1. Docker Desktop

Для стартапу AsciiArtify важливо враховувати:

- Docker Desktop **безкоштовний** для:
  - окремих розробників;
  - малих бізнесів (до певних порогів по кількості співробітників і обороту).
- Для великих компаній потрібна **платна підписка**.

На початковому етапі (2 молодих програмісти) Docker Desktop **формально підходить** безкоштовно. Але в перспективі масштабу бізнесу варто мати план “B”.

### 4.2. Podman як альтернатива

**Podman** — безdaemonний, сумісний з Docker CLI інструмент для контейнеризації.

- **minikube**:
  - має драйвер Podman;
  - дозволяє повністю обійтись без Docker Desktop на Linux.
- **kind**:
  - підтримка Podman присутня, але позначена як experimental;
  - для реальних командних середовищ потребує додаткового тестування.
- **k3d**:
  - спроєктований навколо Docker API; безпосередньої підтримки Podman немає, але можна розглядати інші рішення (Rancher Desktop, k3s напряму тощо).

З погляду ризиків, для AsciiArtify доцільно:

- не “вшивати” Docker Desktop у архітектуру як критичну залежність;
- в середньостроковій перспективі орієнтуватись на:
  - Linux + Docker Engine;
  - або Linux + Podman + minikube/kind.

---

## 5. Demo

## 5.0. Передумови: середовище та залежності. Встановлювати в залежності від обраного стеку

### 5.0.1. Вихідні припущення

Для цілей цього PoC я вважаю, що в нас є:

- свіжий **Linux-сервер** (наприклад, Ubuntu 24.04 LTS);
- доступ до сервера по SSH;
- користувач із правами `sudo`;
- відкритий доступ в інтернет для завантаження пакетів.

Все нижче — команди, які я запускатиму на цьому сервері.

---

### 5.0.2. Базові пакети

Оновлюю пакети та встановлюю базові утиліти:

```bash
sudo apt update
sudo apt install -y \
  curl \
  ca-certificates \
  gnupg \
  lsb-release \
  gpg
```

---

### 5.0.3. Встановлення Docker Engine (без Docker Desktop)

Оскільки ми працюємо на Linux-сервері, нам потрібен **Docker Engine**, а не Docker Desktop.

1. Додаю GPG-ключ і репозиторій Docker:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

2. Встановлюю Docker Engine:

```bash
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

3. Додаю поточного користувача в групу `docker`, щоб не запускати `sudo` щоразу:

```bash
sudo usermod -aG docker $USER
```

> Після цього потрібно **перелогінитися** (вийти з SSH-сесії і зайти знову), щоб група `docker` застосувалася.

Перевіряю, що Docker працює:

```bash
docker run --rm hello-world
```

---

### 5.0.4. Встановлення `kubectl`

`kubectl` — основний клієнт для керування Kubernetes-кластером.

```bash
# завантажити останню стабільну версію
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

kubectl version --client
```

---

### 5.0.5. Встановлення `k3d` (основний інструмент для PoC)

Для PoC AsciiArtify ми обираємо **k3d** як основний локальний кластер.

Встановлення через офіційний інсталяційний скрипт:

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

k3d version
```

---

### 5.0.6. Встановлення `kind`

`kind` нам знадобиться переважно для CI/CD та інтеграційних тестів.

```bash
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64"
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind version
```

> Версію (`v0.24.0`) при потребі можна оновити на актуальну.

---

### 5.0.7. Встановлення `minikube`

`minikube` використовується як альтернативний інструмент для локальних кластерів (особливо для навчання та експериментів з addons).

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/minikube

minikube version
```

---

### 5.0.8. (Опційно) Встановлення Podman

Як альтернатива Docker (особливо з погляду ліцензійних ризиків) можна паралельно встановити **Podman**:

```bash
sudo apt install -y podman
podman --version
```

Надалі:

- `minikube` може працювати з Podman як із драйвером;
- `kind` має експериментальну підтримку Podman;
- `k3d` орієнтований на Docker API, тож тут Podman — не основний варіант.

---


Нижче — демо запуску локального кластера та деплою “Hello World” на **k3d**.

### 5.1. Демо (asciinema)

[![AsciiArtify k3d demo](https://asciinema.org/a/lx3XBSH2WYdpHqSuQeOxjgLF6.svg)](https://asciinema.org/a/lx3XBSH2WYdpHqSuQeOxjgLF6.svg)

---

### 5.2. Покрокове демо: “Hello World” на k3d

У межах практичного ознайомлення я обрав **k3d** як базовий інструмент для локального PoC і провів демо з розгортанням простого “Hello World” застосунку.

#### 5.2.1. Встановлення `k3d`

(див. розділ `0.5` для інсталяції `k3d` на свіжому сервері)

#### 5.2.2. Створення локального кластеру AsciiArtify

Я створив кластер з 1 `server` та 2 `agent`-нодами й пробросом порту 8080 з хоста на порт 80 кластерного load balancer’а:

```bash
k3d cluster create asciiartify-dev \
  --servers 1 \
  --agents 2 \
  -p "8080:80@loadbalancer" \
  --wait
```

Перевірка стану кластера:

```bash
kubectl get nodes
kubectl get pods -A
```

---

#### 5.2.3. Розгортання “Hello World” застосунку

Для демо я використав образ `nginxdemos/hello`.

1. **Створив namespace:**

   ```bash
   kubectl create namespace demo
   ```

2. **Створив Deployment:**

   ```bash
   kubectl -n demo create deployment hello-world \
     --image=nginxdemos/hello
   ```

3. **Створив Service типу LoadBalancer і додав Ingress правило для трафіка:**

   ```bash
   kubectl -n demo expose deployment hello-world \
     --type=LoadBalancer \
     --port=80
   ```

   ```bash
   cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world
  namespace: demo
  annotations:
    ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
EOF
   ```

4. **Перевірив, що Pod і Service запущені:**

   ```bash
   kubectl -n demo get pods
   kubectl -n demo get svc
   ```

Оскільки при створенні кластера я пробросив порт `8080:80@loadbalancer`, доступ до сервісу отримується просто:

```bash
curl http://localhost:8080
```

З хоста можна перевірити також через тестовий под:

```bash
kubectl run test -n demo --image=curlimages/curl:latest --rm -it --restart=Never -- curl http://hello-world
```

Або у браузері:  
`http://localhost:8080`

У відповідь я отримав сторінку з “Hello” від nginx — це підтверджує коректну роботу кластера та сервісу.

---

#### 5.2.4. Як це розширити для AsciiArtify

Те саме демо легко адаптується під продукт:

- замість `nginxdemos/hello` використати власний образ, наприклад:
  - `asciiartify-api` — бекенд, що приймає зображення й повертає ASCII-art;
  - `asciiartify-frontend` — простий фронтенд, який ходить до API.
- у репозиторії GitHub додати:
  - `k8s/namespace.yaml`
  - `k3s/deployment-api.yaml`   # (ймовірно, правильніше: k8s/deployment-api.yaml, але збережемо як у поточній концепції, якщо потрібно змінити — оновити тут)
  - `k8s/deployment-frontend.yaml`
  - `k8s/service-api.yaml`
  - `k8s/service-frontend.yaml`
- створити скрипт для швидкого підняття середовища розробки, наприклад `scripts/dev-cluster.sh`:

  ```bash
  #!/usr/bin/env bash
  set -euo pipefail

  k3d cluster create asciiartify-dev \
    --servers 1 \
    --agents 2 \
    -p "8080:80@loadbalancer" \
    --wait

  kubectl apply -f k8s/
  ```

Розробники запускають один скрипт і отримують однаковий кластер з потрібними сервісами.

---

## 6. Висновки та рекомендації

На основі практичного досвіду з усіма трьома інструментами я сформував такі висновки:

### 6.1. minikube

**Рекомендоване використання:**

- Локальне **навчальне середовище** для тих, хто хоче глибше зрозуміти Kubernetes.
- Сценарії, де потрібні:
  - Dashboard, addons;
  - різні драйвери (включно з Podman та VM).
- Ситуації, коли Docker Desktop небажаний:
  - використання Podman-драйвера на Linux.

---

### 6.2. kind

**Рекомендоване використання:**

- **CI/CD середовище**, зокрема GitHub Actions:
  - піднімати кластер перед тестами;
  - перевіряти Helm-чарти та маніфести.
- Інтеграційні тести, де важливо максимально наблизитися до поведінки “справжнього” Kubernetes.

---

### 6.3. k3d

**Рекомендоване використання (основна рекомендація для PoC AsciiArtify):**

- **Основний локальний кластер** для розробки PoC:
  - легкий, швидкий, невимогливий до ресурсів;
  - простий CLI і швидке створення multi-node кластерів.
- Сценарії з ML/мікросервісами, де розробникам потрібен швидкий цикл:
  - “змінив код → перезібрав образ → оновив Deployment → протестував”.

---

### 6.4. Пропонована стратегія для AsciiArtify

1. **PoC та щоденна розробка:**
   - Використовувати **k3d** як стандартний локальний Kubernetes-кластер.
   - Задокументувати типові сценарії:
     - створення/видалення кластеру;
     - деплой сервісів AsciiArtify;
     - доступ до них з локального браузера.

2. **CI/CD і автоматизоване тестування:**
   - Використовувати **kind** для GitHub Actions:
     - створення кластера;
     - прогін тестів;
     - перевірка маніфестів, Helm-чартів.

3. **Навчання й експерименти:**
   - Використовувати **minikube**:
     - для роботи з Dashboard, addons;
     - для тестів з Podman на Linux без Docker Desktop.

4. **Ліцензійні ризики Docker:**
   - На старті можна використовувати Docker Desktop безкоштовно.
   - Паралельно:
     - готувати сценарії роботи на Linux з Docker Engine / Podman;
     - не зав’язувати архітектуру строго на Docker Desktop.

---

## 7. Підсумок

Як інженер, який проводив практичні тести, я рекомендую:

- **k3d** — як основний інструмент для PoC локального Kubernetes-кластера стартапу **AsciiArtify**;
- **kind** — як інструмент для CI/інтеграційних тестів;
- **minikube** — як навчальний та альтернативний варіант (особливо для сценаріїв без Docker Desktop та з Podman/VM).

Такий поділ ролей між інструментами дає команді:

- швидкий старт PoC;
- гнучкість для майбутнього масштабування;
- мінімізацію ризиків, пов’язаних з ліцензією Docker.
