# AsciiArtify

AsciiArtify — це стартап, який розробляє ML-застосунок для перетворення зображень у ASCII-art.  
Цей репозиторій містить:

- концепцію локального Kubernetes-середовища для PoC;
- інструкції зі встановлення залежностей;
- приклад демо-деплою “Hello World” у кластер k3d;
- GitOps PoC із **Argo CD** для доступу до веб-інтерфейсу та керування деплоєм;
- **MVP**-деплой реального застосунку `go-demo-app` з **auto-sync** в Argo CD.

Основний документ із детальним порівнянням інструментів локального Kubernetes:

👉 див. [`doc/Concept.md`](doc/Concept.md)

Документ з описом GitOps PoC та інструкціями по Argo CD:

👉 див. [`doc/POC.md`](doc/POC.md)

Документ із налаштуванням **MVP** (Argo CD Application → `go-demo-app`, авто-синхронізація):

👉 див. [`doc/MVP.md`](doc/MVP.md)

---

## Структура репозиторію

```text
AsciiArtify/
  doc/
    Concept.md                    # детальний аналіз minikube, kind, k3d + демо
    POC.md                        # GitOps PoC: Argo CD на k3d, доступ до UI
    MVP.md                        # MVP: GitOps-деплой go-demo-app + авто-синхронізація
  img/
    asciiartify-demo.gif          # демо запуску локального кластера та деплою “Hello World” на k3d
  k8s/                            # маніфести Kubernetes для PoC
  README.md
```

---

## Передумови

Повний список залежностей (Docker, kubectl, k3d, kind, minikube, Podman) описаний у  
розділі **0. Передумови: середовище та залежності** файлу [`doc/Concept.md`](doc/Concept.md).

Коротко:

- OS: Linux (наприклад, Ubuntu 22.04 LTS);
- Встановлені:
  - Docker Engine (`docker` CLI);
  - `kubectl`;
  - `k3d` (основний інструмент для PoC).

---

## Швидкий старт (Hello World на k3d)

> Детальні кроки з поясненнями див. в розділі **5. Demo** файлу [`doc/Concept.md`](doc/Concept.md).  
> Тут — стислий “TL;DR”.

1. **Створити кластер k3d:**

   ```bash
   k3d cluster create asciiartify-dev \
     --servers 1 \
     --agents 2 \
     -p "8080:80@loadbalancer" \
     --wait
   ```

2. **Створити namespace:**

   ```bash
   kubectl create namespace demo
   ```

3. **Задеплоїти “Hello World” (nginxdemos/hello):**

   ```bash
   kubectl -n demo create deployment hello-world \
     --image=nginxdemos/hello

   kubectl -n demo expose deployment hello-world \
     --type=LoadBalancer \
     --port=80
   ```

4. **Перевірити, що Pod і Service працюють:**

   ```bash
   kubectl -n demo get pods
   kubectl -n demo get svc
   ```

5. **Відкрити додаток у браузері:**

   ```bash
   curl http://localhost:8080
   ```

   Або просто зайти на:

   - `http://localhost:8080`

---

## GitOps PoC з Argo CD (доступ до UI)

> Детальна покрокова інструкція — у [`doc/POC.md`](doc/POC.md).  
> Нижче — короткий “TL;DR” для розробників.

1. **Створити окремий кластер для GitOps:**

   ```bash
   k3d cluster create asciiartify-gitops      --servers 1      --agents 2      --wait
   ```

2. **Встановити Argo CD:**

   ```bash
   kubectl create namespace argocd

   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

   Перевірити:

   ```bash
   kubectl get pods -n argocd
   ```

   Усі поди мають бути в статусі `Running`.

3. **Простий доступ до UI через port-forward:**

   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```

   Після цього інтерфейс буде доступний за адресою:

   - `https://localhost:8080`

4. **Отримати початковий пароль користувача `admin`:**

   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
   ```

5. **Залогінитись у веб-інтерфейс Argo CD:**

   - **URL:** `https://localhost:8080`
   - **Username:** `admin`
   - **Password:** (пароль з попередньої команди)

   Після входу рекомендується відразу змінити пароль.

Далі можна додати GitOps-додаток (наприклад, демо `guestbook` або майбутні сервіси AsciiArtify) через **`+ New App`** в Argo CD — деталі в `doc/POC.md`.

---

## MVP: GitOps-деплой `go-demo-app` з auto-sync

> Повні деталі — у [`doc/MVP.md`](doc/MVP.md). Тут — **TL;DR**.

1. **Namespace для MVP:**

   ```bash
   kubectl create namespace go-demo
   ```

2. **Argo CD Application (декларативно):**

   Додай файл `k8s/argocd-app-go-demo.yaml` і застосуй:

   ```bash
   kubectl apply -n argocd -f k8s/argocd-app-go-demo.yaml
   ```

   Приклад маніфесту:

   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: go-demo-app
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/den-vasyliev/go-demo-app.git
       targetRevision: main
       path: helm
       helm:
         releaseName: go-demo
     destination:
       server: https://kubernetes.default.svc
       namespace: go-demo
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```

3. **Перевірити деплой:**

   ```bash
   kubectl -n go-demo get pods,svc
   ```

   (за потреби `port-forward` до API Gateway, напр. `ambassador` → `http://localhost:8081`)

4. **Показати авто-синхронізацію:**

   - зроби fork `https://github.com/den-vasyliev/go-demo-app` → `https://github.com/<username>/go-demo-app`;
   - у маніфесті `repoURL` вкажи свій fork;
   - зміни щось очевидне (напр., кількість реплік / тег образу), `git push`;
   - спостерігай у UI Argo CD: `OutOfSync` → `Synced` + rolling update pod’ів.

---

## Як адаптувати під реальний AsciiArtify

На базі цього PoC-кластера можна:

- замінити образ `nginxdemos/hello` на власний бекенд, наприклад `asciiartify-api`;
- додати фронтенд `asciiartify-frontend`, який спілкується з API;
- оформити маніфести у папці `k8s/`:
  - `k8s/namespace.yaml`
  - `k8s/deployment-api.yaml`
  - `k8s/deployment-frontend.yaml`
  - `k8s/service-api.yaml`
  - `k8s/service-frontend.yaml`
- додати скрипт `scripts/dev-cluster.sh`, який:
  - створює кластер k3d;
  - застосовує всі маніфести.

На GitOps-рівні (через Argo CD):

- зберігати маніфести/Helm-чарти AsciiArtify у Git;
- налаштувати Argo CD `Application`, яке відслідковує ці репозиторії та синхронізує їх у кластер.

---

## Ліцензії та подальші кроки

- Для дев-середовища на Linux ми використовуємо **Docker Engine**, що не вимагає Docker Desktop.
- Для майбутнього масштабування:
  - розглянути використання kind у CI (GitHub Actions);
  - зберегти можливість переходу на Podman на Linux (особливо для minikube/kind).
- Для GitOps:
  - використовувати Argo CD як основний інструмент для керування деплоєм;
  - перевести сервіс AsciiArtify на модель “все через Git” (manifests / Helm / Kustomize).

Детальний аналіз та рекомендації див. у [`doc/Concept.md`](doc/Concept.md).

Деталі GitOps PoC та Argo CD — у [`doc/POC.md`](doc/POC.md).

Деталі MVP та auto-sync — у [`doc/MVP.md`](doc/MVP.md).
