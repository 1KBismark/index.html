# index.html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover"/>
<meta name="theme-color" content="#0e1117"/>
<link rel="manifest" href="manifest.webmanifest"/>
<link rel="icon" href="icon.svg" type="image/svg+xml"/>
<link rel="apple-touch-icon" href="icon-192.png"/>
<title>MiniChart</title>
<style>
  :root{color-scheme:dark}
  html,body{height:100%}
  body{
    margin:0;display:flex;flex-direction:column;height:100dvh;overflow:hidden;
    overscroll-behavior:none;background:#0e1117;color:#d1d4dc;
    font:13px/1.4 system-ui,sans-serif;
    padding-top:env(safe-area-inset-top);
    padding-bottom:env(safe-area-inset-bottom);
  }
  header{
    flex:0 0 auto;display:flex;gap:8px;align-items:center;
    padding:8px;border-bottom:1px solid #1f2733;
    overflow-x:auto;white-space:nowrap;scrollbar-width:none;
  }
  header::-webkit-scrollbar{display:none}
  select,button{
    background:#161b22;color:#d1d4dc;border:1px solid #2a2e39;border-radius:6px;
    padding:6px 9px;font:inherit;flex:0 0 auto;
  }
  button{cursor:pointer}
  label.tg{display:flex;gap:4px;align-items:center;flex:0 0 auto;user-select:none}
  #main{flex:1 1 auto;min-height:0}
  #osc{flex:0 0 26vh;min-height:0;border-top:1px solid #1f2733}
  #status{color:#8b949e;flex:0 0 auto}
</style>
</head>
<body>
<header>
  <select id="symbol">
    <option>BTCUSDT</option><option>ETHUSDT</option>
    <option>SOLUSDT</option><option>BNBUSDT</option><option>XRPUSDT</option>
  </select>
  <select id="interval">
    <option value="15m">15m</option>
    <option value="1h" selected>1h</option>
    <option value="4h">4h</option>
    <option value="1d">1d</option>
  </select>
  <label class="tg"><input type="checkbox" id="sma" checked> SMA 20</label>
  <label class="tg"><input type="checkbox" id="ema"> EMA 50</label>
  <label class="tg"><input type="checkbox" id="bb"> BB 20/2</label>
  <select id="oscSel">
    <option value="rsi">RSI 14</option>
    <option value="macd">MACD 12/26/9</option>
  </select>
  <button id="reload">Reload</button>
  <span id="status"></span>
</header>

<div id="main"></div>
<div id="osc"></div>

<script>
  // show a friendly message if the chart library can't load (e.g. first run offline)
  window.addEventListener('error', () => {
    const s = document.getElementById('status');
    if (s) s.textContent = 'offline — reconnect and reload';
  }, true);
</script>
<script src="https://unpkg.com/lightweight-charts@4.1.3/dist/lightweight-charts.standalone.production.js"></script>
<script>
const { createChart } = LightweightCharts;

const statusEl = document.getElementById('status');

const baseOpts = {
  autoSize: true,
  layout: { background: { color: '#0e1117' }, textColor: '#d1d4dc', fontSize: 11 },
  grid: { vertLines: { color: '#1a2029' }, horzLines: { color: '#1a2029' } },
  rightPriceScale: { borderColor: '#2a2e39' },
  timeScale: { borderColor: '#2a2e39', timeVisible: true, secondsVisible: false },
  crosshair: { mode: LightweightCharts.CrosshairMode.Normal }
};

const chart    = createChart(document.getElementById('main'), baseOpts);
const oscChart = createChart(document.getElementById('osc'),  baseOpts);

const candles = chart.addCandlestickSeries({
  upColor: '#26a69a', downColor: '#ef5350',
  wickUpColor: '#26a69a', wickDownColor: '#ef5350',
  borderVisible: false
});
const volume = chart.addHistogramSeries({
  priceFormat: { type: 'volume' },
  priceScaleId: 'vol',
  priceLineVisible: false, lastValueVisible: false
});
chart.priceScale('vol').applyOptions({ scaleMargins: { top: 0.85, bottom: 0 } });

/* ---------- indicators ---------- */
function sma(data, period) {
  const out = []; let sum = 0;
  for (let i = 0; i < data.length; i++) {
    sum += data[i].close;
    if (i >= period) sum -= data[i - period].close;
    if (i >= period - 1) out.push({ time: data[i].time, value: sum / period });
  }
  return out;
}

function ema(data, period) {
  const k = 2 / (period + 1), out = []; let prev = null;
  for (let i = period - 1; i < data.length; i++) {
    if (prev === null) {
      let s = 0;
      for (let j = i - period + 1; j <= i; j++) s += data[j].close;
      prev = s / period;
    } else {
      prev = data[i].close * k + prev * (1 - k);
    }
    out.push({ time: data[i].time, value: prev });
  }
  return out;
}

function emaOfValues(vals, period) {
  const k = 2 / (period + 1), out = []; let prev = null;
  for (let i = period - 1; i < vals.length; i++) {
    if (prev === null) {
      let s = 0;
      for (let j = i - period + 1; j <= i; j++) s += vals[j].value;
      prev = s / period;
    } else {
      prev = vals[i].value * k + prev * (1 - k);
    }
    out.push({ time: vals[i].time, value: prev });
  }
  return out;
}

function bollinger(data, period = 20, mult = 2) {
  const upper = [], lower = [];
  for (let i = period - 1; i < data.length; i++) {
    let sum = 0;
    for (let j = i - period + 1; j <= i; j++) sum += data[j].close;
    const mean = sum / period;
    let vsum = 0;
    for (let j = i - period + 1; j <= i; j++) vsum += (data[j].close - mean) ** 2;
    const sd = Math.sqrt(vsum / period);
    upper.push({ time: data[i].time, value: mean + mult * sd });
    lower.push({ time: data[i].time, value: mean - mult * sd });
  }
  return { upper, lower };
}

function rsi(data, period = 14) {
  const out = []; let gain = 0, loss = 0;
  for (let i = 1; i < data.length; i++) {
    const ch = data[i].close - data[i - 1].close;
    const g = Math.max(ch, 0), l = Math.max(-ch, 0);
    if (i <= period) {
      gain += g; loss += l;
      if (i === period) {
        gain /= period; loss /= period;
        out.push({ time: data[i].time, value: 100 - 100 / (1 + gain / (loss || 1e-9)) });
      }
    } else {
      gain = (gain * (period - 1) + g) / period;
      loss = (loss * (period - 1) + l) / period;
      out.push({ time: data[i].time, value: 100 - 100 / (1 + gain / (loss || 1e-9)) });
    }
  }
  return out;
}

function macd(data, fast = 12, slow = 26, signal = 9) {
  const f = ema(data, fast), s = ema(data, slow);
  const sMap = new Map(s.map(p => [p.time, p.value]));
  const macdLine = f.filter(p => sMap.has(p.time))
                    .map(p => ({ time: p.time, value: p.value - sMap.get(p.time) }));
  const signalLine = emaOfValues(macdLine, signal);
  const sigMap = new Map(signalLine.map(p => [p.time, p.value]));
  const hist = macdLine.filter(p => sigMap.has(p.time))
                       .map(p => ({ time: p.time, value: p.value - sigMap.get(p.time) }));
  return { macdLine, signalLine, hist };
}

/* ---------- overlays ---------- */
const ov = { sma: null, ema: null, bbU: null, bbL: null };
let DATA = [];

function renderOverlays() {
  const wantSma = document.getElementById('sma').checked;
  const wantEma = document.getElementById('ema').checked;
  const wantBb  = document.getElementById('bb').checked;

  if (wantSma && !ov.sma) ov.sma = chart.addLineSeries({ color: '#f5c542', lineWidth: 2, title: 'SMA20', priceLineVisible: false, lastValueVisible: false });
  if (!wantSma && ov.sma) { chart.removeSeries(ov.sma); ov.sma = null; }
  if (ov.sma) ov.sma.setData(sma(DATA, 20));

  if (wantEma && !ov.ema) ov.ema = chart.addLineSeries({ color: '#42a5f5', lineWidth: 2, title: 'EMA50', priceLineVisible: false, lastValueVisible: false });
  if (!wantEma && ov.ema) { chart.removeSeries(ov.ema); ov.ema = null; }
  if (ov.ema) ov.ema.setData(ema(DATA, 50));

  if (wantBb && !ov.bbU) {
    ov.bbU = chart.addLineSeries({ color: '#8b949e', lineWidth: 1, lineStyle: 2, priceLineVisible: false, lastValueVisible: false });
    ov.bbL = chart.addLineSeries({ color: '#8b949e', lineWidth: 1, lineStyle: 2, priceLineVisible: false, lastValueVisible: false });
  }
  if (!wantBb && ov.bbU) {
    chart.removeSeries(ov.bbU); chart.removeSeries(ov.bbL);
    ov.bbU = ov.bbL = null;
  }
  if (ov.bbU) {
    const { upper, lower } = bollinger(DATA, 20, 2);
    ov.bbU.setData(upper); ov.bbL.setData(lower);
  }
}

/* ---------- oscillator pane ---------- */
let oscSeries = [];
function renderOsc() {
  oscSeries.forEach(s => oscChart.removeSeries(s));
  oscSeries = [];
  const mode = document.getElementById('oscSel').value;

  if (mode === 'rsi') {
    const s = oscChart.addLineSeries({ color: '#7e57c2', lineWidth: 2, title: 'RSI', priceLineVisible: false, lastValueVisible: false });
    s.setData(rsi(DATA, 14));
    s.createPriceLine({ price: 70, color: '#ef5350', lineWidth: 1, lineStyle: 2, axisLabelVisible: true, title: '' });
    s.createPriceLine({ price: 30, color: '#26a69a', lineWidth: 1, lineStyle: 2, axisLabelVisible: true, title: '' });
    oscSeries.push(s);
  } else {
    const { macdLine, signalLine, hist } = macd(DATA);
    const h = oscChart.addHistogramSeries({ priceLineVisible: false, lastValueVisible: false, title: 'Hist' });
    h.setData(hist.map(p => ({ time: p.time, value: p.value, color: p.value >= 0 ? 'rgba(38,166,154,.6)' : 'rgba(239,83,80,.6)' })));
    const m = oscChart.addLineSeries({ color: '#42a5f5', lineWidth: 2, priceLineVisible: false, lastValueVisible: false, title: 'MACD' });
    m.setData(macdLine);
    const sig = oscChart.addLineSeries({ color: '#f5c542', lineWidth: 1, priceLineVisible: false, lastValueVisible: false, title: 'Signal' });
    sig.setData(signalLine);
    oscSeries.push(h, m, sig);
  }
}

/* ---------- sync panes ---------- */
let syncing = false;
chart.timeScale().subscribeVisibleLogicalRangeChange(r => {
  if (syncing || !r) return;
  syncing = true; oscChart.timeScale().setVisibleLogicalRange(r); syncing = false;
});
oscChart.timeScale().subscribeVisibleLogicalRangeChange(r => {
  if (syncing || !r) return;
  syncing = true; chart.timeScale().setVisibleLogicalRange(r); syncing = false;
});

/* ---------- data ---------- */
async function load() {
  const symbol   = document.getElementById('symbol').value;
  const interval = document.getElementById('interval').value;
  statusEl.textContent = 'loading…';
  try {
    const res = await fetch(`https://api.binance.com/api/v3/klines?symbol=${symbol}&interval=${interval}&limit=1000`);
    if (!res.ok) throw new Error('HTTP ' + res.status);
    const raw = await res.json();
    DATA = raw.map(k => ({
      time: Math.floor(k[0] / 1000),
      open: +k[1], high: +k[2], low: +k[3], close: +k[4], volume: +k[5]
    }));
    candles.setData(DATA);
    volume.setData(DATA.map(d => ({
      time: d.time, value: d.volume,
      color: d.close >= d.open ? 'rgba(38,166,154,.45)' : 'rgba(239,83,80,.45)'
    })));
    renderOverlays();
    renderOsc();
    chart.timeScale().fitContent();
    statusEl.textContent = `${symbol} · ${interval} · ${DATA.length} bars`;
  } catch (e) {
    statusEl.textContent = 'error: ' + e.message;
  }
}

['symbol', 'interval'].forEach(id => document.getElementById(id).addEventListener('change', load));
['sma', 'ema', 'bb'].forEach(id => document.getElementById(id).addEventListener('change', renderOverlays));
document.getElementById('oscSel').addEventListener('change', renderOsc);
document.getElementById('reload').addEventListener('click', load);

load();

/* ---------- service worker ---------- */
if ('serviceWorker' in navigator && location.protocol === 'https:') {
  window.addEventListener('load', () => navigator.serviceWorker.register('sw.js').catch(() => {}));
}
</script>
</body>
</html>