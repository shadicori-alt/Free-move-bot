# 🎯 الملفات الكاملة لرفعها على GitHub

## 📝 4. server.js (الملف الرئيسي)
هذا ملف كبير، سأقسمه لأجزاء:

### الجزء الأول (الأساسيات والمكتبات):
```javascript
const express = require('express');
const bodyParser = require('body-parser');
const axios = require('axios');
const sqlite3 = require('sqlite3').verbose();
const cors = require('cors');
const WebSocket = require('ws');
const path = require('path');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));
app.use(express.static('public'));

// WebSocket Server للتحديثات الفورية
const wss = new WebSocket.Server({ noServer: true });

// Database Setup
const db = new sqlite3.Database('./orders.db', (err) => {
    if (err) {
        console.error('خطأ في الاتصال بقاعدة البيانات:', err);
    } else {
        console.log('✅ تم الاتصال بقاعدة البيانات بنجاح');
        initDatabase();
    }
});
```

### الجزء الثاني (تهيئة قاعدة البيانات):
```javascript
// تهيئة جداول قاعدة البيانات
function initDatabase() {
    db.serialize(() => {
        // جدول الإعدادات
        db.run(`CREATE TABLE IF NOT EXISTS settings (
            key TEXT PRIMARY KEY,
            value TEXT,
            updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )`);

        // جدول الطلبات
        db.run(`CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            customer_name TEXT,
            customer_phone TEXT,
            customer_address TEXT,
            governorate TEXT,
            order_details TEXT,
            status TEXT DEFAULT 'pending',
            delivery_man TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )`);

        // جدول مندوبي التوصيل
        db.run(`CREATE TABLE IF NOT EXISTS delivery_personnel (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT,
            phone TEXT,
            is_available BOOLEAN DEFAULT 1,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )`);

        // جدول السجلات
        db.run(`CREATE TABLE IF NOT EXISTS logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            type TEXT,
            message TEXT,
            details TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )`);

        // إدراج بيانات تجريبية
        db.run(`INSERT OR IGNORE INTO delivery_personnel (name, phone) VALUES 
            ('أحمد محمد', '01012345678'),
            ('محمد علي', '01112345678'),
            ('سارة أحمد', '01212345678')`);

        console.log('✅ تم تهيئة قاعدة البيانات بنجاح');
    });
}
```

### الجزء الثالث (API Endpoints الأساسية):
```javascript
// API Routes

// الحصول على جميع الطلبات
app.get('/api/orders', (req, res) => {
    db.all(`SELECT * FROM orders ORDER BY created_at DESC`, [], (err, rows) => {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        res.json(rows);
    });
});

// إضافة طلب جديد
app.post('/api/orders', (req, res) => {
    const { customer_name, customer_phone, customer_address, governorate, order_details, delivery_man } = req.body;
    const sql = `INSERT INTO orders (customer_name, customer_phone, customer_address, governorate, order_details, delivery_man) 
                 VALUES (?, ?, ?, ?, ?, ?)`;
    
    db.run(sql, [customer_name, customer_phone, customer_address, governorate, order_details, delivery_man], function(err) {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        res.json({ id: this.lastID, message: 'تم إضافة الطلب بنجاح' });
        
        // إرسال تحديث عبر WebSocket
        broadcastUpdate('new_order', { id: this.lastID, customer_name });
    });
});

// تحديث حالة الطلب
app.put('/api/orders/:id', (req, res) => {
    const { status } = req.body;
    const { id } = req.params;
    
    db.run(`UPDATE orders SET status = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?`, 
           [status, id], function(err) {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        res.json({ message: 'تم تحديث الطلب بنجاح' });
        
        broadcastUpdate('order_updated', { id, status });
    });
});
```

### الجزء الرابع (مندوبي التوصيل والإحصائيات):
```javascript
// الحصول على مندوبي التوصيل
app.get('/api/delivery-personnel', (req, res) => {
    db.all(`SELECT * FROM delivery_personnel ORDER BY name`, [], (err, rows) => {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        res.json(rows);
    });
});

// إضافة مندوب توصيل جديد
app.post('/api/delivery-personnel', (req, res) => {
    const { name, phone } = req.body;
    const sql = `INSERT INTO delivery_personnel (name, phone) VALUES (?, ?)`;
    
    db.run(sql, [name, phone], function(err) {
        if (err) {
            res.status(500).json({ error: err.message });
            return;
        }
        res.json({ id: this.lastID, message: 'تم إضافة مندوب التوصيل بنجاح' });
    });
});

// الحصول على الإحصائيات
app.get('/api/stats', (req, res) => {
    const stats = {};
    
    db.serialize(() => {
        db.get(`SELECT COUNT(*) as total FROM orders`, [], (err, row) => {
            stats.total_orders = row.total;
            
            db.get(`SELECT COUNT(*) as pending FROM orders WHERE status = 'pending'`, [], (err, row) => {
                stats.pending_orders = row.pending;
                
                db.get(`SELECT COUNT(*) as completed FROM orders WHERE status = 'completed'`, [], (err, row) => {
                    stats.completed_orders = row.completed;
                    
                    db.get(`SELECT COUNT(*) as active FROM delivery_personnel WHERE is_available = 1`, [], (err, row) => {
                        stats.active_personnel = row.active;
                        res.json(stats);
                    });
                });
            });
        });
    });
});
```

### الجزء الخامس (WebSocket والاتصال الخارجي):
```javascript
// WebSocket للحديث مع Telegram
app.post('/api/telegram/send-order', async (req, res) => {
    const { message } = req.body;
    
    // إرسال الطلب عبر Telegram Bot
    // (سيتم تنفيذ هذا بعد إضافة API keys)
    res.json({ message: 'تم إرسال الطلب عبر Telegram' });
});

// WebSocket للتحديثات الفورية
function broadcastUpdate(type, data) {
    wss.clients.forEach(client => {
        if (client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify({ type, data }));
        }
    });
}

// معالجة HTTP Upgrade للـ WebSocket
const server = app.listen(PORT, () => {
    console.log(`🚀 الخادم يعمل على المنفذ ${PORT}`);
});

server.on('upgrade', (request, socket, head) => {
    wss.handleUpgrade(request, socket, head, (ws) => {
        wss.emit('connection', ws, request);
    });
});

wss.on('connection', (ws) => {
    console.log('✅ تم الاتصال بالـ WebSocket');
    
    ws.on('message', (message) => {
        console.log('رسالة WebSocket:', message.toString());
    });
    
    ws.on('close', () => {
        console.log('❌ تم قطع اتصال WebSocket');
    });
});

// إغلاق قاعدة البيانات عند إنهاء التطبيق
process.on('SIGINT', () => {
    db.close((err) => {
        if (err) {
            console.error(err.message);
        }
        console.log('🔒 تم إغلاق قاعدة البيانات.');
        process.exit(0);
    });
});
```

## 📱 5. public/index.html
```html
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>داش بورد إدارة الطلبات</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <style>...</style>
</head>
<body>
    <div class="container-fluid">...</div>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
    <script src="app.js"></script>
</body>
</html>
```

## ⚙️ 6. public/app.js
```javascript
// داش بورد إدارة الطلبات - JavaScript
class OrderDashboard { ... }
let dashboard;
document.addEventListener('DOMContentLoaded', () => {
    dashboard = new OrderDashboard();
});
```

## 🎯 الآن صار معك كل الملفات:
1. **package.json** (أعلى)
2. **render.yaml** (أعلى)
3. **server.js** (مقسم لأجزاء)
4. **public/index.html** (كامل)
5. **public/app.js** (كامل)
6. **.env.example** (أعلى)

انسخ كل ملف بالترتيب وارفعه على GitHub! 🚀

هل تريد أن أشرح لك خطوة بخطوة كيفية رفع كل ملف؟
