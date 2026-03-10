# Настройка VM для синхронизации с NAS

Инструкция по настройке автоматической синхронизации с seedbox на локальный NAS.

## Обзор

Система синхронизации:
1. Запускается каждые 30 минут на VM или выделенном Linux-сервере
2. Синхронизирует завершённые загрузки с seedbox через rsync
3. Сортирует файлы в папки Фильмы/Сериалы
4. Уведомляет бота, когда файлы готовы
5. Очищает seedbox

## Требования

- Linux-сервер или VM с доступом к NAS (через SMB/NFS)
- Аккаунт seedbox с SSH-доступом (см. [SEEDBOX_SETUP.md](SEEDBOX_SETUP.md))
- SSH-доступ к seedbox

## 1. Подготовка VM / сервера

Если ваш NAS поддерживает запуск VM (например, Synology, QNAP, или роутер с поддержкой виртуализации):

1. Создай VM с Debian 12 или Ubuntu 22.04
2. Рекомендуемые параметры:
   - **RAM**: 1 ГБ
   - **Диск**: 16 ГБ
   - **Сеть**: Bridge mode
   - Доступ к хранилищу NAS

Альтернативно: используй любой Linux-сервер в локальной сети с доступом к NAS.

## 2. Начальная настройка

Подключись к VM по SSH.

### Установка зависимостей

```bash
sudo apt update
sudo apt install -y rsync sshpass curl jq cron
```

### Создание пользователя и директорий

```bash
# Создаём отдельного пользователя
sudo useradd -m -s /bin/bash syncuser

# Создаём директории для синхронизации
sudo mkdir -p /home/syncuser/sync/logs
sudo chown -R syncuser:syncuser /home/syncuser
```

### Монтирование хранилища NAS

Убедись, что NAS примонтирован по пути типа `/mnt/nas/media/` или аналогичному.
Проверь:

```bash
ls /mnt/nas/media/
```

Если не примонтирован, добавь в `/etc/fstab` (пример для SMB/CIFS):
```
//NAS_IP_OR_HOSTNAME/share /mnt/nas/media cifs credentials=/home/syncuser/.smbcredentials,uid=syncuser,gid=syncuser 0 0
```

Для NFS:
```
NAS_IP:/volume/share /mnt/nas/media nfs defaults 0 0
```

## 3. Установка скриптов синхронизации

### Копирование скриптов

```bash
# Под пользователем syncuser
sudo -u syncuser -i

# Создаём структуру директорий
mkdir -p ~/sync/logs

# Скачиваем скрипты (или копируем из репозитория)
# Вариант 1: Клонируем репозиторий
git clone https://github.com/shipaleks/media-concierge-bot.git /tmp/bot
cp /tmp/bot/scripts/sync_seedbox.sh ~/sync/
cp /tmp/bot/scripts/config.env.template ~/sync/config.env

# Вариант 2: Создаём вручную (копируем содержимое из репозитория)
```

### Настройка credentials

```bash
# Редактируем конфиг
nano ~/sync/config.env

# Защищаем файл
chmod 600 ~/sync/config.env
```

Заполни:
- `SEEDBOX_HOST`: Твой сервер seedbox (например, `john.sb01.usbx.me`)
- `SEEDBOX_USER`: Твой логин
- `SEEDBOX_PASS`: Твой пароль
- `SEEDBOX_PATH`: Путь к завершённым загрузкам (обычно `/home/USERNAME/Downloads/completed`)
- `NAS_MOVIES`: Локальная папка для фильмов
- `NAS_TV`: Локальная папка для сериалов
- `BOT_API_URL`: URL бота на Koyeb
- `SYNC_API_KEY`: API-ключ для уведомлений

### Делаем скрипт исполняемым

```bash
chmod +x ~/sync/sync_seedbox.sh
```

## 4. Тестирование скрипта

Сначала запусти вручную:

```bash
~/sync/sync_seedbox.sh
```

Проверь лог:
```bash
tail -f ~/sync/logs/sync.log
```

## 5. Настройка Cron

```bash
# Редактируем crontab
crontab -e

# Добавляем строку (запуск каждые 30 минут):
*/30 * * * * /home/syncuser/sync/sync_seedbox.sh >> /home/syncuser/sync/logs/cron.log 2>&1
```

## 6. Настройка API-ключа бота

В деплое на Koyeb добавь переменную окружения:

```
SYNC_API_KEY=твой_секретный_ключ
```

Используй тот же ключ в `config.env`.

## Структура папок

После настройки NAS будет организован так:

```
/mnt/nas/media/Фильмы и сериалы/
├── Кино/
│   ├── Movie.Name.2024.2160p.BluRay.mkv
│   └── Another.Movie.1080p.WEB-DL.mkv
└── Сериалы/
    ├── Series.Name.S01E01.720p.WEB.mkv
    └── Other.Show.S02E05.1080p.mkv
```

## Решение проблем

### Скрипт падает с «Permission denied»
```bash
chmod +x ~/sync/sync_seedbox.sh
chmod 600 ~/sync/config.env
```

### «Host key verification failed»
Сначала подключись по SSH вручную, чтобы принять ключ хоста:
```bash
ssh USERNAME@SERVERNAME
# Введи 'yes' для подтверждения
```

### Rsync зависает
Проверь подключение к seedbox:
```bash
ping SEEDBOX_HOST
ssh USERNAME@SEEDBOX_HOST "ls"
```

### Файлы сортируются неправильно
Скрипт определяет сериалы по паттернам типа `S01E01`. Если файлы названы иначе, они попадут в Фильмы по умолчанию.

### Нет уведомлений
1. Проверь, что `SYNC_API_KEY` совпадает в config.env и в Koyeb
2. Проверь правильность URL бота
3. Посмотри ошибки curl в sync.log

## Обслуживание

### Просмотр логов
```bash
tail -100 ~/sync/logs/sync.log
```

### Очистка старых логов
```bash
find ~/sync/logs -name "*.log" -mtime +30 -delete
```

### Проверка свободного места
```bash
df -h /mnt/nas/media/
```
