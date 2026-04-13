
# Инструкция по переносу Open WebUI с настройками между компьютерами через Docker

## Содержание
1. [Введение](#введение)
2. [Часть 1: Резервное копирование данных со старого компьютера](#часть-1-резервное-копирование-данных-со-старого-компьютера)
   - [Шаг 1: Установка и запуск Open WebUI](#шаг-1-установка-и-запуск-open-webui)
   - [Шаг 2: Поиск расположения тома Docker](#шаг-2-поиск-расположения-тома-docker)
   - [Шаг 3: Копирование файла базы данных](#шаг-3-копирование-файла-базы-данных)
   - [Шаг 4: Перенос файла на новый компьютер](#шаг-4-перенос-файла-на-новый-компьютер)
3. [Часть 2: Восстановление данных на новом компьютере](#часть-2-восстановление-данных-на-новом-компьютере)
   - [Шаг 1: Создание Docker-тома](#шаг-1-создание-docker-тома)
   - [Шаг 2: Копирование базы данных в том](#шаг-2-копирование-базы-данных-в-том)
   - [Шаг 3: Запуск контейнера](#шаг-3-запуск-контейнера)
   - [Шаг 4: Проверка работоспособности](#шаг-4-проверка-работоспособности)
4. [Альтернативный метод через временный контейнер](#альтернативный-метод-через-временный-контейнер)
5. [Решение возможных проблем](#решение-возможных-проблем)

---

## Введение

Данная инструкция описывает процесс переноса всех данных Open WebUI (включая пользователей, историю чатов, API-ключи моделей и настройки интерфейса) с одного компьютера на другой с использованием Docker.

**Ключевой файл:** `webui.db` — база данных SQLite, содержащая все настройки и данные.

---

## Часть 1: Резервное копирование данных со старого компьютера

### Шаг 1: Установка и запуск Open WebUI

Если Open WebUI еще не установлен, выполните следующую команду:

```bash
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main


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

### Шаг 3: Копирование файла базы данных

#### Для Linux и macOS

```bash
# Создайте папку для резервной копии на рабочем столе
mkdir -p ~/Desktop/open-webui-backup

# Скопируйте файл базы данных
sudo cp $(docker volume inspect open-webui --format '{{ .Mountpoint }}')/webui.db ~/Desktop/open-webui-backup/

# Измените владельца файла для удобства работы с ним
sudo chown $USER:$USER ~/Desktop/open-webui-backup/webui.db

# Проверьте, что файл скопирован
ls -la ~/Desktop/open-webui-backup/
```

#### Для Windows (PowerShell от имени администратора)

```powershell
# Создайте папку для резервной копии на рабочем столе
New-Item -ItemType Directory -Path "$env:USERPROFILE\Desktop\open-webui-backup" -Force

# Получите путь к тому и скопируйте файл
$volumePath = docker volume inspect open-webui --format '{{ .Mountpoint }}'
$volumePath = $volumePath -replace '/', '\'
Copy-Item "\\wsl$\docker-desktop$volumePath\webui.db" -Destination "$env:USERPROFILE\Desktop\open-webui-backup\webui.db"

# Проверьте, что файл скопирован
Get-ChildItem "$env:USERPROFILE\Desktop\open-webui-backup"
```

> **Примечание для Windows:** Если путь `\\wsl$\docker-desktop` недоступен, убедитесь что:
> 1. Docker Desktop работает в режиме WSL 2
> 2. Включена подсистема Windows для Linux (WSL)

### Шаг 4: Перенос файла на новый компьютер

Скопируйте файл `webui.db` из папки `open-webui-backup` на новый компьютер любым удобным способом:
- USB-флешка
- Облачное хранилище (Google Drive, Яндекс.Диск, Dropbox)
- Сетевая папка
- Электронная почта (если размер файла позволяет)

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

### Шаг 2: Копирование базы данных в том

#### Для Linux и macOS

```bash
# Перейдите в папку с файлом webui.db (например, на рабочем столе)
cd ~/Desktop/open-webui-backup

# Скопируйте файл в том Docker
sudo cp webui.db $(docker volume inspect open-webui --format '{{ .Mountpoint }}')/

# Установите правильные права доступа (Open WebUI работает от пользователя с UID 1000)
sudo chown 1000:1000 $(docker volume inspect open-webui --format '{{ .Mountpoint }}')/webui.db

# Проверьте, что файл на месте
sudo ls -la $(docker volume inspect open-webui --format '{{ .Mountpoint }}')/
```

#### Для Windows (PowerShell от имени администратора)

```powershell
# Перейдите в папку с вашим файлом webui.db
cd "$env:USERPROFILE\Desktop\open-webui-backup"

# Получите путь к тому Docker
$volumePath = docker volume inspect open-webui --format '{{ .Mountpoint }}'
$volumePath = $volumePath -replace '/', '\'

# Скопируйте файл в том
Copy-Item "webui.db" -Destination "\\wsl$\docker-desktop$volumePath\webui.db"

# Проверьте, что файл скопирован
Get-ChildItem "\\wsl$\docker-desktop$volumePath"
```

### Шаг 3: Запуск контейнера

Запустите Open WebUI с примонтированным томом:

```bash
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

**Параметры команды:**
- `-d` — запуск в фоновом режиме
- `-p 3000:8080` — проброс порта (локальный порт 3000 на порт 8080 в контейнере)
- `-v open-webui:/app/backend/data` — монтирование тома с базой данных
- `--name open-webui` — имя контейнера
- `ghcr.io/open-webui/open-webui:main` — образ из GitHub Container Registry

### Шаг 4: Проверка работоспособности

1. Откройте браузер и перейдите по адресу: `http://localhost:3000`
2. Войдите в систему, используя свои учетные данные со старого компьютера
3. Проверьте наличие:
   - Истории чатов
   - Настроенных моделей
   - API-ключей
   - Пользовательских настроек интерфейса

Если все данные на месте — перенос выполнен успешно.

---

## Альтернативный метод через временный контейнер

Этот метод не требует поиска Mountpoint и работает одинаково на всех операционных системах.

### На старом компьютере (копирование ИЗ тома)

```bash
# Создайте папку для резервной копии
mkdir -p ~/open-webui-backup

# Скопируйте файл из тома в текущую папку
docker run --rm -v open-webui:/volume -v $(pwd):/backup alpine cp /volume/webui.db /backup/
```

### На новом компьютере (копирование В том)

```bash
# Перейдите в папку с файлом webui.db
cd ~/open-webui-backup

# Создайте том
docker volume create open-webui

# Скопируйте файл из текущей папки в том
docker run --rm -v open-webui:/volume -v $(pwd):/backup alpine cp /backup/webui.db /volume/

# Установите права (если необходимо)
docker run --rm -v open-webui:/volume alpine chown 1000:1000 /volume/webui.db
```

**Для Windows (PowerShell):**

```powershell
# На старом компьютере
docker run --rm -v open-webui:/volume -v ${PWD}:/backup alpine cp /volume/webui.db /backup/

# На новом компьютере
docker volume create open-webui
docker run --rm -v open-webui:/volume -v ${PWD}:/backup alpine cp /backup/webui.db /volume/
```

---

## Решение возможных проблем

### 1. Контейнер запускается, но настройки сброшены

**Причина:** Файл `webui.db` не найден или имеет неправильные права.

**Решение:**
```bash
# Проверьте наличие файла в томе
docker run --rm -v open-webui:/volume alpine ls -la /volume/

# Если файла нет, повторите копирование
# Если файл есть, проверьте права
docker run --rm -v open-webui:/volume alpine ls -la /volume/webui.db

# Исправьте права при необходимости
docker run --rm -v open-webui:/volume alpine chown 1000:1000 /volume/webui.db
```

### 2. Ошибка "Permission denied" в логах контейнера

**Просмотр логов:**
```bash
docker logs open-webui
```

**Решение:**
```bash
# Остановите контейнер
docker stop open-webui
docker rm open-webui

# Исправьте права на файл
docker run --rm -v open-webui:/volume alpine chown -R 1000:1000 /volume/

# Запустите контейнер заново
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

### 3. В Windows не удается получить доступ к пути \\wsl$\docker-desktop

**Решение 1:** Используйте альтернативный метод через временный контейнер (см. выше).

**Решение 2:** Проверьте статус WSL:
```powershell
wsl --status
wsl --list --verbose
```

Если Docker Desktop использует Hyper-V вместо WSL 2:
1. Откройте Docker Desktop
2. Перейдите в Settings → General
3. Убедитесь, что включена опция "Use WSL 2 based engine"
4. Перезапустите Docker Desktop

### 4. Порт 3000 уже занят на новом компьютере

**Решение:** Используйте другой порт, например 3001:
```bash
docker run -d -p 3001:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

Затем открывайте `http://localhost:3001`

### 5. Контейнер с именем "open-webui" уже существует

**Решение:**
```bash
# Остановите и удалите существующий контейнер
docker stop open-webui
docker rm open-webui

# Запустите новый контейнер
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```

---

## Примечания

- **Регулярное резервное копирование:** Рекомендуется периодически создавать резервные копии файла `webui.db`, особенно перед обновлением Open WebUI.
- **Размер базы данных:** Файл `webui.db` может достигать нескольких гигабайт при активном использовании. Учитывайте это при переносе.
- **Версии Open WebUI:** Данный метод работает для всех версий Open WebUI, использующих SQLite в качестве базы данных.
- **Безопасность:** Файл `webui.db` содержит API-ключи в открытом виде. Храните резервные копии в безопасном месте.

---

## Контакты для поддержки

При возникновении вопросов или проблем обращайтесь к администратору системы или создавайте issue в репозитории Open WebUI на GitHub: https://github.com/open-webui/open-webui/issues

---

**Версия инструкции:** 1.0  
**Дата последнего обновления:** 2026-04-13
```
