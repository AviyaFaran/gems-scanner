# GEMS Daily Scanner — Instructions

## Step 1 — Load All Watchlists

Read ALL .txt files from the `watchlists/` folder. Each file is a watchlist.

Parsing rules:
- Tickers are comma-separated or newline-separated
- Strip exchange prefix: `NASDAQ:NVDA` → `NVDA`, `NYSE:AAPL` → `AAPL`
- Skip tokens starting with `###`
- Skip non-US exchanges: LSE:, EURONEXT:, TSX:, COINBASE:, SEED_
- DEDUPLICATE across all files — scan each ticker once, note which lists it appears in

Announce: `🔍 נטענו X רשימות | Y מניות ייחודיות לסריקה`

## Step 2 — Load Previous Recommendations

Check if `history/recommendations.json` exists. If yes, read it.
Format: `{"recommendations": [{"ticker": "NVDA", "date": "DD.MM.YYYY", "action": "קנה עכשיו", "entry": 120.5, "tp": 145.0, "sl": 112.0}]}`
If missing — continue without history.

## Step 3 — Fetch Data & Pre-Filter

For EACH ticker run this Node.js command (replace TICKER_HERE with plain symbol):

```bash
node -e "
const ticker = 'TICKER_HERE';
const url = 'https://query1.finance.yahoo.com/v8/finance/chart/' + ticker + '?interval=1d&range=1y';
const https = require('https');
https.get(url, {headers: {'User-Agent': 'Mozilla/5.0'}}, (res) => {
  let d = '';
  res.on('data', c => d += c);
  res.on('end', () => {
    try {
      const j = JSON.parse(d);
      const r = j.chart.result[0];
      const q = r.indicators.quote[0];
      const raw = q.close.map((c,i) => ({c, h: q.high[i], l: q.low[i], v: q.volume[i], o: q.open[i]})).filter(b => b.c && b.h && b.l && b.v);
      const closes = raw.map(b => b.c);
      const highs = raw.map(b => b.h);
      const lows = raw.map(b => b.l);
      const vols = raw.map(b => b.v);
      const n = raw.length;
      const ma20 = closes.slice(-20).reduce((a,b) => a+b, 0) / Math.min(20, n);
      const ma150 = closes.slice(-150).reduce((a,b) => a+b, 0) / Math.min(150, n);
      const price = closes[n-1];
      const tp = raw.slice(-20).map(b => (b.h+b.l+b.c)/3);
      const tpAvg = tp.reduce((a,b) => a+b, 0) / 20;
      const md = tp.reduce((a,b) => a + Math.abs(b - tpAvg), 0) / 20;
      const cci = md !== 0 ? (tp[19] - tpAvg) / (0.015 * md) : 0;
      const volLast = vols[n-1];
      const volAvg20 = vols.slice(-20).reduce((a,b) => a+b, 0) / Math.min(20, n);
      const volRatio = volAvg20 > 0 ? volLast / volAvg20 : 0;
      let atrSum = 0;
      for (let i = n-14; i < n; i++) {
        atrSum += Math.max(highs[i]-lows[i], Math.abs(highs[i]-(closes[i-1]||closes[i])), Math.abs(lows[i]-(closes[i-1]||closes[i])));
      }
      const atrPct = (atrSum/14/price)*100;
      const last = raw[n-1];
      const body = Math.abs(last.c - last.o);
      const uw = last.h - Math.max(last.c, last.o);
      const lw = Math.min(last.c, last.o) - last.l;
      let candle = 'Neutral';
      if (lw > 2*body && uw < body) candle = 'Hammer';
      else if (uw > 2*body && lw < body) candle = 'ShootingStar';
      else if (last.c > last.o && body > 0.6*(last.h-last.l)) candle = 'Bullish';
      else if (last.c < last.o && body > 0.6*(last.h-last.l)) candle = 'Bearish';
      const ath = Math.max(...closes);
      console.log('OK|PRICE:'+price.toFixed(2)+'|MA20:'+ma20.toFixed(2)+'|MA150:'+ma150.toFixed(2)+'|CCI:'+cci.toFixed(1)+'|VOL:'+volLast+'|VOLAVG20:'+volAvg20.toFixed(0)+'|VOLRATIO:'+volRatio.toFixed(2)+'|ATR_PCT:'+atrPct.toFixed(1)+'|CANDLE:'+candle+'|ATH:'+ath.toFixed(2));
    } catch(e) { console.log('ERROR|' + e.message); }
  });
}).on('error', e => console.log('ERROR|' + e.message));
"
```

Pre-filter (all 3 must pass):
- PRICE > MA150
- CCI > -150
- VOLRATIO > 0.30

Bonus signal (noted in report, not a filter): PRICE > MA20 = short-term momentum confirmed.

Show progress every 50 stocks. Skip ERROR tickers.

## Step 4 — Deep GEMS Analysis (max 20 candidates)

For each candidate run full GEMS 5-stage analysis:

1. **Trend Guardian** — Price > MA150? Price > MA20? Distance from each in %? Both confirm uptrend?
2. **Trampoline Setup** — Pullback to MA20 or support? CCI bouncing from -100 zone?
3. **R:R Mathematics** — TP = nearest resistance. SL = below swing low or -5%. R:R >= 1:3 required.
4. **Trigger** — Bullish candle / Hammer / VOLRATIO > 1.2?
5. **Defense Wall** — ATR < 8%?

If ticker in history add flag:
`🔁 **מחזק מגמה** — הומלץ ב-[DATE] בכניסה $[ENTRY] | כעת $[PRICE] ([+/-X%])`

If R:R < 1:3 → "לעקוב" category.

## Step 5 — Save New Recommendations

Create/update `history/recommendations.json`:
- Append all "קנה עכשיו" and "המתן ל" tickers
- Keep last 90 days only
- Format: `{"recommendations": [{"ticker": "X", "date": "DD.MM.YYYY", "action": "קנה עכשיו", "entry": X, "tp": X, "sl": X}]}`
- Save with Write tool

## Step 6 — Email Report

Send to info@tnx-aviya.com via Gmail MCP.

Subject: `📊 GEMS Scan [DD.MM.YYYY] — [Z] מניות לטרייד`

Full detailed Hebrew email body:
```
סרקתי [X] מניות מ-[N] רשימות | עברו pre-filter: [Y] | אישורי כניסה: [Z]

━━━━━━━━━━━━━━━━━━━━━━━━
🟢 קנה עכשיו ([Z]):
━━━━━━━━━━━━━━━━━━━━━━━━
[TICKER] — [Company Name] | רשימות: [list names]
[🔁 **מחזק מגמה** — הומלץ ב-DD.MM.YYYY בכניסה $X | כעת $Y ([+Z%])]
🚦 שורה תחתונה: קנה עכשיו
⚖️ R:R: [X:Y] | זמן משוער: [X-Y ימים]
📍 כניסה: $[X] | SL: $[X] | TP: $[X]
📝 MA20=$[X] | MA150=$[X] | CCI=[X] | ווליום [X]% מממוצע | ATR=[X]%
💡 [one-line setup summary]
[Full 5-stage analysis]

━━━━━━━━━━━━━━━━━━━━━━━━
🟡 המתן ל... ([N]):
━━━━━━━━━━━━━━━━━━━━━━━━
[TICKER] — ממתין ל: [condition]
📍 כניסה פוטנציאלית: $[X] | SL: $[X] | TP: $[X]

━━━━━━━━━━━━━━━━━━━━━━━━
⚪ לעקוב ([N]):
━━━━━━━━━━━━━━━━━━━━━━━━
[TICKER] — [reason]

❌ נדחו: [X] מתחת MA150 | [Y] CCI שלילי | [Z] ווליום נמוך

---
GEMS Protocol | אוטומטי | info@tnx-aviya.com
```

⚠️ ניתוח טכני אוטומטי בלבד — אינו מהווה המלצת השקעה.