# LinkedIn Posts — TLS From First Principles

---

## 🇬🇧 English Post → Medium (International audience)

Every engineer has typed `ssl = yes` in a config file.

Almost no one knows what happens next.

I published a new article that fixes that — not a "how to install certbot" tutorial, but a full mental model: from why HTTP is broken by design, to how two machines negotiate a secret key in plain sight of the entire internet without anyone being able to steal it.

**What's inside:**

🔐 The three threats TLS defeats — eavesdropping, tampering, impersonation — with a live `tcpdump` demo that shows credentials flying across the wire in cleartext

🔑 How the TLS handshake actually works — the padlock analogy for asymmetric encryption, Diffie-Hellman key exchange in plain English, and the six things your browser silently checks on every certificate

📜 The honest certificate landscape — self-signed vs. commercial DV/OV/EV vs. Let's Encrypt, what you're actually paying for, and why 90-day certs are a security *feature*, not a limitation

🛠 Four fully-worked practice labs:
→ Apache + Nginx with Certbot
→ Postfix + Dovecot (reusing the same cert, with auto-reload hooks on renewal)
→ cert-manager on GKE (ClusterIssuer → Certificate → Secret, fully automated)
→ Google-managed certificates + Cloud Run zero-config TLS

📊 A Prometheus alert rule for certificate expiry — because the answer to "never let a cert expire" is architecture, not vigilance

---

One thing I didn't plan to write but turned into the sharpest line in the piece:

> *"If you find yourself manually copying certificate files between servers, stop. That process will fail the moment no one is watching."*

Certificate expiry is one of the most avoidable incidents in infrastructure. It is also one of the most common. The fix isn't a reminder in your calendar — it's removing yourself from the renewal loop entirely.

Full article on Medium → [LINK]

Part of the d3ep0ps series — a structured engineering journey from the first `uname -a` on a fresh OS all the way to intelligent, self-healing cloud-native systems.

#DevOps #Security #TLS #Kubernetes #Linux #SRE #CloudEngineering #GCP #SystemArchitecture #LetsEncrypt

---

---

## 🇺🇦 Ukrainian Post → d3ep0ps.com (Ukrainian tech audience)

Кожен інженер хоч раз писав `ssl = yes` у конфіг-файлі.

Майже ніхто не знає, що відбувається далі.

Новий матеріал на d3ep0ps.com — не черговий туторіал "як встановити certbot", а повна ментальна модель: від того, чому HTTP зламаний за своєю природою, до того, як два сервери домовляються про спільний секретний ключ — на очах у всього інтернету — і ніхто не може його вкрасти.

**Що всередині:**

🔐 Три загрози, від яких захищає TLS — підслуховування, підміна даних, імітація — з живою демонстрацією через `tcpdump`, де паролі летять по мережі у відкритому вигляді

🔑 Як насправді працює TLS-рукостискання — аналогія з навісним замком для асиметричного шифрування, обмін ключами Діффі-Хеллмана простими словами, і шість речей, які браузер перевіряє у кожному сертифікаті

📜 Чесна картина ринку сертифікатів — self-signed проти комерційних DV/OV/EV проти Let's Encrypt, за що ви насправді платите, і чому 90-денні сертифікати — це свідоме рішення з безпеки, а не обмеження

🛠 Чотири повністю розібрані практичні лаби:
→ Apache + Nginx через Certbot
→ Postfix + Dovecot (один сертифікат на обидва, з автоматичним перезавантаженням після поновлення)
→ cert-manager на GKE (ClusterIssuer → Certificate → Secret — без жодного ручного кроку)
→ Google-managed certificates + Cloud Run без жодного налаштування TLS

📊 Правило алерту для Prometheus на закінчення терміну дії сертифіката — бо правильна відповідь на "сертифікат не повинен протікати" — це архітектура, а не пильність

---

Фраза, яку я не планував писати, але яка стала найгострішою у статті:

> *"Якщо ви вручну копіюєте файли сертифікатів між серверами — зупиніться. Цей процес дасть збій саме тоді, коли за ним ніхто не стежить."*

Закінчення терміну дії сертифіката — один із найбільш передбачуваних інцидентів в інфраструктурі. І водночас — один із найпоширеніших. Рішення не в нагадуванні в календарі, а в тому, щоб повністю прибрати людину з процесу поновлення.

Стаття українською → https://d3ep0ps.com

Частина серії d3ep0ps — структурований інженерний шлях від першого `uname -a` на свіжій системі до інтелектуальних, самовідновлюваних хмарних систем.

#DevOps #Security #TLS #Kubernetes #Linux #Безпека #Інфраструктура #GCP #SRE #d3ep0ps #УкраїнськіІнженери
