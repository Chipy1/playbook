Старого playbook не было, пришлось всё делать с нуля.

## Что сделано

### 1. requirements.yml

Создан в директории со старой версией (roles-1/requirements.yml). Содержит ссылку на clickhouse-роль от AlexeySetevoi.

### 2. Установка clickhouse-роли

```bash
ansible-galaxy install -r requirements.yml
```

### 3. Создание vector-role

```bash
ansible-galaxy role init vector-role
```

Роль лежит в `roles-1/vector-role/`.

### 4. Наполнение vector-role

Задачи:

- Скачать RPM пакет Vector по версии
- Установить его через yum
- Создать директорию для данных
- Задеплоить конфиг из шаблона
- Запустить и включить сервис

Переменные разнесены: версия, пути, состояние сервиса - в `defaults/`. Внутренние ссылки на пакет и конфиг - в `vars/`.

### 5. Шаблоны vector-role

Конфиг `vector.toml.j2` лежит в `templates/`. Параметризирован через переменные из `defaults/`.

### 6. README для ролей

Обе роли (vector-role и lighthouse-role) содержат README с описанием переменных, требований и примером использования.

### 7. Создание lighthouse-role

```bash
ansible-galaxy role init lighthouse-role
```

Роль в `roles-2/lighthouse-role/`. Отдельно от vector-role, как и просили - одна роль = один продукт.

Задачи:

- Установить EPEL и nginx
- Скачать Lighthouse с GitHub
- Распаковать архив
- Задеплоить конфиг nginx из шаблона
- Запустить nginx

### 8. Репозитории и теги

vector-role: https://github.com/Chipy1/vector-role (тег v1.0.0)
lighthouse-role: https://github.com/Chipy1/lighthouse-role (тег v1.0.0)

Обе добавлены в `playbook/requirements.yml` вместе с clickhouse. Все три роли подтягиваются по SSH.

### 9. Playbook

`playbook/site.yml` использует три роли. Для LightHouse добавлены pre_tasks (проверка, что ClickHouse отвечает - зависимость) и post_tasks (проверка, что Lighthouse отдаёт страницу). Так выполняется требование совмещать `roles` с `tasks` в одном play.

```yaml
- name: Install ClickHouse
  hosts: clickhouse
  roles:
    - clickhouse

- name: Install Vector
  hosts: vector
  roles:
    - vector-role

- name: Install LightHouse
  hosts: lighthouse
  pre_tasks:
    - name: Ensure ClickHouse is reachable
      ansible.builtin.wait_for:
        host: "{{ clickhouse_host | default('127.0.0.1') }}"
        port: "{{ clickhouse_tcp_port | default(9000) }}"
        timeout: 30
  roles:
    - lighthouse-role
  post_tasks:
    - name: Check LightHouse is serving
      ansible.builtin.uri:
        url: "http://{{ ansible_default_ipv4.address }}/"
        status_code: 200
```

### 10. Репозиторий с playbook

playbook: https://github.com/Chipy1/playbook

### 11. Ссылки

vector-role: https://github.com/Chipy1/vector-role
lighthouse-role: https://github.com/Chipy1/lighthouse-role
playbook: https://github.com/Chipy1/playbook

## Что пошло не так

Сначала SSH ключа на GitHub не было - ansible-galaxy отказывался клонировать. Сгенерировал ключ, добавил в known_hosts, всё заработало.

Старого playbook не нашлось, так что задачи для vector и lighthouse писал по документации продуктов.
