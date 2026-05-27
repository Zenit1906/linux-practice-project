# Linux Server Administration — Учебный проект

Практика по развёртыванию и администрированию Linux-сервера с контейнеризацией и базовым мониторингом.
Продолжительность: 2 недели (10 рабочих дней). Все задачи выполнялись самостоятельно на виртуальной машине.

---

## Окружение

| Параметр | Значение |
|---|---|
| ОС хоста | Windows 10 |
| Гипервизор | VirtualBox |
| ОС гостя | Ubuntu Server 22.04 LTS |
| RAM / HDD VM | 2 ГБ / 20 ГБ |
| Веб-сервер | Nginx + Apache2 (оба проверены) |
| База данных | PostgreSQL 15 (Docker-контейнер) |
| Контейнеризация | Docker CE |
| Мониторинг | Prometheus + Grafana + node_exporter |
| Контроль версий | Git + GitHub |
| Firewall | UFW |

---

## Структура репозитория

```
project/
├── scripts/
│   ├── backup.sh          # автоматический бэкап /etc и домашних каталогов
│   └── site_check.sh      # проверка доступности Nginx и PostgreSQL
├── screenshots/           # скриншоты по каждому дню практики
├── README.md
└── .gitignore
```

---

## Скрипты

### backup.sh

Копирует `/etc` и домашние каталоги в `/backup/$(date +%F)`. Запускается автоматически каждый день в 02:00 через cron.

```bash
#!/bin/bash
DATE=$(date +%F)
BACKUP_DIR="/backup/$DATE"

mkdir -p "$BACKUP_DIR"
cp -r /etc "$BACKUP_DIR/etc"
cp -r /home "$BACKUP_DIR/home"

echo "[$DATE] Бэкап завершён: $BACKUP_DIR"
```

### site_check.sh

Проверяет, отвечает ли Nginx на `http://localhost` и запущен ли контейнер PostgreSQL.

```bash
#!/bin/bash

# Проверка HTTP
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost)

if [ "$HTTP_CODE" == "200" ]; then
    echo "[OK] Nginx доступен (HTTP $HTTP_CODE)"
else
    echo "[FAIL] Nginx недоступен (HTTP $HTTP_CODE)"
fi

# Проверка PostgreSQL
PG_RESULT=$(docker exec -i pg psql -U postgres -c "SELECT 1;" 2>&1)

if echo "$PG_RESULT" | grep -q "1 row"; then
    echo "[OK] PostgreSQL работает нормально"
else
    echo "[FAIL] PostgreSQL не отвечает"
    echo "$PG_RESULT"
fi
```

---

## Инструкция по развёртыванию

### 1. Подготовка системы

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Пользователи и группы

```bash
sudo adduser student1
sudo adduser student2
sudo adduser admin
sudo usermod -aG sudo admin
sudo groupadd dev
sudo usermod -aG dev student1
sudo chmod 750 /home/student1
sudo chmod 750 /home/student2
```

### 3. Бэкап и cron

```bash
sudo cp scripts/backup.sh /usr/local/bin/backup.sh
sudo chmod +x /usr/local/bin/backup.sh

# Добавить в cron (запустить crontab -e и вставить строку ниже)
# 0 2 * * * /usr/local/bin/backup.sh
```

### 4. Установка Nginx

```bash
sudo apt install nginx -y
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl start nginx

# Проверка
curl -I http://localhost
```

### 5. Установка Apache2 (опционально)

```bash
sudo apt install apache2 -y
sudo apache2ctl configtest
sudo systemctl enable apache2
sudo systemctl start apache2
```

### 6. Установка Docker и запуск PostgreSQL

```bash
# Установка Docker CE
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io

# Запуск PostgreSQL
docker pull postgres
docker run -d \
  --name pg \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  postgres

# Проверка подключения
docker exec -it pg psql -U postgres -c "SELECT version();"
```

### 7. SSH и Firewall (UFW)

```bash
# Генерация ключа
ssh-keygen -t ed25519 -C "student@server"

# Настройка прав
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Firewall
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 5432
sudo ufw enable
sudo ufw status
```

### 8. Мониторинг (Prometheus + Grafana)

```bash
# Prometheus
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  prom/prometheus

# node_exporter
docker run -d \
  --name node_exporter \
  -p 9100:9100 \
  prom/node-exporter

# Grafana
docker run -d \
  --name grafana \
  -p 3000:3000 \
  grafana/grafana
```

Grafana открывается по адресу `http://localhost:3000` (логин: `admin`, пароль: `admin`).
В дашборде настроены метрики: CPU, RAM, диск, сетевой трафик и статус PostgreSQL.

### 9. Git

```bash
git init
echo "backup/" >> .gitignore
echo "*.log" >> .gitignore
git add .
git commit -m "Initial commit: scripts and docs"
git remote add origin <URL>
git push -u origin main
```

### 10. Итоговая проверка и архив

```bash
# Запуск итогового скрипта
chmod +x scripts/site_check.sh
bash scripts/site_check.sh

# Создание архива для наставника
tar -czf project.tar.gz ./project
```

---

## Список основных команд (шпаргалка)

```bash
# Система
sudo apt update && sudo apt upgrade -y

# Пользователи
sudo adduser <name>
sudo usermod -aG sudo <name>
sudo groupadd <group>

# Nginx
sudo nginx -t
sudo systemctl reload nginx
curl -I http://localhost

# Docker
docker ps -a
docker start/stop <name>
docker exec -it pg psql -U postgres

# UFW
sudo ufw status
sudo ufw allow <port>
sudo ufw enable

# Cron
crontab -l
crontab -e

# Git
git status
git add .
git commit -m "message"
git push

# Архив
tar -czf archive.tar.gz ./folder
tar -xzf archive.tar.gz
```

---

## Скриншоты

Все скриншоты хранятся в папке `screenshots/`.

| День | Что показано |
|------|-------------|
| День 2 | Создание пользователей и настройка групп |
| День 3 | Скрипт backup.sh + настройка crontab в nano |
| День 4 | Установка Nginx + настройка виртуального хоста |
| День 5 | Установка Apache2 + проверка работы сервера |
| День 6 | Установка Docker + первый запущенный контейнер + подключение к PostgreSQL |
| День 7 | Генерация SSH-ключа + настройка UFW (Firewall is active) |
| День 8 | Дашборд Grafana + вывод `ip a` |

> Скриншоты по дням 1, 9, 10 добавить вручную после финальной проверки.

---

## Итоги практики

За 10 дней практики я прошёл весь базовый цикл администрирования Linux-сервера:
развернул систему с нуля, настроил пользователей и права доступа, написал скрипт автоматического бэкапа,
поднял веб-сервер на Nginx и Apache2, запустил базу данных в Docker-контейнере,
настроил SSH-доступ с ключами и Firewall, подключил мониторинг через Prometheus и Grafana,
зафиксировал всё в Git и написал итоговый скрипт проверки.

---

*Практика выполнена в рамках учебного плана. Период: 2 недели. Ubuntu Server 22.04 LTS + Docker CE + Nginx + PostgreSQL.*
