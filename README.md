# Интернет-магазин (React + Vite) + Python backend + Telegram-бот

Проект состоит из двух частей:

```
project/
├── src/                  # React (существующий магазин)
├── public/
├── backend/              # Python backend (новый)
│   ├── server.py         # FastAPI: POST /api/order
│   ├── bot.py            # Форматирование и отправка заказа в Telegram
│   ├── requirements.txt  # Python-зависимости
│   ├── run_backend.bat   # Наблюдатель: автозапуск + автorestart сервера
│   ├── start_backend_hidden.vbs  # Скрытый запуск (без окна консоли)
│   ├── server.log        # Лог backend (создаётся автоматически)
│   └── .env              # Секреты (TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID) — не коммитится
├── .env                  # Frontend: VITE_API_URL
├── package.json
└── .gitignore
```

**Как работает отправка заказа:**

```
React Cart → POST /api/order (Python/FastAPI) → Telegram Bot API → ваш Telegram
```

React НИКОГДА не обращается к Telegram напрямую и не содержит токен бота.

---

## 1. Установка Python

Скачайте Python 3.10+ с https://www.python.org/downloads/ и установите.

**Windows:** при установке обязательно поставьте галочку **"Add python.exe to PATH"**.

Проверка в терминале:

```bash
python --version
python -m pip --version
```

## 2. Установка зависимостей backend

Из корня проекта:

```bash
python -m pip install -r backend/requirements.txt
```

## 3. Создание `backend/.env`

Файл `backend/.env` уже создан в проекте. Если его нет — создайте рядом с `server.py`:

```
TELEGRAM_BOT_TOKEN=YOUR_TOKEN
TELEGRAM_CHAT_ID=YOUR_CHAT_ID
```

`.env` добавлен в `.gitignore` и не попадёт в git.

## 4. Куда вставить Telegram Bot Token

В строку `TELEGRAM_BOT_TOKEN=` файла `backend/.env`.

Токен выдаёт [@BotFather](https://t.me/BotFather) в Telegram (команда `/newbot`).
Токен выглядит примерно так: `1234567890:AAH...`.

**Никогда** не вставляйте токен в React-код, во frontend-файлы или в git.

## 5. Куда вставить Telegram Chat ID

В строку `TELEGRAM_CHAT_ID=` файла `backend/.env`.

Как узнать свой chat id:

1. Откройте своего бота в Telegram и нажмите **Start** (отправьте ему любое сообщение).
2. Откройте в браузере:
   `https://api.telegram.org/bot<ВАШ_ТОКЕН>/getUpdates`
3. В ответе найдите `"chat":{"id": <ВАШ_CHAT_ID>, ...}` — это и есть ваш Chat ID.
4. Вставьте число в `TELEGRAM_CHAT_ID=` и перезапустите backend.

## 6. Запуск Python backend

Из корня проекта:

```bash
python backend/server.py
```

Или:

```bash
cd backend
python -m uvicorn server:app --host 0.0.0.0 --port 8000
```

Сервер запустится на **http://localhost:8000**.
Проверка: откройте http://localhost:8000 — должен вернуться `{"status":"ok",...}`.

CORS уже настроен: React (http://localhost:5173 и любой localhost-порт) может отправлять запросы.

### Автозапуск: заказы приходят даже если закрыть VS Code

Backend работает как **отдельный фоновый процесс Windows** и не зависит от VS Code/терминала:

1. **`backend/run_backend.bat`** — «наблюдатель»: следит за портом 8000, сам запускает `server.py` и перезапускает его через ~5 секунд после любого падения. Несколько наблюдателей могут работать одновременно — не конфликтуют.
2. **`backend/start_backend_hidden.vbs`** — запускает наблюдателя скрыто (без чёрного окна консоли).
3. **Автозагрузка** — копия `start_backend_hidden.vbs` установлена в:
   `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup`
   — backend стартует автоматически при каждом входе в Windows.

Проверить: откройте http://localhost:8000 — должен вернуться `{"status":"ok",...}`.
Логи работы: `backend/server.log`.

**Остановить полностью:**
- Диспетчер задач → завершить процессы `python.exe` (server.py) и `cmd.exe` с `run_backend.bat`;
- удалить `start_backend_hidden.vbs` из папки Автозагрузки (`shell:startup`).

**Запустить вручную:**

```bash
wscript backend\start_backend_hidden.vbs   # скрыто, фоном (как в автозагрузке)
python backend/server.py                   # в консоли (удобно для отладки)
```

> Автозагрузка срабатывает **при входе в систему**. Если нужен запуск ещё до входа (сразу после загрузки Windows) — создайте задачу Планировщика `ONSTART` из терминала **от имени администратора** (прав у обычной учётки недостаточно). Для работы магазина автозагрузки достаточно.

## 7. Запуск React

В отдельном терминале, из корня проекта:

```bash
npm install
npm run dev
```

Откройте http://localhost:5173.

Frontend `.env` (создан в корне):

```
VITE_API_URL=http://localhost:8000
```

React отправляет заказ в `${VITE_API_URL}/api/order`. После изменения `.env` перезапустите `npm run dev`.

## 8. Как проверить оформление заказа

### Через сайт

1. Запустите backend (пункт 6) и frontend (пункт 7).
2. Добавьте товары в корзину, откройте корзину.
3. Нажмите **«Оформить заказ»** — откроется страница оформления.
4. Заполните **Имя**, **Телефон**, **Адрес** (улица и/или город).
5. Нажмите **«Оформить заказ»** в блоке итога.
6. Успех: появится **«Заказ успешно отправлен! ✅»**, корзина очистится, придёт Telegram-сообщение.
7. Ошибка: **«Не удалось отправить заказ. Попробуйте ещё раз.»** (корзина НЕ очищается).

### Через API напрямую

PowerShell:

```powershell
$body = @{
  items   = @(@{ name = "iPhone 15"; price = 800; quantity = 2; total = 1600 })
  total   = 1600
  name    = "Ali"
  phone   = "+998901234567"
  address = "Toshkent"
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri http://localhost:8000/api/order -Method Post -ContentType "application/json" -Body $body
```

curl / macOS / Linux:

```bash
curl -X POST http://localhost:8000/api/order \
  -H "Content-Type: application/json" \
  -d '{"items":[{"name":"iPhone 15","price":800,"quantity":2,"total":1600}],"total":1600,"name":"Ali","phone":"+998901234567","address":"Toshkent"}'
```

Пример сообщения в Telegram:

```
🛍 НОВЫЙ ЗАКАЗ

📦 ТОВАРЫ:

• iPhone 15
  2 шт. × $800 = $1600

━━━━━━━━━━━━
💰 ИТОГО: $1600

👤 Имя: Ali
📞 Телефон: +998901234567
📍 Адрес: Toshkent

🕐 Время: 06.10.2026 14:30
```

### Возможные ошибки

| Ответ | Причина |
|---|---|
| HTTP 422 | Не заполнены/некорректны поля заказа |
| HTTP 502, `TELEGRAM_CHAT_ID is not configured` | В `backend/.env` не указан Chat ID (пункт 5) |
| HTTP 502, `Telegram API error` | Неверный токен или бот заблокирован |

---

## Полезные ссылки

- Swagger-документация backend: http://localhost:8000/docs
- Health-check: http://localhost:8000/

---

## Оригинальный шаблон (React + Vite)

This template provides a minimal setup to get React working in Vite with HMR and some ESLint rules.

Currently, two official plugins are available:

- [@vitejs/plugin-react](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react) uses [Oxc](https://oxc.rs/)
- [@vitejs/plugin-react-swc](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react-swc) uses [SWC](https://swc.rs/)

## Expanding the ESLint configuration

If you are developing a production application, we recommend using TypeScript with type-aware lint rules enabled. Check out the [TS template](https://github.com/vitejs/vite/tree/main/packages/create-vite/template-react-ts) for information on how to integrate TypeScript and [`typescript-eslint`](https://typescript-eslint.io/).
