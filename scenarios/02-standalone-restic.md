# Работа с Restic: бэкап на диск и SFTP

Как делать бэкапы на флешки, внешние диски или свои серверы напрямую.

---

## 1. Подготовка

Перед запуском скриптов требуется выполнение следующих предварительных условий:

*   **Установка программ:** Установка Restic (см. [Установку](../README.md#1-установка)).
*   **Инициализация хранилища:** Создание репозитория Restic командой `restic init` перед первым использованием (указана в комментариях к скрипту).

---

## 2. Скрипты для Linux

### Базовый скрипт (`backup.sh`)

```bash
#!/bin/bash
set -euo pipefail
# --- НАСТРОЙКИ ---
# Где лежит хранилище
export RESTIC_REPOSITORY="/mnt/usb_drive/backup"
# Если используете SFTP:
# export RESTIC_REPOSITORY="sftp:user@host:/path/to/repo"
# Если используете облако (Google Drive) через Rclone:
# export RESTIC_REPOSITORY="rclone:gdrive:backup_repo"

export RESTIC_PASSWORD="ваш_пароль"
# Можно хранить пароль в файле:
# export RESTIC_PASSWORD_FILE="/path/to/password.txt"

SOURCE_DIR="/home/user/data"
LOG_FILE="/var/log/restic_backup.log"

# ВАЖНО: Перед первым бэкапом создайте хранилище (один раз):
# restic init

# --- РАБОТА ---
echo "$(date): Start" >> "$LOG_FILE"

# Переходим в папку с данными, чтобы пути в бэкапе были короткими
PARENT_DIR=$(dirname "$SOURCE_DIR")
TARGET_DIR=$(basename "$SOURCE_DIR")
cd "$PARENT_DIR" || exit

# Варианты запуска:
# restic backup "$TARGET_DIR"                                     # Обычный бэкап
# restic backup "$TARGET_DIR" --exclude "*.tmp"                   # Бэкап без временных файлов
# restic backup "$TARGET_DIR" --tag "local"                       # Бэкап с меткой (удобно для поиска)

# Делаем бэкап
restic backup "$TARGET_DIR" --tag "local" >> "$LOG_FILE" 2>&1

# Варианты очистки старых копий:
# restic forget --keep-last 10 --prune                            # Оставить только 10 последних
# restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune # Хранить 7 дней, 4 недели, 12 месяцев

# Удаляем старые копии
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune >> "$LOG_FILE" 2>&1

# Проверка данных на ошибки
restic check >> "$LOG_FILE" 2>&1

echo "$(date): Done" >> "$LOG_FILE"
```

### Скрипт с уведомлениями Ntfy (`backup_ntfy.sh`)

Полная версия скрипта, отправляющая статус работы на телефон или компьютер. Инструкция по настройке: **[Оповещения через Ntfy.sh](07-monitoring-ntfy.md)**.

```bash
#!/bin/bash
set -euo pipefail

# --- НАСТРОЙКИ ---
NTFY_TOPIC="имя_темы"

# Где лежит хранилище
export RESTIC_REPOSITORY="/mnt/usb_drive/backup"
# Если используете облако (Google Drive) через Rclone:
# export RESTIC_REPOSITORY="rclone:gdrive:backup_repo"

export RESTIC_PASSWORD="ваш_пароль"
SOURCE_DIR="/home/user/data"
LOG_FILE="/var/log/restic_backup.log"

# --- ФУНКЦИЯ УВЕДОМЛЕНИЙ ---
send_ntfy() {
    local MESSAGE="$1"
    local TITLE="${2:-Notification}"
    local TAGS="${3:-}"

    curl -s \
        -H "Title: $TITLE" \
        -H "Tags: $TAGS" \
        -d "$MESSAGE" \
        "https://ntfy.sh/$NTFY_TOPIC" > /dev/null || echo "Failed to send notification"
}

# --- МОНИТОРИНГ ОШИБОК ---
trap 'send_ntfy "❌ Backup error on $(hostname). Check log: $LOG_FILE" "Backup Failed" "warning,skull"' ERR

# --- РАБОТА ---
echo "$(date): Start" >> "$LOG_FILE"

PARENT_DIR=$(dirname "$SOURCE_DIR")
TARGET_DIR=$(basename "$SOURCE_DIR")
cd "$PARENT_DIR" || exit

# Бэкап
restic backup "$TARGET_DIR" --tag "local" >> "$LOG_FILE" 2>&1

# Очистка
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune >> "$LOG_FILE" 2>&1

# Проверка
restic check >> "$LOG_FILE" 2>&1

echo "$(date): Done" >> "$LOG_FILE"

send_ntfy "✅ Backup on $(hostname) completed successfully." "Success" "heavy_check_mark"
```

---

## 3. Скрипты для Windows

### Базовый скрипт (`backup.ps1`)

```powershell
# --- НАСТРОЙКИ ---
$ErrorActionPreference = "Stop"

# Полный путь к программе (обязательно для Планировщика)
# ВАЖНО: Если Планировщик работает от имени SYSTEM, $env:USERPROFILE использовать нельзя!
# Укажите жесткий путь к программе:
$ResticExe = "C:\Users\Admin\scoop\shims\restic.exe"
# Если установлен системно: "C:\Program Files\restic\restic.exe"

# Где лежит хранилище
$env:RESTIC_REPOSITORY = "D:\Backup\repo"
# Если используете SFTP:
# $env:RESTIC_REPOSITORY = "sftp:user@host:/path/to/repo"
# Если используете облако (Google Drive) через Rclone:
# $env:RESTIC_REPOSITORY = "rclone:gdrive:backup_repo"
# ВАЖНО: Чтобы Планировщик (SYSTEM) нашел Rclone из Scoop, добавьте пути:
# $env:PATH += ";C:\Users\Admin\scoop\shims"
# $env:RCLONE_CONFIG = "C:\Users\Admin\scoop\apps\rclone\current\rclone.conf"

$env:RESTIC_PASSWORD   = "ваш_пароль"
# Можно хранить пароль в файле:
# $env:RESTIC_PASSWORD_FILE = "C:\path\to\password.txt"

$SourceDir             = "C:\Users\Admin\Documents"
$LogFile               = "C:\Logs\backup.log"

# ВАЖНО: Перед первым бэкапом создайте хранилище (один раз):
# & $ResticExe init

# --- РАБОТА ---
Add-Content $LogFile "$(Get-Date): Start"

# Переходим в папку с данными, чтобы пути в бэкапе были короткими
$ParentDir = Split-Path -Path $SourceDir -Parent
$TargetDir = Split-Path -Path $SourceDir -Leaf
Set-Location -Path $ParentDir

# Временно отключаем остановку на stderr от native-утилит (restic пишет статусы в stderr)
$ErrorActionPreference = "Continue"

# Варианты запуска:
# & $ResticExe backup "$TargetDir" 2>&1 | Out-File -FilePath "$LogFile" -Append
# & $ResticExe backup "$TargetDir" --exclude "*.tmp" 2>&1 | Out-File -FilePath "$LogFile" -Append
# & $ResticExe backup "$TargetDir" --tag "local" 2>&1 | Out-File -FilePath "$LogFile" -Append

# Делаем бэкап
& $ResticExe backup "$TargetDir" --tag "local" 2>&1 | Out-File -FilePath "$LogFile" -Append
if ($LASTEXITCODE -ne 0) { Write-Warning "Restic backup error (code: $LASTEXITCODE)" }

# Варианты очистки старых копий:
# & $ResticExe forget --keep-last 10 --prune 2>&1 | Out-File -FilePath "$LogFile" -Append
# & $ResticExe forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune 2>&1 | Out-File -FilePath "$LogFile" -Append

# Удаляем старые копии
& $ResticExe forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune 2>&1 | Out-File -FilePath "$LogFile" -Append
if ($LASTEXITCODE -ne 0) { Write-Warning "Restic forget error (code: $LASTEXITCODE)" }

# Проверка данных на ошибки
& $ResticExe check 2>&1 | Out-File -FilePath "$LogFile" -Append
if ($LASTEXITCODE -ne 0) { Write-Warning "Restic check error (code: $LASTEXITCODE)" }

$ErrorActionPreference = "Stop"

Add-Content $LogFile "$(Get-Date): Done"
```

### Скрипт с уведомлениями Ntfy (`backup_ntfy.ps1`)

Полная версия скрипта, отправляющая статус работы на телефон или компьютер. Инструкция по настройке: **[Оповещения через Ntfy.sh](07-monitoring-ntfy.md)**.

```powershell
$ErrorActionPreference = "Stop"

# --- НАСТРОЙКИ ---
$NtfyTopic = "ваше_имя_темы"

# ВАЖНО: Если Планировщик работает от имени SYSTEM, $env:USERPROFILE использовать нельзя!
$ResticExe = "C:\Users\Admin\scoop\shims\restic.exe"

# Где лежит хранилище
$env:RESTIC_REPOSITORY = "D:\Backup\repo"
# Если используете облако (Google Drive) через Rclone:
# $env:RESTIC_REPOSITORY = "rclone:gdrive:backup_repo"
# ВАЖНО: Чтобы Планировщик (SYSTEM) нашел Rclone из Scoop, добавьте пути:
# $env:PATH += ";C:\Users\Admin\scoop\shims"
# $env:RCLONE_CONFIG = "C:\Users\Admin\scoop\apps\rclone\current\rclone.conf"

$env:RESTIC_PASSWORD   = "ваш_пароль"
$SourceDir             = "C:\Users\Admin\Documents"
$LogFile               = "C:\Logs\backup.log"

# --- ФУНКЦИЯ УВЕДОМЛЕНИЙ ---
function Send-Ntfy ($Message, $Title = "Notification", $Tags = "") {
    $Topic = "$NtfyTopic".Trim().Trim("/")
    if (-not $Topic) { return }

    $Url = "https://ntfy.sh/$Topic"
    $Params = @()
    if ($Title) { $Params += "title=$([Uri]::EscapeDataString($Title))" }
    if ($Tags)  { $Params += "tags=$([Uri]::EscapeDataString($Tags))" }
    if ($Params.Count -gt 0) { $Url += "?" + ($Params -join "&") }

    try {
        $Body = [System.Text.Encoding]::UTF8.GetBytes($Message)
        Invoke-RestMethod -Uri $Url -Method Post -Body $Body -ErrorAction Stop | Out-Null
    }
    catch {
        Write-Warning "Failed to send notification: $($_.Exception.Message)"
    }
}

# --- РАБОТА (С МОНИТОРИНГОМ) ---
try {
    Add-Content $LogFile "$(Get-Date): Start"

    $ParentDir = Split-Path -Path $SourceDir -Parent
    $TargetDir = Split-Path -Path $SourceDir -Leaf
    Set-Location -Path $ParentDir

    # Отключаем остановку при выводе restic в stderr
    $ErrorActionPreference = "Continue"

    # Бэкап
    & $ResticExe backup "$TargetDir" --tag "local" 2>&1 | Out-File -FilePath "$LogFile" -Append
    if ($LASTEXITCODE -ne 0) { throw "Restic backup failed (code: $LASTEXITCODE)." }

    # Очистка
    & $ResticExe forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune 2>&1 | Out-File -FilePath "$LogFile" -Append
    if ($LASTEXITCODE -ne 0) { throw "Restic forget failed (code: $LASTEXITCODE)." }

    # Проверка
    & $ResticExe check 2>&1 | Out-File -FilePath "$LogFile" -Append
    if ($LASTEXITCODE -ne 0) { throw "Restic check failed (code: $LASTEXITCODE)." }

    $ErrorActionPreference = "Stop"

    Add-Content $LogFile "$(Get-Date): Done"
    Send-Ntfy "✅ Backup on $($env:COMPUTERNAME) completed successfully." "Success" "heavy_check_mark"
}
catch {
    Send-Ntfy "❌ Error on $($env:COMPUTERNAME): $($_.Exception.Message). Check log: $LogFile" "Backup Failed" "warning,skull"
    throw
}
```

---

## 4. Автозапуск

Настройка запуска по расписанию:
**[Автоматизация бэкапов](00-automation.md)**

---

## 5. Когда использовать

* При наличии прямого доступа к диску или серверу по SFTP.
* Высокая скорость работы за счет отсутствия программ-посредников.
* **Важно:** Предварительный запуск `restic init` для подготовки места под бэкапы.