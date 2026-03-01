# Data-hub
Is about data bundle reselling platform 

// ====================================================== // COMPLETE PRODUCTION SYSTEM // Backend: Node.js + Express + MySQL // Features Added: // ✔ Datamart Purchase // ✔ Webhook Verification // ✔ Reseller Commission Engine // ✔ Wallet Funding (Paystack example structure) // ✔ Admin Analytics // ✔ JWT Authentication // ======================================================

require('dotenv').config(); const express = require('express'); const axios = require('axios'); const mysql = require('mysql2/promise'); const jwt = require('jsonwebtoken'); const bcrypt = require('bcryptjs'); const crypto = require('crypto');

const app = express(); app.use(express.json());

// ================= DATABASE ================= const db = mysql.createPool({ host: 'localhost', user: 'root', password: 'password', database: 'dataplatform' });

// ================= AUTH ================= function generateToken(user) { return jwt.sign({ id: user.id, role: user.role }, process.env.JWT_SECRET, { expiresIn: '7d' }); }

async function authMiddleware(req, res, next) { const token = req.headers.authorization?.split(' ')[1]; if (!token) return res.status(401).json({ error: 'Unauthorized' }); try { req.user = jwt.verify(token, process.env.JWT_SECRET); next(); } catch { res.status(401).json({ error: 'Invalid token' }); } }

// ================= REGISTER ================= app.post('/api/register', async (req, res) => { const { username, password, role } = req.body; const hash = await bcrypt.hash(password, 10); const [result] = await db.execute( 'INSERT INTO users (username,password,role) VALUES (?,?,?)', [username, hash, role] ); res.json({ token: generateToken({ id: result.insertId, role }) }); });

// ================= LOGIN ================= app.post('/api/login', async (req, res) => { const { username, password } = req.body; const [rows] = await db.execute('SELECT * FROM users WHERE username=?', [username]); if (!rows.length) return res.status(400).json({ error: 'User not found' }); const valid = await bcrypt.compare(password, rows[0].password); if (!valid) return res.status(400).json({ error: 'Invalid password' }); res.json({ token: generateToken(rows[0]) }); });

// ================= DATAMART PURCHASE ================= app.post('/api/purchase', authMiddleware, async (req, res) => { const { phoneNumber, network, capacity } = req.body; try { const response = await axios.post( 'https://api.datamartgh.shop/api/developer/purchase', { phoneNumber, network, capacity, gateway: 'wallet' }, { headers: { 'X-API-Key': process.env.DATAMART_API_KEY } } );

const price = response.data.data.price;
let profit = 0;

if (req.user.role === 'reseller') {
  profit = price * 0.05; // 5% commission example
}

await db.execute(
  'INSERT INTO transactions (user_id,phone,network,capacity,amount,profit,status,reference) VALUES (?,?,?,?,?,?,?,?)',
  [req.user.id, phoneNumber, network, capacity, price, profit, 'completed', response.data.data.transactionReference]
);

res.json(response.data);

} catch (err) { res.status(500).json({ error: 'Purchase failed', details: err.response?.data }); } });

// ================= PAYSTACK WALLET FUNDING ================= app.post('/api/fund/verify', async (req, res) => { const { reference, userId } = req.body; try { const verify = await axios.get( https://api.paystack.co/transaction/verify/${reference}, { headers: { Authorization: Bearer ${process.env.PAYSTACK_SECRET} } } );

if (verify.data.data.status === 'success') {
  const amount = verify.data.data.amount / 100;
  await db.execute('UPDATE users SET wallet = wallet + ? WHERE id=?', [amount, userId]);
  res.json({ success: true });
}

} catch { res.status(500).json({ error: 'Verification failed' }); } });

// ================= WEBHOOK ================= app.post('/webhook/datamart', express.json(), (req, res) => { const signature = req.headers['x-datamart-signature']; const expected = crypto .createHmac('sha256', process.env.DATAMART_WEBHOOK_SECRET) .update(JSON.stringify(req.body)) .digest('hex');

if (signature !== expected) return res.status(401).send('Invalid');

// Process event res.json({ received: true }); });

// ================= ADMIN ANALYTICS ================= app.get('/api/admin/analytics', authMiddleware, async (req, res) => { if (req.user.role !== 'admin') return res.status(403).json({ error: 'Forbidden' });

const [sales] = await db.execute('SELECT SUM(amount) as totalSales FROM transactions'); const [profits] = await db.execute('SELECT SUM(profit) as totalProfit FROM transactions'); const [users] = await db.execute('SELECT COUNT(*) as totalUsers FROM users');

res.json({ totalSales: sales[0].totalSales || 0, totalProfit: profits[0].totalProfit || 0, totalUsers: users[0].totalUsers }); });

app.listen(5000, () => console.log('Production server running on port 5000'));

// ================= REQUIRED ENV (.env) ================= /* DATAMART_API_KEY=NEW_SECRET_KEY DATAMART_WEBHOOK_SECRET=YOUR_WEBHOOK_SECRET JWT_SECRET=SUPER_RANDOM_SECRET PAYSTACK_SECRET=PAYSTACK_SECRET_KEY */

// ================= MYSQL TABLES ================= /* CREATE TABLE users ( id INT AUTO_INCREMENT PRIMARY KEY, username VARCHAR(100), password VARCHAR(255), role ENUM('admin','reseller','user'), wallet DECIMAL(10,2) DEFAULT 0, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP );

CREATE TABLE transactions ( id INT AUTO_INCREMENT PRIMARY KEY, user_id INT, phone VARCHAR(20), network VARCHAR(20), capacity VARCHAR(10), amount DECIMAL(10,2), profit DECIMAL(10,2), status VARCHAR(20), reference VARCHAR(100), created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ); */
