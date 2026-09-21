# Elasticsearch кластер на Ansible

Тестове завдання — підняти Elasticsearch кластер через Ansible. Раніше не мала практичного досвіду з Ansible, тільки теорію, тому цей проєкт — ще й спосіб перевірити, чи справді цікавий мені DevOps напрям.

## Що вийшло

Підняла 3 EC2-інстанси на AWS, поставила туди Elasticsearch через Ansible-роль, налаштувала кластеризацію (discovery, master election) і ввімкнула security автентифікацію та TLS для зв'язку між нодами.

Спершу зробила все без security, щоб переконатись що кластер взагалі піднімається і працює (записує/читає дані, шарди розподіляються по нодах). Коли це запрацювало, додала security.

## Архітектура

Три ноди спілкуються між собою по приватних IP всередині AWS VPC (порт 9300, тепер з TLS). Публічні IP використовую тільки щоб підключитись по SSH і постукатись через curl ззовні.

## Структура репозиторію
```
.
├── ansible.cfg
├── inventory.ini
├── site.yml
├── group_vars/elasticsearch/vault.yml     # пароль elastic-юзера, зашифрований
└── roles/elasticsearch/
    ├── tasks/main.yml
    ├── templates/
    │   ├── elasticsearch.yml.j2
    │   └── index-settings.json.j2         # shards, replicas, mapping для logs-index
    ├── handlers/main.yml
    └── files/elastic-certs.p12            # TLS-сертифікат, генерується локально
```

## Як розгорнути

1. Клонувати репо, заповнити `inventory.ini` своїми IP і шляхом до SSH-ключа.
2. Згенерувати транспортний TLS-сертифікат на одній з нод (`elasticsearch-certutil cert --out elastic-certs.p12`), скопіювати в `roles/elasticsearch/files/`.
3. `ansible-playbook site.yml --ask-vault-pass`
4. На одній із нод (по SSH) виконати `sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -a -b`, скопіювати пароль і зберегти в `group_vars/elasticsearch/vault.yml` через ansible-vault.
5. Перевірити: `curl -u elastic:пароль http://<IP>:9200/_cluster/health?pretty` (пароль передається відкритим текстом по HTTP без TLS — прийнятно для тестового кластера, для production варто увімкнути `xpack.security.http.ssl.enabled`)

## Про security

Спочатку кластер був повністю без захисту тому що так найшвидше піднімається MVP. Але залишати таке рішення фінальним не хотіла.

Що зробила:
- xpack.security увімкнена, без пароля доступу нема
- TLS між нодами (порт 9300)
- Пароль не лежить в git відкритим текстом, а тільки через ansible-vault

## Про індекси

Додала explicit конфігурацію через Ansible (`index-settings.json.j2`):
- `number_of_shards: 1` — для такого обсягу даних більше не потрібно, шардинг має сенс тільки коли дані не вміщаються на одну ноду
- `number_of_replicas: 2` — по репліці на кожну з двох інших нод, щоб втрата будь-якої однієї ноди не призвела до втрати даних
- Явний mapping замість dynamic: `name` як `keyword` (точні значення, для фільтрів), `message` як `text` (повнотекстовий пошук), `timestamp` як `date`

Перевірила через `_cat/shards`, що primary і обидві репліки реально лежать на різних нодах, і що запис документа реплікується одразу на всі три (`_shards.successful: 3`).

## Перевірка роботи

Записала тестовий документ, прочитала його назад, перевірила що primary-шард і репліка лежать на різних нодах, тобто це реальний розподілений кластер, а не три окремих інстанси з однаковим ім'ям. Також перевірила explicit index settings: shards і replicas розподілені по нодах як очікувалось, mapping застосувався правильно (не dynamic).
