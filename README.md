Excelente 🔥, retomemos desde ahí para que puedas crear


---

🧱 1️⃣ Estructura general del proyecto

Crea esta estructura (puedes hacerlo directamente en Replit):

supabase-saas/
├─ backend/
│  ├─ index.js
│  ├─ routes/
│  │   ├─ auth.js
│  │   ├─ data.js
│  ├─ middleware/
│  │   ├─ authMiddleware.js
│  ├─ utils/
│  │   ├─ generateApiKey.js
│  ├─ package.json
├─ .env
└─ README.md


---

⚙️ 2️⃣ Contenido de los archivos principales

backend/package.json

{
  "name": "supabase-saas-backend",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "body-parser": "^1.20.2",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.2",
    "pg": "^8.11.1",
    "stripe": "^14.10.0"
  }
}


---

backend/index.js

require('dotenv').config();
const express = require('express');
const bodyParser = require('body-parser');
const authRoutes = require('./routes/auth');
const dataRoutes = require('./routes/data');
const { authMiddleware } = require('./middleware/authMiddleware');

const app = express();
app.use(bodyParser.json());

// rutas públicas
app.use('/auth', authRoutes);

// rutas protegidas (requieren API_KEY)
app.use('/api', authMiddleware, dataRoutes);

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`🚀 API corriendo en puerto ${PORT}`));


---

backend/middleware/authMiddleware.js

const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function authMiddleware(req, res, next) {
  const token = req.headers['authorization'];
  if (!token) return res.status(401).json({ error: 'Token requerido' });

  const result = await pool.query('SELECT * FROM api_keys WHERE key=$1', [token]);
  if (result.rows.length === 0) return res.status(403).json({ error: 'Token inválido' });

  const api = result.rows[0];

  if (!api.ilimitado) {
    const limite = getLimit(api.plan);
    if (api.consumido >= limite) return res.status(429).json({ error: 'Límite alcanzado' });
    await pool.query('UPDATE api_keys SET consumido=consumido+1 WHERE id=$1', [api.id]);
  }

  req.api = api;
  next();
}

function getLimit(plan) {
  switch (plan) {
    case 'gratis': return 100;
    case 'pro': return 10000;
    case 'business': return 99999999;
    default: return 0;
  }
}

module.exports = { authMiddleware };


---

backend/routes/auth.js

const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const { Pool } = require('pg');
const { generarApiKey } = require('../utils/generateApiKey');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Registro
router.post('/register', async (req, res) => {
  const { email, password, plan } = req.body;
  try {
    const user = await pool.query(
      'INSERT INTO users (email, password) VALUES ($1, $2) RETURNING id',
      [email, password]
    );
    const apiKey = generarApiKey();
    const ilimitado = (plan === 'admin');
    await pool.query(
      'INSERT INTO api_keys (usuario_id, key, plan, ilimitado) VALUES ($1, $2, $3, $4)',
      [user.rows[0].id, apiKey, plan || 'gratis', ilimitado]
    );
    res.json({ apiKey });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Error en registro' });
  }
});

// Login
router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const result = await pool.query('SELECT * FROM users WHERE email=$1 AND password=$2', [email, password]);
  if (result.rows.length === 0) return res.status(403).json({ error: 'Credenciales inválidas' });

  const token = jwt.sign({ id: result.rows[0].id }, process.env.JWT_SECRET, { expiresIn: '1h' });
  res.json({ token });
});

module.exports = router;


---

backend/utils/generateApiKey.js

const crypto = require('crypto');
function generarApiKey() {
  return crypto.randomBytes(32).toString('hex');
}
module.exports = { generarApiKey };


---

backend/routes/data.js

const express = require('express');
const router = express.Router();
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Ejemplo de endpoint protegido
router.get('/users', async (req, res) => {
  const result = await pool.query('SELECT id, email FROM users');
  res.json(result.rows);
});

router.post('/insert', async (req, res) => {
  const { table, data } = req.body;
  try {
    const keys = Object.keys(data).join(',');
    const values = Object.values(data);
    const placeholders = values.map((_, i) => `$${i + 1}`).join(',');
    const query = `INSERT INTO ${table} (${keys}) VALUES (${placeholders}) RETURNING *`;
    const result = await pool.query(query, values);
    res.json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Error insertando datos' });
  }
});

module.exports = router;


---

.env

MASTER_API_KEY=mi_master_key_fija
JWT_SECRET=mi_jwt_secreto
STRIPE_SECRET=sk_test_tu_stripe_secret
PRICE_PRO=price_abc123
PRICE_BUSINESS=price_xyz456
DATABASE_URL=postgres://usuario:password@host:port/db
PORT=3000


---

🧩 3️⃣ Crear base de datos PostgreSQL

Crea tablas básicas (puedes hacerlo con Supabase, Neon o Render):

CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(100) UNIQUE NOT NULL,
  password VARCHAR(100) NOT NULL
);

CREATE TABLE api_keys (
  id SERIAL PRIMARY KEY,
  usuario_id INT,
  key VARCHAR(255) UNIQUE NOT NULL,
  plan VARCHAR(50) DEFAULT 'gratis',
  consumido INT DEFAULT 0,
  ilimitado BOOLEAN DEFAULT FALSE,
  creado_at TIMESTAMP DEFAULT NOW()
);


---

🧠 4️⃣ Cómo probarlo

1. Inicia el backend:

cd backend
npm install
npm start


2. Prueba en Postman o curl:

# Registro
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@test.com","password":"1234","plan":"admin"}'

👉 Te devolverá tu apiKey ilimitada (MASTER).


3. Usa esa apiKey en los headers:

curl -X GET http://localhost:3000/api/users \
  -H "Authorization: tu_api_key"




---

💳 5️⃣ Activar Stripe y monetizar

En Stripe, crea 2 precios (price_abc123, price_xyz456)

Configura tus STRIPE_SECRET y PRICE_ID n) 
