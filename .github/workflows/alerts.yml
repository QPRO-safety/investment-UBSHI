// Investment Agent - cloud alert checker (runs in GitHub Actions)
// Reads the portfolio from the private Gist, checks market alerts, sends them to Telegram.
// Alert state is kept in the same Gist (file: alert_state.json) so each crossing fires only once.

const GIST_TOKEN = process.env.GIST_TOKEN;
const GIST_ID    = process.env.GIST_ID;
const TG_TOKEN   = process.env.TG_TOKEN;
const TG_CHAT    = process.env.TG_CHAT;
const DRY_RUN    = process.env.DRY_RUN === '1';
const COOLDOWN_MS = 6 * 60 * 60 * 1000; // same ticker+type at most once per 6 hours
const ENABLE_RSI_SUPPORT = false;       // set to true to also get RSI and support-break alerts

// ---------- indicators (same math as the app) ----------
export function calcMA(arr, n) { if (arr.length < n) return null; return arr.slice(-n).reduce((a, b) => a + b, 0) / n; }
export function calcRSI(arr, n = 14) {
  if (arr.length < n + 1) return null;
  const ch = arr.slice(1).map((c, i) => c - arr[i]);
  const rc = ch.slice(-n);
  let g = 0, l = 0;
  rc.forEach(c => c > 0 ? g += c : l += Math.abs(c));
  const ag = g / n, al = l / n;
  return al === 0 ? 100 : 100 - (100 / (1 + ag / al));
}
export function analyze(closes) {
  if (!closes || closes.length < 20) return null;
  const ma150 = calcMA(closes, 150);
  const ma200 = calcMA(closes, 200);
  const rsi = calcRSI(closes);
  // support = lowest close of the previous 20 completed bars (excludes the live bar, otherwise a break can never be detected)
  const prior = closes.slice(-21, -1);
  const support = prior.length ? Math.min(...prior) : null;
  return { ma150, ma200, rsi, support };
}

// ---------- alert rules (same rules as the app's checkAlerts) ----------
export function evaluate(ticker, price, a, prev) {
  const out = [];
  if (!a || !price || !prev) return out;
  if (a.ma150 && prev.ma150) {
    if (prev.price < prev.ma150 && price >= a.ma150) out.push({ type: 'MA150_UP', text: `${ticker} חצה מעלה את ממוצע 150 ($${a.ma150.toFixed(2)}) 🚀` });
    if (prev.price > prev.ma150 && price <= a.ma150) out.push({ type: 'MA150_DN', text: `${ticker} ירד מתחת לממוצע 150 ($${a.ma150.toFixed(2)}) ⚠` });
  }
  if (a.ma200 && prev.ma200) {
    if (prev.price < prev.ma200 && price >= a.ma200) out.push({ type: 'MA200_UP', text: `${ticker} חצה מעלה את ממוצע 200 ($${a.ma200.toFixed(2)}) 🚀` });
    if (prev.price > prev.ma200 && price <= a.ma200) out.push({ type: 'MA200_DN', text: `${ticker} ירד מתחת לממוצע 200 ($${a.ma200.toFixed(2)}) ⚠` });
  }
  if (ENABLE_RSI_SUPPORT) {
    if (a.rsi > 75 && prev.rsi <= 75) out.push({ type: 'RSI_OB', text: `${ticker} RSI הגיע ל-${a.rsi.toFixed(1)} — אזור over-bought` });
    if (a.support && price < a.support && prev.price >= a.support) out.push({ type: 'SUP_BRK', text: `${ticker} שבר תמיכה ב-$${a.support.toFixed(2)}` });
  }
  return out;
}

// ---------- data ----------
async function fetchYahoo(ticker) {
  const urls = [
    `https://query1.finance.yahoo.com/v8/finance/chart/${encodeURIComponent(ticker)}?interval=1d&range=1y`,
    `https://query2.finance.yahoo.com/v8/finance/chart/${encodeURIComponent(ticker)}?interval=1d&range=1y`,
  ];
  for (const url of urls) {
    try {
      const r = await fetch(url, { headers: { 'User-Agent': 'Mozilla/5.0 (compatible; InvestmentAgent/1.0)' } });
      if (!r.ok) continue;
      const d = await r.json();
      const res = d?.chart?.result?.[0];
      const closes = (res?.indicators?.quote?.[0]?.close || []).filter(c => c != null);
      const price = res?.meta?.regularMarketPrice ?? closes[closes.length - 1];
      if (closes.length >= 20 && price) return { closes, price };
    } catch (e) { /* try next */ }
  }
  return null;
}

async function gistGet() {
  const r = await fetch(`https://api.github.com/gists/${GIST_ID}`, {
    headers: { Authorization: `token ${GIST_TOKEN}`, 'User-Agent': 'investment-agent' }
  });
  if (!r.ok) throw new Error(`Gist read failed: HTTP ${r.status}`);
  return r.json();
}

async function gistSaveState(state) {
  const r = await fetch(`https://api.github.com/gists/${GIST_ID}`, {
    method: 'PATCH',
    headers: { Authorization: `token ${GIST_TOKEN}`, 'User-Agent': 'investment-agent', 'Content-Type': 'application/json' },
    body: JSON.stringify({ files: { 'alert_state.json': { content: JSON.stringify(state, null, 2) } } })
  });
  if (!r.ok) throw new Error(`Gist state save failed: HTTP ${r.status}`);
}

async function tgSend(text) {
  if (DRY_RUN) { console.log('[DRY_RUN] would send:', text); return; }
  const r = await fetch(`https://api.telegram.org/bot${TG_TOKEN}/sendMessage`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ chat_id: TG_CHAT, text })
  });
  const d = await r.json().catch(() => ({}));
  if (!r.ok || !d.ok) throw new Error(`Telegram failed: ${d.description || r.status}`);
}

// ---------- main ----------
export async function main() {
  for (const [k, v] of Object.entries({ GIST_TOKEN, GIST_ID, TG_TOKEN, TG_CHAT })) {
    if (!v) throw new Error(`Missing secret: ${k}`);
  }
  const gist = await gistGet();
  const raw = gist.files?.['investment_data.json']?.content;
  if (!raw) throw new Error('investment_data.json not found in Gist');
  const securities = JSON.parse(raw).securities || [];
  const state = gist.files?.['alert_state.json']?.content ? JSON.parse(gist.files['alert_state.json'].content) : { tickers: {}, lastSent: {} };
  state.tickers ||= {}; state.lastSent ||= {};

  const tickers = [...new Set(securities.map(s => s.ticker).filter(Boolean))];
  let ok = 0, failed = [], sent = 0;
  const now = Date.now();

  for (const ticker of tickers) {
    const data = await fetchYahoo(ticker);
    await new Promise(r => setTimeout(r, 300));
    if (!data) { failed.push(ticker); continue; }
    ok++;
    const a = analyze(data.closes);
    const alerts = evaluate(ticker, data.price, a, state.tickers[ticker]);
    for (const al of alerts) {
      const key = `${ticker}|${al.type}`;
      if (state.lastSent[key] && now - state.lastSent[key] < COOLDOWN_MS) continue;
      await tgSend(`📈 ${ticker}\n${al.text}`);
      state.lastSent[key] = now;
      sent++;
    }
    if (a) state.tickers[ticker] = { price: data.price, ma150: a.ma150 || 0, ma200: a.ma200 || 0, rsi: a.rsi ?? 50 };
  }

  console.log(`checked=${ok} failed=${failed.length} sent=${sent}`);
  if (failed.length) console.log('no data for:', failed.join(', '));
  if (ok === 0) throw new Error('No price data for any ticker');
  if (!DRY_RUN) await gistSaveState(state);
}

if (import.meta.url === `file://${process.argv[1]}`) {
  main().catch(e => { console.error(e.message); process.exit(1); });
}
