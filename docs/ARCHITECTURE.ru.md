# Архитектура

Этот документ описывает архитектуру и сценарии установки, поддерживаемые remnawave-installer.

## Сценарии установки

### Вариант 1: Два сервера (Панель и Нода раздельно)

Рекомендуется для надёжности и гибкости.

**Сервер Панели:**
- Remnawave Panel
- Subscription-Page
- Caddy (для панели и подписок)
- PostgreSQL, Redis

**Сервер Ноды:**
- Remnawave Node (с Xray)
- Caddy для домена Selfsteal

**Настройка DNS:**
- Домены панели и подписок → IP сервера панели
- Домен Selfsteal → IP сервера ноды

**Порядок установки:**
1. На сервере панели: выберите "Panel Only". Сохраните публичный ключ (`SSL_CERT="..."`)
2. На сервере ноды: выберите "Node only". Введите домен selfsteal, IP панели и сохранённый ключ

### Вариант 2: Всё-в-одном (Панель и Нода на одном сервере)

Упрощённый вариант для тестирования или малых нагрузок.

**Один сервер содержит:**
- Remnawave Panel, Node, Subscription-Page
- Caddy, PostgreSQL, Redis

**DNS:** Все три домена (разные!) → IP этого сервера

### Маршрутизация трафика (режим Всё-в-одном)

```
Клиент → Порт 443 → Xray (локальная Remnawave Node)
                      ├─ (VLESS прокси-трафик) → Обрабатывается Xray
                      └─ (Не-VLESS трафик, fallback) → Caddy (порт 9443)
                                                        ├─ SNI: Домен панели → Remnawave Panel (порт 3000)
                                                        ├─ SNI: Домен подписок → Subscription Page (порт 3010)
                                                        └─ SNI: Домен Selfsteal → Статическая HTML-страница
```

> **Примечание**: В режиме Всё-в-одном, если остановить ноду или сломать конфиг Xray, панель и другие веб-сервисы станут недоступны через доменные имена.

## Структура директорий

### Установка панели (`/opt/remnawave/`)
```
/opt/remnawave/
├── .env                    # Переменные окружения панели
├── docker-compose.yml      # Сервисы панели
├── credentials.txt         # Сгенерированные учётные данные
├── Makefile                # Управление сервисами
├── caddy/
│   ├── Caddyfile          # Конфигурация Caddy
│   ├── docker-compose.yml
│   └── html/              # Статические файлы
├── subscription-page/
│   └── docker-compose.yml
└── node/                   # (только Всё-в-одном)
    ├── .env
    └── docker-compose.yml
```

### Отдельная установка ноды (`/opt/remnanode/`)
```
/opt/remnanode/
├── .env
├── docker-compose.yml
├── Makefile
└── selfsteal/
    ├── Caddyfile
    ├── docker-compose.yml
    └── html/
```

## Защита доступа к панели

### SIMPLE Cookie Security
- Доступ по URL с секретным ключом: `https://panel.example.com/auth/login?caddy=SECRET`
- Caddy устанавливает cookie при первом посещении
- Без валидного cookie/параметра показывается страница-заглушка Selfsteal

### FULL Caddy Security (рекомендуется)
- Использует образ `remnawave/caddy-with-auth` с модулем `caddy-security`
- **Двухуровневая аутентификация:**
  1. Caddy Auth Portal (логин/пароль + MFA при первом входе)
  2. Вход в Remnawave Panel
- Панель доступна по случайному пути: `https://panel.example.com/<RANDOM_PATH>/auth`

## Сетевая конфигурация

### Открытые порты
| Порт | Назначение |
|------|------------|
| 80/tcp | Caddy HTTP |
| 443/tcp | Caddy HTTPS / Xray |
| 22/tcp | SSH (или ваш порт) |
| 2222/tcp | API ноды (ограничен) |

### Ограничения порта 2222
- **Всё-в-одном**: Открыт только для 172.30.0.0/16 (подсеть Docker)
- **Отдельная нода**: Открыт только для IP панели
