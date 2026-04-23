# Универсальный бэкап: Локально, SFTP, Облако и Гибридная схема

Описание процессов создания резервных копий в любые типы хранилищ и настройки автоматического копирования между ними. Настройка скриптов под конкретные задачи выполняется посредством комментирования или раскомментирования строк в блоке параметров.

---

## 1. Подготовка

Перед запуском скриптов учитываются следующие условия:

*   **Установка программ:** Наличие установленных Restic и Rclone (см. [Установку](../README.md#1-установка)).
*   **Настройка облака:** **Если** используются облачные хранилища, требуется наличие созданного подключения в Rclone (см. [Настройку облака](00-rclone-config.md)).
*   **Файл настроек:** **Если** используется облако, требуется знание абсолютного пути к файлу `rclone.conf` (путь можно узнать командой `rclone config file`).
*   **Инициализация:** Выполнение команды `restic init` для основного репозитория. **Если** используется гибридная схема (копирование), инициализация также требуется для дополнительного репозитория.
*   **Журнал:** Путь из `LOG_FILE` существует и доступен для записи. Примеры путей в шаблонах показаны как образец и при необходимости заменяются.
*   **Пароли:** Для каждого репозитория используется свой пароль или свой файл пароля. При гибридной схеме доступ требуется и к основному, и к дополнительному хранилищу.
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
# Пути передаются в restic как есть и сохраняются в snapshot в исходном виде
SOURCE="/home/user/data"
# SOURCE=("/home/user/data" "/var/www/html")

# Опции Restic
RESTIC_OPTIONS=()
# Настройка необходимых флагов. Одна ФС / Кэш / Исключения / Файл исключений
# RESTIC_OPTIONS=(--one-file-system --exclude-caches --exclude="/var/tmp" --exclude-file="/etc/restic/excludes.txt")

# Политика хранения копий (Варианты: Расширенная / Простая)
RETENTION_POLICY="--keep-daily 7 --keep-weekly 4 --keep-monthly 12"
# RETENTION_POLICY="--keep-last 5"

# Логирование
LOG_FILE="/var/log/restic_backup.log"


# ВАЖНО: Создание хранилища перед первым бэкапом (выполняется один раз):
# restic init
# RESTIC_PASSWORD="$RESTIC_PASSWORD_TO" restic -r "$SECONDARY_REPOSITORY" init

# --- РАБОТА ---
echo "$(date): --- Start Backup ---" >> "$LOG_FILE"

# 1. Выполнение бэкапа в основное хранилище
if [[ "$(declare -p SOURCE 2>/dev/null)" =~ "declare -a" ]]; then
    SOURCES_ARR=("${SOURCE[@]}")
else
    SOURCES_ARR=("$SOURCE")
fi

for src in "${SOURCES_ARR[@]}"; do
    if [ ! -e "$src" ]; then
        echo "$(date): Missing source: $src" >> "$LOG_FILE"
        exit 1
    fi
done

restic backup "${SOURCES_ARR[@]}" "${RESTIC_OPTIONS[@]}" --tag "local" >> "$LOG_FILE" 2>&1

# 2. Копирование в дополнительное хранилище (при наличии настроек)
if [ -n "$SECONDARY_REPOSITORY" ]; then
    echo "$(date): --- Start Copy to Secondary ---" >> "$LOG_FILE"
    PRIMARY_REPOSITORY="$RESTIC_REPOSITORY"
    PRIMARY_PASSWORD="${RESTIC_PASSWORD:-}"
    PRIMARY_PASSWORD_FILE="${RESTIC_PASSWORD_FILE:-}"

    export RESTIC_FROM_REPOSITORY="$RESTIC_REPOSITORY"
    export RESTIC_FROM_PASSWORD="${RESTIC_PASSWORD:-}"
    export RESTIC_FROM_PASSWORD_FILE="${RESTIC_PASSWORD_FILE:-}"

    if [ -n "${RESTIC_PASSWORD_TO:-}" ]; then
        export RESTIC_PASSWORD="$RESTIC_PASSWORD_TO"
        unset RESTIC_PASSWORD_FILE
    elif [ -n "${RESTIC_PASSWORD_FILE_TO:-}" ]; then
        export RESTIC_PASSWORD_FILE="$RESTIC_PASSWORD_FILE_TO"
        unset RESTIC_PASSWORD
    fi

    restic -r "$SECONDARY_REPOSITORY" copy --from-repo "$RESTIC_FROM_REPOSITORY" >> "$LOG_FILE" 2>&1

    export RESTIC_REPOSITORY="$PRIMARY_REPOSITORY"
    if [ -n "$PRIMARY_PASSWORD" ]; then
        export RESTIC_PASSWORD="$PRIMARY_PASSWORD"
    else
        unset RESTIC_PASSWORD
    fi
    if [ -n "$PRIMARY_PASSWORD_FILE" ]; then
        export RESTIC_PASSWORD_FILE="$PRIMARY_PASSWORD_FILE"
    else
        unset RESTIC_PASSWORD_FILE
    fi
fi

# 3. Очистка старых копий (в основном хранилище)
# shellcheck disable=SC2086
restic forget $RETENTION_POLICY --prune >> "$LOG_FILE" 2>&1

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
# Пути передаются в restic как есть и сохраняются в snapshot в исходном виде
SOURCE="/home/user/data"
# SOURCE=("/home/user/data" "/var/www/html")

# Опции Restic
RESTIC_OPTIONS=()
# Настройка необходимых флагов. Одна ФС / Кэш / Исключения / Файл исключений
# RESTIC_OPTIONS=(--one-file-system --exclude-caches --exclude="/var/tmp" --exclude-file="/etc/restic/excludes.txt")

# Политика хранения копий (Варианты: Расширенная / Простая)
RETENTION_POLICY="--keep-daily 7 --keep-weekly 4 --keep-monthly 12"
# RETENTION_POLICY="--keep-last 5"

# Логирование
LOG_FILE="/var/log/restic_backup.log"


# ВАЖНО: Создание хранилища перед первым бэкапом (выполняется один раз):
# restic init
# RESTIC_PASSWORD="$RESTIC_PASSWORD_TO" restic -r "$SECONDARY_REPOSITORY" init

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

fail_with_ntfy() {
    local MESSAGE="$1"
    echo "$(date): $MESSAGE" >> "$LOG_FILE"
    send_ntfy "❌ Error during $CURRENT_STAGE on $(hostname): $MESSAGE. Check log: $LOG_FILE" "Backup Failed" "warning,skull"
    exit 1
}

# --- МОНИТОРИНГ ОШИБОК ---
CURRENT_STAGE="Initialization"
trap 'send_ntfy "❌ Error during $CURRENT_STAGE on $(hostname). Check log: $LOG_FILE" "Backup Failed" "warning,skull"' ERR

# --- РАБОТА ---
echo "$(date): --- Start ---" >> "$LOG_FILE"

# 1. Бэкап
CURRENT_STAGE="Backup"
if [[ "$(declare -p SOURCE 2>/dev/null)" =~ "declare -a" ]]; then
    SOURCES_ARR=("${SOURCE[@]}")
else
    SOURCES_ARR=("$SOURCE")
fi

for src in "${SOURCES_ARR[@]}"; do
    if [ ! -e "$src" ]; then
        fail_with_ntfy "Missing source: $src"
    fi
done

restic backup "${SOURCES_ARR[@]}" "${RESTIC_OPTIONS[@]}" --tag "local" >> "$LOG_FILE" 2>&1

# 2. Копирование (при наличии настроек)
COPY_STATUS=""
if [ -n "$SECONDARY_REPOSITORY" ]; then
    CURRENT_STAGE="Copy to Secondary"
    echo "$(date): --- Copying to Secondary ---" >> "$LOG_FILE"
    PRIMARY_REPOSITORY="$RESTIC_REPOSITORY"
    PRIMARY_PASSWORD="${RESTIC_PASSWORD:-}"
    PRIMARY_PASSWORD_FILE="${RESTIC_PASSWORD_FILE:-}"

    export RESTIC_FROM_REPOSITORY="$RESTIC_REPOSITORY"
    export RESTIC_FROM_PASSWORD="${RESTIC_PASSWORD:-}"
    export RESTIC_FROM_PASSWORD_FILE="${RESTIC_PASSWORD_FILE:-}"

    if [ -n "${RESTIC_PASSWORD_TO:-}" ]; then
        export RESTIC_PASSWORD="$RESTIC_PASSWORD_TO"
        unset RESTIC_PASSWORD_FILE
    elif [ -n "${RESTIC_PASSWORD_FILE_TO:-}" ]; then
        export RESTIC_PASSWORD_FILE="$RESTIC_PASSWORD_FILE_TO"
        unset RESTIC_PASSWORD
    fi

    restic -r "$SECONDARY_REPOSITORY" copy --from-repo "$RESTIC_FROM_REPOSITORY" >> "$LOG_FILE" 2>&1

    export RESTIC_REPOSITORY="$PRIMARY_REPOSITORY"
    if [ -n "$PRIMARY_PASSWORD" ]; then
        export RESTIC_PASSWORD="$PRIMARY_PASSWORD"
    else
        unset RESTIC_PASSWORD
    fi
    if [ -n "$PRIMARY_PASSWORD_FILE" ]; then
        export RESTIC_PASSWORD_FILE="$PRIMARY_PASSWORD_FILE"
    else
        unset RESTIC_PASSWORD_FILE
    fi
    COPY_STATUS=" and Copy"
fi

# 3. Обслуживание
CURRENT_STAGE="Maintenance (Forget/Check)"
# shellcheck disable=SC2086
restic forget $RETENTION_POLICY --prune >> "$LOG_FILE" 2>&1
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
# Пути передаются в restic как есть и сохраняются в snapshot в исходном виде
$Source = "C:\Users\Admin\Documents"
# $Source = @("C:\Data", "D:\Projects")

# Опции Restic
$ResticOptions = @()
# Настройка необходимых флагов. VSS / Кэш / Исключения / Файл исключений
# $ResticOptions = @("--use-fs-snapshot", "--exclude-caches", "--exclude=C:\Windows\Temp", "--exclude-file=C:\bk\excludes.txt")

# Логирование
$LogFile = "C:\Logs\backup.log"

# ВАЖНО: Создание хранилища перед первым бэкапом (выполняется один раз):
# & $ResticExe init
# $env:RESTIC_PASSWORD = $env:RESTIC_PASSWORD_TO
# & $ResticExe -r $SecondaryRepo init

# --- ФУНКЦИЯ ЗАПУСКА ---
function Invoke-ResticLogged {
    param (
        [Parameter(ValueFromRemainingArguments = $true)]
        [string[]]$Arguments
    )

    & $ResticExe @Arguments 2>&1 | Out-File -FilePath "$LogFile" -Append
    if ($LASTEXITCODE -ne 0) {
        throw "Restic failed: $($Arguments -join ' ')"
    }
}

function Test-BackupSources {
    param (
        [string[]]$Paths
    )

    foreach ($PathItem in $Paths) {
        if (-not (Test-Path -LiteralPath $PathItem)) {
            Add-Content $LogFile "$(Get-Date): Missing source: $PathItem"
            throw "Missing source: $PathItem"
        }
    }
}


# --- РАБОТА ---
Add-Content $LogFile "$(Get-Date): --- Start Backup ---"

# Подготовка путей
$Targets = [array]$Source

# Временно отключаем остановку на stderr
$ErrorActionPreference = "Continue"

try {
    # 1. Выполнение бэкапа
    Test-BackupSources -Paths $Targets
    $BackupArgs = @("backup") + $Targets + $ResticOptions + @("--tag", "local")
    Invoke-ResticLogged -Arguments $BackupArgs

    # 2. Копирование (при наличии настроек)
    if ($SecondaryRepo) {
        Add-Content $LogFile "$(Get-Date): --- Start Copy to Secondary ---"
        $PrimaryRepository = $env:RESTIC_REPOSITORY
        $PrimaryPassword = $env:RESTIC_PASSWORD
        $PrimaryPasswordFile = $env:RESTIC_PASSWORD_FILE

        $env:RESTIC_FROM_REPOSITORY = $env:RESTIC_REPOSITORY
        $env:RESTIC_FROM_PASSWORD = $env:RESTIC_PASSWORD
        $env:RESTIC_FROM_PASSWORD_FILE = $env:RESTIC_PASSWORD_FILE
        $env:RESTIC_REPOSITORY = $SecondaryRepo

        if ($env:RESTIC_PASSWORD_TO) {
            $env:RESTIC_PASSWORD = $env:RESTIC_PASSWORD_TO
            Remove-Item Env:RESTIC_PASSWORD_FILE -ErrorAction SilentlyContinue
        }
        elseif ($env:RESTIC_PASSWORD_FILE_TO) {
            $env:RESTIC_PASSWORD_FILE = $env:RESTIC_PASSWORD_FILE_TO
            Remove-Item Env:RESTIC_PASSWORD -ErrorAction SilentlyContinue
        }

        $CopyArgs = @("copy", "--from-repo", $env:RESTIC_FROM_REPOSITORY)
        Invoke-ResticLogged -Arguments $CopyArgs

        $env:RESTIC_REPOSITORY = $PrimaryRepository
        if ($PrimaryPassword) {
            $env:RESTIC_PASSWORD = $PrimaryPassword
        }
        else {
            Remove-Item Env:RESTIC_PASSWORD -ErrorAction SilentlyContinue
        }
        if ($PrimaryPasswordFile) {
            $env:RESTIC_PASSWORD_FILE = $PrimaryPasswordFile
        }
        else {
            Remove-Item Env:RESTIC_PASSWORD_FILE -ErrorAction SilentlyContinue
        }
    }

    # 3. Удаление старых копий
    $ForgetArgs = @("forget") + $RetentionPolicy + @("--prune")
    Invoke-ResticLogged -Arguments $ForgetArgs

    # 4. Проверка данных
    $CheckArgs = @("check")
    Invoke-ResticLogged -Arguments $CheckArgs

    Add-Content $LogFile "$(Get-Date): --- Done ---"
}
finally {
    $ErrorActionPreference = "Stop"
}
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
# Пути передаются в restic как есть и сохраняются в snapshot в исходном виде
$Source = "C:\Users\Admin\Documents"
# $Source = @("C:\Data", "D:\Projects")

# Опции Restic
$ResticOptions = @()
# Настройка необходимых флагов. VSS / Кэш / Исключения / Файл исключений
# $ResticOptions = @("--use-fs-snapshot", "--exclude-caches", "--exclude=C:\Windows\Temp", "--exclude-file=C:\bk\excludes.txt")

# Логирование
$LogFile = "C:\Logs\backup.log"

# ВАЖНО: Создание хранилища перед первым бэкапом (выполняется один раз):
# & $ResticExe init
# $env:RESTIC_PASSWORD = $env:RESTIC_PASSWORD_TO
# & $ResticExe -r $SecondaryRepo init

# --- ФУНКЦИЯ ЗАПУСКА ---
function Invoke-ResticLogged {
    param (
        [Parameter(ValueFromRemainingArguments = $true)]
        [string[]]$Arguments
    )

    & $ResticExe @Arguments 2>&1 | Out-File -FilePath "$LogFile" -Append
    if ($LASTEXITCODE -ne 0) {
        throw "Restic failed: $($Arguments -join ' ')"
    }
}

function Test-BackupSources {
    param (
        [string[]]$Paths
    )

    foreach ($PathItem in $Paths) {
        if (-not (Test-Path -LiteralPath $PathItem)) {
            Add-Content $LogFile "$(Get-Date): Missing source: $PathItem"
            throw "Missing source: $PathItem"
        }
    }
}


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

    $Targets = [array]$Source

    $ErrorActionPreference = "Continue"

    try {
        # 1. Бэкап
        $CurrentStage = "Backup"
        Test-BackupSources -Paths $Targets
        $BackupArgs = @("backup") + $Targets + $ResticOptions + @("--tag", "local")
        Invoke-ResticLogged -Arguments $BackupArgs

        # 2. Копирование (при наличии настроек)
        $CopyStatus = ""
        if ($SecondaryRepo) {
            $CurrentStage = "Copy to Secondary"
            Add-Content $LogFile "$(Get-Date): --- Copying ---"
            $PrimaryRepository = $env:RESTIC_REPOSITORY
            $PrimaryPassword = $env:RESTIC_PASSWORD
            $PrimaryPasswordFile = $env:RESTIC_PASSWORD_FILE

            $env:RESTIC_FROM_REPOSITORY = $env:RESTIC_REPOSITORY
            $env:RESTIC_FROM_PASSWORD = $env:RESTIC_PASSWORD
            $env:RESTIC_FROM_PASSWORD_FILE = $env:RESTIC_PASSWORD_FILE
            $env:RESTIC_REPOSITORY = $SecondaryRepo

            if ($env:RESTIC_PASSWORD_TO) {
                $env:RESTIC_PASSWORD = $env:RESTIC_PASSWORD_TO
                Remove-Item Env:RESTIC_PASSWORD_FILE -ErrorAction SilentlyContinue
            }
            elseif ($env:RESTIC_PASSWORD_FILE_TO) {
                $env:RESTIC_PASSWORD_FILE = $env:RESTIC_PASSWORD_FILE_TO
                Remove-Item Env:RESTIC_PASSWORD -ErrorAction SilentlyContinue
            }

            $CopyArgs = @("copy", "--from-repo", $env:RESTIC_FROM_REPOSITORY)
            Invoke-ResticLogged -Arguments $CopyArgs

            $env:RESTIC_REPOSITORY = $PrimaryRepository
            if ($PrimaryPassword) {
                $env:RESTIC_PASSWORD = $PrimaryPassword
            }
            else {
                Remove-Item Env:RESTIC_PASSWORD -ErrorAction SilentlyContinue
            }
            if ($PrimaryPasswordFile) {
                $env:RESTIC_PASSWORD_FILE = $PrimaryPasswordFile
            }
            else {
                Remove-Item Env:RESTIC_PASSWORD_FILE -ErrorAction SilentlyContinue
            }
            $CopyStatus = " and Copy"
        }

        # 3. Очистка и Проверка
        $CurrentStage = "Maintenance (Forget/Check)"
        $ForgetArgs = @("forget") + $RetentionPolicy + @("--prune")
        Invoke-ResticLogged -Arguments $ForgetArgs
        $CheckArgs = @("check")
        Invoke-ResticLogged -Arguments $CheckArgs

        Add-Content $LogFile "$(Get-Date): --- Done ---"
        Send-Ntfy "✅ Backup$CopyStatus on $($env:COMPUTERNAME) completed successfully." "Success" "heavy_check_mark"
    }
    finally {
        $ErrorActionPreference = "Stop"
    }
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
