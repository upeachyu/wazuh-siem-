# Wazuh SIEM — развёртывание и первые детекты в домашней лаборатории

**Автор:** upeachyu
**Дата:** ноябрь 2026
**Стек:** Wazuh 4.9 , Ubuntu Server, Windows Server 2022 (Evaluation), QEMU/KVM

## Цель

Поднять собственную SIEM-систему в изолированной лаборатории и пройти полный цикл: развёртывание → подключение агентов → настройка аудита → анализ первых алертов. Практика для позиции SOC-аналитика (L1).

## Топология стенда

- Хост: Ubuntu (16 ГБ RAM), гипервизор QEMU/KVM (virt-manager).
- Сеть: libvirt NAT (192.168.122.0/24), проброс порта 8443 → Wazuh Dashboard (443).

| VM | ОС | Роль |
|---|---|---|
| wazuh-server | Ubuntu Server | Wazuh All-in-One (Indexer + Manager + Dashboard) |
| ubuntu-lab | Ubuntu Server | Агент Wazuh (Linux) |
| win-dc | Windows Server 2022 | Контроллер домена `lab.local`, агент Wazuh (Windows) |

## Развёртывание Wazuh

1. Установка All-in-One:
   ```bash
   curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
   sudo bash ./wazuh-install.sh -a
   ```
2. Сброс пароля `admin` через `wazuh-passwords-tool.sh`.
3. Проброс порта 8443 на хосте (`iptables DNAT`) для доступа к Dashboard.

## Подключение агентов

- **Linux (`ubuntu-lab`):** регистрация агента через `manage_agents`, установка `.deb` с указанием IP менеджера.
- **Windows (`win-dc`):** добавление агента через Dashboard (Deploy new agent), установка `.msi`.

## Настройка аудита Windows

- Включён аудит создания процессов:
  ```powershell
  auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
  ```
- Включён вывод командной строки: `ProcessCreationIncludeCmdLine_Enabled = 1`.

## Первые детекты

| Event ID | Событие | Правило Wazuh | Уровень | Наблюдение |
|---|---|---|---|---|
| 4624 | Успешный вход | 60106 | 3 (low) | 116 срабатываний — в основном служебные входы, шум |
| 4625 | Неудачный вход | 60122 | 5 (medium) | 2 срабатывания — подозрительно, но не брутфорс |
| 4688 | Создание процесса | 67027 (WEF baseline) | 3 (low) | процесс + командная строка; описание «A process was created» без имени процесса — точка для тюнинга |
| 4720 | Создание учётной записи | 60109 | 8 (medium) | наиболее значимое событие из этой группы |

Помимо этого, в алертах Linux-агента присутствовали ложные срабатывания (AppArmor DENIED, Dpkg). Это иллюстрирует ключевую задачу SOC L1 — триаж: из сотен алертов реально значимых единицы.

## Выводы

1. SIEM-стек развёрнут и собирает события с Linux и Windows.
2. Windows-события корректно маппятся в правила Wazuh с уровнями важности.
3. Ключевой навык — триаж: отделение шума (4624, AppArmor, Dpkg) от значимых событий (4720, брутфорс).

## Дальнейшие шаги

- Брутфорс SSH/RDP через `hydra` (Kali) → алерт уровня 10+.
- Маппинг детектов на MITRE ATT&CK.
- Написание собственных правил корреляции и снижение числа ложных срабатываний.
- Кастомизация правила 67027: по умолчанию в описании нет имени процесса («A process was created»). Добавить `win.eventdata.newProcessName` и `commandLine` через локальное правило/оверрайд — это ускоряет триаж.
