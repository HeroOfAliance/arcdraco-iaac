# ArcDraco IaC
- Создайте ноду и сервера Minecraft через панель (или автоматизируйте через API)
- Создайте первого администратора в панели
## Ручные шаги

- `.github/workflows` — CI/CD
- `infra/ansible` — роли, плейбуки, переменные
## Структура

   ```
   ansible-playbook playbooks/site.yml -i inventories/production/hosts.ini
   ```bash
4. Запустите деплой:
3. Добавьте секреты (db_password и др.) через Ansible Vault или GitHub Secrets
2. Задайте переменные в `infra/ansible/group_vars/all.yml`
1. Настройте DNS для panel.arcdraco.com и arcdraco.com
## Быстрый старт

- Открытые порты: 22, 80, 443, 25565–25600
- Сеть: 1 Гбит/с
- SSD NVMe: 500+ ГБ
- RAM: 64–128 ГБ
- CPU: 12–16 ядер
- Ubuntu 22.04
## Требования к серверу

- **Minecraft серверы** (через Wings/Docker)
- **Pterodactyl Wings** (Docker)
- **Pterodactyl Panel** (PHP, MariaDB, Redis, Nginx, Certbot)
## Архитектура

Инфраструктура как код для автоматического развертывания Pterodactyl Panel, Wings и Minecraft серверов на одной машине.


