# For-sumii
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>For Susmitha</title>
<style>
  :root{
    --ink: #100a18;
    --plum: #241432;
  }
  *{ box-sizing:border-box; -webkit-tap-highlight-color: transparent; }
  html,body{
    margin:0; padding:0; width:100%; height:100%;
    background: radial-gradient(120% 100% at 50% 12%, var(--plum) 0%, var(--ink) 65%, #070410 100%);
    overflow:hidden;
    font-family: Georgia, 'Times New Roman', serif;
  }
  #stage{ position:fixed; inset:0; width:100%; height:100%; display:block; }

  #caption{
    position:fixed;
    left:0; right:0;
    bottom:5%;
    text-align:center;
    color:#f4e3d3;
    opacity:0;
    transform: translateY(10px);
    transition: opacity 1.6s ease, transform 1.6s ease;
    pointer-events:none;
    padding:0 6%;
  }
  #caption .name{
    font-size: clamp(22px, 6.5vw, 34px);
    letter-spacing: 0.06em;
    color: #fce3cf;
    text-shadow: 0 0 18px rgba(232,178,61,0.35);
  }
  #caption .sub{
    display:block;
    margin-top:6px;
    font-size: clamp(12px, 3.4vw, 15px);
    letter-spacing: 0.18em;
    text-transform: uppercase;
    color: #cfa8b6;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  }
  #caption.show{ opacity:1; transform:translateY(0); }

  #hint{
    position:fixed;
    top:16px; left:0; right:0;
    text-align:center;
    color:#c9b8c8;
    font-size:11px;
    letter-spacing:0.14em;
    text-transform:uppercase;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    opacity:0;
    transition: opacity 1s ease;
    pointer-events:none;
  }
</style>
</head>
<body>

<canvas id="stage"></canvas>
<div id="hint">tap to bloom again</div>
<div id="caption">
  <span class="name">Susmitha</span>
  <span class="sub">every letter of your name, blooming into one</span>
</div>

<script>
/* ============================================================
   Customize here:
   - NAME: the word whose letters make up the flower
   - The message text is in the #caption div above
   ============================================================ */
const NAME = "Susmitha";

const canvas = document.getElementById('stage');
const ctx = canvas.getContext('2d');
const captionEl = document.getElementById('caption');
const hintEl = document.getElementById('hint');

let W, H, DPR;
let letters = [];
let startTime = 0;
const FORM_DURATION = 3000; // ms for the main formation
let animId = null;
let formed = false;

function resize(){
  DPR = Math.min(window.devicePixelRatio || 1, 2);
  W = window.innerWidth;
  H = window.innerHeight;
  canvas.width = W * DPR;
  canvas.height = H * DPR;
  canvas.style.width = W + 'px';
  canvas.style.height = H + 'px';
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
}

function rand(a, b){ return a + Math.random() * (b - a); }

// rotate a local (lx, ly) offset by `angle`, anchored at (cx, cy)
function rot(cx, cy, lx, ly, angle){
  const x = lx * Math.cos(angle) - ly * Math.sin(angle);
  const y = lx * Math.sin(angle) + ly * Math.cos(angle);
  return { x: cx + x, y: cy + y };
}

/* A single petal, filled with letters laid out on structured rings
   (not random noise) so the outline reads as a clean, recognizable
   petal shape: narrow at the base, wide in the middle, rounded tip. */
function gridPetal(cx, cy, angle, length, widthFactor, spacing, hue, hueSpread, lightBase, lightSpan, sat){
  const pts = [];
  const numRings = Math.max(7, Math.round(length / spacing));
  for(let ring = 1; ring <= numRings; ring++){
    const u = ring / numRings;
    const halfWidth = length * widthFactor * Math.sin(Math.PI * u);
    const count = Math.max(1, Math.round((2 * halfWidth) / spacing));
    for(let i = 0; i < count; i++){
      const v = count === 1 ? 0 : -halfWidth + (i + 0.5) * (2 * halfWidth / count);
      const p = rot(cx, cy, u * length, v, angle);
      pts.push({
        x: p.x, y: p.y,
        hue: hue + rand(-hueSpread, hueSpread),
        sat: sat + rand(-4, 4),
        light: lightBase + (1 - u) * lightSpan + rand(-3, 3),
        fontPx: spacing * rand(0.95, 1.15)
      });
    }
  }
  return pts;
}

/* A round disc (the flower's center) filled with concentric rings of
   letters, evenly spaced by angle on each ring. */
function gridDisc(cx, cy, radius, spacing, hue, hueSpread, lightBase, lightSpan, sat){
  const pts = [];
  const numRings = Math.max(3, Math.round(radius / spacing));
  for(let ring = 0; ring <= numRings; ring++){
    const r = (ring / numRings) * radius;
    if(r < spacing * 0.5){
      pts.push({ x: cx, y: cy, hue: hue+rand(-hueSpread,hueSpread), sat, light: lightBase+lightSpan, fontPx: spacing*1.05 });
      continue;
    }
    const count = Math.max(5, Math.round((2 * Math.PI * r) / spacing));
    const stagger = ring * 0.35;
    for(let i = 0; i < count; i++){
      const a = (i / count) * Math.PI * 2 + stagger;
      pts.push({
        x: cx + Math.cos(a) * r, y: cy + Math.sin(a) * r,
        hue: hue + rand(-hueSpread, hueSpread),
        sat: sat + rand(-5, 5),
        light: lightBase - (r / radius) * lightSpan + rand(-3, 3),
        fontPx: spacing * rand(0.95, 1.15)
      });
    }
  }
  return pts;
}

/* A straight-ish stem: letters evenly spaced along a gentle curve. */
function gridStem(x1, y1, x2, y2, bendX, spacing, hue, sat, light){
  const pts = [];
  const approxLen = Math.hypot(x2 - x1, y2 - y1);
  const count = Math.max(4, Math.round(approxLen / spacing));
  const cx1 = x1 + bendX, cy1 = y1 + (y2 - y1) * 0.33;
  const cx2 = x2 - bendX * 0.4, cy2 = y1 + (y2 - y1) * 0.7;
  for(let i = 0; i <= count; i++){
    const t = i / count;
    const mt = 1 - t;
    const x = mt*mt*mt*x1 + 3*mt*mt*t*cx1 + 3*mt*t*t*cx2 + t*t*t*x2;
    const y = mt*mt*mt*y1 + 3*mt*mt*t*cy1 + 3*mt*t*t*cy2 + t*t*t*y2;
    pts.push({ x, y, hue: hue+rand(-6,6), sat: sat+rand(-5,5), light: light+rand(-4,4), fontPx: spacing*1.05 });
  }
  return pts;
}

function buildTargets(){
  const scale = Math.min(W, H);
  const cx = W / 2;
  const cy = H * 0.36;

  const petalLength = scale * 0.33;
  const petalWidthFactor = 0.40;
  const petalSpacing = petalLength * 0.10;
  const numPetals = 6;

  const petalHue = 340;      // rose pink
  const centerHue = 44;      // golden center
  const stemHue = 138;       // green

  let pts = [];

  for(let p = 0; p < numPetals; p++){
    const angle = -Math.PI/2 + (p / numPetals) * Math.PI * 2; // start pointing up
    pts.push(...gridPetal(
      cx, cy, angle, petalLength, petalWidthFactor, petalSpacing,
      petalHue, 6, 42, 26, 68
    ));
  }

  pts.push(...gridDisc(cx, cy, petalLength * 0.24, petalSpacing * 0.85, centerHue, 8, 46, 16, 78));

  const stemTopX = cx, stemTopY = cy + petalLength * 0.82;
  const stemBotX = cx, stemBotY = H * 0.88;
  pts.push(...gridStem(stemTopX, stemTopY, stemBotX, stemBotY, scale*0.02, scale*0.035, stemHue, 45, 28));

  const leafSpecs = [
    { t: 0.32, side: -1 }, { t: 0.58, side: 1 }
  ];
  leafSpecs.forEach(spec=>{
    const lx = stemTopX;
    const ly = stemTopY + (stemBotY - stemTopY) * spec.t;
    // left leaves point left-and-down, right leaves point right-and-down
    const angle = spec.side < 0 ? (Math.PI - 0.45) : 0.45;
    pts.push(...gridPetal(
      lx, ly, angle, scale*0.17, 0.40, scale*0.028,
      stemHue, 8, 24, 16, 42
    ));
  });

  return pts;
}

function easeOutBack(t){
  const c1 = 1.4, c3 = c1 + 1;
  return 1 + c3 * Math.pow(t-1,3) + c1*Math.pow(t-1,2);
}

function init(scatter){
  const targets = buildTargets();
  letters = targets.map((pt, i)=>{
    const angle = rand(0, Math.PI*2);
    const dist = Math.max(W,H) * rand(0.65, 1.2);
    const sx = scatter ? W/2 + Math.cos(angle)*dist : W/2 + rand(-40,40);
    const sy = scatter ? H/2 + Math.sin(angle)*dist : H/2 + rand(-40,40);
    return {
      ch: NAME[i % NAME.length],
      sx, sy,
      tx: pt.x, ty: pt.y,
      hue: pt.hue, sat: pt.sat, light: pt.light, fontPx: pt.fontPx,
      delay: rand(0, 800),
      phase: rand(0, Math.PI*2),
      freq: rand(0.6, 1.3)
    };
  });
  formed = false;
  captionEl.classList.remove('show');
  startTime = performance.now();
  if(animId) cancelAnimationFrame(animId);
  animId = requestAnimationFrame(tick);
}

function tick(now){
  const elapsed = now - startTime;
  ctx.clearRect(0,0,W,H);
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';

  let allDone = true;
  for(const L of letters){
    let t = (elapsed - L.delay) / FORM_DURATION;
    if(t < 0){ t = 0; allDone = false; }
    if(t < 1) allDone = false;
    t = Math.min(1, Math.max(0, t));
    const e = easeOutBack(t);

    let x = L.sx + (L.tx - L.sx) * e;
    let y = L.sy + (L.ty - L.sy) * e;

    if(t >= 1){
      const floatT = (now/1000) * L.freq + L.phase;
      x += Math.sin(floatT) * 1.6;
      y += Math.cos(floatT*0.8) * 1.6;
    }

    const alpha = 0.55 + 0.45 * t;
    ctx.font = `${L.fontPx}px Georgia, serif`;
    ctx.fillStyle = `hsla(${L.hue}, ${L.sat}%, ${L.light}%, ${alpha})`;
    ctx.fillText(L.ch, x, y);
  }

  if(allDone && !formed){
    formed = true;
    captionEl.classList.add('show');
    hintEl.style.opacity = 0.55;
  }

  animId = requestAnimationFrame(tick);
}

function start(){
  resize();
  hintEl.style.opacity = 0;
  init(true);
}

window.addEventListener('resize', ()=>{ resize(); init(false); });
window.addEventListener('orientationchange', ()=>{ setTimeout(()=>{ resize(); init(false); }, 200); });

canvas.addEventListener('click', ()=>{ init(true); });
canvas.addEventListener('touchstart', (e)=>{ e.preventDefault(); init(true); }, { passive:false });

start();
</script>
</body>
</html>
