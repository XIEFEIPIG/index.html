# index.html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>圣诞树 · 北欧夜晚背景远景</title>
  <style>
    html,body{margin:0;height:100%;background:#070a12;color:#e8edf7;font-family:system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial,"Noto Sans","PingFang SC","Microsoft YaHei",sans-serif}
    #wrap{height:100%;display:grid;place-items:center}
    canvas{width:min(96vw,1100px);height:auto;aspect-ratio:16/9;display:block;border-radius:14px;box-shadow:0 12px 40px rgba(0,0,0,.55);background:#070a12}
    #hud{position:fixed;left:16px;top:14px;z-index:10;background:rgba(10,14,24,.55);border:1px solid rgba(255,255,255,.08);border-radius:12px;padding:10px 12px;backdrop-filter:blur(8px)}
    #title{font-weight:650;font-size:14px;letter-spacing:.2px}
    #hint{margin-top:6px;font-size:12px;opacity:.85}
  </style>
</head>
<body>
  <div id="hud">
    <div id="title">圣诞树 · 北欧夜景远眺</div>
    <div id="hint">点击圣诞树 → 圣诞树旋转（背景为北欧夜晚远景）</div>
  </div>

  <div id="wrap">
    <canvas id="c"></canvas>
  </div>

<script>
(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');

  // --- DPI-aware resize ---
  function resize(){
    const dpr = Math.max(1, window.devicePixelRatio || 1);
    const cssW = Math.min(window.innerWidth*0.96, 1100);
    const cssH = cssW * 9/16; // 16:9
    canvas.style.width = cssW + 'px';
    canvas.style.height = cssH + 'px';
    canvas.width = Math.floor(cssW * dpr);
    canvas.height = Math.floor(cssH * dpr);
    ctx.setTransform(dpr,0,0,dpr,0,0);
  }
  window.addEventListener('resize', resize);
  resize();

  // --- Scene params ---
  let t = 0;
  let rotating = false;
  let angle = 0;

  // Snow particles
  const N = 900;
  const snow = Array.from({length:N}, () => ({
    x: rand(-0.1, 1.1),
    y: rand(-0.2, 1.2),
    z: rand(0, 1),         // depth-ish
    v: rand(0.12, 0.40),   // fall speed
    drift: rand(-0.12, 0.12),
    s: rand(0.6, 1.6)
  }));

  function rand(a,b){ return a + Math.random()*(b-a); }
  function W(){ return parseFloat(canvas.style.width); }
  function H(){ return parseFloat(canvas.style.height); }
  function lerp(a,b,u){ return a + (b-a)*u; }

  function clearBackground(){
    const w=W(), h=H();

    // Nordic night gradient
    const g = ctx.createLinearGradient(0,0,0,h);
    g.addColorStop(0, '#061024');
    g.addColorStop(0.55, '#070a12');
    g.addColorStop(1, '#0b0f18');
    ctx.fillStyle = g;
    ctx.fillRect(0,0,w,h);

    // subtle stars
    ctx.globalAlpha = 0.35;
    ctx.fillStyle = '#dbe6ff';
    for(let i=0;i<70;i++){
      const x = (i*97 % 997)/997 * w;
      const y = (i*191 % 733)/733 * h*0.55;
      ctx.fillRect(x,y,1,1);
    }
    ctx.globalAlpha = 1;
  }

  function poly(points){
    ctx.beginPath();
    ctx.moveTo(points[0][0], points[0][1]);
    for(let i=1;i<points.length;i++) ctx.lineTo(points[i][0], points[i][1]);
    ctx.closePath();
    ctx.fill();
  }

  function drawMountains(){
    const w=W(), h=H();
    const baseY = h*0.62;

    // far ridge (valley silhouette)
    ctx.fillStyle = '#0b1426';
    poly([
      [0, baseY+20],
      [w*0.12, baseY-60],
      [w*0.28, baseY-10],
      [w*0.45, baseY-90],
      [w*0.62, baseY-25],
      [w*0.78, baseY-75],
      [w*0.92, baseY-15],
      [w, baseY+20],
      [w, h],
      [0, h]
    ]);

    // nearer ridge
    ctx.fillStyle = '#0a1020';
    poly([
      [0, baseY+45],
      [w*0.18, baseY-20],
      [w*0.33, baseY+20],
      [w*0.52, baseY-35],
      [w*0.70, baseY+15],
      [w*0.88, baseY-10],
      [w, baseY+50],
      [w, h],
      [0, h]
    ]);
  }

  function drawSnowfield(){
    const w=W(), h=H();
    const y0 = h*0.70;
    const g = ctx.createLinearGradient(0,y0,0,h);
    g.addColorStop(0, 'rgba(40,45,60,0.9)');
    g.addColorStop(1, 'rgba(16,18,26,1)');
    ctx.fillStyle = g;
    ctx.beginPath();
    ctx.moveTo(0,y0);
    ctx.quadraticCurveTo(w*0.35, y0-18, w*0.55, y0+10);
    ctx.quadraticCurveTo(w*0.78, y0+35, w, y0+18);
    ctx.lineTo(w,h);
    ctx.lineTo(0,h);
    ctx.closePath();
    ctx.fill();
  }

  function drawHouse(x,y,s){
    // body
    ctx.fillStyle = '#1b2230';
    ctx.fillRect(x - 26*s, y - 18*s, 52*s, 36*s);

    // roof
    ctx.fillStyle = '#121827';
    ctx.beginPath();
    ctx.moveTo(x - 30*s, y - 18*s);
    ctx.lineTo(x, y - 40*s);
    ctx.lineTo(x + 30*s, y - 18*s);
    ctx.closePath();
    ctx.fill();

    // warm windows
    ctx.fillStyle = 'rgba(255,220,140,0.95)';
    ctx.fillRect(x - 18*s, y - 6*s, 12*s, 10*s);
    ctx.fillRect(x + 6*s,  y - 6*s, 12*s, 10*s);

    // soft glow
    ctx.globalAlpha = 0.35;
    ctx.fillStyle = 'rgba(255,210,120,0.35)';
    ctx.beginPath();
    ctx.ellipse(x, y-2*s, 44*s, 26*s, 0, 0, Math.PI*2);
    ctx.fill();
    ctx.globalAlpha = 1;
  }

  function drawVillage(){
    const w=W(), h=H();
    const y = h*0.70;
    drawHouse(w*0.22, y-20, 1.05);
    drawHouse(w*0.32, y-12, 0.85);
    drawHouse(w*0.50, y-18, 1.00);
    drawHouse(w*0.64, y-10, 0.80);
    drawHouse(w*0.78, y-22, 1.10);
  }

  // Tree rotation: faux y-axis rotation via X scale (cos(angle))
  function drawTree(){
    const w=W(), h=H();
    const cx = w*0.50;
    const groundY = h*0.78;

    const rot = Math.cos(angle);
    const squash = lerp(0.55, 1.0, Math.abs(rot));
    const shade = rot > 0 ? 1.0 : 0.92;

    ctx.save();
    ctx.translate(cx, groundY);
    ctx.scale(squash, 1);

    // trunk
    ctx.fillStyle = `rgba(${Math.floor(90*shade)},${Math.floor(60*shade)},${Math.floor(35*shade)},1)`;
    roundRect(ctx, -14, -80, 28, 80, 10);
    ctx.fill();

    function layer(y, w0, h0, col){
      ctx.fillStyle = col;
      ctx.beginPath();
      ctx.moveTo(0, y - h0);
      ctx.lineTo(-w0, y);
      ctx.lineTo(w0, y);
      ctx.closePath();
      ctx.fill();
    }

    const c1 = colorMul('#1e7a4c', shade);
    const c2 = colorMul('#186a40', shade);
    const c3 = colorMul('#145c38', shade);

    layer(-70, 110, 110, c1);
    layer(-140, 90, 95, c2);
    layer(-205, 70, 80, c3);

    // snow caps
    function snowCap(y, w0, h0){
      ctx.fillStyle = 'rgba(238,242,250,0.97)';
      ctx.beginPath();
      ctx.moveTo(0, y - h0);
      ctx.lineTo(-w0, y);
      ctx.lineTo(w0, y);
      ctx.closePath();
      ctx.fill();
    }
    snowCap(-150, 26, 26);
    snowCap(-215, 22, 22);
    snowCap(-260, 18, 18);

    // subtle ground snow mound near trunk
    ctx.globalAlpha = 0.9;
    ctx.fillStyle = 'rgba(230,235,245,0.35)';
    ctx.beginPath();
    ctx.ellipse(0, 2, 70, 18, 0, 0, Math.PI*2);
    ctx.fill();
    ctx.globalAlpha = 1;

    ctx.restore();
  }

  function colorMul(hex, k){
    const r = parseInt(hex.slice(1,3),16);
    const g = parseInt(hex.slice(3,5),16);
    const b = parseInt(hex.slice(5,7),16);
    const rr = Math.max(0, Math.min(255, Math.round(r*k)));
    const gg = Math.max(0, Math.min(255, Math.round(g*k)));
    const bb = Math.max(0, Math.min(255, Math.round(b*k)));
    return `rgb(${rr},${gg},${bb})`;
  }

  function roundRect(ctx, x, y, w, h, r){
    ctx.beginPath();
    ctx.moveTo(x+r, y);
    ctx.arcTo(x+w, y, x+w, y+h, r);
    ctx.arcTo(x+w, y+h, x, y+h, r);
    ctx.arcTo(x, y+h, x, y, r);
    ctx.arcTo(x, y, x+w, y, r);
    ctx.closePath();
  }

  function drawSnow(){
    const w=W(), h=H();
    const wind = 0.06*Math.sin(t*0.6);

    ctx.save();
    ctx.fillStyle = 'rgba(245,248,255,0.95)';

    for(const p of snow){
      // update
      p.y += p.v/60;
      p.x += (p.drift + wind)*0.006;

      if(p.y > 1.15){
        p.y = rand(-0.15, 0.0);
        p.x = rand(-0.05, 1.05);
        p.z = rand(0,1);
        p.v = rand(0.12, 0.42);
        p.drift = rand(-0.12, 0.12);
        p.s = rand(0.6, 1.6);
      }
      if(p.x < -0.1) p.x = 1.1;
      if(p.x > 1.1)  p.x = -0.1;

      // depth -> size/alpha
      const size = lerp(0.8, 2.4, 1-p.z) * p.s;
      const alpha = lerp(0.35, 0.95, 1-p.z);

      ctx.globalAlpha = alpha;
      const x = p.x*w;
      const y = p.y*h;
      ctx.beginPath();
      ctx.arc(x, y, size, 0, Math.PI*2);
      ctx.fill();
    }
    ctx.restore();
  }

  function inTreeHit(mx,my){
    const w=W(), h=H();
    const cx = w*0.50;
    const gy = h*0.78;
    return (mx >= cx-140 && mx <= cx+140 && my >= gy-300 && my <= gy+310);
  }

  canvas.addEventListener('click', (e) => {
    const rect = canvas.getBoundingClientRect();
    const mx = e.clientX - rect.left;
    const my = e.clientY - rect.top;
    if(inTreeHit(mx,my)){
      rotating = !rotating;
    }
  });

  function tick(){
    t += 1/60;
    if(rotating) angle += 0.04;

    clearBackground();
    drawMountains();
    drawVillage();
    drawSnowfield();
    drawTree();
    drawSnow();

    requestAnimationFrame(tick);
  }
  tick();
})();
</script>
</body>
</html>
