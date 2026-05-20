# Безпека як код: GKE Binary Authorization та операційний DevSecOps (Частина 3 з 3)

> **"Політика, яку можна обійти, є лише порадою. Binary Authorization робить конвеєр збірки єдиним шляхом до продакшену."**

У [Частині 1](sbom-part1.ua.md) та [Частині 2](sbom-part2.ua.md) цієї серії ми убезпечили цикл розробника (за допомогою `uv` та `syft`), налаштували безперервне сканування реєстру та конфігурували **OPA Gatekeeper** для обмеження запусків у GKE лише тими образами, які зафіксовані в реєстрі за sha256 дайджестом.

Проте навіть із цими політиками, як ви можете гарантувати, що образ контейнера дійсно пройшов усі сканування безпеки *до того*, як потрапив у GKE? Що, якщо підписаний образ містить критичні вразливості, але його все одно завантажили в реєстр?

Для досягнення **Рівня 3 (Корпоративний захист)** нам потрібно замінити рекомендаційні обмеження на криптографічні політики. У цій фінальній статті ми впровадимо **GCP Binary Authorization** — систему, яка забороняє запуск будь-якого контейнера в GKE, якщо він не має підтвердженої криптографічної атестації (attestation) від нашого конвеєра збірки. Ми також розглянемо автоматичне оновлення залежностей за допомогою **Renovate** та операційні практики керування SBOM та CVE у великих масштабах.

---

## 1. Binary Authorization на GKE

Binary Authorization — це нативна служба GCP для контролю допуску (admission control), яка оцінює політики розгортання в реальному часі. Політика гарантує, що жоден образ не буде розгорнутий у вашому кластері GKE, якщо він не має **атестації** (цифрового підпису, який підтверджує успішне проходження певного кроку безпеки, наприклад сканування вразливостей).

Перш ніж створювати атестатор, ми маємо зареєструвати **Container Analysis Note** у GCP. Ця нотатка є реєстром метаданих, де записуються факти проходження атестації. Оскільки ми керуємо інфраструктурою GCP декларативно за допомогою Terraform, ми визначаємо її так:

```hcl
# terraform/security.tf
resource "google_container_analysis_note" "attestation_note" {
  name = "cloud-build-attestation"
  
  attestation_authority {
    hint {
      human_readable_name = "Cloud Build Attestation Authority"
    }
  }
}

# Надання сервісному акаунту Cloud Build прав на прикріплення атестацій до нотатки
resource "google_container_analysis_note_iam_member" "cloud_build_attacher" {
  note   = google_container_analysis_note.attestation_note.name
  role   = "roles/containeranalysis.notes.attacher"
  member = "serviceAccount:cloud-build@${var.project_id}.iam.gserviceaccount.com"
}
```

Далі увімкніть Binary Authorization у вашому кластері GKE та імпортуйте політику:

```bash
# Увімкнення Binary Authorization у вашому кластері
gcloud container clusters update agents-cluster --region us-central1 --enable-binauthz

# Створення атестатора
gcloud container binauthz attestors create cloud-build-attestor \
  --attestation-authority-note=projects/my-project/notes/cloud-build-attestation \
  --attestation-authority-note-project=my-project

# Додавання публічного ключа перевірки (який відповідає підпису нашого конвеєра)
gcloud container binauthz attestors public-keys add \
  --attestor=cloud-build-attestor \
  --pkix-public-key-file=cosign.pub \
  --pkix-public-key-algorithm=ecdsa-p256-sha256
```

Визначте політику, яка вимагає атестації від `cloud-build-attestor` для робочих навантажень GKE:

```yaml
# binauthz-policy.yaml
defaultAdmissionRule:
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  evaluationMode: REQUIRE_ATTESTATION
  requireAttestationsBy:
    - projects/my-project/attestors/cloud-build-attestor
```

```bash
# Імпорт політики у ваш проєкт GCP
gcloud container binauthz policy import binauthz-policy.yaml
```

Після застосування політики GKE блокуватиме будь-який образ, якщо він не був атестований вашим конвеєром збірки. Локально зібрані образи, публічні образи з Docker Hub та неавторизовані обходи будуть повністю заблоковані.

---

## 2. Повний конвеєр Cloud Build Рівня 3

Ось повний безпечний конвеєр збірки. Кожен етап є фільтром. Якщо будь-яка перевірка зазнає невдачі, збірка зупиняється, атестація не генерується, і розгортання блокується.

```yaml
# cloudbuild.yaml — Рівень 3: Повний ланцюжок атестації
steps:
  # 1. Запуск tfsec для перевірки конфігурації Terraform
  - name: 'aquasec/tfsec'
    args: ['--minimum-severity', 'HIGH', '/workspace/terraform/']
    id: 'iac-tfsec'

  # 2. Запуск Checkov для перевірки всіх шаблонів
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

  # 5. Сканування SBOM через Grype — зупинка збірки при CRITICAL або HIGH вразливостях
  - name: 'anchore/grype'
    args: ['sbom:/workspace/sbom.json', '--fail-on', 'high']
    id: 'scan'
    waitFor: ['sbom']

  # 6. Push образу в Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '$_IMAGE_TAG']
    id: 'push'
    waitFor: ['scan']

  # 7. Прикріплення SBOM як реферера OCI
  - name: 'gcr.io/projectsigstore/cosign'
    args: ['attach', 'sbom', '--sbom', '/workspace/sbom.json', '--type', 'cyclonedx', '$_IMAGE_TAG']
    id: 'attach-sbom'
    waitFor: ['push']

  # 8. Безключовий підпис образу через Cosign
  - name: 'gcr.io/projectsigstore/cosign'
    args: ['sign', '--oidc-provider=google', '$_IMAGE_TAG']
    id: 'sign'
    waitFor: ['attach-sbom']

  # 9. Створення атестації Binary Authorization для GKE
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
          IMAGE_DIGEST=$(gcloud container images describe $_IMAGE_TAG --format='get(image_summary.digest)')
          gcloud container binauthz attestations create \
            --artifact-url="$_IMAGE_BASE@$$IMAGE_DIGEST" \
            --attestor=projects/$PROJECT_ID/attestors/cloud-build-attestor \
            --signature-algorithm=ECDSA_P256_SHA256 \
            --public-key-id-override=$_COSIGN_KEY_ID
    id: 'attest'
    waitFor: ['sign']

substitutions:
  _IMAGE_TAG: 'us-central1-docker.pkg.dev/my-project/agents/orchestrator:$SHORT_SHA'
  _IMAGE_BASE: 'us-central1-docker.pkg.dev/my-project/agents/orchestrator'
  _COSIGN_KEY_ID: 'projects/my-project/locations/global/keyRings/cosign/cryptoKeys/signing-key'
```

---

## 3. Renovate для автоматичного оновлення залежностей

Налаштування сканерів — це лише половина справи; інша половина — виправлення вразливостей. З сотнями транзитивних залежностей відстежувати патчі вручну є надто трудомістким завданням.

**Renovate** автоматизує цей процес: він зчитує ваші файли залежностей (наприклад, `requirements.txt` або `pyproject.toml`) і автоматично відкриває pull-запити, коли з'являються нові оновлення. Створіть файл `renovate.json` у вашому репозиторії:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchPackagePatterns": ["*"],
      "matchUpdateTypes": ["patch"],
      "automerge": true
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  }
}
```

Коли Renovate виявляє безпекове сповіщення, яке впливає на вашу зафіксовану версію пакета, він миттєво відкриває PR. Якщо патч змінює лише патч-версію, він може автоматично зливатися після успішного проходження тестів вашого проєкту.

---

## 4. Операційна дисципліна: Життя з SBOM

Коли ваш конвеєр автоматично генерує SBOM та блокує збірки, DevSecOps стає щоденним операційним процесом.

### Тріаж CVE: Не кожна знахідка є критичною пожежею
Сканери на кшталт Grype часто виявлятимуть вразливості, які ви не можете виправити миттєво. Під час аналізу дайте відповідь на три запитання:
1.  **Чи є виправлення?** Якщо фіксована версія відсутня (`FIXED-IN` порожнє), задокументуйте це як відомий ризик та продовжуйте роботу.
2.  **Чи доступний вразливий шлях виконання коду?** Якщо вразливість стосується компонента чи функції, яку ваш AI-агент ніколи не визиває, реальний ризик є мінімальним.
3.  **Яке мережеве оточення?** Бібліотека з вразливістю всередині ізольованої мережі VPC під суворою політикою IAM несе набагато менше ризику, ніж бібліотека на публічному інтернет-шлюзі.

### SBOM як інструмент реагування на інциденти
Коли стає відомо про вразливість нульового дня, вам потрібно терміново дізнатися, чи зачіпає вона ваші сервіси. Замість ручного пошуку по коду, зробіть запит до збережених файлів SBOM:

```bash
# Пошук конкретного пакета по всіх збережених SBOM
for sbom in ./sboms/*.json; do
  echo "=== $sbom ==="
  cat "$sbom" | jq --arg pkg "vulnerable-package-name" \
    '.components[] | select(.name == $pkg) | {name, version, image: input_filename}'
done
```

### Головна метрика: Середній час до виправлення (MTTP)
Оптимізуйте вашу безпеку під одну ключову метрику: **Mean Time to Patch (MTTP)** — середній час між публікацією CVE та запуском виправленої версії у продакшені. З автоматичним оновленням через Renovate та суворим admission-контролем ви можете скоротити MTTP з тижнів до кількох годин.

---

## Підсумок 3-рівневого конвеєра безпеки

*   **Рівень 1 (Solo/Startup)**: Локальна генерація SBOM через `syft`, сканування CVE через `grype` (CI-фільтр), підпис образів `cosign` за допомогою ключів KMS.
*   **Рівень 2 (Production)**: Безключовий підпис через OIDC, збереження SBOM як OCI-артефактів реєстру, та блокування сторонніх реєстрів за допомогою **OPA Gatekeeper**.
*   **Рівень 3 (Enterprise)**: Admission-контроль на рівні всього GKE через **GCP Binary Authorization**, що вимагає криптографічних атестацій від Cloud Build.

---

## Що далі: AI-специфічні загрози

Ця серія статей дозволила повністю убезпечити інфраструктурний рівень — від робочої станції розробника до admission-контролю GKE. Проте AI-агенти, які виконуються на цій інфраструктурі, мають додаткову, специфічну поверхню атак, яку контейнерні сканери та Binary Authorization не здатні виявити.

У наступній серії статей ми перейдемо до безпеки самих AI-систем: атак **ін'єкції промптів (prompt injection)**, що дозволяють перехопити хід думок агента, **отруєння ваг моделей (model poisoning)** та вразливостей у **протоколі взаємодії агентів (A2A)**, які можуть дозволити скомпрометованому агенту отримати підвищені привілеї в мережі.
