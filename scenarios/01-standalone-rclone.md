# Работа с Rclone: Синхронизация данных

Как копировать файлы один-в-один без сохранения старых версий.

---

## 1. Подготовка

Перед запуском скриптов требуется выполнение следующих предварительных условий:

*   **Установка программ:** Установка Rclone (см. [Установку](../README.md#1-установка)).
*   **Настройка облака:** Создание подключения в Rclone (см. [Настройку облака](00-rclone-config.md)).

---

## 2. Скрипты для Linux

### Базовый скрипт (`rclone_sync.sh`)

```bash
#!/bin/bash
set -euo pipefail
# --- НАСТРОЙКИ ---
SOURCE="/home/user/documents"
# Путь к настройкам rclone (узнать путь: rclone config file)
RCLONE_CONF="/home/user/.config/rclone/rclone.conf"
# Файл журнала
LOG_FILE="/var/log/rclone_sync.log"

# Куда копируем:
DEST="gdrive:backup_mirror"      # В облако
# DEST="local_drive:/mnt/backup" # На диск

# --- РАБОТА ---
echo "$(date): Start" >> "$LOG_FILE"

# Варианты копирования:
# rclone --config "$RCLONE_CONF" sync "$SOURCE" "$DEST"        # Синхронизация (удаляет лишнее в копии)
# rclone --config "$RCLONE_CONF" copy "$SOURCE" "$DEST"        # Копирование (не удаляет лишнее)
# rclone --config "$RCLONE_CONF" move "$SOURCE" "$DEST"        # Перемещение (удаляет оригинал)

# Запуск:
# --progress: показывать процесс; --log-file: писать в журнал; 
# --checksum: проверять файлы; --transfers: число потоков.
rclone --config "$RCLONE_CONF" sync "$SOURCE" "$DEST" \
    --progress \
    --log-file="$LOG_FILE" \
    --checksum \
    --transfers 4

echo "$(date): Done" >> "$LOG_FILE"
```

### Скрипт с уведомлениями Ntfy (`rclone_sync_ntfy.sh`)

Полная версия скрипта, отправляющая статус работы на телефон или компьютер. Инструкция по настройке: **[Оповещения через Ntfy.sh](07-monitoring-ntfy.md)**.

```bash
#!/bin/bash
set -euo pipefail

# --- НАСТРОЙКИ ---
# Уникальное имя темы (topic) для подписки в приложении ntfy:
NTFY_TOPIC="имя_темы"
SOURCE="/home/user/documents"
RCLONE_CONF="/home/user/.config/rclone/rclone.conf"
DEST="gdrive:backup_mirror"
LOG_FILE="/var/log/rclone_sync.log"

# --- ФУНКЦИЯ УВЕДОМЛЕНИЙ ---
send_ntfy() {
    local MESSAGE="$1"
    local TITLE="${2:-Notification}"
    local TAGS="${3:-}"

    # Отправка данных в ntfy.sh:
    curl -s \
        -H "Title: $TITLE" \
        -H "Tags: $TAGS" \
        -d "$MESSAGE" \
        "https://ntfy.sh/$NTFY_TOPIC" > /dev/null || echo "Failed to send notification"
}

# --- МОНИТОРИНГ ОШИБОК ---
# Перехват сбоев для отправки уведомления:
trap 'send_ntfy "❌ Sync error on $(hostname). Check log: $LOG_FILE" "Backup Failed" "warning,skull"' ERR

# --- РАБОТА ---
echo "$(date): --- Start sync ---" >> "$LOG_FILE"

# Синхронизация данных:
rclone --config "$RCLONE_CONF" sync "$SOURCE" "$DEST" \
    --log-file="$LOG_FILE" \
    --checksum \
    --transfers 4

echo "$(date): --- Done ---" >> "$LOG_FILE"

send_ntfy "✅ Sync on $(hostname) completed successfully." "Success" "heavy_check_mark"
```

---

## 3. Скрипты для Windows

### Базовый скрипт (`rclone_sync.ps1`)

```powershell
# --- НАСТРОЙКИ ---
$ErrorActionPreference = "Stop"
# Полный путь к программе (обязательно для Планировщика)
$RcloneExe  = "$env:USERPROFILE\scoop\shims\rclone.exe" 
# Если установлен не через Scoop, укажите точный путь, например: "C:\Program Files\rclone\rclone.exe"
$SourceDir  = "C:\Users\Admin\Documents"
# Путь к настройкам rclone
$RcloneConf = "$env:USERPROFILE\scoop\apps\rclone\current\rclone.conf"
# Куда копируем:
$DestDir    = "gdrive:backup_mirror"
$LogFile    = "C:\Logs\rclone_sync.log"

# --- РАБОТА ---
Add-Content $LogFile "$(Get-Date): Start"

# Варианты копирования:
# & $RcloneExe --config $RcloneConf sync "$SourceDir" "$DestDir"        # Синхронизация (удаляет лишнее в копии)
# & $RcloneExe --config $RcloneConf copy "$SourceDir" "$DestDir"        # Копирование (не удаляет лишнее)
# & $RcloneExe --config $RcloneConf move "$SourceDir" "$DestDir"        # Перемещение (удаляет оригинал)

# Запуск с полным путем:
& $RcloneExe --config $RcloneConf sync "$SourceDir" "$DestDir" `
    --progress `
    --log-file "$LogFile" `
    --checksum `
    --transfers 4

Add-Content $LogFile "$(Get-Date): Done"
```

### Скрипт с уведомлениями Ntfy (`rclone_sync_ntfy.ps1`)

Полная версия скрипта, отправляющая статус работы на телефон или компьютер. Инструкция по настройке: **[Оповещения через Ntfy.sh](07-monitoring-ntfy.md)**.

```powershell
$ErrorActionPreference = "Stop"

# --- НАСТРОЙКИ ---
$NtfyTopic = "ваше_имя_темы"

# Полный путь к программе (обязательно для работы в Планировщике)
$RcloneExe  = "$env:USERPROFILE\scoop\shims\rclone.exe" 

$SourceDir  = "C:\Users\Admin\Documents"
$RcloneConf = "$env:USERPROFILE\scoop\apps\rclone\current\rclone.conf"
$DestDir    = "gdrive:backup_mirror"
$LogFile    = "C:\Logs\rclone_sync.log"

# --- ФУНКЦИЯ УВЕДОМЛЕНИЙ ---
function Send-Ntfy ($Message, $Title = "Notification", $Tags = "") {
    $Topic = "$NtfyTopic".Trim().Trim("/")
    if (-not $Topic) { return }

    # Формируем URL с параметрами (обход проблемы с кодировкой заголовков в PowerShell)
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

    # Запуск rclone через оператор & с полным путем
    & $RcloneExe --config $RcloneConf sync "$SourceDir" "$DestDir" `
        --log-file "$LogFile" `
        --checksum `
        --transfers 4

    # Проверка: если rclone вернул не 0, значит случилась ошибка
    if ($LASTEXITCODE -ne 0) {
        throw "Rclone failed with exit code $LASTEXITCODE. Check log: $LogFile"
    }

    Add-Content $LogFile "$(Get-Date): Done"
    Send-Ntfy "✅ Sync on $($env:COMPUTERNAME) completed successfully." "Success" "heavy_check_mark"
}
catch {
    Send-Ntfy "❌ Error on $($env:COMPUTERNAME): $($_.Exception.Message)" "Backup Failed" "warning,skull"
    throw
}
```

---

## 4. Автозапуск

Как настроить запуск по расписанию:
**[Автоматизация бэкапов](00-automation.md)**

---

## 5. Особенности работы

* Создает точную копию данных. Старые версии файлов не хранятся.
* Файлы в облаке лежат в обычном виде, их можно открыть без специальных программ.
* Синхронизация не следит за файлами постоянно. Её нужно запускать самому или через планировщик.
* **Удобство:** Можно легко поменять Google Drive на другой диск, просто изменив одну строчку в настройках скрипта.