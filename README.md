# 📷 Camera Auto Configurator

Инструмент для автоматической массовой настройки IP-камер через HTTP API.  
Поддерживает несколько моделей камер с разными протоколами — через систему профилей.

---

## Поддерживаемые модели

| Модель | Профиль | Авторизация | Протокол |
|---|---|---|---|
| CROSS (RTL838x) | `cross` | Custom POST + encrypt1 | XML form-data |
| Apix VDome E8 EXT | `apix_e8` | HTTP Digest MD5 | JSON PUT (LAPI v1.0) |
| Apix VDome S8 Panoramic | `apix_s8` | HTTP Digest | JSON POST (CGI) |

---

## Возможности

- ✅ Настройка **сетевых параметров** (IP, маска, шлюз, DNS)
- ✅ Настройка **часового пояса**
- ✅ Настройка **NTP-сервера**
- ✅ Настройка **детектора движения** (расписание, чувствительность, сетка зон)
- ✅ **Массовая настройка** — список камер или диапазон IP (`10.97.242.1-50`)
- ✅ **Интерактивное меню** — выбор нужных разделов перед применением
- ✅ **Профили моделей** — добавление новой модели без изменения кода

---

## Структура проекта

```
camera-auto/
├── camera_configurator.py   # основной скрипт
├── camera_config.yaml       # настройки и список камер
└── profiles/
    ├── cross.yaml           # профиль CROSS
    ├── apix_e8.yaml         # профиль Apix E8
    └── apix_s8.yaml         # профиль Apix S8
```

---

## Установка

**Требования:** Python 3.10+

```bash
git clone https://github.com/ВАШ_НИК/camera-auto.git
cd camera-auto
pip install requests pyyaml questionary
```

---

## Настройка

Отредактируй `camera_config.yaml`:

### Список камер

```yaml
cameras:
  # Диапазон IP — развернётся в 50 камер
  - ip: "10.97.242.1-50"
    username: "admin"
    password: "ВАШ_ПАРОЛЬ"
    profile: cross

  # Одиночная камера с другой моделью
  - ip: "10.81.238.34"
    username: "Admin"
    password: "ВАШ_ПАРОЛЬ"
    profile: apix_e8
```

### Сетевые параметры

```yaml
network:
  ip_address:  "10.97.242.219"
  subnet_mask: "255.255.252.0"
  gateway:     "10.97.240.1"
  dns_main:    "8.8.8.8"
  dns_spare:   "192.168.0.2"
  dhcp:        false
  mtu:         1500
```

### Часовой пояс

Каждая модель использует свой формат — укажи все три:

```yaml
timezone:
  timezone:      "(GMT+03:00) Москва"   # CROSS
  timezone_int:  45                      # Apix E8 (целое число, уточни по вебке)
  timezone_utc:  "UTC+3"                 # Apix S8
  timezone_name: "MSK"                   # Apix S8
  language:      3
```

### NTP

```yaml
ntp:
  server:   "10.97.200.6"
  port:     123
  enabled:  true
  interval: 3600   # секунды
```

### Детектор движения

```yaml
motion:
  enabled:     true
  sensitivity: 5        # 1–10
  record:      true
  smtp:        false
  ftp:         false
  # schedule:           # пусто = круглосуточно все дни
  # grid_data:          # пусто = вся зона активна
```

---

## Запуск

```bash
python camera_configurator.py
```

После запуска появится интерактивное меню:

```
? Выберите разделы для применения:
  (↑↓ — навигация, Пробел — выбор, Enter — подтвердить)
 ❯ ● Сеть
   ● Часовой пояс
   ● NTP
   ● Детектор движения
```

Пробел — вкл/выкл раздел, Enter — применить к камерам.

**Пример вывода:**

```
Камер для настройки: 52

[1/52]
==========================================
  Камера : 10.97.242.1
  Профиль: cross
==========================================
→ Подключение...
  ✓ Session key: mYMMdhUYKg
  ✓ DeviceID: 03A4F1
→ Сеть...
  ✓ Ответ: 0||Выполнено
→ Детектор движения...
  ✓ Ответ: 0||Выполнено
  Готово.

==========================================
  Итог: 51/52 камер обработано
==========================================
```

---

## Добавление новой модели

1. Перехвати HTTP-запросы с помощью **mitmproxy**:
   ```bash
   mitmweb --listen-port 8080
   # В Firefox: прокси 127.0.0.1:8080
   ```
   Нужно поймать запросы: авторизация, сеть, таймзона, NTP, детектор движения.

2. Создай профиль `profiles/новая_модель.yaml`:
   ```yaml
   driver: новая_модель
   auth:
     method: digest   # digest | custom_encrypt
   endpoints:
     network:
       url: /api/v1/network
       method: PUT
     ...
   ```

3. Добавь класс драйвера в `camera_configurator.py`:
   ```python
   class НоваяМодельDriver(BaseDriver):
       def connect(self)        -> bool: ...
       def apply_network(self)  -> bool: ...
       def apply_timezone(self) -> bool: ...
       def apply_ntp(self)      -> bool: ...
       def apply_motion(self)   -> bool: ...
   ```

4. Зарегистрируй в словаре `DRIVERS`:
   ```python
   DRIVERS = {
       "cross":        CrossDriver,
       "apix_e8":      ApixE8Driver,
       "apix_s8":      ApixS8Driver,
       "новая_модель": НоваяМодельDriver,  # ← добавить
   }
   ```

5. Укажи профиль в `camera_config.yaml`:
   ```yaml
   cameras:
     - ip: "192.168.1.100"
       username: "admin"
       password: "pass"
       profile: новая_модель
   ```

---

## Зависимости

| Библиотека | Назначение |
|---|---|
| `requests` | HTTP-запросы, Digest-авторизация |
| `pyyaml` | Чтение конфигов |
| `questionary` | Интерактивное меню в консоли |

---
