# Автоматизация бэкапов

Настройка автоматического запуска скриптов в Linux и Windows.

---

## 1. Linux: Планировщик Cron

Запуск задач по расписанию.

* **Выбор пользователя:**
    * `sudo crontab -e` — настройка задач от имени суперпользователя (root). Требуется для доступа к системным папкам, базам данных и файлам других пользователей.
    * `crontab -e` — настройка задач текущего пользователя. Подходит для бэкапа личных документов и настроек.
* **Пример записи (каждый день в 03:00):**
  ```bash
  0 3 * * * /path/to/your/backup.sh
  ```
* **Особенности:**
    * **Логи:** Если в скрипте настроена запись в `LOG_FILE`, внешнее перенаправление (`>> /log/file`) в Cron не требуется. Это помогает избежать дублирования информации.
    * **Права:** Скрипт помечается как исполняемый (`chmod +x backup.sh`).
    * **Переменные:** Cron имеет ограниченный набор переменных окружения. В скриптах используются полные пути к программам и файлам.

---

## 2. Linux: Таймеры Systemd

Автоматизация с разделением логики запуска и расписания.

### Принцип работы:
* **Сервис (`.service`):** Описывает задачу (что запускать).
* **Таймер (`.timer`):** Описывает расписание (когда запускать).
* **Связь:** Таймер автоматически управляет сервисом с таким же именем.

### Расположение конфигураций:
* **Системные (от имени root):** `/etc/systemd/system/`
* **Пользовательские:** `~/.config/systemd/user/`

### Файл сервиса (`backup-restic.service`):
```ini
[Unit]
Description=Restic Backup Service
# Ожидание доступности сети перед запуском
After=network.target

[Service]
# Тип oneshot подходит для задач, которые завершаются после выполнения
Type=oneshot
# Полный путь к скрипту бэкапа
ExecStart=/path/to/your/backup.sh
```

### Файл таймера (`backup-restic.timer`):
```ini
[Unit]
Description=Run Restic Backup Daily

[Timer]
# Запуск раз в сутки. Для отладки (каждую минуту) используется: *:0/1
OnCalendar=daily
# Запуск задачи сразу, если время было пропущено (например, компьютер был выключен)
Persistent=true

[Install]
# Привязка к системным таймерам
WantedBy=timers.target
```

**Активация:** `systemctl enable --now backup-restic.timer`

---

## 3. Windows: Планировщик задач

Запуск скриптов PowerShell в фоновом режиме.

### Параметры в интерфейсе:
1. **Действие:** запуск программы (`powershell.exe`).
2. **Аргументы:** `-ExecutionPolicy Bypass -WindowStyle Hidden -File "C:\Scripts\backup.ps1"`
3. **Права:** «Выполнять с наивысшими правами» (для доступа к VSS).

### Создание задачи через PowerShell:
```powershell
$action = New-ScheduledTaskAction -Execute 'powershell.exe' `
    -Argument '-ExecutionPolicy Bypass -WindowStyle Hidden -File "C:\Scripts\backup.ps1"'

$trigger = New-ScheduledTaskTrigger -Daily -At 3am

Register-ScheduledTask -Action $action -Trigger $trigger -TaskName "DailyBackup" -User "SYSTEM" -RunLevel Highest
```
