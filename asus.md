

# Полная Ключ-Система с Чекпоинтами через Linkvertise

## Архитектура

```
┌─────────────────┐     ┌────────────────────┐     ┌─────────────┐
│  Roblox Script  │────▶│  API /key/validate  │◀───▶│   MySQL DB  │
│  (проверка)     │     │                    │     │             │
└────────┬────────┘     │  semyaware.ru      │     │  sessions   │
         │              │                    │     │  keys_table │
         │ открывает    │  /api/session      │     │  settings   │
         ▼              │  /api/checkpoint   │     └─────────────┘
┌─────────────────┐     │  /api/key          │
│  getkey.html    │────▶│                    │
│  (фронтенд)    │     └────────┬───────────┘
└────────┬────────┘              │
         │                       │  генерирует URL
         ▼                       ▼
┌─────────────────────────────────────────┐
│          Linkvertise                     │
│  Чекпоинт 1 → Чекпоинт 2 → Чекпоинт 3 │
│          ↓ callback на каждом шаге       │
└─────────────────────────────────────────┘
```

---

## 1. Подготовка проекта

### Структура файлов

```
key-system/
├── server.js
├── .env
├── package.json
├── schema.sql
├── public/
│   └── getkey.html
└── lua/
    └── keysystem.lua
```

### package.json

```json
{
  "name": "semyaware-keysystem",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mysql2": "^3.6.0",
    "cors": "^2.8.5",
    "express-rate-limit": "^7.1.0",
    "dotenv": "^16.3.1"
  }
}
```

```bash
npm install
```

---

## 2. Конфигурация (.env)

```env
PORT=3001
DB_HOST=localhost
DB_USER=keysystem_user
DB_PASS=СильныйПароль123!
DB_NAME=keysystem

# Секретный ключ для подписи токенов (сгенерируй свой!)
SECRET_KEY=a1b2c3d4e5f6g7h8i9j0_CHANGE_THIS_TO_RANDOM_STRING

# Linkvertise
LINKVERTISE_PUBLISHER_ID=123456

# Настройки ключей
SITE_URL=https://semyaware.ru
KEY_PREFIX=SEMYA
KEY_DURATION_HOURS=24
CHECKPOINT_COUNT=3
MIN_CHECKPOINT_DELAY_SECONDS=15
```

---

## 3. База данных (schema.sql)

```sql
CREATE DATABASE IF NOT EXISTS keysystem
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE keysystem;

-- ══════════════════════════════════════
-- Сессии чекпоинтов
-- ══════════════════════════════════════
CREATE TABLE sessions (
    id               INT AUTO_INCREMENT PRIMARY KEY,
    session_id       VARCHAR(64)  NOT NULL UNIQUE,
    hwid             VARCHAR(255) NOT NULL,
    ip_address       VARCHAR(45),
    user_agent       TEXT,

    -- Чекпоинт 1
    checkpoint_1             TINYINT(1) DEFAULT 0,
    checkpoint_1_token       VARCHAR(200),
    checkpoint_1_completed_at DATETIME,

    -- Чекпоинт 2
    checkpoint_2             TINYINT(1) DEFAULT 0,
    checkpoint_2_token       VARCHAR(200),
    checkpoint_2_completed_at DATETIME,

    -- Чекпоинт 3
    checkpoint_3             TINYINT(1) DEFAULT 0,
    checkpoint_3_token       VARCHAR(200),
    checkpoint_3_completed_at DATETIME,

    created_at       DATETIME DEFAULT CURRENT_TIMESTAMP,
    expires_at       DATETIME NOT NULL,

    INDEX idx_sid  (session_id),
    INDEX idx_hwid (hwid),
    INDEX idx_exp  (expires_at)
);

-- ══════════════════════════════════════
-- Сгенерированные ключи
-- ══════════════════════════════════════
CREATE TABLE keys_table (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    key_value   VARCHAR(128) NOT NULL UNIQUE,
    hwid        VARCHAR(255) NOT NULL,
    session_id  VARCHAR(64),
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    expires_at  DATETIME NOT NULL,
    is_active   TINYINT(1) DEFAULT 1,

    INDEX idx_key    (key_value),
    INDEX idx_hwid   (hwid),
    INDEX idx_active (is_active, expires_at)
);

-- ══════════════════════════════════════
-- Лог попыток обхода (опционально)
-- ══════════════════════════════════════
CREATE TABLE bypass_log (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    ip_address VARCHAR(45),
    reason     VARCHAR(255),
    details    TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

```bash
mysql -u root -p < schema.sql
```

---

## 4. Backend — server.js (полный код)

```javascript
/* ═══════════════════════════════════════════════════════
   SemyAware Key System — server.js
   Полная реализация: сессии, чекпоинты, ключи, валидация
   ═══════════════════════════════════════════════════════ */

require('dotenv').config();
const express    = require('express');
const cors       = require('cors');
const crypto     = require('crypto');
const mysql      = require('mysql2/promise');
const rateLimit  = require('express-rate-limit');
const path       = require('path');

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.set('trust proxy', 1);

// ── CORS — разрешаем запросы с сайта и из Roblox-эксплойтов ──
app.use(cors({
    origin: '*',        // для Roblox request() нужен *
    methods: ['GET', 'POST']
}));

// ── Раздача статики ──
app.use(express.static(path.join(__dirname, 'public')));

// ═══════════════════════════════════════
//  КОНФИГУРАЦИЯ
// ═══════════════════════════════════════
const CONFIG = {
    SECRET:           process.env.SECRET_KEY,
    PUBLISHER_ID:     process.env.LINKVERTISE_PUBLISHER_ID,
    SITE_URL:         process.env.SITE_URL,
    KEY_PREFIX:       process.env.KEY_PREFIX            || 'SEMYA',
    KEY_DURATION_H:   parseInt(process.env.KEY_DURATION_HOURS)          || 24,
    CHECKPOINTS:      parseInt(process.env.CHECKPOINT_COUNT)            || 3,
    MIN_DELAY_SEC:    parseInt(process.env.MIN_CHECKPOINT_DELAY_SECONDS)|| 15,
    SESSION_TTL_H:    2,          // сессия живёт 2 часа
    TOKEN_MAX_AGE_S:  600,        // токен действителен 10 минут
};

// Белый список столбцов — защита от SQL-инъекций
const CP_COLUMNS = {};
for (let i = 1; i <= CONFIG.CHECKPOINTS; i++) {
    CP_COLUMNS[i] = {
        status:    `checkpoint_${i}`,
        token:     `checkpoint_${i}_token`,
        completed: `checkpoint_${i}_completed_at`,
    };
}

// ═══════════════════════════════════════
//  БАЗА ДАННЫХ
// ═══════════════════════════════════════
const pool = mysql.createPool({
    host:     process.env.DB_HOST,
    user:     process.env.DB_USER,
    password: process.env.DB_PASS,
    database: process.env.DB_NAME,
    waitForConnections: true,
    connectionLimit: 20,
    charset: 'utf8mb4',
});

// ═══════════════════════════════════════
//  RATE LIMITING
// ═══════════════════════════════════════
const apiLimiter = rateLimit({
    windowMs: 5 * 60 * 1000,     // 5 минут
    max: 60,
    standardHeaders: true,
    message: { success: false, error: 'Слишком много запросов. Подождите 5 минут.' }
});

const validateLimiter = rateLimit({
    windowMs: 1 * 60 * 1000,
    max: 30,
    message: { valid: false, error: 'Rate limit' }
});

app.use('/api/session',    apiLimiter);
app.use('/api/checkpoint', apiLimiter);
app.use('/api/key/validate', validateLimiter);

// ═══════════════════════════════════════
//  УТИЛИТЫ
// ═══════════════════════════════════════

function hmac(data) {
    return crypto.createHmac('sha256', CONFIG.SECRET).update(data).digest('hex');
}

function randomHex(bytes = 32) {
    return crypto.randomBytes(bytes).toString('hex');
}

function generateKey() {
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    const parts = [];
    for (let p = 0; p < 4; p++) {
        let seg = '';
        for (let c = 0; c < 5; c++) {
            seg += chars[crypto.randomInt(chars.length)];
        }
        parts.push(seg);
    }
    return `${CONFIG.KEY_PREFIX}-${parts.join('-')}`;
    // Пример: SEMYA-A3K9F-BX72M-QW5TZ-8NP1L
}

function linkvertiseUrl(targetUrl) {
    const b64  = Buffer.from(targetUrl).toString('base64');
    const rand = crypto.randomInt(100000, 999999);
    return `https://link-to.net/${CONFIG.PUBLISHER_ID}/${rand}/dynamic/?r=${b64}`;
}

function clientIp(req) {
    return (req.headers['x-forwarded-for'] || '').split(',')[0].trim()
        || req.socket?.remoteAddress
        || req.ip;
}

function hoursFromNow(h) {
    return new Date(Date.now() + h * 3600000);
}

// ── Логирование попыток обхода ──
async function logBypass(ip, reason, details = '') {
    try {
        await pool.query(
            'INSERT INTO bypass_log (ip_address, reason, details) VALUES (?,?,?)',
            [ip, reason, details]
        );
    } catch (_) { /* не критично */ }
}

// ═══════════════════════════════════════════════════════
//  1.  POST /api/session/create
//      Создание или возобновление сессии
//      Body: { hwid: string }
// ═══════════════════════════════════════════════════════
app.post('/api/session/create', async (req, res) => {
    try {
        const { hwid } = req.body;
        if (!hwid || typeof hwid !== 'string' || hwid.length < 5 || hwid.length > 255) {
            return res.json({ success: false, error: 'Неверный HWID' });
        }

        const ip = clientIp(req);

        // ── Есть ли действующий ключ? ──
        const [activeKeys] = await pool.query(
            `SELECT key_value, expires_at FROM keys_table
             WHERE hwid = ? AND is_active = 1 AND expires_at > NOW()
             ORDER BY created_at DESC LIMIT 1`,
            [hwid]
        );
        if (activeKeys.length > 0) {
            return res.json({
                success:    true,
                has_key:    true,
                key:        activeKeys[0].key_value,
                expires_at: activeKeys[0].expires_at,
            });
        }

        // ── Есть ли незавершённая сессия? ──
        const [existing] = await pool.query(
            `SELECT * FROM sessions
             WHERE hwid = ? AND expires_at > NOW()
             ORDER BY created_at DESC LIMIT 1`,
            [hwid]
        );

        if (existing.length > 0) {
            const s = existing[0];
            let done = 0;
            for (let i = 1; i <= CONFIG.CHECKPOINTS; i++) {
                if (s[CP_COLUMNS[i].status]) done++;
            }
            return res.json({
                success:      true,
                has_key:      false,
                session_id:   s.session_id,
                current_step: done + 1,
                total_steps:  CONFIG.CHECKPOINTS,
            });
        }

        // ── Новая сессия ──
        const sessionId = randomHex(32);
        await pool.query(
            `INSERT INTO sessions (session_id, hwid, ip_address, user_agent, expires_at)
             VALUES (?, ?, ?, ?, ?)`,
            [sessionId, hwid, ip, req.headers['user-agent'] || '', hoursFromNow(CONFIG.SESSION_TTL_H)]
        );

        res.json({
            success:      true,
            has_key:      false,
            session_id:   sessionId,
            current_step: 1,
            total_steps:  CONFIG.CHECKPOINTS,
        });

    } catch (err) {
        console.error('[session/create]', err);
        res.status(500).json({ success: false, error: 'Ошибка сервера' });
    }
});

// ═══════════════════════════════════════════════════════
//  2.  POST /api/checkpoint/start
//      Генерирует Linkvertise-ссылку для текущего шага
//      Body: { session_id, step }
// ═══════════════════════════════════════════════════════
app.post('/api/checkpoint/start', async (req, res) => {
    try {
        const { session_id, step } = req.body;
        const stepNum = parseInt(step);

        // ── Валидация входных данных ──
        if (!session_id || isNaN(stepNum) || stepNum < 1 || stepNum > CONFIG.CHECKPOINTS) {
            return res.json({ success: false, error: 'Неверные параметры' });
        }

        const [rows] = await pool.query(
            'SELECT * FROM sessions WHERE session_id = ? AND expires_at > NOW()',
            [session_id]
        );
        if (rows.length === 0) {
            return res.json({ success: false, error: 'Сессия не найдена или истекла' });
        }

        const session = rows[0];
        const col     = CP_COLUMNS[stepNum];

        // ── Уже пройден? ──
        if (session[col.status]) {
            return res.json({ success: false, error: 'Этот чекпоинт уже пройден' });
        }

        // ── Предыдущий чекпоинт должен быть пройден ──
        if (stepNum > 1 && !session[CP_COLUMNS[stepNum - 1].status]) {
            return res.json({ success: false, error: 'Сначала пройдите предыдущий чекпоинт' });
        }

        // ── Минимальная задержка между чекпоинтами ──
        if (stepNum > 1) {
            const prevDone = session[CP_COLUMNS[stepNum - 1].completed];
            if (prevDone) {
                const elapsed = (Date.now() - new Date(prevDone).getTime()) / 1000;
                if (elapsed < CONFIG.MIN_DELAY_SEC) {
                    const wait = Math.ceil(CONFIG.MIN_DELAY_SEC - elapsed);
                    return res.json({ success: false, error: `Подождите ${wait} сек.`, wait });
                }
            }
        }

        // ── Создаём подписанный токен ──
        const ts    = Date.now().toString();
        const token = hmac(`${session_id}:${stepNum}:${ts}:${session.hwid}`);

        // Сохраняем токен в БД (одноразовый)
        await pool.query(
            `UPDATE sessions SET ${col.token} = ? WHERE session_id = ?`,
            [`${token}:${ts}`, session_id]
        );

        // ── Формируем callback URL ──
        const callback = `${CONFIG.SITE_URL}/api/checkpoint/verify`
            + `?s=${stepNum}&t=${token}&ts=${ts}&sid=${session_id}`;

        // ── Оборачиваем в Linkvertise ──
        const url = linkvertiseUrl(callback);

        res.json({ success: true, url, step: stepNum });

    } catch (err) {
        console.error('[checkpoint/start]', err);
        res.status(500).json({ success: false, error: 'Ошибка сервера' });
    }
});

// ═══════════════════════════════════════════════════════
//  3.  GET /api/checkpoint/verify
//      Callback после прохождения Linkvertise
//      Query: ?s=STEP&t=TOKEN&ts=TIMESTAMP&sid=SESSION_ID
// ═══════════════════════════════════════════════════════
app.get('/api/checkpoint/verify', async (req, res) => {
    const redir = (msg) => res.redirect(`${CONFIG.SITE_URL}/getkey.html?error=${encodeURIComponent(msg)}`);
    const ok    = (sid, extra = '') =>
        res.redirect(`${CONFIG.SITE_URL}/getkey.html?session=${sid}${extra}`);

    try {
        const { s, t: token, ts, sid } = req.query;
        const stepNum = parseInt(s);
        const ip      = clientIp(req);

        // ── Базовая проверка параметров ──
        if (!s || !token || !ts || !sid || isNaN(stepNum)
            || stepNum < 1 || stepNum > CONFIG.CHECKPOINTS) {
            await logBypass(ip, 'invalid_params', JSON.stringify(req.query));
            return redir('Неверные параметры');
        }

        // ── Загружаем сессию ──
        const [rows] = await pool.query(
            'SELECT * FROM sessions WHERE session_id = ? AND expires_at > NOW()',
            [sid]
        );
        if (rows.length === 0) {
            return redir('Сессия истекла');
        }
        const session = rows[0];
        const col     = CP_COLUMNS[stepNum];

        // ── Уже пройден ──
        if (session[col.status]) {
            return ok(sid, '&info=already_done');
        }

        // ── Проверяем подпись токена ──
        const expected = hmac(`${sid}:${stepNum}:${ts}:${session.hwid}`);
        if (token !== expected) {
            await logBypass(ip, 'invalid_token', `sid=${sid} step=${stepNum}`);
            return redir('Неверный токен');
        }

        // ── Проверяем совпадение с сохранённым токеном (одноразовость) ──
        const stored = session[col.token];
        if (!stored || stored !== `${token}:${ts}`) {
            await logBypass(ip, 'token_reuse', `sid=${sid} step=${stepNum}`);
            return redir('Токен уже использован');
        }

        // ── Проверяем возраст токена ──
        const age = (Date.now() - parseInt(ts)) / 1000;
        if (age > CONFIG.TOKEN_MAX_AGE_S) {
            return redir('Токен истёк, попробуйте снова');
        }

        // ── Anti-bypass: минимальное время прохождения ──
        if (age < CONFIG.MIN_DELAY_SEC) {
            await logBypass(ip, 'too_fast', `age=${age.toFixed(1)}s sid=${sid} step=${stepNum}`);
            return redir('Слишком быстро. Пройдите Linkvertise полностью.');
        }

        // ── Помечаем чекпоинт пройденным ──
        await pool.query(
            `UPDATE sessions
             SET ${col.status} = 1,
                 ${col.completed} = NOW(),
                 ${col.token} = NULL
             WHERE session_id = ?`,
            [sid]
        );

        // ── Все чекпоинты пройдены? ──
        // Перечитываем сессию после обновления
        const [updated] = await pool.query(
            'SELECT * FROM sessions WHERE session_id = ?',
            [sid]
        );
        const sess = updated[0];
        let allDone = true;
        for (let i = 1; i <= CONFIG.CHECKPOINTS; i++) {
            if (!sess[CP_COLUMNS[i].status] && i !== stepNum) {
                // stepNum мы только что обновили, но перечитали из БД
            }
            if (!sess[CP_COLUMNS[i].status]) { allDone = false; break; }
        }

        if (allDone) {
            // ── Генерируем ключ ──
            const keyValue  = generateKey();
            const expiresAt = hoursFromNow(CONFIG.KEY_DURATION_H);

            // Деактивируем старые ключи этого HWID
            await pool.query(
                'UPDATE keys_table SET is_active = 0 WHERE hwid = ?',
                [sess.hwid]
            );

            await pool.query(
                `INSERT INTO keys_table (key_value, hwid, session_id, expires_at)
                 VALUES (?, ?, ?, ?)`,
                [keyValue, sess.hwid, sid, expiresAt]
            );

            return ok(sid, '&key_ready=1');
        }

        return ok(sid, `&done=${stepNum}`);

    } catch (err) {
        console.error('[checkpoint/verify]', err);
        return redir('Ошибка сервера');
    }
});

// ═══════════════════════════════════════════════════════
//  4.  POST /api/checkpoint/status
//      Текущий прогресс сессии
//      Body: { session_id }
// ═══════════════════════════════════════════════════════
app.post('/api/checkpoint/status', async (req, res) => {
    try {
        const { session_id } = req.body;
        if (!session_id) return res.json({ success: false });

        const [rows] = await pool.query(
            'SELECT * FROM sessions WHERE session_id = ? AND expires_at > NOW()',
            [session_id]
        );
        if (rows.length === 0) {
            return res.json({ success: false, error: 'Сессия не найдена' });
        }

        const s = rows[0];
        const steps = [];
        let done = 0;

        for (let i = 1; i <= CONFIG.CHECKPOINTS; i++) {
            const completed = !!s[CP_COLUMNS[i].status];
            if (completed) done++;
            steps.push({
                step: i,
                completed,
                completed_at: s[CP_COLUMNS[i].completed] || null,
            });
        }

        // Если всё пройдено — отдаём ключ
        let key = null;
        let expires_at = null;
        if (done === CONFIG.CHECKPOINTS) {
            const [keys] = await pool.query(
                `SELECT key_value, expires_at FROM keys_table
                 WHERE hwid = ? AND is_active = 1 AND expires_at > NOW()
                 ORDER BY created_at DESC LIMIT 1`,
                [s.hwid]
            );
            if (keys.length > 0) {
                key = keys[0].key_value;
                expires_at = keys[0].expires_at;
            }
        }

        res.json({
            success:      true,
            session_id:   s.session_id,
            steps,
            current_step: Math.min(done + 1, CONFIG.CHECKPOINTS),
            total_steps:  CONFIG.CHECKPOINTS,
            all_done:     done === CONFIG.CHECKPOINTS,
            key,
            expires_at,
        });

    } catch (err) {
        console.error('[checkpoint/status]', err);
        res.status(500).json({ success: false, error: 'Ошибка сервера' });
    }
});

// ═══════════════════════════════════════════════════════
//  5.  POST /api/key/validate
//      Валидация ключа (вызывается из Roblox-скрипта)
//      Body: { key, hwid }
// ═══════════════════════════════════════════════════════
app.post('/api/key/validate', async (req, res) => {
    try {
        const { key, hwid } = req.body;

        if (!key || !hwid) {
            return res.json({ valid: false, error: 'Missing key or HWID' });
        }

        const [rows] = await pool.query(
            `SELECT key_value, expires_at FROM keys_table
             WHERE key_value = ? AND hwid = ? AND is_active = 1 AND expires_at > NOW()`,
            [key.trim(), hwid.trim()]
        );

        if (rows.length === 0) {
            return res.json({ valid: false, error: 'Invalid or expired key' });
        }

        res.json({
            valid:      true,
            expires_at: rows[0].expires_at,
        });

    } catch (err) {
        console.error('[key/validate]', err);
        res.status(500).json({ valid: false, error: 'Server error' });
    }
});

// ═══════════════════════════════════════════════════════
//  6.  POST /api/key/check-hwid
//      Проверка: есть ли действующий ключ по HWID
//      Body: { hwid }
// ═══════════════════════════════════════════════════════
app.post('/api/key/check-hwid', async (req, res) => {
    try {
        const { hwid } = req.body;
        if (!hwid) return res.json({ has_key: false });

        const [rows] = await pool.query(
            `SELECT key_value, expires_at FROM keys_table
             WHERE hwid = ? AND is_active = 1 AND expires_at > NOW()
             ORDER BY created_at DESC LIMIT 1`,
            [hwid]
        );

        if (rows.length === 0) {
            return res.json({ has_key: false });
        }

        res.json({
            has_key:    true,
            key:        rows[0].key_value,
            expires_at: rows[0].expires_at,
        });
    } catch (err) {
        res.status(500).json({ has_key: false });
    }
});

// ── Очистка просроченных данных (раз в час) ──
setInterval(async () => {
    try {
        await pool.query('DELETE FROM sessions WHERE expires_at < NOW()');
        await pool.query('UPDATE keys_table SET is_active = 0 WHERE expires_at < NOW()');
        console.log('[cleanup] Expired data cleaned');
    } catch (_) {}
}, 3600000);

// ═══════════════════════════════════════
//  ЗАПУСК
// ═══════════════════════════════════════
const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
    console.log(`🔑 SemyAware Key System API → http://localhost:${PORT}`);
});
```

---

## 5. Фронтенд — public/getkey.html

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SemyAware — Получить ключ</title>
    <style>
        /* ══════════ RESET & BASE ══════════ */
        *, *::before, *::after { margin:0; padding:0; box-sizing:border-box; }
        body {
            font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
            background: #07070e;
            color: #e2e2e2;
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            overflow: hidden;
        }

        /* ══════════ BACKGROUND GLOW ══════════ */
        body::before {
            content: '';
            position: fixed;
            top: -50%; left: -50%;
            width: 200%; height: 200%;
            background: radial-gradient(circle at 30% 40%, rgba(111, 66, 193, .12) 0%, transparent 50%),
                        radial-gradient(circle at 70% 60%, rgba(59, 130, 246, .08) 0%, transparent 50%);
            animation: bgMove 20s ease-in-out infinite alternate;
            z-index: -1;
        }
        @keyframes bgMove {
            0%   { transform: translate(0, 0); }
            100% { transform: translate(-5%, -3%); }
        }

        /* ══════════ CARD ══════════ */
        .card {
            background: rgba(15, 15, 25, .85);
            backdrop-filter: blur(20px);
            border: 1px solid rgba(111, 66, 193, .25);
            border-radius: 20px;
            padding: 40px;
            width: 480px;
            max-width: 95vw;
            box-shadow: 0 0 60px rgba(111, 66, 193, .08);
            transition: all .3s;
        }

        /* ══════════ HEADER ══════════ */
        .logo {
            text-align: center;
            margin-bottom: 30px;
        }
        .logo h1 {
            font-size: 28px;
            font-weight: 700;
            background: linear-gradient(135deg, #8b5cf6, #3b82f6);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            background-clip: text;
        }
        .logo p {
            color: #888;
            font-size: 14px;
            margin-top: 6px;
        }

        /* ══════════ HWID INPUT ══════════ */
        .hwid-section { text-align: center; }
        .hwid-section input {
            width: 100%;
            padding: 14px 18px;
            background: rgba(255,255,255,.04);
            border: 1px solid rgba(255,255,255,.1);
            border-radius: 12px;
            color: #fff;
            font-size: 14px;
            outline: none;
            transition: border .3s;
            margin-bottom: 16px;
        }
        .hwid-section input:focus {
            border-color: #8b5cf6;
        }

        /* ══════════ BUTTONS ══════════ */
        .btn {
            display: inline-flex;
            align-items: center;
            justify-content: center;
            gap: 8px;
            width: 100%;
            padding: 14px 24px;
            border: none;
            border-radius: 12px;
            font-size: 15px;
            font-weight: 600;
            cursor: pointer;
            transition: all .25s;
            text-decoration: none;
            color: #fff;
        }
        .btn-primary {
            background: linear-gradient(135deg, #8b5cf6, #6d28d9);
        }
        .btn-primary:hover {
            transform: translateY(-2px);
            box-shadow: 0 8px 25px rgba(139, 92, 246, .35);
        }
        .btn-primary:disabled {
            opacity: .5;
            cursor: not-allowed;
            transform: none;
            box-shadow: none;
        }
        .btn-success {
            background: linear-gradient(135deg, #10b981, #059669);
        }
        .btn-success:hover {
            box-shadow: 0 8px 25px rgba(16, 185, 129, .35);
        }

        /* ══════════ PROGRESS ══════════ */
        .progress-section {
            display: none;
        }
        .progress-bar-bg {
            width: 100%;
            height: 6px;
            background: rgba(255,255,255,.08);
            border-radius: 3px;
            margin: 20px 0;
            overflow: hidden;
        }
        .progress-bar-fill {
            height: 100%;
            background: linear-gradient(90deg, #8b5cf6, #3b82f6);
            border-radius: 3px;
            transition: width .6s ease;
            width: 0%;
        }

        /* ══════════ STEPS LIST ══════════ */
        .steps { list-style: none; margin: 24px 0; }
        .steps li {
            display: flex;
            align-items: center;
            gap: 14px;
            padding: 14px 16px;
            background: rgba(255,255,255,.02);
            border: 1px solid rgba(255,255,255,.06);
            border-radius: 12px;
            margin-bottom: 10px;
            transition: all .3s;
        }
        .steps li.active {
            border-color: rgba(139, 92, 246, .4);
            background: rgba(139, 92, 246, .06);
        }
        .steps li.done {
            border-color: rgba(16, 185, 129, .3);
            background: rgba(16, 185, 129, .05);
        }

        .step-circle {
            width: 36px;
            height: 36px;
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            font-weight: 700;
            font-size: 14px;
            flex-shrink: 0;
            border: 2px solid rgba(255,255,255,.15);
            color: #888;
            transition: all .3s;
        }
        .steps li.active .step-circle {
            border-color: #8b5cf6;
            color: #8b5cf6;
            box-shadow: 0 0 12px rgba(139, 92, 246, .3);
        }
        .steps li.done .step-circle {
            border-color: #10b981;
            background: #10b981;
            color: #fff;
        }

        .step-label {
            flex: 1;
            font-size: 14px;
            color: #999;
        }
        .steps li.active .step-label { color: #e2e2e2; }
        .steps li.done .step-label   { color: #10b981; }

        .step-badge {
            font-size: 11px;
            padding: 4px 10px;
            border-radius: 20px;
            font-weight: 600;
        }
        .badge-pending { background: rgba(255,255,255,.06); color: #666; }
        .badge-active  { background: rgba(139,92,246,.15);  color: #a78bfa; }
        .badge-done    { background: rgba(16,185,129,.15);  color: #34d399; }

        /* ══════════ KEY DISPLAY ══════════ */
        .key-section {
            display: none;
            text-align: center;
        }
        .key-display {
            background: rgba(16, 185, 129, .08);
            border: 1px solid rgba(16, 185, 129, .25);
            border-radius: 12px;
            padding: 20px;
            margin: 20px 0;
        }
        .key-display .label {
            font-size: 12px;
            color: #34d399;
            text-transform: uppercase;
            letter-spacing: 1px;
            margin-bottom: 8px;
        }
        .key-display .key-text {
            font-family: 'Cascadia Code', 'Fira Code', monospace;
            font-size: 20px;
            color: #fff;
            font-weight: 700;
            letter-spacing: 2px;
            word-break: break-all;
            user-select: all;
        }
        .key-display .expires {
            font-size: 12px;
            color: #888;
            margin-top: 10px;
        }

        /* ══════════ STATUS MESSAGES ══════════ */
        .msg {
            padding: 12px 16px;
            border-radius: 10px;
            font-size: 13px;
            margin-top: 16px;
            display: none;
        }
        .msg-error {
            background: rgba(239, 68, 68, .1);
            border: 1px solid rgba(239, 68, 68, .2);
            color: #f87171;
        }
        .msg-info {
            background: rgba(59, 130, 246, .1);
            border: 1px solid rgba(59, 130, 246, .2);
            color: #60a5fa;
        }

        /* ══════════ TIMER ══════════ */
        .timer {
            text-align: center;
            color: #a78bfa;
            font-size: 24px;
            font-weight: 700;
            font-family: monospace;
            margin: 10px 0;
            display: none;
        }

        /* ══════════ SPINNER ══════════ */
        .spinner {
            display: inline-block;
            width: 18px; height: 18px;
            border: 2px solid rgba(255,255,255,.2);
            border-top-color: #8b5cf6;
            border-radius: 50%;
            animation: spin .7s linear infinite;
        }
        @keyframes spin { to { transform: rotate(360deg); } }

        /* ══════════ ANIMATIONS ══════════ */
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(12px); }
            to   { opacity: 1; transform: translateY(0); }
        }
        .fade-in { animation: fadeIn .4s ease forwards; }

        /* ══════════ COPY FEEDBACK ══════════ */
        .copied-toast {
            position: fixed;
            bottom: 30px;
            left: 50%;
            transform: translateX(-50%) translateY(20px);
            background: #10b981;
            color: #fff;
            padding: 12px 28px;
            border-radius: 10px;
            font-weight: 600;
            font-size: 14px;
            opacity: 0;
            transition: all .3s;
            pointer-events: none;
            z-index: 999;
        }
        .copied-toast.show {
            opacity: 1;
            transform: translateX(-50%) translateY(0);
        }
    </style>
</head>
<body>

<div class="card">
    <!-- ═══ ВВОД HWID ═══ -->
    <div id="hwidSection" class="hwid-section">
        <div class="logo">
            <h1>🔑 SemyAware Hub</h1>
            <p>Система получения ключа</p>
        </div>

        <input type="text" id="hwidInput"
               placeholder="Вставьте ваш HWID"
               autocomplete="off" spellcheck="false">

        <button id="startBtn" class="btn btn-primary" onclick="startSession()">
            Начать получение ключа
        </button>

        <div id="hwidMsg" class="msg msg-error"></div>
    </div>

    <!-- ═══ ЧЕКПОИНТЫ ═══ -->
    <div id="progressSection" class="progress-section">
        <div class="logo">
            <h1>🔑 SemyAware Hub</h1>
            <p>Пройдите все чекпоинты для получения ключа</p>
        </div>

        <div class="progress-bar-bg">
            <div id="progressFill" class="progress-bar-fill"></div>
        </div>

        <ul id="stepsList" class="steps"></ul>

        <div id="timerDisplay" class="timer"></div>

        <button id="checkpointBtn" class="btn btn-primary" onclick="startCheckpoint()">
            ▶ Пройти чекпоинт 1
        </button>

        <div id="progressMsg" class="msg"></div>
    </div>

    <!-- ═══ КЛЮЧ ГОТОВ ═══ -->
    <div id="keySection" class="key-section">
        <div class="logo">
            <h1>✅ Ключ получен!</h1>
            <p>Скопируйте и вставьте в скрипт</p>
        </div>

        <div class="key-display">
            <div class="label">Ваш ключ</div>
            <div id="keyText" class="key-text"></div>
            <div id="keyExpires" class="expires"></div>
        </div>

        <button class="btn btn-success" onclick="copyKey()">
            📋 Скопировать ключ
        </button>
    </div>
</div>

<div id="copiedToast" class="copied-toast">✓ Ключ скопирован!</div>

<script>
/* ═══════════════════════════════════════════════
   SemyAware Key System — Frontend Logic
   ═══════════════════════════════════════════════ */

const API          = window.location.origin + '/api';
const TOTAL_STEPS  = 3;

let sessionId      = null;
let currentStep    = 1;
let timerInterval  = null;

// ── Инициализация: проверяем URL-параметры ──
document.addEventListener('DOMContentLoaded', () => {
    const params  = new URLSearchParams(window.location.search);
    const hwid    = params.get('hwid');
    const error   = params.get('error');
    const session = params.get('session');
    const keyReady= params.get('key_ready');

    // Если есть HWID из скрипта — вставляем
    if (hwid) {
        document.getElementById('hwidInput').value = hwid;
    }

    // Если вернулись с Linkvertise с ошибкой
    if (error) {
        showMsg('hwidMsg', decodeURIComponent(error), 'error');
    }

    // Если вернулись после чекпоинта
    if (session) {
        sessionId = session;
        loadStatus();
    }
});

// ═══════════════════════════════════════
//  Создание / возобновление сессии
// ═══════════════════════════════════════
async function startSession() {
    const hwid = document.getElementById('hwidInput').value.trim();
    if (!hwid || hwid.length < 5) {
        return showMsg('hwidMsg', 'Введите корректный HWID (минимум 5 символов)', 'error');
    }

    const btn = document.getElementById('startBtn');
    btn.disabled = true;
    btn.innerHTML = '<span class="spinner"></span> Загрузка...';

    try {
        const resp = await fetch(API + '/session/create', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ hwid })
        });
        const data = await resp.json();

        if (!data.success) {
            showMsg('hwidMsg', data.error || 'Ошибка', 'error');
            btn.disabled = false;
            btn.textContent = 'Начать получение ключа';
            return;
        }

        // Если уже есть ключ
        if (data.has_key) {
            showKey(data.key, data.expires_at);
            return;
        }

        sessionId   = data.session_id;
        currentStep = data.current_step;

        showCheckpoints();

    } catch (err) {
        showMsg('hwidMsg', 'Ошибка сети. Попробуйте позже.', 'error');
        btn.disabled = false;
        btn.textContent = 'Начать получение ключа';
    }
}

// ═══════════════════════════════════════
//  Загрузка текущего статуса
// ═══════════════════════════════════════
async function loadStatus() {
    try {
        const resp = await fetch(API + '/checkpoint/status', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ session_id: sessionId })
        });
        const data = await resp.json();

        if (!data.success) {
            // Сессия истекла — показать ввод HWID
            return;
        }

        if (data.key) {
            showKey(data.key, data.expires_at);
            return;
        }

        currentStep = data.current_step;
        renderSteps(data.steps);
        showCheckpoints();

    } catch (err) {
        console.error('loadStatus error:', err);
    }
}

// ═══════════════════════════════════════
//  Отображение чекпоинтов
// ═══════════════════════════════════════
function showCheckpoints() {
    document.getElementById('hwidSection').style.display    = 'none';
    document.getElementById('progressSection').style.display = 'block';
    document.getElementById('progressSection').classList.add('fade-in');

    if (!document.getElementById('stepsList').children.length) {
        renderSteps(null);
    }

    updateUI();
}

function renderSteps(stepsData) {
    const list = document.getElementById('stepsList');
    list.innerHTML = '';

    for (let i = 1; i <= TOTAL_STEPS; i++) {
        const completed = stepsData ? stepsData[i-1]?.completed : false;
        const isActive  = !completed && i === currentStep;
        const isPending = !completed && i !== currentStep;

        const li = document.createElement('li');
        if (completed) li.className = 'done';
        else if (isActive) li.className = 'active';

        const stateClass = completed ? 'badge-done'
                         : isActive  ? 'badge-active'
                         : 'badge-pending';
        const stateText  = completed ? '✓ Пройден'
                         : isActive  ? '● Текущий'
                         : 'Ожидание';

        li.innerHTML = `
            <div class="step-circle">${completed ? '✓' : i}</div>
            <div class="step-label">Чекпоинт ${i}</div>
            <span class="step-badge ${stateClass}">${stateText}</span>
        `;
        list.appendChild(li);
    }
}

function updateUI() {
    // Прогресс-бар
    const pct = ((currentStep - 1) / TOTAL_STEPS) * 100;
    document.getElementById('progressFill').style.width = pct + '%';

    // Кнопка
    const btn = document.getElementById('checkpointBtn');
    if (currentStep > TOTAL_STEPS) {
        btn.textContent = '✓ Все чекпоинты пройдены!';
        btn.disabled = true;
    } else {
        btn.innerHTML = `▶ Пройти чекпоинт ${currentStep}`;
        btn.disabled = false;
    }

    // Обновляем список
    renderSteps(null);

    // Отмечаем пройденные
    const items = document.querySelectorAll('#stepsList li');
    items.forEach((li, idx) => {
        const step = idx + 1;
        if (step < currentStep) {
            li.className = 'done';
            li.querySelector('.step-circle').textContent = '✓';
            li.querySelector('.step-badge').className = 'step-badge badge-done';
            li.querySelector('.step-badge').textContent = '✓ Пройден';
        } else if (step === currentStep) {
            li.className = 'active';
            li.querySelector('.step-badge').className = 'step-badge badge-active';
            li.querySelector('.step-badge').textContent = '● Текущий';
        }
    });
}

// ═══════════════════════════════════════
//  Запуск чекпоинта (→ Linkvertise)
// ═══════════════════════════════════════
async function startCheckpoint() {
    const btn = document.getElementById('checkpointBtn');
    btn.disabled = true;
    btn.innerHTML = '<span class="spinner"></span> Генерация ссылки...';

    try {
        const resp = await fetch(API + '/checkpoint/start', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                session_id: sessionId,
                step: currentStep
            })
        });
        const data = await resp.json();

        if (!data.success) {
            // Если нужно подождать — показать таймер
            if (data.wait) {
                startTimer(data.wait);
            }
            showMsg('progressMsg', data.error, 'error');
            btn.disabled = false;
            btn.innerHTML = `▶ Пройти чекпоинт ${currentStep}`;
            return;
        }

        // Открываем Linkvertise
        showMsg('progressMsg',
            'Вы будете перенаправлены на Linkvertise. Пройдите все шаги и вернётесь обратно автоматически.',
            'info');

        setTimeout(() => {
            window.location.href = data.url;
        }, 1500);

    } catch (err) {
        showMsg('progressMsg', 'Ошибка сети', 'error');
        btn.disabled = false;
        btn.innerHTML = `▶ Пройти чекпоинт ${currentStep}`;
    }
}

// ═══════════════════════════════════════
//  Таймер ожидания
// ═══════════════════════════════════════
function startTimer(seconds) {
    const display = document.getElementById('timerDisplay');
    const btn     = document.getElementById('checkpointBtn');
    let remaining = Math.ceil(seconds);

    display.style.display = 'block';
    btn.disabled = true;

    clearInterval(timerInterval);
    timerInterval = setInterval(() => {
        remaining--;
        display.textContent = `⏳ ${remaining} сек.`;

        if (remaining <= 0) {
            clearInterval(timerInterval);
            display.style.display = 'none';
            btn.disabled = false;
            btn.innerHTML = `▶ Пройти чекпоинт ${currentStep}`;
            hideMsg('progressMsg');
        }
    }, 1000);

    display.textContent = `⏳ ${remaining} сек.`;
}

// ═══════════════════════════════════════
//  Показ ключа
// ═══════════════════════════════════════
function showKey(key, expiresAt) {
    document.getElementById('hwidSection').style.display     = 'none';
    document.getElementById('progressSection').style.display  = 'none';
    document.getElementById('keySection').style.display       = 'block';
    document.getElementById('keySection').classList.add('fade-in');

    document.getElementById('keyText').textContent = key;

    if (expiresAt) {
        const exp = new Date(expiresAt);
        document.getElementById('keyExpires').textContent =
            `Действителен до: ${exp.toLocaleString('ru-RU')}`;
    }
}

// ═══════════════════════════════════════
//  Копирование ключа
// ═══════════════════════════════════════
function copyKey() {
    const key = document.getElementById('keyText').textContent;
    navigator.clipboard.writeText(key).then(() => {
        const toast = document.getElementById('copiedToast');
        toast.classList.add('show');
        setTimeout(() => toast.classList.remove('show'), 2500);
    });
}

// ═══════════════════════════════════════
//  Утилиты
// ═══════════════════════════════════════
function showMsg(id, text, type) {
    const el = document.getElementById(id);
    el.textContent = text;
    el.className = `msg msg-${type}`;
    el.style.display = 'block';
}

function hideMsg(id) {
    document.getElementById(id).style.display = 'none';
}
</script>

</body>
</html>
```

---

## 6. Roblox Lua скрипт — lua/keysystem.lua

```lua
--[[
    ═══════════════════════════════════════════
    SemyAware Hub — Key System (Client-Side)
    Вставить в начало главного скрипта хаба
    ═══════════════════════════════════════════
--]]

local KeySystem = {}

-- ══════ НАСТРОЙКИ ══════
local API_URL     = "https://semyaware.ru/api"
local GETKEY_URL  = "https://semyaware.ru/getkey.html"
local KEY_FILE    = "semyaware/key.txt"
local HWID_FILE   = "semyaware/hwid.txt"

-- ══════ ПОЛУЧЕНИЕ HWID ══════
local function getHWID()
    -- Поддержка разных эксплойтов
    local ok, hwid

    -- Synapse X / Fluxus / Hydrogen и др.
    ok, hwid = pcall(function()
        if gethwid then return gethwid() end
        if getexecutorname and game.PlaceId then
            -- Фолбэк: используем RbxAnalyticsService
            return game:GetService("RbxAnalyticsService"):GetClientId()
        end
        return nil
    end)

    if ok and hwid then return hwid end

    -- Ещё один фолбэк
    ok, hwid = pcall(function()
        return game:GetService("RbxAnalyticsService"):GetClientId()
    end)

    if ok and hwid then return hwid end

    -- Генерируем и сохраняем свой
    if isfile and isfile(HWID_FILE) then
        return readfile(HWID_FILE)
    end

    local generated = game:GetService("HttpService"):GenerateGUID(false)
    if writefile then
        pcall(function()
            makefolder("semyaware")
            writefile(HWID_FILE, generated)
        end)
    end

    return generated
end

-- ══════ HTTP ЗАПРОС ══════
local function httpPost(endpoint, body)
    local HttpService = game:GetService("HttpService")

    local ok, response = pcall(function()
        return request({
            Url     = API_URL .. endpoint,
            Method  = "POST",
            Headers = { ["Content-Type"] = "application/json" },
            Body    = HttpService:JSONEncode(body)
        })
    end)

    if not ok then
        -- Некоторые эксплойты используют другие функции
        ok, response = pcall(function()
            return http_request({
                Url     = API_URL .. endpoint,
                Method  = "POST",
                Headers = { ["Content-Type"] = "application/json" },
                Body    = HttpService:JSONEncode(body)
            })
        end)
    end

    if not ok or not response then
        return nil
    end

    local success, data = pcall(function()
        return HttpService:JSONDecode(response.Body)
    end)

    return success and data or nil
end

-- ══════ ВАЛИДАЦИЯ КЛЮЧА ══════
function KeySystem:ValidateKey(key)
    local hwid = getHWID()
    local data = httpPost("/key/validate", {
        key  = key,
        hwid = hwid
    })

    if data and data.valid == true then
        return true, data.expires_at
    end

    return false, data and data.error or "Ошибка проверки"
end

-- ══════ ПРОВЕРКА ПО HWID ══════
function KeySystem:CheckHWID()
    local hwid = getHWID()
    local data = httpPost("/key/check-hwid", { hwid = hwid })

    if data and data.has_key == true then
        return true, data.key, data.expires_at
    end

    return false
end

-- ══════ СОХРАНЕНИЕ / ЗАГРУЗКА КЛЮЧА ══════
function KeySystem:SaveKey(key)
    pcall(function()
        if writefile then
            makefolder("semyaware")
            writefile(KEY_FILE, key)
        end
    end)
end

function KeySystem:LoadKey()
    local ok, key = pcall(function()
        if isfile and isfile(KEY_FILE) then
            return readfile(KEY_FILE)
        end
        return nil
    end)
    return ok and key or nil
end

-- ══════ УВЕДОМЛЕНИЯ ══════
local function notify(title, text, duration)
    pcall(function()
        game:GetService("StarterGui"):SetCore("SendNotification", {
            Title    = title,
            Text     = text,
            Duration = duration or 8
        })
    end)
end

-- ══════════════════════════════════════════════
--  ГЛАВНАЯ ФУНКЦИЯ: KeySystem:Init()
--  Возвращает true если ключ валиден,
--  иначе открывает страницу получения ключа
-- ══════════════════════════════════════════════
function KeySystem:Init()
    local hwid = getHWID()

    notify("SemyAware Hub", "⏳ Проверка ключа...", 5)

    -- 1) Проверяем сохранённый ключ
    local savedKey = self:LoadKey()
    if savedKey and savedKey ~= "" then
        local valid, expires = self:ValidateKey(savedKey)
        if valid then
            notify("SemyAware Hub", "✅ Ключ действителен!", 5)
            return true
        end
    end

    -- 2) Проверяем по HWID (может ключ получен с другого устройства)
    local hasKey, serverKey, expires = self:CheckHWID()
    if hasKey then
        self:SaveKey(serverKey)
        notify("SemyAware Hub", "✅ Ключ найден и сохранён!", 5)
        return true
    end

    -- 3) Ключа нет — отправляем на сайт
    local keyUrl = GETKEY_URL .. "?hwid=" .. hwid

    pcall(function()
        setclipboard(keyUrl)
    end)

    notify("SemyAware Hub",
        "🔑 Ключ не найден!\n" ..
        "Ссылка скопирована в буфер обмена.\n" ..
        "Получите ключ на сайте и перезапустите скрипт.",
        15)

    -- 4) Показываем GUI для ввода ключа
    self:ShowKeyGUI(hwid)

    return false
end

-- ══════════════════════════════════════════════
--  GUI для ввода ключа прямо в игре
-- ══════════════════════════════════════════════
function KeySystem:ShowKeyGUI(hwid)
    -- Удаляем старый GUI если есть
    local coreGui = game:GetService("CoreGui")
    if coreGui:FindFirstChild("SemyAwareKeyGUI") then
        coreGui.SemyAwareKeyGUI:Destroy()
    end

    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name  = "SemyAwareKeyGUI"
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

    local Frame = Instance.new("Frame")
    Frame.Size = UDim2.new(0, 360, 0, 220)
    Frame.Position = UDim2.new(0.5, -180, 0.5, -110)
    Frame.BackgroundColor3 = Color3.fromRGB(15, 15, 25)
    Frame.BorderSizePixel = 0
    Frame.Parent = ScreenGui

    local Corner = Instance.new("UICorner")
    Corner.CornerRadius = UDim.new(0, 14)
    Corner.Parent = Frame

    local Stroke = Instance.new("UIStroke")
    Stroke.Color = Color3.fromRGB(111, 66, 193)
    Stroke.Thickness = 1.5
    Stroke.Transparency = 0.6
    Stroke.Parent = Frame

    local Title = Instance.new("TextLabel")
    Title.Size = UDim2.new(1, 0, 0, 40)
    Title.Position = UDim2.new(0, 0, 0, 10)
    Title.BackgroundTransparency = 1
    Title.Text = "🔑 SemyAware Hub"
    Title.TextColor3 = Color3.fromRGB(139, 92, 246)
    Title.TextSize = 20
    Title.Font = Enum.Font.GothamBold
    Title.Parent = Frame

    local Sub = Instance.new("TextLabel")
    Sub.Size = UDim2.new(1, -20, 0, 20)
    Sub.Position = UDim2.new(0, 10, 0, 50)
    Sub.BackgroundTransparency = 1
    Sub.Text = "Вставьте ключ с сайта semyaware.ru"
    Sub.TextColor3 = Color3.fromRGB(150, 150, 150)
    Sub.TextSize = 13
    Sub.Font = Enum.Font.Gotham
    Sub.Parent = Frame

    local Input = Instance.new("TextBox")
    Input.Size = UDim2.new(1, -30, 0, 38)
    Input.Position = UDim2.new(0, 15, 0, 80)
    Input.BackgroundColor3 = Color3.fromRGB(25, 25, 40)
    Input.BorderSizePixel = 0
    Input.PlaceholderText = "SEMYA-XXXXX-XXXXX-XXXXX-XXXXX"
    Input.PlaceholderColor3 = Color3.fromRGB(80, 80, 80)
    Input.Text = ""
    Input.TextColor3 = Color3.fromRGB(255, 255, 255)
    Input.TextSize = 14
    Input.Font = Enum.Font.Code
    Input.ClearTextOnFocus = false
    Input.Parent = Frame

    local InputCorner = Instance.new("UICorner")
    InputCorner.CornerRadius = UDim.new(0, 8)
    InputCorner.Parent = Input

    local SubmitBtn = Instance.new("TextButton")
    SubmitBtn.Size = UDim2.new(1, -30, 0, 38)
    SubmitBtn.Position = UDim2.new(0, 15, 0, 130)
    SubmitBtn.BackgroundColor3 = Color3.fromRGB(111, 66, 193)
    SubmitBtn.BorderSizePixel = 0
    SubmitBtn.Text = "Активировать"
    SubmitBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    SubmitBtn.TextSize = 15
    SubmitBtn.Font = Enum.Font.GothamBold
    SubmitBtn.Parent = Frame

    local BtnCorner = Instance.new("UICorner")
    BtnCorner.CornerRadius = UDim.new(0, 8)
    BtnCorner.Parent = SubmitBtn

    local GetKeyBtn = Instance.new("TextButton")
    GetKeyBtn.Size = UDim2.new(1, -30, 0, 25)
    GetKeyBtn.Position = UDim2.new(0, 15, 0, 178)
    GetKeyBtn.BackgroundTransparency = 1
    GetKeyBtn.Text = "📎 Скопировать ссылку на получение ключа"
    GetKeyBtn.TextColor3 = Color3.fromRGB(96, 165, 250)
    GetKeyBtn.TextSize = 12
    GetKeyBtn.Font = Enum.Font.Gotham
    GetKeyBtn.Parent = Frame

    ScreenGui.Parent = coreGui

    -- ── Обработка кнопки активации ──
    local self_ = self
    SubmitBtn.MouseButton1Click:Connect(function()
        local key = Input.Text:match("^%s*(.-)%s*$") -- trim
        if not key or #key < 10 then
            Sub.Text = "❌ Введите корректный ключ"
            Sub.TextColor3 = Color3.fromRGB(248, 113, 113)
            return
        end

        SubmitBtn.Text = "⏳ Проверка..."
        SubmitBtn.BackgroundColor3 = Color3.fromRGB(80, 50, 140)

        local valid, info = self_:ValidateKey(key)
        if valid then
            self_:SaveKey(key)
            Sub.Text = "✅ Ключ принят! Загрузка хаба..."
            Sub.TextColor3 = Color3.fromRGB(52, 211, 153)
            SubmitBtn.Text = "✅ Успешно!"
            SubmitBtn.BackgroundColor3 = Color3.fromRGB(16, 185, 129)

            task.wait(1.5)
            ScreenGui:Destroy()

            -- ═══ ЗАГРУЖАЕМ ОСНОВНОЙ СКРИПТ ХАБА ═══
            loadstring(game:HttpGet("https://semyaware.ru/hub.lua"))()

        else
            Sub.Text = "❌ " .. tostring(info)
            Sub.TextColor3 = Color3.fromRGB(248, 113, 113)
            SubmitBtn.Text = "Активировать"
            SubmitBtn.BackgroundColor3 = Color3.fromRGB(111, 66, 193)
        end
    end)

    -- ── Кнопка копирования ссылки ──
    GetKeyBtn.MouseButton1Click:Connect(function()
        pcall(function()
            setclipboard(GETKEY_URL .. "?hwid=" .. hwid)
        end)
        GetKeyBtn.Text = "✓ Ссылка скопирована!"
        task.wait(2)
        GetKeyBtn.Text = "📎 Скопировать ссылку на получение ключа"
    end)

    -- ── Перетаскивание окна ──
    local dragging, dragStart, startPos
    Frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = Frame.Position
        end
    end)
    Frame.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    game:GetService("UserInputService").InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = input.Position - dragStart
            Frame.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y
            )
        end
    end)
end

return KeySystem
```

### Использование в главном скрипте хаба:

```lua
-- loader.lua (точка входа)
local KeySystem = loadstring(game:HttpGet(
    "https://semyaware.ru/lua/keysystem.lua"
))()

-- Если ключ валиден — Init вернёт true и загрузит хаб
-- Если нет — покажет GUI для ввода ключа
if KeySystem:Init() then
    -- Ключ уже был валиден, грузим хаб
    loadstring(game:HttpGet("https://semyaware.ru/hub.lua"))()
end
-- Если Init вернул false, GUI уже показан и хаб
-- загрузится автоматически после ввода ключа
```

---

## 7. Настройка Nginx (проксирование)

Если основной сайт на PHP/Nginx, добавь в конфиг:

```nginx
# /etc/nginx/sites-available/semyaware.ru

server {
    listen 443 ssl;
    server_name semyaware.ru;

    # SSL сертификаты
    ssl_certificate     /etc/letsencrypt/live/semyaware.ru/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/semyaware.ru/privkey.pem;

    # ══════ Основной сайт (PHP) ══════
    root /var/www/semyaware.ru;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # ══════ Key System API (Node.js) ══════
    location /api/ {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ══════ Страница получения ключа ══════
    location /getkey.html {
        proxy_pass http://127.0.0.1:3001/getkey.html;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # ══════ Lua скрипты для Roblox ══════
    location /lua/ {
        root /var/www/semyaware.ru;
        add_header Access-Control-Allow-Origin *;
        add_header Content-Type "text/plain; charset=utf-8";
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 8. Запуск и деплой

```bash
# 1. Установить зависимости
cd /opt/semyaware-keysystem
npm install

# 2. Создать базу данных
mysql -u root -p < schema.sql

# 3. Настроить .env файл
nano .env
# Заполнить все переменные (особенно LINKVERTISE_PUBLISHER_ID и SECRET_KEY)

# 4. Тестовый запуск
node server.js

# 5. Настроить systemd для автозапуска
sudo nano /etc/systemd/system/keysystem.service
```

```ini
# /etc/systemd/system/keysystem.service
[Unit]
Description=SemyAware Key System API
After=network.target mysql.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/semyaware-keysystem
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable keysystem
sudo systemctl start keysystem
sudo systemctl status keysystem
```

---

## Как всё работает (итого)

```
1. Игрок запускает скрипт в Roblox
         │
2. Скрипт проверяет сохранённый ключ → POST /api/key/validate
         │
         ├── ✅ Ключ валиден → загружается хаб
         │
         └── ❌ Нет ключа → показывает GUI + копирует ссылку
                    │
3. Игрок открывает semyaware.ru/getkey.html?hwid=XXX
         │
4. POST /api/session/create → создаётся сессия
         │
5. Чекпоинт 1: POST /api/checkpoint/start → Linkvertise URL
         │  ↓ пользователь проходит Linkvertise ↓
         │  GET /api/checkpoint/verify → отмечает чекпоинт ✓
         │
6. Чекпоинт 2: то же самое (мин. 15 сек задержка)
         │
7. Чекпоинт 3: то же самое
         │
8. Все 3 пройдены → генерируется ключ SEMYA-XXXXX-XXXXX-XXXXX-XXXXX
         │
9. Игрок копирует ключ → вставляет в GUI в Roblox
         │
10. POST /api/key/validate → ✅ → хаб загружается
```

**Защита от обхода:**
- HMAC-подписанные одноразовые токены
- Минимальная задержка между чекпоинтами (15 сек)
- Токен действителен максимум 10 минут
- Привязка ключа к HWID
- Rate limiting на все эндпоинты
- Логирование попыток обхода
- Автоочистка просроченных данных
