# Универсальный бэкап: Локально, SFTP, Облако и Гибридная схема

Описание процессов создания резервных копий в любые типы хранилищ и настройки автоматического копирования между ними. Настройка скриптов под конкретные задачи выполняется посредством комментирования или раскомментирования строк в блоке параметров.

---

## 1. Подготовка

Перед запуском скриптов учитываются следующие условия:

*   **Установка программ:** Наличие установленных Restic и Rclone (см. [Установку](../README.md#1-установка)).
*   **Настройка облака:** **Если** используются облачные хранилища, требуется наличие созданного подключения в Rclone (см. [Настройку облака](00-rclone-config.md)).
*   **Файл настроек:** **Если** используется облако, требуется знание абсолютного пути к файлу `rclone.conf` (путь можно узнать командой `rclone config file`).
*   **Инициализация:** Выполнение команды `restic init` для основного репозитория. **Если** используется гибридная схема (копирование), инициализация также требуется для дополнительного репозитория.
*   **Права доступа:** **Если** в Windows используется параметр VSS (`--use-fs-snapshot`), запуск скрипта выполняется от имени администратора.

---

## 2. Скрипты для Linux

### Базовый скрипт (`backup.sh`)

```bash
#!/bin/bash
set -euo pipefail

# --- НАСТРОЙКИ ---

# Хранилище (Варианты: Локально / SFTP / Облако)
export RESTIC_REPOSITORY="/mnt/usb_drive/backup"
# export RESTIC_REPOSITORY="sftp:user@host:/path/to/repo"
# export RESTIC_REPOSITORY="rclone:gdrive:backup_repo"

# Второе хранилище (Опционально: Пусто / Облако / Другой диск)
# При заполнении выполняется команда restic copy
SECONDARY_REPOSITORY=""
# SECONDARY_REPOSITORY="rclone:gdrive:backup_cloud"

# Безопасность (Варианты: Пароль / Файл / Второй пароль / Второй файл)
export RESTIC_PASSWORD="ваш_пароль"
# export RESTIC_PASSWORD_FILE="/path/to/password.txt"
# export RESTIC_PASSWORD_TO="пароль_от_второго_хранилища"
# export RESTIC_PASSWORD_FILE_TO="/path/to/password2.txt"

# Источники (Варианты: Один путь / Массив путей)
SOURCE="/home/user/data"
# SOURCE=("/home/user/data" "/var/www/html")

# Параметры бэкапа (Варианты: Нет / Пропуск кэша / Одна файловая система)
RESTIC_OPTIONS=""
# RESTIC_OPTIONS="--exclude-caches"
# RESTIC_OPTIONS="--one-file-system"

# Логирование
LOG_FILE="/var/log/restic_backup.log"


# ВАЖНО: Создание хранилища перед первым бэкапом (выполняется один раз):
# restic init
# restic -r "$SECONDARY_REPOSITORY" init

# --- РАБОТА ---
echo "$(date): --- Start Backup ---" >> "$LOG_FILE"

# 1. Выполнение бэкапа в основное хранилище
if [[ "$(declare -p SOURCE 2>/dev/null)" =~ "declare -a" ]]; then
    # Использование полных путей для массива
    restic backup "${SOURCE[@]}" $RESTIC_OPTIONS --tag "local" >> "$LOG_FILE" 2>&1
else
    # Переход в родительский каталог для красоты имен в бэкапе
    PARENT_DIR=$(dirname "$SOURCE")
    TARGET_NAME=$(basename "$SOURCE")
    cd "$PARENT_DIR" || exit
    restic backup "$TARGET_NAME" $RESTIC_OPTIONS --tag "local" >> "$LOG_FILE" 2>&1
fi

# 2. Копирование в дополнительное хранилище (при наличии настроек)
if [ -n "$SECONDARY_REPOSITORY" ]; then
    echo "$(date): --- Start Copy to Secondary ---" >> "$LOG_FILE"
    restic copy --to-repo "$SECONDARY_REPOSITORY" >> "$LOG_FILE" 2>&1
fi

# 3. Очистка старых копий (в основном хранилище)
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune >> "$LOG_FILE" 2>&1

# 4. Проверка на ошибки
restic check >> "$LOG_FILE" 2>&1

echo "$(date): --- Done ---" >> "$LOG_FILE"
```

### Скрипт с уведомлениями Ntfy (`backup_ntfy.sh`)

```bash
#!/bin/bash
set -euo pipefail

# --- НАСТРОЙКИ ---

# Оповещения (Вариант: Тема ntfy.sh)
NTFY_TOPIC="имя_темы"

# Хранилище (Варианты: Локально / SFTP / Облако)
export RESTIC_REPOSITORY="/mnt/usb_drive/backup"
# export RESTIC_REPOSITORY="sftp:user@host:/path/to/repo"
# export RESTIC_REPOSITORY="rclone:gdrive:backup_repo"

# Второе хранилище (Опционально: Пусто / Облако / Другой диск)
# При заполнении выполняется команда restic copy
SECONDARY_REPOSITORY=""
# SECONDARY_REPOSITORY="rclone:gdrive:backup_cloud"

# Безопасность (Варианты: Пароль / Файл / Второй пароль / Второй файл)
export RESTIC_PASSWORD="ваш_пароль"
# export RESTIC_PASSWORD_FILE="/path/to/password.txt"
# export RESTIC_PASSWORD_TO="пароль_от_второго_хранилища"
# export RESTIC_PASSWORD_FILE_TO="/path/to/password2.txt"

# Источники (Варианты: Один путь / Массив)
SOURCE="/home/user/data"
# SOURCE=("/home/user/data" "/var/www/html")

# Параметры бэкапа (Варианты: Нет / Пропуск кэша / Одна файловая система)
RESTIC_OPTIONS=""
# RESTIC_OPTIONS="--exclude-caches"
# RESTIC_OPTIONS="--one-file-system"

# Логирование
LOG_FILE="/var/log/restic_backup.log"


# ВАЖНО: Создание хранилища перед первым бэкапом (выполняется один раз):
# restic init
# restic -r "$SECONDARY_REPOSITORY" init

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
CURRENT_STAGE="Initialization"
trap 'send_ntfy "❌ Error during $CURRENT_STAGE on $(hostname). Check log: $LOG_FILE" "Backup Failed" "warning,skull"' ERR

# --- РАБОТА ---
echo "$(date): --- Start ---" >> "$LOG_FILE"

# 1. Бэкап
CURRENT_STAGE="Backup"
if [[ "$(declare -p SOURCE 2>/dev/null)" =~ "declare -a" ]]; then
    restic backup "${SOURCE[@]}" $RESTIC_OPTIONS --tag "local" >> "$LOG_FILE" 2>&1
else
    PARENT_DIR=$(dirname "$SOURCE")
    TARGET_NAME=$(basename "$SOURCE")
    cd "$PARENT_DIR" || exit
    restic backup "$TARGET_NAME" $RESTIC_OPTIONS --tag "local" >> "$LOG_FILE" 2>&1
fi

# 2. Копирование (при наличии настроек)
COPY_STATUS=""
if [ -n "$SECONDARY_REPOSITORY" ]; then
    CURRENT_STAGE="Copy to Secondary"
    echo "$(date): --- Copying to Secondary ---" >> "$LOG_FILE"
    restic copy --to-repo "$SECONDARY_REPOSITORY" >> "$LOG_FILE" 2>&1
    COPY_STATUS=" and Copy"
fi

# 3. Обслуживание
CURRENT_STAGE="Maintenance (Forget/Check)"
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune >> "$LOG_FILE" 2>&1
restic check >> "$LOG_FILE" 2>&1

echo "$(date): --- Done ---" >> "$LOG_FILE"
send_ntfy "✅ Backup$COPY_STATUS on $(hostname) completed successfully." "Success" "heavy_check_mark"
```

---

## 3. Скрипты для Windows

### Базовый скрипт (`backup.ps1`)

```powershell
# --- НАСТРОЙКИ ---
$ErrorActionPreference = "Stop"
$ResticExe = "C:\Users\Admin\scoop\shims\restic.exe"

# Хранилище (Варианты: Локально / Облако / SFTP)
$env:RESTIC_REPOSITORY = "D:\Backup\repo"
# $env:RESTIC_REPOSITORY = "rclone:gdrive:backup_repo"
# $env:RESTIC_REPOSITORY = "sftp:user@host:/path/to/repo"

# Второе хранилище (Опционально: Пусто / Облако)
# При заполнении выполняется команда restic copy
$SecondaryRepo = ""
# $SecondaryRepo = "rclone:gdrive:backup_cloud"

# Программа и конфиг (Пути / Конфиг Rclone: узнать через rclone config file)
# $env:PATH += ";C:\Users\Admin\scoop\shims"
# $env:RCLONE_CONFIG = "C:\Users\Admin\scoop\apps\rclone\current\rclone.conf"

# Безопасность (Варианты: Пароль / Файл / Второй пароль / Второй файл)
$env:RESTIC_PASSWORD = "ваш_пароль"
# $env:RESTIC_PASSWORD_FILE = "C:\path\to\password.txt"
# $env:RESTIC_PASSWORD_TO = "пароль_от_второго_хранилища"
# $env:RESTIC_PASSWORD_FILE_TO = "C:\path\to\password2.txt"

# Источники (Варианты: Один путь / Список путей)
$Source = "C:\Users\Admin\Documents"
# $Source = @("C:\Data", "D:\Projects")

# Параметры бэкапа (Варианты: Нет / VSS снимок / Пропуск кэша)
$ResticOptions = ""
# $ResticOptions = "--use-fs-snapshot"
# $ResticOptions = "--exclude-caches"

# Логирование
$LogFile = "C:\Logs\backup.log"


# ВАЖНО: Создание хранилища перед первым бэкапом (выполняется один раз):
# & $ResticExe init
# & $ResticExe -r $SecondaryRepo init

# --- РАБОТА ---
Add-Content $LogFile "$(Get-Date): --- Start Backup ---"

# Подготовка путей
if ($Source -is [array]) {
    $Targets = $Source
} else {
    $ParentDir = Split-Path -Path $Source -Parent
    $TargetName = Split-Path -Path $Source -Leaf
    Set-Location -Path $ParentDir
    $Targets = $TargetName
}

# Временно отключаем остановку на stderr
$ErrorActionPreference = "Continue"

# 1. Выполнение бэкапа
& $ResticExe backup $Targets $ResticOptions --tag "local" 2>&1 | Out-File -FilePath "$LogFile" -Append
if ($LASTEXITCODE -ne 0) { throw "Backup failed" }

# 2. Копирование (при наличии настроек)
if ($SecondaryRepo) {
    Add-Content $LogFile "$(Get-Date): --- Start Copy to Secondary ---"
    & $ResticExe copy --to-repo $SecondaryRepo 2>&1 | Out-File -FilePath "$LogFile" -Append
}

# 3. Удаление старых копий
& $ResticExe forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune 2>&1 | Out-File -FilePath "$LogFile" -Append

# 4. Проверка данных
& $ResticExe check 2>&1 | Out-File -FilePath "$LogFile" -Append

$ErrorActionPreference = "Stop"
Add-Content $LogFile "$(Get-Date): --- Done ---"
```

### Скрипт с уведомлениями Ntfy (`backup_ntfy.ps1`)

```powershell
# --- НАСТРОЙКИ ---
$ErrorActionPreference = "Stop"
$ResticExe = "C:\Users\Admin\scoop\shims\restic.exe"

# Оповещения (Вариант: Тема ntfy.sh)
$NtfyTopic = "ваше_имя_темы"

# Хранилище (Варианты: Локально / Облако / SFTP)
$env:RESTIC_REPOSITORY = "D:\Backup\repo"
# $env:RESTIC_REPOSITORY = "rclone:gdrive:backup_repo"
# $env:RESTIC_REPOSITORY = "sftp:user@host:/path/to/repo"

# Второе хранилище (Опционально: Пусто / Облако)
# При заполнении выполняется команда restic copy
$SecondaryRepo = ""
# $SecondaryRepo = "rclone:gdrive:backup_cloud"

# Программа и конфиг (Пути / Конфиг Rclone: узнать через rclone config file)
# $env:PATH += ";C:\Users\Admin\scoop\shims"
# $env:RCLONE_CONFIG = "C:\Users\Admin\scoop\apps\rclone\current\rclone.conf"

# Безопасность (Варианты: Пароль / Файл / Второй пароль / Второй файл)
$env:RESTIC_PASSWORD = "ваш_пароль"
# $env:RESTIC_PASSWORD_FILE = "C:\path\to\password.txt"
# $env:RESTIC_PASSWORD_TO = "пароль_от_второго_хранилища"
# $env:RESTIC_PASSWORD_FILE_TO = "C:\path\to\password2.txt"

# Источники (Варианты: Один путь / Список путей)
$Source = "C:\Users\Admin\Documents"
# $Source = @("C:\Data", "D:\Projects")

# Параметры бэкапа (Варианты: Нет / VSS снимок / Пропуск кэша)
$ResticOptions = ""
# $ResticOptions = "--use-fs-snapshot"
# $ResticOptions = "--exclude-caches"

# Логирование
$LogFile = "C:\Logs\backup.log"


# ВАЖНО: Создание хранилища перед первым бэкапом (выполняется один раз):
# & $ResticExe init
# & $ResticExe -r $SecondaryRepo init

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
$CurrentStage = "Initialization"
try {
    Add-Content $LogFile "$(Get-Date): --- Start ---"

    if ($Source -is [array]) {
        $Targets = $Source
    } else {
        $ParentDir = Split-Path -Path $Source -Parent
        $TargetName = Split-Path -Path $Source -Leaf
        Set-Location -Path $ParentDir
        $Targets = $TargetName
    }

    $ErrorActionPreference = "Continue"

    # 1. Бэкап
    $CurrentStage = "Backup"
    & $ResticExe backup $Targets $ResticOptions --tag "local" 2>&1 | Out-File -FilePath "$LogFile" -Append
    if ($LASTEXITCODE -ne 0) { throw "Backup failed." }

    # 2. Копирование (при наличии настроек)
    $CopyStatus = ""
    if ($SecondaryRepo) {
        $CurrentStage = "Copy to Secondary"
        Add-Content $LogFile "$(Get-Date): --- Copying ---"
        & $ResticExe copy --to-repo $SecondaryRepo 2>&1 | Out-File -FilePath "$LogFile" -Append
        if ($LASTEXITCODE -ne 0) { throw "Copy to secondary repository failed." }
        $CopyStatus = " and Copy"
    }

    # 3. Очистка и Проверка
    $CurrentStage = "Maintenance (Forget/Check)"
    & $ResticExe forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune 2>&1 | Out-File -FilePath "$LogFile" -Append
    & $ResticExe check 2>&1 | Out-File -FilePath "$LogFile" -Append

    $ErrorActionPreference = "Stop"
    Add-Content $LogFile "$(Get-Date): --- Done ---"
    Send-Ntfy "✅ Backup$CopyStatus on $($env:COMPUTERNAME) completed successfully." "Success" "heavy_check_mark"
}
catch {
    Send-Ntfy "❌ Error during $CurrentStage on $($env:COMPUTERNAME): $($_.Exception.Message). Check log: $LogFile" "Backup Failed" "warning,skull"
    throw
}
```

---

## 4. Автозапуск

Настройка запуска по расписанию:
**[Автоматизация бэкапов](00-automation.md)**
