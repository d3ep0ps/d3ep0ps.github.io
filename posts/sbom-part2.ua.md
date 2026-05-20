# Безпека як код: Блокування розгортань GKE криптографічною довірою (Частина 2 з 3)

> **"Непідписаний образ у реєстрі — це обіцянка без свідка. Безключовий підпис робить конвеєр збірки свідком."**

У [Частині 1](sbom-part1.ua.md) цієї серії ми розібрали **айсберг залежностей** та побудували локальний цикл аудиту безпеки: зафіксували залежності за допомогою `uv`, згенерували CycloneDX SBOM за допомогою `syft` та просканували код на CVE за допомогою `grype`.

Локальні перевірки — це перша лінія оборони, але вони залежать від дисципліни розробника. Для масштабування цього підходу на всю команду та надійного захисту продакшену нам потрібно перейти на **Рівень 2 (Промисловий захист)**.

У цій статті ми автоматизуємо цей конвеєр: впровадимо **безключовий підпис образів (keyless image signing)** через OIDC у Google Cloud Build, опублікуємо SBOM безпосередньо в Artifact Registry як **реферери OCI (OCI referrers)**, проведемо аудит конфігурацій інфраструктури за допомогою інструментів IaC та налаштуємо **OPA Gatekeeper** для блокування неперевірених або мінливих (mutable) образів у нашому кластері GKE.

---

## 1. Безключовий підпис образів через OIDC (Cloud Build)

Головною проблемою традиційного підпису образів є керування ключами: ротація, запобігання витоку та безпечна передача ключів у CI/CD-раннери.

**Sigstore Cosign** вирішує це за допомогою **безключового підпису**. Замість статичного приватного ключа Cosign використовує OpenID Connect (OIDC) ідентифікатор самого раннера. У Google Cloud Build завдання виконується від імені сервісного акаунту GCP. Cosign отримує OIDC-токен для цього сервісного акаунту, генерує короткочасну пару ключів, підписує дайджест образу та записує підпис у **Rekor** — публічний лог прозорості Sigstore, який працює за принципом "тільки для запису".

Підпис може бути перевірений будь-ким, хто знає ідентифікатор сервісного акаунту Cloud Build:

```yaml
# У cloudbuild.yaml — крок безключового підпису:
- name: 'gcr.io/projectsigstore/cosign'
  args:
    - 'sign'
    - '--oidc-provider=google'
    - '$_IMAGE_TAG'
```

Для перевірки цього підпису виконайте:

```bash
cosign verify \
  --certificate-oidc-issuer=https://accounts.google.com \
  --certificate-identity=serviceAccount:cloud-build@my-project.iam.gserviceaccount.com \
  us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0
```

Перевірка пройде успішно лише в тому випадку, якщо образ був підписаний завданням Cloud Build, яке запускалося саме від імені цього сервісного акаунту. Жодна інша сутність не зможе підробити цей сертифікат.

---

## 2. Прикріплення SBOM як OCI-артефакту

Замість того щоб зберігати файли SBOM у віддалених сховищах (storage buckets), ми можемо прикріпити їх безпосередньо до образу контейнера всередині Artifact Registry як реферер OCI. SBOM та образ переміщуються разом:

```bash
# Прикріплення SBOM до образу як OCI-артефакту
cosign attach sbom \
  --sbom orchestrator-sbom.json \
  --type cyclonedx \
  us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0

# Отримання SBOM з образу пізніше
cosign download sbom \
  us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0 > retrieved-sbom.json
```

Тепер будь-який інструмент чи розробник, що має доступ на читання образу контейнера, може отримати його повний маніфест компонентів програмним шляхом.

---

## 3. Увімкнення безперервного сканування Artifact Registry

Безперервне сканування працює на іншому часовому рівні, ніж перевірки в CI. У той час як Grype блокує вразливі збірки безпосередньо в CI, **сканування в Artifact Registry** перевіряє образи, що вже знаходяться в реєстрі, коли публікуються нові CVE.

Увімкніть його разом із сповіщеннями через Pub/Sub для створення швидкої системи реагування:

```bash
# Увімкнення Container Scanning API (одноразово на рівні проєкту)
gcloud services enable containerscanning.googleapis.com

# Увімкнення безперервного сканування репозиторію Artifact Registry
gcloud artifacts repositories update agents \
  --location=us-central1 \
  --vulnerability-scanning=enabled

# Створення топіка Pub/Sub для сповіщень
gcloud pubsub topics create container-vulnerabilities
```

Ви можете переглянути знайдені вразливості програмним шляхом (наприклад, для щотижневого аудиту):

```bash
gcloud artifacts docker images list-vulnerabilities \
  us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0 \
  --format="table(vulnerability.effectiveSeverity, vulnerability.shortDescription, packageIssue[0].affectedPackage)" \
  --filter="vulnerability.effectiveSeverity=CRITICAL OR vulnerability.effectiveSeverity=HIGH"
```

---

## 4. Сканування IaC: Terraform та Kubernetes

Ваші інфраструктурні файли — це також код. Неправильно налаштований пул вузлів GKE чи база даних AlloyDB є вразливостями на рівні розгортання. Додайте інструменти **tfsec** та **Checkov** до вашого конвеєра лінтингу/CI:

```bash
# Запуск tfsec для каталогу Terraform
tfsec ./terraform/ --minimum-severity HIGH

# Запуск Checkov для всієї інфраструктури: Terraform + маніфести GKE + Dockerfiles
checkov -d . --framework terraform,kubernetes,dockerfile --compact --quiet
```

Checkov підсвітить проблеми на кшталт цієї:

```
Check: CKV_GCP_69: "Ensure Kubernetes Cluster is not publicly accessible via master_authorized_networks_config"
FAILED for resource: google_container_cluster.agents
File: terraform/gke.tf:12-45
```

---

## 5. Встановлення OPA Gatekeeper на GKE

**OPA Gatekeeper** працює як Kubernetes admission webhook. Кожен запит `kubectl apply` чи створення Pod перевіряється на відповідність вашим політикам безпеки, перш ніж API-сервер прийме його.

Встановіть Gatekeeper:

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
helm install gatekeeper gatekeeper/gatekeeper --namespace gatekeeper-system --create-namespace --set replicas=2
```

### Написання ConstraintTemplate: Вимога фіксації реєстру та використання дайджестів

Gatekeeper не може перевіряти підписи образів безпосередньо (оскільки він не повинен звертатися до зовнішніх реєстрів на етапі admission-контролю). Проте він може забезпечити виконання критично важливих передумов:
1.  Образи мають завантажуватися лише з вашого довіреного Artifact Registry.
2.  Образи повинні посилатися на незмінний **sha256 дайджест**, а не на теги (як-от `:latest`).

Визначте шаблон:

```yaml
# constraint-template-require-registry.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequireapprovedregistry
spec:
  crd:
    spec:
      names:
        kind: K8sRequireApprovedRegistry
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequireapprovedregistry
 
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          image := container.image
          not startswith(image, "us-central1-docker.pkg.dev/my-project/")
          msg := sprintf("Image '%v' is not from the approved registry", [image])
        }
 
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          image := container.image
          not contains(image, "@sha256:")
          msg := sprintf("Image '%v' must be referenced by digest, not tag", [image])
        }
```

Застосуйте обмеження до GKE:

```yaml
# constraint-require-registry.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireApprovedRegistry
metadata:
  name: require-approved-registry
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["agents", "production"]
  enforcementAction: deny
```

```bash
kubectl apply -f constraint-template-require-registry.yaml
kubectl apply -f constraint-require-registry.yaml
```

Якщо ви спробуєте запустити сторонній неперевірений образ, наприклад `nginx:latest`, GKE заблокує цей запит:
```bash
kubectl run test --image=nginx:latest -n agents
# Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: Image 'nginx:latest' is not from the approved registry
```

---

## 6. Повний конвеєр Cloud Build Рівня 2

Окремі кроки, описані вище, об'єднуються в єдиний послідовний конвеєр. Кожен етап є блокуючим — помилка на будь-якому кроці зупиняє збірку до того, як образ потрапить у реєстр.

```yaml
# cloudbuild.yaml — Рівень 2: Підпис + SBOM + перевірки інфраструктури
steps:
  # 1. Запуск tfsec для перевірки конфігурації Terraform
  - name: 'aquasec/tfsec'
    args: ['--minimum-severity', 'HIGH', '/workspace/terraform/']
    id: 'iac-tfsec'

  # 2. Запуск Checkov для перевірки всіх IaC-шаблонів
  - name: 'bridgecrew/checkov'
    args: ['-d', '/workspace', '--framework', 'terraform,kubernetes,dockerfile', '--compact']
    id: 'iac-checkov'
    waitFor: ['iac-tfsec']

  # 3. Збірка образу контейнера агента
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '$_IMAGE_TAG', '-f', 'agents/orchestrator/Dockerfile', '.']
    id: 'build'
    waitFor: ['iac-checkov']

  # 4. Генерація CycloneDX SBOM за допомогою syft
  - name: 'anchore/syft'
    args: ['$_IMAGE_TAG', '-o', 'cyclonedx-json', '--file', '/workspace/sbom.json']
    id: 'sbom'
    waitFor: ['build']

  # 5. Сканування SBOM через Grype — зупинка збірки при HIGH або CRITICAL вразливостях
  - name: 'anchore/grype'
    args: ['sbom:/workspace/sbom.json', '--fail-on', 'high']
    id: 'scan'
    waitFor: ['sbom']

  # 6. Push образу в Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '$_IMAGE_TAG']
    id: 'push'
    waitFor: ['scan']

  # 7. Прикріплення SBOM як реферера OCI (переміщується разом із образом)
  - name: 'gcr.io/projectsigstore/cosign'
    args: ['attach', 'sbom', '--sbom', '/workspace/sbom.json', '--type', 'cyclonedx', '$_IMAGE_TAG']
    id: 'attach-sbom'
    waitFor: ['push']

  # 8. Безключовий підпис образу через Cosign (OIDC від Sigstore — без статичних ключів)
  - name: 'gcr.io/projectsigstore/cosign'
    args: ['sign', '--oidc-provider=google', '$_IMAGE_TAG']
    id: 'sign'
    waitFor: ['attach-sbom']

substitutions:
  _IMAGE_TAG: 'us-central1-docker.pkg.dev/my-project/agents/orchestrator:$SHORT_SHA'
```

Що ви отримуєте на Рівні 2: кожен образ у вашому реєстрі має підпис Sigstore та прикріплений SBOM. Будь-який розробник або аудиторський інструмент може завантажити обидва файли з єдиної точки реєстру. Повне криптографічне блокування непідписаних образів на етапі допуску в GKE реалізується на Рівні 3.

---

## Що далі?

Маючи налаштований **Рівень 2 (Промисловий захист)**, ви убезпечили свої образи, інфраструктурні шаблони та допуски GKE. Але що, якщо підписаний образ буде скомпрометований *після* успішного проходження перевірок, чи якщо хтось спробує обійти політики Gatekeeper?

У **Частині 3** ми розберемо **GCP Binary Authorization** — фінальний криптографічний щит. Ми побудуємо повний ланцюжок атестації (attestation chain) від Cloud Build до GKE Autopilot, налаштуємо Renovate для автоматичного оновлення залежностей та розберемо алгоритм реагування на інциденти за допомогою SBOM.
