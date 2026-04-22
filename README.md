# Restic + Rclone: Система бэкапов

Резервное копирование для Windows и Linux с защитой данных и оповещениями.

---

## Основные возможности

*   **Экономия места:** дедупликация данных (хранение только уникальных частей файлов).
*   **Безопасность:** шифрование и защита от вирусов-вымогателей.
*   **Гибкость:** поддержка облачных сервисов (Google Drive, S3) и локальных носителей.
*   **Контроль:** оповещения через Ntfy.sh и ведение журналов (логов).

---

## Надежность скриптов

Сценарии включают механизмы для немедленной остановки при возникновении ошибок:
*   **Linux (Bash):** использование `set -euo pipefail`.
*   **Windows (PowerShell):** настройка `$ErrorActionPreference = 'Stop'`.

---

## 1. Установка программ

### Windows (через Scoop)
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
scoop install restic rclone
```

### Linux (через пакетный менеджер)
```bash
sudo apt update && sudo apt install restic rclone -y
```

---

## 2. Настройка системы

*   **[Облако](scenarios/00-rclone-config.md):** создание и проверка подключений в Rclone.
*   **[Автоматизация](scenarios/00-automation.md):** настройка расписания через Cron, Systemd или Планировщик Windows.
*   **[Мониторинг](scenarios/00-monitoring-ntfy.md):** работа с оповещениями через Ntfy.sh.

---

## 3. Сценарии использования

*   **[01. Облачное зеркало](scenarios/01-universal-rclone.md):** простая синхронизация папок с облаком через Rclone (без версий).
*   **[02. Универсальный Restic](scenarios/02-universal-restic.md):** основной скрипт для бэкапов. 
    *   **Локально:** бэкап на диски, флешки или SFTP.
    *   **В облако:** бэкап напрямую через Rclone.
    *   **Гибридно:** сначала локальный бэкап, затем копия в облако (`restic copy`).

---

## 4. Обслуживание

*   **[Управление данными](scenarios/00-manage-data.md):** просмотр списка копий, восстановление, ротация данных и монтирование репозитория как диска.

---

## Ссылки на проекты
*   **Restic:** [Официальный сайт](https://restic.net/) | [GitHub](https://github.com/restic/restic)
*   **Rclone:** [Официальный сайт](https://rclone.org/) | [GitHub](https://github.com/rclone/rclone)
