# Техническое задание  
IaC для развертывания Pterodactyl Panel + Wings + Minecraft серверов
для проекта ArcDraco (panel.arcdraco.com / arcdraco.com)

## 1. Цель

Реализовать инфраструктуру как код (IaC), которая позволяет:

1) Развернуть с нуля рабочее окружение с Pterodactyl Panel и Wings на целевой Linux машине  
2) Описать конфигурацию сервера полностью в репозитории (git)  
3) Запускать разворачивание и обновления:
   - либо вручную с целевой машины (git pull + команда)
   - либо через GitHub Actions по push или manual dispatch  
4) Обеспечить воспроизводимость и идемпотентность разворачивания

## 2. Целевая архитектура

Окружение на одной машине (VPS или dedicated) под Ubuntu 22.04:

- Pterodactyl Panel  
  - Доступна по HTTPS на домене `panel.arcdraco.com`  
  - Работает через PHP-FPM, MariaDB, Redis, Nginx, Certbot  
- Pterodactyl Wings  
  - Работает на той же машине  
  - Использует Docker для игровых контейнеров  
- Minecraft серверы  
  - Запускаются через Wings в Docker-контейнерах  
  - Доступны по домену `arcdraco.com` с разными портами (например: 25565, 25566, 25567 и т д)

Сетевое окружение:

- Панель доступна по HTTPS на `panel.arcdraco.com`  
- Игровые сервера доступны по `arcdraco.com:PORT` (портовая маршрутизация)  
- Административный доступ по SSH на IP сервера

## 3. Требования

### 3.1. Функциональные

1) Один репозиторий описывает все действия по развертыванию:
   - установка системных пакетов
   - установка и настройка Pterodactyl Panel
   - установка и настройка Wings
   - установка и настройка Nginx + HTTPS
   - базовая конфигурация панели (APP_URL, подключение к БД, очередь, cron)
2) Возможность задать параметры окружения через переменные:
   - домен панели `PANEL_DOMAIN` (по умолчанию `panel.arcdraco.com`)
   - домен проекта `PROJECT_DOMAIN` (по умолчанию `arcdraco.com`)
   - версия Ubuntu (минимум 20.04, целевая 22.04)
   - параметры БД (имя, пользователь, пароль)
   - лимиты CPU / RAM / Disk для ноды (конфиг Pterodactyl)
   - диапазон игровых портов на `arcdraco.com`
3) Возможность запустить разворачивание:
   - локально на сервере одной командой (например `make deploy` или `ansible-playbook ...`)
   - через GitHub Actions workflow
4) Все операции должны быть идемпотентными:
   - повторный запуск не ломает окружение
   - настройки не дублируются
   - пакеты не переустанавливаются без необходимости

### 3.2. Нефункциональные

1) Репозиторий должен быть читаемым и расширяемым  
2) Секреты (пароли БД, API токены, SSH ключи) не хранятся в открытом виде:
   - либо используются GitHub Actions secrets
   - либо Ansible Vault
3) Обновление версии Pterodactyl не требует полного пересоздания сервера  
4) Код должен быть разбит на логические модули (system, panel, wings, nginx, ssl)

## 4. Технологический стек IaC

Обязательные инструменты:

- Ansible как основной инструмент конфигурации сервера  
- GitHub Actions как CI pipeline для запуска Ansible

Опционально (если генератор захочет расширить):

- Terraform для создания самой виртуальной машины в облаке  
  - В ТЗ это можно описать как отдельный слой, но реализация не обязательна на первом этапе

## 5. Структура репозитория

Требуемая примерная структура:

```text
.
├── README.md
├── infra
│   ├── ansible
│   │   ├── inventories
│   │   │   ├── production
│   │   │   │   └── hosts.ini
│   │   │   └── staging (опционально)
│   │   ├── group_vars
│   │   │   └── all.yml
│   │   ├── roles
│   │   │   ├── common
│   │   │   ├── docker
│   │   │   ├── pterodactyl_panel
│   │   │   ├── pterodactyl_wings
│   │   │   ├── nginx
│   │   │   └── ssl
│   │   ├── playbooks
│   │   │   ├── site.yml
│   │   │   ├── panel.yml
│   │   │   └── wings.yml
│   │   └── ansible.cfg
│   └── terraform (опционально)
│       └── main.tf
└── .github
    └── workflows
        ├── deploy.yml
        └── check.yml
```

Пояснения:

- `infra/ansible/roles/common`  
  - базовые пакеты, обновления, настройки UFW, создание пользователя админа
- `infra/ansible/roles/docker`  
  - установка Docker и настройка systemd unit
- `infra/ansible/roles/pterodactyl_panel`  
  - скачивание панели, установка PHP, MariaDB, Redis  
  - конфиг `.env`  
  - выполнение миграций
  - настройка очередей и cron
- `infra/ansible/roles/pterodactyl_wings`  
  - установка бинарника Wings  
  - создание config.yml из шаблона  
  - настройка systemd unit
- `infra/ansible/roles/nginx`  
  - установка Nginx  
  - разворачивание конфигов для панели
- `infra/ansible/roles/ssl`  
  - установка Certbot  
  - получение сертификата Let’s Encrypt для `panel.arcdraco.com`

## 6. Переменные и конфигурация

Все важные параметры вынести в переменные Ansible:

```yaml
# infra/ansible/group_vars/all.yml

panel_domain: "panel.arcdraco.com"
project_domain: "arcdraco.com"

panel_install_dir: "/var/www/pterodactyl"

db_name: "panel"
db_user: "paneluser"
db_password: "CHANGE_ME_SECURE"

php_version: "8.1"

pterodactyl_panel_version: "latest"
pterodactyl_wings_version: "latest"

node_name: "arcdraco-main-node"
node_public_fqdn: "arcdraco.com"
node_memory_mb: 8192
node_disk_mb: 102400

# диапазон игровых портов, выдаваемых Pterodactyl для серверов на arcdraco.com
game_ports:
  start: 25565
  end: 25600

firewall_allowed_ports:
  - 22
  - 80
  - 443
  - 25565
  - 25566
  - 25567
```

Секреты, которые нельзя хранить в репозитории:

- `db_password`
- возможные API ключи панели
- SSH private key

Для них использовать:

- либо Ansible Vault  
- либо GitHub Actions secrets и подстановку через env

## 7. Ansible playbooks

### 7.1. Главный плейбук `site.yml`

Назначение: полный деплой с нуля.

```yaml
# infra/ansible/playbooks/site.yml
- hosts: all
  become: true

  roles:
    - role: common
    - role: docker
    - role: pterodactyl_panel
    - role: pterodactyl_wings
    - role: nginx
    - role: ssl
```

### 7.2. Плейбук для обновления панели

```yaml
# infra/ansible/playbooks/panel.yml
- hosts: all
  become: true

  roles:
    - role: pterodactyl_panel
```

### 7.3. Плейбук для обновления Wings

```yaml
# infra/ansible/playbooks/wings.yml
- hosts: all
  become: true

  roles:
    - role: pterodactyl_wings
```

## 8. Роли Ansible (высокоуровневое описание)

### 8.1. Роль `common`

Задачи:

1) Обновление пакетов  
2) Установка базовых утилит (curl, git, unzip, tar, htop, ufw)  
3) Настройка часового пояса  
4) Включение UFW и открытие портов из списка `firewall_allowed_ports`  

### 8.2. Роль `docker`

Задачи:

1) Установка Docker через официальный скрипт или через apt  
2) Включение и запуск сервиса  
3) Проверка, что `docker info` выполняется без ошибок  

### 8.3. Роль `pterodactyl_panel`

Задачи:

1) Установка PHP выбранной версии и расширений  
2) Установка и базовая настройка MariaDB  
3) Создание базы `db_name` и пользователя `db_user`  
4) Установка Redis  
5) Загрузка архива панели из GitHub releases по `pterodactyl_panel_version`  
6) Распаковка в `panel_install_dir`  
7) Установка зависимостей через Composer  
8) Создание `.env` из шаблона и заполнение:
   - `APP_URL=https://{{ panel_domain }}`
   - настройки БД и Redis  
9) Выполнение миграций и сидов  
10) Создание systemd сервиса для очередей `pteroq`  
11) Настройка cron для `artisan schedule:run`

Создание первого администратора можно оставить на ручной шаг или реализовать скриптом с ожидаемыми параметрами.

### 8.4. Роль `pterodactyl_wings`

Задачи:

1) Скачивание бинарника Wings нужной версии  
2) Создание директории конфигурации `/etc/pterodactyl`  
3) Шаблон `config.yml` с параметрами ноды:
   - память, диск, FQDN = `{{ project_domain }}`, порт демона (по умолчанию 8080)  
4) Создание и запуск systemd unit `wings.service`

Первичное создание ноды и выдача конфига можно сделать:

- либо half-автоматом (описать в README ручные шаги через панель)  
- либо с помощью API Pterodactyl, если будет реализован вызов к API

### 8.5. Роль `nginx`

Задачи:

1) Установка Nginx  
2) Шаблон `pterodactyl.conf` в `sites-available`:
   - `server_name {{ panel_domain }}`
   - `root {{ panel_install_dir }}/public`
   - конфигурация PHP FPM  
3) Включение сайта через symlink в `sites-enabled`  
4) Проверка конфига (`nginx -t`) и перезагрузка сервиса  

### 8.6. Роль `ssl`

Задачи:

1) Установка Certbot и плагина для Nginx  
2) Получение сертификата Let’s Encrypt для `panel.arcdraco.com`  
3) Настройка автоматического обновления сертификатов  
4) Опционально тестовый вызов `certbot renew --dry-run`

## 9. GitHub Actions

### 9.1. Workflow `deploy.yml`

Назначение: запуск Ansible деплоя на целевой машине по кнопке или по push в main.

Примерная логика:

- Триггеры:
  - `workflow_dispatch`
  - `push` в `main` (опционально)
- Шаги:
  1) Checkout репозитория  
  2) Настройка SSH доступа к серверу:
     - использование GitHub Secret с приватным ключом  
  3) Установка Ansible в раннере  
  4) Запуск `ansible-playbook infra/ansible/playbooks/site.yml -i infra/ansible/inventories/production/hosts.ini`

Пример `hosts.ini`:

```ini
[panel]
arcdraco-main ansible_host=YOUR_SERVER_IP ansible_user=ubuntu
```

### 9.2. Workflow `check.yml`

Назначение: базовые проверки кода перед merge:

- yamllint  
- ansible-lint  
- возможно, terraform validate (если будет terraform)

## 10. Локальный запуск (без GitHub Actions)

На целевой машине должны быть:

- git  
- Ansible  

Шаги:

```bash
git clone <repo_url> /opt/arcdraco-infra
cd /opt/arcdraco-infra/infra/ansible

ansible-playbook playbooks/site.yml -i inventories/production/hosts.ini
```

## 11. Документация

В `README.md` требуется:

1) Краткое описание проекта и архитектуры (ArcDraco, Pterodactyl, Minecraft сервера)  
2) Требования к целевой машине (OS, RAM, Disk, открытые порты)  
3) Инструкция:
   - как настроить домены и DNS (`panel.arcdraco.com` и `arcdraco.com`)  
   - как задать переменные в `group_vars/all.yml`  
   - как заводить secrets в GitHub Actions  
   - как запустить деплой локально и через GitHub Actions  
4) Описание ручных шагов:
   - создание первого админа в панели  
   - создание ноды и серверов Minecraft (если это не автоматизировано)
