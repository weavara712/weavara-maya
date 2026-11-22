// Weavara backend for Vercel (replaces the Google Sheet script).
// Storage: Upstash Redis (connect it from Vercel > Storage). Env vars needed: ADMIN_KEY (you set it).
// Upstash adds KV_REST_API_URL and KV_REST_API_TOKEN by itself.
const crypto = require('crypto');

const REDIS_URL = process.env.KV_REST_API_URL || process.env.UPSTASH_REDIS_REST_URL;
const REDIS_TOKEN = process.env.KV_REST_API_TOKEN || process.env.UPSTASH_REDIS_REST_TOKEN;
const ADMIN_KEY = process.env.ADMIN_KEY;

const VCOLS = ['id','ts','type','page','route','title','lang','referrer','utm','entryUrl','vid','sid','device'];
const COLS = ['id','ts','type','name','phone','address','pincode','note','productId','productName','price','collection','bookedFromUrl','entryUrl','referrer','utm','lang','whatsapp','status'];
const BOOKING_TYPES = ['order', 'consultation', 'whatsapp_click'];
const K_VISITS = 'weavara:visits';
const K_BOOKINGS = 'weavara:bookings';

async function redis(cmd) {
  const r = await fetch(REDIS_URL, {
    method: 'POST',
    headers: { Authorization: 'Bearer ' + REDIS_TOKEN, 'Content-Type': 'application/json' },
    body: JSON.stringify(cmd)
  });
  const j = await r.json();
  if (j.error) throw new Error(j.error);
  return j.result;
}
function pick(d, cols) {
  const o = {};
  cols.forEach(c => { o[c] = d[c] == null ? '' : String(d[c]).slice(0, 1000); });
  return o;
}
function authOk(key) {
  if (!ADMIN_KEY || typeof key !== 'string') return false;
  const a = Buffer.from(key), b = Buffer.from(ADMIN_KEY);
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
function parseBody(req) {
  let b = req.body;
  if (Buffer.isBuffer(b)) b = b.toString('utf8');
  if (typeof b === 'string') { try { return JSON.parse(b); } catch (e) { return null; } }
  return b && typeof b === 'object' ? b : null;
}
function send(res, o, code) {
  res.setHeader('Content-Type', 'application/json');
  res.setHeader('Cache-Control', 'no-store');
  res.status(code || 200).send(JSON.stringify(o));
}
function rows(list) {
  return (list || []).map(s => { try { return JSON.parse(s); } catch (e) { return null; } }).filter(Boolean);
}

module.exports = async function handler(req, res) {
  try {
    if (!REDIS_URL || !REDIS_TOKEN || !ADMIN_KEY) return send(res, { ok: false, error: 'config' });

    if (req.method === 'GET') {
      const q = req.query || {};
      if (!authOk(q.key)) return send(res, { ok: false, error: 'auth' });
      if (q.action === 'visits') return send(res, { ok: true, rows: rows(await redis(['LRANGE', K_VISITS, -5000, -1])) });
      return send(res, { ok: true, rows: rows(await redis(['LRANGE', K_BOOKINGS, 0, -1])) });
    }

    if (req.method === 'POST') {
      const d = parseBody(req);
      if (!d) return send(res, { ok: false });

      if (d.type === 'page_view') {
        await redis(['RPUSH', K_VISITS, JSON.stringify(pick(d, VCOLS))]);
        await redis(['LTRIM', K_VISITS, -20000, -1]);
        return send(res, { ok: true });
      }

      if (d.action === 'update') {
        if (!authOk(d.key)) return send(res, { ok: false, error: 'auth' });
        const list = await redis(['LRANGE', K_BOOKINGS, 0, -1]) || [];
        for (let i = 0; i < list.length; i++) {
          let o; try { o = JSON.parse(list[i]); } catch (e) { continue; }
          if (o.id === d.id) {
            o.status = String(d.status || '').slice(0, 40);
            await redis(['LSET', K_BOOKINGS, i, JSON.stringify(o)]);
            return send(res, { ok: true });
          }
        }
        return send(res, { ok: false, error: 'not found' });
      }

      if (!BOOKING_TYPES.includes(d.type)) return send(res, { ok: false });
      await redis(['RPUSH', K_BOOKINGS, JSON.stringify(pick(d, COLS))]);
      await redis(['LTRIM', K_BOOKINGS, -5000, -1]);
      return send(res, { ok: true });
    }

    return send(res, { ok: false }, 405);
  } catch (e) {
    return send(res, { ok: false, error: 'server' }, 500);
  }
};
