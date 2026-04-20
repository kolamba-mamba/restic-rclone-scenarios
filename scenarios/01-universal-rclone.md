# Работа с Rclone: Синхронизация данных

Описание процессов копирования файлов один-в-один без сохранения версий. Данный метод создает зеркальную копию данных в одном или нескольких хранилищах. Настройка скриптов под конкретные задачи выполняется посредством комментирования или раскомментирования строк в блоке параметров.

---

## 1. Подготовка

Перед запуском скриптов учитываются следующие условия:

*   **Установка программ:** Наличие установленного Rclone (см. [Установку](../README.md#1-установка)).
*   **Настройка облака:** **Если** планируется использование облака, требуется наличие созданного подключения в Rclone (см. [Настройку облака](00-rclone-config.md)).
*   **Файл настроек:** Знание абсолютного пути к файлу `rclone.conf` (путь можно узнать командой `rclone config file`).

---

## 2. Скрипты для Linux

### Базовый скрипт (`rclone_sync.sh`)

```bash
#!/bin/bash
set -euo pipefail

# --- НАСТРОЙКИ ---

# Конфигурация (Путь к конфигу: узнать через rclone config file)
RCLONE_CONF="/home/user/.config/rclone/rclone.conf"

# Источники (Варианты: Один / Массив папок)
SOURCE="/home/user/documents"
# SOURCE=("/home/user/data" "/var/www/html")

# Назначение (Варианты: Одно / Массив назначений)
DEST="gdrive:backup_mirror"
# DEST=("gdrive:backup" "onedrive:backup" "/mnt/usb_drive/backup")

# Параметры (Варианты: sync: зеркало / copy: копирование / move: перемещение)
RCLONE_CMD="sync"
# RCLONE_CMD="copy"
# RCLONE_CMD="move"

# Дополнительно (Варианты: Проверка контрольных сумм / Потоки)
RCLONE_OPTS="--checksum --transfers 4 --progress"

# Логирование
LOG_FILE="/var/log/rclone_sync.log"


# --- РАБОТА ---
echo "$(date): --- Start ---" >> "$LOG_FILE"

# Приведение источников и назначений к массивам
if [[ "$(declare -p SOURCE 2>/dev/null)" =~ "declare -a" ]]; then
    SOURCES_ARR=("${SOURCE[@]}")
else
    SOURCES_ARR=("$SOURCE")
fi

if [[ "$(declare -p DEST 2>/dev/null)" =~ "declare -a" ]]; then
    DESTS_ARR=("${DEST[@]}")
else
    DESTS_ARR=("$DEST")
fi

# Цикл по источникам и назначениям
for src in "${SOURCES_ARR[@]}"; do
    for dst in "${DESTS_ARR[@]}"; do
        echo "$(date): Processing $src -> $dst" >> "$LOG_FILE"
        rclone --config "$RCLONE_CONF" "$RCLONE_CMD" "$src" "$dst" \
            --log-file="$LOG_FILE" \
            $RCLONE_OPTS
    done
done

echo "$(date): --- Done ---" >> "$LOG_FILE"
```

### Скрипт с уведомлениями Ntfy (`rclone_sync_ntfy.sh`)

```bash
#!/bin/bash
set -euo pipefail

# --- НАСТРОЙКИ ---

# Оповещения (Вариант: Тема ntfy.sh)
NTFY_TOPIC="имя_темы"

# Конфигурация (Путь к конфигу: узнать через rclone config file)
RCLONE_CONF="/home/user/.config/rclone/rclone.conf"

# Источники (Варианты: Один / Массив)
SOURCE="/home/user/documents"
# SOURCE=("/home/user/data" "/var/www/html")

# Назначение (Варианты: Одно / Массив)
DEST="gdrive:backup_mirror"
# DEST=("gdrive:backup" "onedrive:backup")

# Параметры (Варианты: sync / copy)
RCLONE_CMD="sync"

# Дополнительно (Варианты: Проверка контрольных сумм / Потоки)
RCLONE_OPTS="--checksum --transfers 4"

# Логирование
LOG_FILE="/var/log/rclone_sync.log"


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
CURRENT_PAIR="Initialization"
trap 'send_ntfy "❌ Error: $CURRENT_PAIR failed. Check log: $LOG_FILE" "Sync Failed" "warning,skull"' ERR

# --- РАБОТА ---
echo "$(date): --- Start ---" >> "$LOG_FILE"

if [[ "$(declare -p SOURCE 2>/dev/null)" =~ "declare -a" ]]; then
    SOURCES_ARR=("${SOURCE[@]}")
else
    SOURCES_ARR=("$SOURCE")
fi

if [[ "$(declare -p DEST 2>/dev/null)" =~ "declare -a" ]]; then
    DESTS_ARR=("${DEST[@]}")
else
    DESTS_ARR=("$DEST")
fi

COUNT=0
for src in "${SOURCES_ARR[@]}"; do
    for dst in "${DESTS_ARR[@]}"; do
        CURRENT_PAIR="$src -> $dst"
        rclone --config "$RCLONE_CONF" "$RCLONE_CMD" "$src" "$dst" \
            --log-file="$LOG_FILE" \
            $RCLONE_OPTS
        ((COUNT++))
    done
done

echo "$(date): --- Done ---" >> "$LOG_FILE"
send_ntfy "✅ Sync on $(hostname) completed successfully. Total pairs: $COUNT." "Success" "heavy_check_mark"
```

---

## 3. Скрипты для Windows

### Базовый скрипт (`rclone_sync.ps1`)

```powershell
# --- НАСТРОЙКИ ---
$ErrorActionPreference = "Stop"

# Программа и конфиг (EXE / Путь к конфигу: узнать через rclone config file)
$RcloneExe  = "C:\Users\Admin\scoop\shims\rclone.exe" 
$RcloneConf = "C:\Users\Admin\scoop\apps\rclone\current\rclone.conf"

# Источники (Варианты: Один / Массив)
$SourceDir  = "C:\Users\Admin\Documents"
# $SourceDir  = @("C:\Data", "D:\Projects")

# Назначение (Варианты: Одно / Массив)
$DestDir    = "gdrive:backup_mirror"
# $DestDir    = @("gdrive:backup", "onedrive:backup")

# Параметры (Варианты: sync: зеркало / copy: копирование / move: перемещение)
$RcloneCmd  = "sync"
# $RcloneCmd  = "copy"
# $RcloneCmd  = "move"

# Дополнительно (Варианты: Проверка контрольных сумм / Потоки)
$RcloneOpts = @("--checksum", "--transfers", "4", "--progress")

# Логирование
$LogFile    = "C:\Logs\rclone_sync.log"


# --- РАБОТА ---
Add-Content $LogFile "$(Get-Date): --- Start ---"

# Приведение к массивам для единообразной обработки
$Sources = [array]$SourceDir
$Destinations = [array]$DestDir

foreach ($src in $Sources) {
    foreach ($dst in $Destinations) {
        Add-Content $LogFile "$(Get-Date): Processing $src -> $dst"
        & $RcloneExe --config $RcloneConf $RcloneCmd "$src" "$dst" `
            --log-file "$LogFile" `
            @RcloneOpts
    }
}

Add-Content $LogFile "$(Get-Date): --- Done ---"
```

### Скрипт с уведомлениями Ntfy (`rclone_sync_ntfy.ps1`)

```powershell
# --- НАСТРОЙКИ ---
$ErrorActionPreference = "Stop"

# Оповещения (Вариант: Тема ntfy.sh)
$NtfyTopic = "ваше_имя_темы"

# Программа и конфиг (EXE / Путь к конфигу: узнать через rclone config file)
$RcloneExe  = "C:\Users\Admin\scoop\shims\rclone.exe" 
$RcloneConf = "C:\Users\Admin\scoop\apps\rclone\current\rclone.conf"

# Источники (Варианты: Один / Массив)
$SourceDir  = "C:\Users\Admin\Documents"
# $SourceDir  = @("C:\Data", "D:\Projects")

# Назначение (Варианты: Одно / Массив)
$DestDir    = "gdrive:backup_mirror"
# $DestDir    = @("gdrive:backup", "onedrive:backup")

# Параметры (Варианты: sync / copy)
$RcloneCmd  = "sync"

# Дополнительно (Варианты: Проверка контрольных сумм / Потоки)
$RcloneOpts = @("--checksum", "--transfers", "4")

# Логирование
$LogFile    = "C:\Logs\rclone_sync.log"


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

# --- РАБОТА ---
$CurrentPair = "Initialization"
try {
    Add-Content $LogFile "$(Get-Date): --- Start ---"

    $Sources = [array]$SourceDir
    $Destinations = [array]$DestDir
    $Count = 0

    foreach ($src in $Sources) {
        foreach ($dst in $Destinations) {
            $CurrentPair = "$src -> $dst"
            & $RcloneExe --config $RcloneConf $RcloneCmd "$src" "$dst" `
                --log-file "$LogFile" `
                @RcloneOpts
            
            if ($LASTEXITCODE -ne 0) {
                throw "Rclone failed with exit code $LASTEXITCODE."
            }
            $Count++
        }
    }

    Add-Content $LogFile "$(Get-Date): --- Done ---"
    Send-Ntfy "✅ Sync on $($env:COMPUTERNAME) completed successfully. Total pairs: $Count." "Success" "heavy_check_mark"
}
catch {
    Send-Ntfy "❌ Error: $CurrentPair failed. $($_.Exception.Message)" "Sync Failed" "warning,skull"
    throw
}
```

---

## 4. Автозапуск

Настройка запуска по расписанию:
**[Автоматизация бэкапов](00-automation.md)**
