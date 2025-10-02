require('dotenv').config();
const express = require('express');
const cors = require('cors');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const bodyParser = require('body-parser');
const axios = require('axios');

const app = express();
const PORT = process.env.PORT || 3000;
const JWT_SECRET = process.env.JWT_SECRET || 'replace_this_secret';
const OPENAI_KEY = process.env.OPENAI_API_KEY;

if (!OPENAI_KEY) {
  console.warn('WARNING: OPENAI_API_KEY is not set. The /api/chat endpoint will fail until you set it in .env');
}

app.use(cors());
app.use(bodyParser.json());

// Simple in-memory "database" for demo purposes.
const users = {}; // { email: { passwordHash } }

function generateToken(email) {
  return jwt.sign({ email }, JWT_SECRET, { expiresIn: '7d' });
}

// Signup
app.post('/api/signup', async (req, res) => {
  try {
    const { email, password } = req.body;
    if (!email || !password) return res.status(400).json({ message: 'Missing email or password' });
    if (users[email]) return res.status(409).json({ message: 'User already exists' });

    const hash = await bcrypt.hash(password, 10);
    users[email] = { passwordHash: hash };
    const token = generateToken(email);
    return res.json({ token });
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

// Login
app.post('/api/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    if (!email || !password) return res.status(400).json({ message: 'Missing email or password' });
    const user = users[email];
    if (!user) return res.status(401).json({ message: 'Invalid credentials' });

    const ok = await bcrypt.compare(password, user.passwordHash);
    if (!ok) return res.status(401).json({ message: 'Invalid credentials' });

    const token = generateToken(email);
    return res.json({ token });
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

// Middleware: protect endpoints
function authenticate(req, res, next) {
  const auth = req.headers.authorization;
  if (!auth) return res.status(401).json({ message: 'No token provided' });
  const parts = auth.split(' ');
  const token = parts.length === 2 ? parts[1] : parts[0];
  try {
    const payload = jwt.verify(token, JWT_SECRET);
    req.user = payload;
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Invalid token' });
  }
}

// Chat proxy -> calls OpenAI
app.post('/api/chat', authenticate, async (req, res) => {
  try {
    const { message } = req.body;
    if (!message) return res.status(400).json({ message: 'No message provided' });

    const openaiUrl = 'https://api.openai.com/v1/chat/completions';
    const payload = {
      model: process.env.OPENAI_MODEL || 'gpt-4o-mini',
      messages: [
        { role: 'system', content: 'You are SDK Chat, a helpful assistant.' },
        { role: 'user', content: message }
      ],
      max_tokens: 800,
      temperature: 0.2
    };

    const r = await axios.post(openaiUrl, payload, {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${OPENAI_KEY}`
      }
    });

    const reply = r.data?.choices?.[0]?.message?.content || 'No response';
    return res.json({ response: reply });
  } catch (err) {
    console.error('OpenAI error:', err?.response?.data || err.message || err);
    const msg = err?.response?.data?.error?.message || 'Error contacting AI';
    return res.status(500).json({ message: msg });
  }
});

app.get('/', (req, res) => res.send('SDK Chat backend running'));

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
