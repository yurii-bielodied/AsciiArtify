# MVP: GitOps-деплой `go-demo-app` через Argo CD на k3d

---

## 0. Цілі MVP

На етапі MVP ми:

1. Підключаємо **реальний продукт** (`go-demo-app`) до вже налаштованої GitOps-інфраструктури.
2. Створюємо **додаток в Argo CD**, який відслідковує репозиторій:  
   `https://github.com/den-vasyliev/go-demo-app`
3. Вмикаємо **автоматичну синхронізацію** (auto-sync) між Git та Kubernetes-кластером.
4. Демонструємо **повний цикл**:
   - зміна в Git → Argo CD її бачить → автоматично застосовує в кластері;
   - користувачі отримують оновлену версію додатку.
5. Готуємо базу для показу **фокус-групі** та подальшого розвитку AsciiArtify.

---

## 1. Передумови

Перед стартом MVP вважаємо, що:

- PoC GitOps-інфраструктури **вже виконаний** (див. [`doc/POC.md`](./POC.md)):
  - є кластер k3d `asciiartify-gitops`;
  - Argo CD встановлений у namespace `argocd`;
  - доступ до веб-інтерфейсу Argo CD (через `kubectl port-forward` або інший спосіб) налаштовано.
- На робочій машині встановлено:
  - **Docker Engine**
  - **kubectl**
  - **k3d**
  - (опційно) **git**

> Якщо PoC ще не пройдений — спочатку виконуємо кроки з `doc/POC.md`, а потім повертаємось до цього документу.

---

## 2. Підготовка Kubernetes-кластера для `go-demo-app`

Працюємо з уже існуючим кластером `asciiartify-gitops`:

```bash
kubectl config use-context k3d-asciiartify-gitops

kubectl cluster-info
kubectl get nodes
```

Створюємо окремий namespace для MVP:

```bash
kubectl create namespace go-demo
```

> Ім'я namespace можна змінити на `asciiartify-mvp`, але в цьому документі для простоти використовуємо `go-demo`.

---

## 3. Argo CD Application для `go-demo-app` (декларативний варіант)

### 3.1. Маніфест Argo CD Application

Приклад `k8s/argocd-app-go-demo.yaml`:

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
      prune: true       # видаляти зайві ресурси
      selfHeal: true    # самовідновлення при ручних змінах у кластері
    syncOptions:
      - CreateNamespace=true
```

> `path: helm` — використовує вбудований в `go-demo-app` Helm-чарт.  
> `automated + prune + selfHeal` — саме це забезпечує повноцінну **авто-синхронізацію**.

### 3.2. Застосування маніфесту Application

```bash
kubectl apply -n argocd -f k8s/argocd-app-go-demo.yaml
```

Перевіряємо, що додаток з’явився в Argo CD:

```bash
kubectl get applications -n argocd
```

або через веб-інтерфейс Argo CD → має бути `go-demo-app` зі статусом `Synced` / `Healthy` після першої синхронізації.

---

## 4. Створення Application через веб-інтерфейс Argo CD (альтернатива)

Якщо зручніше створювати додаток руками через UI:

1. Відкрити веб-інтерфейс Argo CD (див. `POC.md`, розділ про доступ до UI).
2. Натиснути **`+ New App`**.
3. Заповнити форму:

**GENERAL**

- *Application Name*: `go-demo-app`
- *Project*: `default`
- *Sync Policy*: `Automatic` (поставити галочку `AUTO-CREATE NAMESPACE` / `Self Heal` / `Prune` — залежно від версії UI).

**SOURCE**

- *Repository URL*: `https://github.com/den-vasyliev/go-demo-app`
- *Revision*: `main`
- *Path*: `helm`

**DESTINATION**

- *Cluster URL*: `https://kubernetes.default.svc`
- *Namespace*: `go-demo`

**SYNC OPTIONS**

- *AUTO-CREATE NAMESPACE*: ☑️

1. Натиснути **Create**.
2. Почекати, поки Argo CD ініціалізує перший деплой (статус має стати `Synced` + `Healthy`).

---

## 5. Перевірка роботи додатку (функціональне демо)

Після того, як `go-demo-app` задеплоєний Argo CD:

### 5.1. Перевірка подів та сервісів

```bash
kubectl -n go-demo get pods
kubectl -n go-demo get svc
```

Очікування:

- декілька pod’ів додатку у статусі `Running`;
- сервіс на базі API Gateway (**зазвичай це `ambassador`**), через який ходитимуть запити.

### 5.2. Port-forward до API Gateway

Щоб продемонструвати роботу продукту локально, зробимо `port-forward` на сервіс, який приймає HTTP-трафік від користувачів (наприклад, `ambassador`):

```bash
kubectl -n go-demo port-forward svc/ambassador 8081:80
```

Тепер додаток доступний з локальної машини:

- Базова URL: `http://localhost:8081`

### 5.3. Приклади запитів до `go-demo-app`

1. **Перетворення тексту в ASCII-art**:

   ```bash
   curl -XPOST --data '{"text":"AsciiArtify MVP"}' \
     http://localhost:8081/ascii/
   ```

2. **Перетворення зображення в ASCII-art** (приклад з локальним файлом):

   ```bash
   wget -O /tmp/demo.png https://via.placeholder.com/200x80.png?text=AsciiArtify

   curl -F 'image=@/tmp/demo.png' http://localhost:8081/img/
   ```

3. **Веб-інтерфейс ML/демо** (якщо підтримується у конкретній версії чарта):

   - Відкрити в браузері: `http://localhost:8081/ml5/`

---

## 6. Демонстрація автоматичної синхронізації (повний GitOps-цикл)

Щоб показати всю магію GitOps, потрібно продемонструвати:

> **Зміни в Git → Argo CD виявив → автоматично синхронізував → Kubernetes оновив додаток.**

### 6.1. Підготовка моніторингу в терміналі

У окремому терміналі:

```bash
watch -n 2 'kubectl -n go-demo get pods'
```

Це дозволить побачити rolling-update (старі pod’и завершуються, нові стартують).

### 6.2. (Опційно) Створити fork для власних змін

Щоб не комітити напряму в `https://github.com/den-vasyliev/go-demo-app`, можна:

1. Зробити **fork** репозиторію на GitHub у свій акаунт, наприклад:  
   `https://github.com/yurii-bielodied/go-demo-app`
2. Оновити `repoURL` в маніфесті Argo CD Application / у UI на `https://github.com/yurii-bielodied/go-demo-app`.
3. Далі працювати зі своїм форком (push змін, теги, гілки).

> У завданні явно вказано `https://github.com/den-vasyliev/go-demo-app` як продукт —  
> для demo авто-синхронізації досить тимчасово перемкнутись на fork, а потім повернутись, якщо потрібно.

### 6.3. Внести зміну в Git

У локальному клону репозиторію (форку):

```bash
git clone https://github.com/yurii-bielodied/go-demo-app
cd go-demo-app
```

Далі — **будь-яка несумісна зміна, яку легко видно**. Наприклад:

- збільшити кількість реплік сервісу;
- змінити тег Docker-образу додатку (наприклад, з `v5.0.0` на `v5.0.1`);
- змінити текст версії / банера, який потім видно у відповіді API.

Приклад (умовний, структура `values.yaml` може відрізнятись):

```bash
vim helm/values.yaml
# змінити:
# replicaCount: 1  →  replicaCount: 2
# або змінити тег образу
```

Коміт та push:

```bash
git commit -am "Increase replicas for MVP demo"
git push origin main
```

### 6.4. Поведінка Argo CD

Після push:

1. Argo CD побачить зміну в Git.
2. Статус додатку в UI короткий час буде `OutOfSync`, але завдяки `syncPolicy.automated`:
   - Argo CD автоматично запустить **Sync**;
   - статус зміниться на `Synced` / `Healthy`.
3. У терміналі з `watch -n 2 'kubectl -n go-demo get pods'` буде видно rolling-update:
   - старі pod’и йдуть у `Terminating`;
   - нові pod’и з новою конфігурацією (інший тег/репліки) створюються.

Додатково можна:

- зробити `curl` до `http://localhost:8081/api/` / `ascii/` і перевірити, що версія/поведінка змінилася;
- зняти короткий запис екрану, де одночасно видно:
  - UI Argo CD (авто-sync),
  - термінал з pod’ами,
  - curl/браузер із відповіддю додатку.

---

## 7. Вбудоване демо для оцінювання

## Демо: робота продукту на інфраструктурі AsciiArtify

![MVP demo: Argo CD auto-sync](img/asciiartify-mvp-argocd.mp4)

<video src="img/asciiartify-mvp-argocd.mp4" controls width="640"></video>

> На демо видно роботу `go-demo-app`:
> - інтерфейс Argo CD: статус додатку, auto-sync;
> - зміну в Git-репозиторії та автоматичне оновлення подів у кластері.
> - демонструє **автоматичну синхронізацію Argo CD ←→ Git ←→ Kubernetes**.

---

## 8. Висновок

На етапі MVP ми:

- Використали ті ж самі принципи, що були закладені в `Concept` та `PoC` (k3d + Argo CD + GitOps).
- Підключили **реальний багатокомпонентний додаток** `go-demo-app` як прототип AsciiArtify.
- Налаштували **авто-синхронізацію** Argo CD з Git-репозиторієм.
- Показали повний GitOps-цикл “зміна в Git → оновлений деплой в кластері” на живому прикладі.

Інфраструктура готова до:

- подальшої заміни `go-demo-app` на власні сервіси AsciiArtify;
- розширення (моніторинг, логування, ingress, TLS, тощо);
- тестування на фокус-групі користувачів з наступними ітераціями продукту.
