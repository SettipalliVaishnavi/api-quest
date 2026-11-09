const express = require('express');
const cors = require('cors');
const crypto = require('crypto');

const app = express();
const PORT = process.env.PORT || 3000;
const SECRET = process.env.TOKEN_SECRET || 'api-quest-dev-secret';
// auto  = GET /books becomes protected after the first token is issued (works for every level in order)
// true  = always protected, false = never protected
const PROTECT_BOOKS = (process.env.PROTECT_BOOKS || 'auto').toLowerCase();
const DELETE_STATUS = parseInt(process.env.DELETE_STATUS || '204', 10);
const PAGINATION_ENVELOPE = process.env.PAGINATION_ENVELOPE === 'true';

app.disable('x-powered-by');
app.use(cors());
app.use(express.json({ limit: '1mb' }));

// ---------- helpers ----------
let tokenIssued = false;
const b64 = (s) => Buffer.from(s).toString('base64url');
const sign = (data) => crypto.createHmac('sha256', SECRET).update(data).digest('base64url');

function createToken(sub) {
  const payload = b64(JSON.stringify({ sub, exp: Date.now() + 24 * 3600 * 1000 }));
  return `${payload}.${sign(payload)}`;
}
function verifyToken(token) {
  const [payload, sig] = (token || '').split('.');
  if (!payload || !sig) return false;
  const expected = sign(payload);
  if (sig.length !== expected.length) return false;
  if (!crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(expected))) return false;
  try { return JSON.parse(Buffer.from(payload, 'base64url').toString()).exp > Date.now(); }
  catch { return false; }
}
function requireAuth(req, res, next) {
  const active = PROTECT_BOOKS === 'true' || (PROTECT_BOOKS === 'auto' && tokenIssued);
  if (!active) return next();
  const m = (req.headers.authorization || '').match(/^Bearer\s+(.+)$/i);
  if (!m || !verifyToken(m[1])) return res.status(401).json({ error: 'Unauthorized: missing or invalid token' });
  next();
}
const isStr = (v) => typeof v === 'string' && v.trim().length > 0;

// ---------- Level 1: Ping ----------
app.get('/ping', (req, res) => res.status(200).json({ message: 'pong', status: 'ok' }));

// ---------- Level 2: Echo ----------
app.post('/echo', (req, res) => res.status(200).json(req.body === undefined ? {} : req.body));

// ---------- Level 5: Auth ----------
app.post('/auth/token', (req, res) => {
  const { username } = req.body || {};
  const token = createToken(username || 'user');
  tokenIssued = true;
  res.status(200).json({ token, access_token: token, token_type: 'Bearer', expires_in: 86400 });
});

// ---------- Books (in-memory) ----------
const books = [];
let nextId = 1;

function validateBook(body, partial = false) {
  if (!body || typeof body !== 'object' || Array.isArray(body)) return 'Request body must be a JSON object';
  if (!partial || 'title' in body) if (!isStr(body.title)) return 'title is required and must be a non-empty string';
  if (!partial || 'author' in body) if (!isStr(body.author)) return 'author is required and must be a non-empty string';
  if ('year' in body && body.year !== null && !Number.isInteger(body.year)) return 'year must be an integer';
  return null;
}

// Level 3 + 6: create, list (filter + paginate), read
app.post('/books', (req, res) => {
  const err = validateBook(req.body);
  if (err) return res.status(400).json({ error: err });
  const { title, author, year } = req.body;
  const book = { id: nextId++, title: title.trim(), author: author.trim() };
  if (year !== undefined) book.year = year;
  books.push(book);
  res.status(201).json(book);
});

app.get('/books', requireAuth, (req, res) => {
  let result = books;
  if (isStr(req.query.author)) {
    const a = req.query.author.trim().toLowerCase();
    result = result.filter((b) => b.author.toLowerCase() === a);
  }
  const total = result.length;
  const hasPaging = req.query.page !== undefined || req.query.limit !== undefined;
  let page = 1, limit = total || 1;
  if (hasPaging) {
    page = Math.max(1, parseInt(req.query.page, 10) || 1);
    limit = Math.min(100, Math.max(1, parseInt(req.query.limit, 10) || 10));
    result = result.slice((page - 1) * limit, page * limit);
  }
  res.set({ 'X-Total-Count': total, 'X-Page': page, 'X-Limit': limit });
  if (PAGINATION_ENVELOPE && hasPaging) {
    return res.json({ data: result, page, limit, total, totalPages: Math.ceil(total / limit) });
  }
  res.json(result);
});

app.get('/books/:id', (req, res) => {
  const book = books.find((b) => String(b.id) === req.params.id);
  if (!book) return res.status(404).json({ error: 'Book not found' });
  res.json(book);
});

// Level 4: update & delete
app.put('/books/:id', (req, res) => {
  const book = books.find((b) => String(b.id) === req.params.id);
  if (!book) return res.status(404).json({ error: 'Book not found' });
  const err = validateBook(req.body, true);
  if (err) return res.status(400).json({ error: err });
  for (const k of ['title', 'author', 'year']) if (k in req.body) book[k] = typeof req.body[k] === 'string' ? req.body[k].trim() : req.body[k];
  res.json(book);
});

app.delete('/books/:id', (req, res) => {
  const i = books.findIndex((b) => String(b.id) === req.params.id);
  if (i === -1) return res.status(404).json({ error: 'Book not found' });
  const [removed] = books.splice(i, 1);
  if (DELETE_STATUS === 204) return res.status(204).end();
  res.status(DELETE_STATUS).json({ message: 'Book deleted', book: removed });
});

// ---------- Level 7: error handling ----------
app.use((req, res) => res.status(404).json({ error: 'Not found' }));
app.use((err, req, res, next) => {
  if (err.type === 'entity.parse.failed') return res.status(400).json({ error: 'Invalid JSON body' });
  console.error(err);
  res.status(500).json({ error: 'Internal server error' });
});

app.listen(PORT, () => console.log(`API Quest listening on port ${PORT}`));