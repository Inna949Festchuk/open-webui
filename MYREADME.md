# Инструкция по переносу Open WebUI с настройками между компьютерами через Docker

## Содержание
- [Инструкция по переносу Open WebUI с настройками между компьютерами через Docker](#инструкция-по-переносу-open-webui-с-настройками-между-компьютерами-через-docker)
  - [Содержание](#содержание)
  - [Введение](#введение)
  - [Часть 1: Резервное копирование данных со старого компьютера](#часть-1-резервное-копирование-данных-со-старого-компьютера)
    - [Шаг 1: Установка и запуск Open WebUI](#шаг-1-установка-и-запуск-open-webui)
    - [Шаг 2: Поиск расположения тома Docker](#шаг-2-поиск-расположения-тома-docker)
    - [Шаг 3: Копирование ВСЕХ данных тома](#шаг-3-копирование-всех-данных-тома)
      - [Для Linux и macOS](#для-linux-и-macos)
      - [Для Windows (PowerShell от имени администратора)](#для-windows-powershell-от-имени-администратора)
    - [Шаг 4: Перенос архива на новый компьютер](#шаг-4-перенос-архива-на-новый-компьютер)
  - [Часть 2: Восстановление данных на новом компьютере](#часть-2-восстановление-данных-на-новом-компьютере)
    - [Шаг 1: Создание Docker-тома](#шаг-1-создание-docker-тома)
    - [Шаг 2: Восстановление ВСЕХ данных в том](#шаг-2-восстановление-всех-данных-в-том)
      - [Для Linux и macOS](#для-linux-и-macos-1)
      - [Для Windows (PowerShell от имени администратора)](#для-windows-powershell-от-имени-администратора-1)
    - [Шаг 3: Запуск контейнера](#шаг-3-запуск-контейнера)
    - [Шаг 4: Проверка работоспособности](#шаг-4-проверка-работоспособности)
  - [Альтернативный метод через временный контейнер (рекомендуемый)](#альтернативный-метод-через-временный-контейнер-рекомендуемый)
    - [На старом компьютере (создание полного архива)](#на-старом-компьютере-создание-полного-архива)
    - [На новом компьютере (восстановление из архива)](#на-новом-компьютере-восстановление-из-архива)
  - [Часть 3: Подготовка к поставке заказчику (быстрое развёртывание)](#часть-3-подготовка-к-поставке-заказчику-быстрое-развёртывание)
    - [Создание деплоймент-репозитория](#создание-деплоймент-репозитория)
    - [Структура проекта для заказчика](#структура-проекта-для-заказчика)
    - [Файл docker-compose.yml](#файл-docker-composeyml)
    - [Файл .env](#файл-env)
    - [Скрипты автоматизации](#скрипты-автоматизации)
      - [scripts/backup.sh (для создания полного бэкапа)](#scriptsbackupsh-для-создания-полного-бэкапа)
      - [scripts/restore.sh (для быстрого восстановления у заказчика)](#scriptsrestoresh-для-быстрого-восстановления-у-заказчика)
      - [Makefile (для удобства)](#makefile-для-удобства)
    - [Файл .gitignore](#файл-gitignore)
    - [Передача и запуск у заказчика](#передача-и-запуск-у-заказчика)
    - [Возможности доработки без изменения кода](#возможности-доработки-без-изменения-кода)
  - [Решение возможных проблем](#решение-возможных-проблем)
    - [1. Контейнер запускается, но настройки сброшены или нет файлов](#1-контейнер-запускается-но-настройки-сброшены-или-нет-файлов)
    - [2. Ошибка "Permission denied" в логах контейнера](#2-ошибка-permission-denied-в-логах-контейнера)
    - [3. Ошибка миграции базы данных (no such column: user.username)](#3-ошибка-миграции-базы-данных-no-such-column-userusername)
    - [4. В Windows не удается получить доступ к пути \\wsl$\\docker-desktop](#4-в-windows-не-удается-получить-доступ-к-пути-wsldocker-desktop)
    - [5. Порт 3000 уже занят на новом компьютере](#5-порт-3000-уже-занят-на-новом-компьютере)
    - [6. Контейнер с именем "open-webui" уже существует](#6-контейнер-с-именем-open-webui-уже-существует)
  - [Примечания](#примечания)
  - [Контакты для поддержки](#контакты-для-поддержки)

---

## Введение

Данная инструкция описывает процесс переноса **всех данных** Open WebUI с одного компьютера на другой с использованием Docker.

**Важно:** Open WebUI хранит не только базу данных `webui.db`, но и другие критически важные файлы в директории данных:

| Папка/Файл | Назначение |
| :--- | :--- |
| `webui.db` | База данных SQLite (пользователи, чаты, API-ключи, настройки) |
| `uploads/` | Загруженные пользователями файлы (изображения, документы) |
| `cache/` | Кэш моделей эмбеддингов (например, `all-MiniLM-L6-v2`) |
| `vector_db/` | Векторная база данных для RAG (Retrieval-Augmented Generation) |

**Потеря любой из этих папок приведёт к неполной работе системы** (пропадут загруженные файлы, потребуется перезагрузка моделей, исчезнут проиндексированные документы). Поэтому мы копируем **всю директорию целиком**.

---

## Часть 1: Резервное копирование данных со старого компьютера

### Шаг 1: Установка и запуск Open WebUI

Если Open WebUI еще не установлен, выполните следующую команду:

```bash
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

Если контейнер уже существует, но остановлен, запустите его:

```bash
docker start open-webui
```

**Проверка работы:** Откройте браузер и перейдите по адресу `http://localhost:3000`

### Шаг 2: Поиск расположения тома Docker

Выполните команду для получения информации о томе `open-webui`:

```bash
docker volume inspect open-webui
```

В выводе найдите поле `"Mountpoint"`. Пример вывода:

```json
[
    {
        "CreatedAt": "2024-01-15T10:30:00+03:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/open-webui/_data",
        "Name": "open-webui",
        "Options": {},
        "Scope": "local"
    }
]
```

Запомните путь из поля `"Mountpoint"`:
- **Linux:** `/var/lib/docker/volumes/open-webui/_data`
- **Windows/macOS:** Путь внутри виртуальной машины Docker

### Шаг 3: Копирование ВСЕХ данных тома

#### Для Linux и macOS

```bash
# Создайте папку для резервной копии
mkdir -p ~/Desktop/open-webui-full-backup

# Скопируйте ВСЁ содержимое тома
sudo cp -r $(docker volume inspect open-webui --format '{{ .Mountpoint }}')/. ~/Desktop/open-webui-full-backup/

# Исправьте права для удобства работы
sudo chown -R $USER:$USER ~/Desktop/open-webui-full-backup/

# Проверьте, что все папки скопировались (должны быть: webui.db, uploads/, cache/, vector_db/)
ls -la ~/Desktop/open-webui-full-backup/
```

#### Для Windows (PowerShell от имени администратора)

```powershell
# Создайте папку для резервной копии
$backupPath = "$env:USERPROFILE\Desktop\open-webui-full-backup"
New-Item -ItemType Directory -Path $backupPath -Force

# Получите путь к тому и скопируйте ВСЁ содержимое
$volumePath = docker volume inspect open-webui --format '{{ .Mountpoint }}'
$volumePath = $volumePath -replace '/', '\'
Copy-Item "\\wsl$\docker-desktop$volumePath\*" -Destination $backupPath -Recurse

# Проверьте, что всё скопировалось
Get-ChildItem $backupPath
```

### Шаг 4: Перенос архива на новый компьютер

**Рекомендуется создать архив** для удобства переноса:

```bash
# На Linux/macOS
cd ~/Desktop
tar -czf open-webui-full-backup.tar.gz open-webui-full-backup/

# На Windows (PowerShell)
Compress-Archive -Path "$env:USERPROFILE\Desktop\open-webui-full-backup\*" -DestinationPath "$env:USERPROFILE\Desktop\open-webui-full-backup.zip"
```

Скопируйте полученный архив на новый компьютер любым удобным способом:
- USB-флешка
- Облачное хранилище (Google Drive, Яндекс.Диск, Dropbox)
- Сетевая папка

---

## Часть 2: Восстановление данных на новом компьютере

### Шаг 1: Создание Docker-тома

На новом компьютере создайте том с именем `open-webui`:

```bash
docker volume create open-webui
```

Проверьте, что том создан:

```bash
docker volume ls | grep open-webui
```

### Шаг 2: Восстановление ВСЕХ данных в том

#### Для Linux и macOS

```bash
# Распакуйте архив (если запаковывали)
cd ~/Desktop
tar -xzf open-webui-full-backup.tar.gz

# Перейдите в папку с данными
cd ~/Desktop/open-webui-full-backup

# Скопируйте ВСЁ содержимое в том Docker
sudo cp -r * $(docker volume inspect open-webui --format '{{ .Mountpoint }}')/

# Установите правильные права доступа (Open WebUI работает от пользователя с UID 1000)
sudo chown -R 1000:1000 $(docker volume inspect open-webui --format '{{ .Mountpoint }}')/

# Проверьте, что всё на месте
sudo ls -la $(docker volume inspect open-webui --format '{{ .Mountpoint }}')/
```

#### Для Windows (PowerShell от имени администратора)

```powershell
# Распакуйте архив (если запаковывали)
Expand-Archive -Path "$env:USERPROFILE\Desktop\open-webui-full-backup.zip" -DestinationPath "$env:USERPROFILE\Desktop\open-webui-full-backup"

# Перейдите в папку с данными
cd "$env:USERPROFILE\Desktop\open-webui-full-backup"

# Получите путь к тому Docker
$volumePath = docker volume inspect open-webui --format '{{ .Mountpoint }}'
$volumePath = $volumePath -replace '/', '\'

# Скопируйте ВСЁ содержимое в том
Copy-Item "*" -Destination "\\wsl$\docker-desktop$volumePath\" -Recurse

# Проверьте, что всё скопировалось
Get-ChildItem "\\wsl$\docker-desktop$volumePath"
```

### Шаг 3: Запуск контейнера

Запустите Open WebUI с примонтированным томом:

```bash
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

### Шаг 4: Проверка работоспособности

1. Откройте браузер и перейдите по адресу: `http://localhost:3000`
2. Войдите в систему, используя свои учетные данные со старого компьютера
3. Проверьте наличие:
   - Истории чатов (включая прикреплённые файлы)
   - Настроенных моделей
   - API-ключей
   - Пользовательских настроек интерфейса
   - Загруженных ранее файлов в чатах

Если все данные на месте — перенос выполнен успешно.

---

## Альтернативный метод через временный контейнер (рекомендуемый)

Этот метод **не требует поиска Mountpoint, не требует прав sudo** и работает одинаково на всех операционных системах. Это самый надёжный способ.

### На старом компьютере (создание полного архива)

```bash
# Создайте папку для бэкапа
mkdir -p ~/open-webui-full-backup

# Создайте полный архив всего содержимого тома
docker run --rm \
    -v open-webui:/volume:ro \
    -v $(pwd)/open-webui-full-backup:/backup \
    alpine \
    tar czf /backup/open-webui-full.tar.gz -C /volume .

# Проверьте, что архив создался
ls -la ~/open-webui-full-backup/
```

**Для Windows (PowerShell):**

```powershell
# Создайте папку для бэкапа
New-Item -ItemType Directory -Path "$env:USERPROFILE\open-webui-full-backup" -Force

# Создайте архив
docker run --rm `
    -v open-webui:/volume:ro `
    -v "$env:USERPROFILE\open-webui-full-backup:/backup" `
    alpine `
    tar czf /backup/open-webui-full.tar.gz -C /volume .
```

### На новом компьютере (восстановление из архива)

```bash
# Перейдите в папку с архивом
cd ~/open-webui-full-backup

# Создайте том
docker volume create open-webui

# Распакуйте архив в том
docker run --rm \
    -v open-webui:/volume \
    -v $(pwd):/backup \
    alpine \
    sh -c "tar xzf /backup/open-webui-full.tar.gz -C /volume/ && chown -R 1000:1000 /volume/"

# Запустите контейнер
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

**Для Windows (PowerShell):**

```powershell
# Перейдите в папку с архивом
cd "$env:USERPROFILE\open-webui-full-backup"

# Создайте том
docker volume create open-webui

# Распакуйте архив в том
docker run --rm `
    -v open-webui:/volume `
    -v "${PWD}:/backup" `
    alpine `
    sh -c "tar xzf /backup/open-webui-full.tar.gz -C /volume/ && chown -R 1000:1000 /volume/"

# Запустите контейнер
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

---

## Часть 3: Подготовка к поставке заказчику (быстрое развёртывание)

После того как вы успешно перенесли и проверили Open WebUI на новом компьютере, можно подготовить конфигурацию для передачи заказчику. Это позволит ему запустить полностью настроенную систему одной командой, не вникая в детали.

### Создание деплоймент-репозитория

Создайте отдельный Git-репозиторий (например, `open-webui-deploy`), который будет содержать только файлы, необходимые для запуска. **Форк исходного репозитория Open WebUI не требуется**.

### Структура проекта для заказчика

```
open-webui-deploy/
├── docker-compose.yml      # конфигурация Docker Compose
├── .env                    # переменные окружения
├── .gitignore              # исключаем data/ и архивы из Git
├── data/                   # папка с данными (создаётся при восстановлении)
│   ├── webui.db
│   ├── uploads/
│   ├── cache/
│   └── vector_db/
├── backups/                # папка для хранения архивов бэкапов
│   └── open-webui-full-20260414_120000.tar.gz
├── scripts/
│   ├── backup.sh           # скрипт для создания полного бэкапа
│   └── restore.sh          # скрипт для восстановления из архива
├── Makefile                # команды для удобства
└── README.md               # инструкция для заказчика
```

### Файл docker-compose.yml

```yaml
version: '3.8'

services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:v0.8.6  # Рекомендуется использовать стабильную версию!
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "${WEBUI_PORT:-3000}:8080"
    volumes:
      # Монтируем локальную папку data со ВСЕМИ данными
      - ./data:/app/backend/data
    environment:
      - WEBUI_NAME=${WEBUI_NAME:-Open WebUI}
      - WEBUI_URL=${WEBUI_URL:-http://localhost:3000}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

> **Важно:** Рекомендуется использовать конкретную стабильную версию образа (например, `v0.8.6`), а не `:main`. Тег `:main` может содержать несовместимые изменения схемы базы данных. Если вы используете `:main`, убедитесь, что у заказчика есть свежий бэкап перед обновлением.

### Файл .env

```env
# Порт для доступа к веб-интерфейсу
WEBUI_PORT=3000

# Название вашего инстанса (отображается в интерфейсе)
WEBUI_NAME=My Company AI

# URL для доступа (можно изменить при использовании домена)
WEBUI_URL=http://localhost:3000
```

### Скрипты автоматизации

#### scripts/backup.sh (для создания полного бэкапа)

```bash
#!/bin/bash
# Полное резервное копирование Open WebUI (включая uploads, cache, vector_db)

set -e

BACKUP_DIR="./backups"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_NAME="open-webui-full-${TIMESTAMP}"

GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

echo -e "${YELLOW}🔄 Создание полной резервной копии Open WebUI...${NC}"

mkdir -p "${BACKUP_DIR}"

# Проверяем, существует ли том open-webui
if ! docker volume inspect open-webui >/dev/null 2>&1; then
    echo -e "${RED}❌ Том open-webui не найден!${NC}"
    echo -e "${YELLOW}💡 Сначала запустите Open WebUI для создания тома.${NC}"
    exit 1
fi

# Создаём архив всего содержимого тома
docker run --rm \
    -v open-webui:/volume:ro \
    -v "$(pwd)/${BACKUP_DIR}":/backup \
    alpine \
    tar czf "/backup/${BACKUP_NAME}.tar.gz" -C /volume .

echo -e "${GREEN}✅ Полный бэкап создан: ${BACKUP_DIR}/${BACKUP_NAME}.tar.gz${NC}"
echo -e "${GREEN}📦 Размер архива: $(du -h ${BACKUP_DIR}/${BACKUP_NAME}.tar.gz | cut -f1)${NC}"
```

#### scripts/restore.sh (для быстрого восстановления у заказчика)

```bash
#!/bin/bash
# Полное восстановление Open WebUI из архива

set -e

GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
RED='\033[0;31m'
NC='\033[0m'

echo -e "${BLUE}🔄 Восстановление полной резервной копии Open WebUI...${NC}"

# Ищем последний архив в папке backups
ARCHIVE=$(ls -t ./backups/*.tar.gz 2>/dev/null | head -n1)

# Если архива нет в backups, ищем в корне
if [ -z "$ARCHIVE" ] && [ -f ./open-webui-full.tar.gz ]; then
    ARCHIVE="./open-webui-full.tar.gz"
fi

if [ -z "$ARCHIVE" ]; then
    echo -e "${RED}❌ Архив не найден!${NC}"
    echo -e "${YELLOW}💡 Поместите архив .tar.gz в папку backups/ или в корень проекта.${NC}"
    echo -e "   Или запустите: $0 /путь/к/бэкапу.tar.gz"
    exit 1
fi

echo -e "${GREEN}📁 Найден архив: ${ARCHIVE}${NC}"

# Создаём папку data
mkdir -p ./data

# Очищаем папку data перед восстановлением (с подтверждением)
if [ "$(ls -A ./data 2>/dev/null)" ]; then
    echo -e "${YELLOW}⚠️  Папка ./data не пуста.${NC}"
    read -p "Очистить папку перед восстановлением? [y/N] " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        rm -rf ./data/*
        echo -e "${YELLOW}🧹 Папка ./data очищена.${NC}"
    fi
fi

# Распаковываем архив
echo -e "${YELLOW}📦 Распаковка данных...${NC}"
tar xzf "$ARCHIVE" -C ./data/

# Устанавливаем правильные права (для Linux/macOS)
if [[ "$OSTYPE" == "linux-gnu"* ]] || [[ "$OSTYPE" == "darwin"* ]]; then
    chmod -R 755 ./data/
    echo -e "${GREEN}✅ Права доступа установлены${NC}"
fi

echo -e "${GREEN}✅ Данные восстановлены в ./data/${NC}"
echo -e "${BLUE}🚀 Запуск Open WebUI...${NC}"

docker-compose up -d

echo -e "${GREEN}✨ Open WebUI запущен! Доступен по адресу: http://localhost:3000${NC}"
```

Не забудьте сделать скрипты исполняемыми: `chmod +x scripts/*.sh`

#### Makefile (для удобства)

```makefile
.PHONY: up down logs backup restore clean

up:     ## Запустить сервис
	docker-compose up -d
	@echo "✅ Open WebUI запущен: http://localhost:3000"

down:   ## Остановить сервис
	docker-compose down

logs:   ## Показать логи
	docker-compose logs -f

backup: ## Создать полный бэкап (архив .tar.gz)
	@./scripts/backup.sh

restore: ## Восстановить из последнего бэкапа и запустить
	@./scripts/restore.sh

clean:  ## Очистить все данные (ТРЕБУЕТ ПОДТВЕРЖДЕНИЯ)
	@echo "⚠️  ВНИМАНИЕ: Это удалит ВСЕ данные Open WebUI!"
	@read -p "Вы уверены? [y/N] " -n 1 -r; \
	echo ""; \
	if [[ $$REPLY =~ ^[Yy]$$ ]]; then \
		docker-compose down -v 2>/dev/null || true; \
		rm -rf ./data/*; \
		echo "✅ Данные удалены"; \
	else \
		echo "❌ Операция отменена"; \
	fi
```

### Файл .gitignore

```
# Данные (не храним в Git)
data/
*.db

# Архивы с бэкапами
backups/*.tar.gz
backups/*.db
*.tar.gz
*.zip

# Системные файлы
.DS_Store
.env.local
```

### Передача и запуск у заказчика

1. **Заказчик клонирует репозиторий**:
   ```bash
   git clone <url-репозитория>
   cd open-webui-deploy
   ```

2. **Помещает архив с данными** (если он не был включён в репозиторий) в папку `backups/`:
   ```bash
   mkdir -p backups
   cp /путь/к/open-webui-full-*.tar.gz ./backups/
   ```

3. **Запускает восстановление и сервис одной командой**:
   ```bash
   make restore
   ```
   или вручную:
   ```bash
   chmod +x scripts/*.sh
   ./scripts/restore.sh
   ```

4. **Открывает браузер** по адресу `http://localhost:3000` и видит полностью настроенную систему со всеми данными.

### Возможности доработки без изменения кода

Заказчик может кастомизировать Open WebUI **не трогая исходный код**, используя встроенные возможности:

- **Подключение новых моделей**: Admin Panel → Settings → Connections (поддерживаются Ollama, OpenAI API, LiteLLM, Groq и другие).
- **Настройка интерфейса**: Темы, промпты по умолчанию, создание кастомных инструментов (Tools) и функций (Functions).
- **Управление пользователями**: Создание учётных записей, распределение прав доступа.

Если потребуется модификация исходного кода, заказчик может:
- Сделать форк оригинального репозитория `open-webui/open-webui`.
- В `docker-compose.yml` заменить строку `image: ...` на `build: /путь/к/форку`.
- Пересобрать контейнер.

Таким образом, деплоймент-репозиторий не ограничивает дальнейшее развитие системы.

---

## Решение возможных проблем

### 1. Контейнер запускается, но настройки сброшены или нет файлов

**Причина:** Данные не были скопированы полностью или имеют неправильные права.

**Решение:**
```bash
# Проверьте содержимое тома
docker run --rm -v open-webui:/volume alpine ls -la /volume/

# Должны быть видны: webui.db, uploads/, cache/, vector_db/
# Если чего-то нет — повторите копирование ВСЕХ данных.

# Исправьте права при необходимости
docker run --rm -v open-webui:/volume alpine chown -R 1000:1000 /volume/
```

### 2. Ошибка "Permission denied" в логах контейнера

**Просмотр логов:**
```bash
docker logs open-webui
```

**Решение:**
```bash
# Остановите и удалите контейнер
docker stop open-webui
docker rm open-webui

# Исправьте права на все файлы
docker run --rm -v open-webui:/volume alpine chown -R 1000:1000 /volume/

# Запустите контейнер заново
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

### 3. Ошибка миграции базы данных (no such column: user.username)

**Причина:** Несоответствие схемы базы данных версии приложения. Ваша база данных (`webui.db`) была создана старой версией Open WebUI, а вы пытаетесь запустить новую версию с изменённой структурой таблиц.

**Решение 1 (простое): Используйте стабильную версию образа**
```bash
docker stop open-webui
docker rm open-webui
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:v0.8.6
```

**Решение 2 (сложное): Ручная миграция базы данных**
```bash
# Запустите временный контейнер
docker run --rm -it \
    -v open-webui:/app/backend/data \
    --entrypoint /bin/bash \
    ghcr.io/open-webui/open-webui:main

# Внутри контейнера выполните:
cd /app/backend/open_webui
export DATABASE_URL="sqlite:////app/backend/data/webui.db"
export WEBUI_SECRET_KEY=$(cat /app/backend/data/.webui_secret_key 2>/dev/null || cat /app/backend/.webui_secret_key 2>/dev/null || echo "temp-key")
alembic upgrade head

# Выйдите из контейнера (exit) и запустите основной контейнер
```

### 4. В Windows не удается получить доступ к пути \\wsl$\docker-desktop

**Решение:** Используйте **Альтернативный метод через временный контейнер** (см. выше). Он полностью обходит эту проблему.

### 5. Порт 3000 уже занят на новом компьютере

**Решение:** Используйте другой порт, например 3001:
```bash
docker run -d -p 3001:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```
Затем открывайте `http://localhost:3001`

### 6. Контейнер с именем "open-webui" уже существует

**Решение:**
```bash
# Остановите и удалите существующий контейнер
docker stop open-webui 2>/dev/null || true
docker rm open-webui 2>/dev/null || true

# Запустите новый контейнер
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

---

## Примечания

- **Регулярное резервное копирование:** Рекомендуется периодически создавать полные резервные копии (архив `.tar.gz`), особенно перед обновлением Open WebUI.
- **Размер данных:** Папка с данными может достигать нескольких гигабайт из-за кэша моделей и векторной базы данных. Учитывайте это при переносе.
- **Версии Open WebUI:** Для production-использования рекомендуется указывать конкретную стабильную версию образа (например, `v0.8.6`), а не `:main`.
- **Безопасность:** Файл `webui.db` содержит API-ключи в открытом виде. Храните резервные копии в безопасном месте.
- **Полнота данных:** Всегда копируйте **всю директорию данных**, а не только `webui.db`. Иначе вы потеряете загруженные файлы, кэш моделей и векторную базу знаний.

---

## Контакты для поддержки

При возникновении вопросов или проблем обращайтесь к администратору системы или создавайте issue в репозитории Open WebUI на GitHub: https://github.com/open-webui/open-webui/issues

---

**Версия инструкции:** 2.0  
**Дата последнего обновления:** 2026-04-14
```

## Дополнительные заметки

```
curl -X GET https://api.gpt.mws.ru/v1/models \
 -H "Authorization: Bearer API-KEY"
```

## Подключение моделей 

### 🖼 Мультимодальные модели (Изображения и Видео)
- gemma-3-27b-it: Новое поколение от Google, понимает текст и изображения.
- qwen2.5-vl / qwen2.5-vl-72b: Специализированные модели от Alibaba для глубокого анализа изображений и видео.
- qwen3-vl-30b-a3b-instruct: Свежая версия Qwen для работы с визуальным контентом.
- cotype-pro-vl-32b: Еще одна модель с поддержкой зрения (Vision-Language).
- qwen-image / qwen-image-lightning: Модели, оптимизированные исключительно под быструю обработку картинок.

### 🧠 Мощные текстовые модели (LLM)
- deepseek-r1-distill-qwen-32b: Модель-рассуждатель (Reasoning), дистиллированная из DeepSeek для логических задач.
- QwQ-32B: Экспериментальная модель от Qwen, ориентированная на сложные рассуждения (аналог OpenAI o1).
- llama-3.3-70b-instruct / llama-3.1-8b: Актуальные флагманы от Meta, универсальные и надежные.
- qwen2.5-72b / qwen3-32b / qwen3-235B: Семейство моделей Alibaba, лидеры в тестах на кодинг и математику.
- qwen3-coder-480b-a35b: Огромная специализированная модель для написания сложного кода.
- kimi-k2-instruct: Китайская модель с огромным контекстным окном для анализа длинных текстов.
- glm-4.6-357b: Мощная универсальная модель от Zhipu AI.
- gpt-oss (20b/120b): Open-source интерпретации архитектур типа GPT.

### 🔊 Работа со звуком (Аудио)
- whisper-medium / whisper-turbo-local: Мировые стандарты для перевода речи в текст (транскрибация) от OpenAI.

### 🔍 Эмбеддинги (Поиск и RAG)
- BAAI/bge-multilingual-gemma2: Модель для создания векторных представлений текста на многих языках.
- bge-m3: Универсальный инструмент для семантического поиска.
- qwen3-embedding-8b: Свежая модель эмбеддингов от команды Qwen.

## Создание Tools

# Проверьте, что инструмент загрузился:

```
curl -s http://localhost:3000/api/v1/tools/ \
  -H "Authorization: Bearer APIKEY" | grep web_search
```

# Удаление и Запуск

```
docker stop open-webui

docker rm - f open-webui

docker run -d -p 3000:8080 \                                                                                  
  -e RAG_EMBEDDING_MODEL_ID="" \
  -e ENABLE_RAG_LOCAL_WEB_FETCH=False \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  ghcr.io/open-webui/open-webui:main

```
