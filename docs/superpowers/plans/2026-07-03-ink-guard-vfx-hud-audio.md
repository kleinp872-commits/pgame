# 墨守 Ink Guard 视觉/HUD/音效升级 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为单文件水墨游戏 `mo-shou_3.html` 加入打击感、合成音效、水墨渲染升级、全屏氛围特效、HUD 重做与最高分持久化。

**Architecture:** 全部改动都在 `mo-shou_3.html` 的一个 IIFE 内，按注释分区新增模块（audio / juice / fx / hud / persist），通过在现有函数（`hitEnemy`、`killEnemy`、`resolveCast`、`update`、`render` 等）内插入挂接调用接入。新增第三层 canvas `#stain` 做残墨。

**Tech Stack:** 原生 Canvas 2D、Web Audio API（程序合成）、Vibration API、localStorage。零依赖、零构建。

**Spec:** `docs/superpowers/specs/2026-07-03-ink-guard-vfx-hud-audio-design.md`

## Global Constraints

- 保持单 HTML 文件，双击 `file://` 即玩；不引入任何外部资源/构建。
- 不改玩法数值与判定：`LEVELS`、`ANIMAL_DEFS`、`spawnEnemy` 速度公式、`distToSeg` 碰撞、越线扣 1 命 + 14 HP 均不动（combo 加分是**新增**分，不改 `ptsPerHit`）。
- Web Audio / Vibration / localStorage 一律特性检测 + try/catch 静默降级。
- 所有运动量以 px/ms 计并乘 `dt`；禁止帧率相关运动。
- 热路径（每帧）避免新建对象/渐变/闭包；粒子、飘字、碎片走环形池。
- 验证方式：浏览器手动冒烟（无测试框架）。每个任务的验证步骤都必须"打开 DevTools Console 确认无报错"。
- 每个任务完成后 git commit 一次。

## 文件结构

只有一个源文件。各任务修改区域（按文件内顺序）：

| 区域 | 位置锚点 | 任务 |
|---|---|---|
| CSS 新增/修改 | `<style>` 内 | 2, 5, 6 |
| HTML 新增元素 | `#stage` 内 | 2, 3, 5 |
| audio 模块 | IIFE 开头、`resize()` 之前 | 2 |
| juice 模块 | audio 之后 | 1 |
| fx 模块（splatter/shards/stain/lightning/wave/fog/floaters） | game state 区之后 | 1, 3, 4, 5 |
| 挂接点 | `hitEnemy`/`killEnemy`/`resolveCast`/`beginLevelClear`/`startLevel`/`update`/`render`/`gameOver`/`resetGame`/按钮 handlers | 全部 |

---

### Task 1: 打击感（震屏 / 顿帧 / 方向性飞溅 / 震动）

**Files:**
- Modify: `mo-shou_3.html`

**Interfaces:**
- Produces: `shake(power)`, `buzz(ms)`, `freezeT`（模块级变量）, `lastStrokeAngle`, `spawnParticle(x,y,vx,vy,life,r)`, `splatter(x,y,ang)`, `updateFx(dt)`。后续任务在挂接点直接调用 `shake`/`buzz`，并把自己的每帧特效更新加进 `updateFx`。
- Consumes: 现有 `particles` 数组、`update`、`render`、`loop`、`hitEnemy`、`killEnemy`、`pointerMove`。

- [ ] **Step 1: 新增 juice 模块**

在 `// ---------- game state ----------` 注释块**之前**插入：

```js
  // ---------- juice: shake / hit-stop / vibrate ----------
  let shakeT=0, shakePow=0, freezeT=0;
  const SHAKE_DUR=220;
  function shake(p){
    shakePow = Math.min(12, Math.max(shakePow, p));
    shakeT = SHAKE_DUR;
  }
  function buzz(ms){
    if(navigator.vibrate){ try{ navigator.vibrate(ms); }catch(e){} }
  }
```

- [ ] **Step 2: 粒子系统改造为带重力/半径 + 方向性飞溅**

替换现有 `burst` 函数（保留函数名，签名兼容旧调用点）并新增 `spawnParticle`/`splatter`：

```js
  // ---------- particles ----------
  function spawnParticle(x,y,vx,vy,life,r){
    particles.push({x,y,vx,vy,life,maxLife:life,r:r||2,color:'rgba(44,38,32,0.7)'});
  }
  function burst(x,y,color){
    for(let i=0;i<8;i++){
      const a = Math.random()*Math.PI*2;
      const sp = 0.05+Math.random()*0.12;
      particles.push({x,y,vx:Math.cos(a)*sp,vy:Math.sin(a)*sp,life:500,maxLife:500,r:2,color:color||'rgba(44,38,32,0.7)'});
    }
  }
  function splatter(x,y,ang){
    for(let i=0;i<10;i++){
      const a = ang + (Math.random()-0.5)*1.4;           // ±40°
      const sp = 0.08+Math.random()*0.22;
      spawnParticle(x,y, Math.cos(a)*sp, Math.sin(a)*sp, 320+Math.random()*280, 1+Math.random()*2.2);
    }
  }
```

粒子 update（现有 `particles.forEach(...)` 行）替换为带重力版本，render 中粒子绘制半径 `2` 改为 `p.r`：

```js
    particles.forEach(p=>{ p.vy += 0.0004*dt; p.x+=p.vx*dt; p.y+=p.vy*dt; p.life-=dt; });
```

```js
      gctx.beginPath(); gctx.arc(p.x,p.y,p.r,0,Math.PI*2); gctx.fill();
```

- [ ] **Step 3: 顿帧 — update 拆分出 updateFx**

把 `update(dt)` 里的粒子/拖尾/墨晕更新三段移入新函数 `updateFx(dt)`，`update` 开头改为：

```js
  function updateFx(dt){
    particles.forEach(p=>{ p.vy += 0.0004*dt; p.x+=p.vx*dt; p.y+=p.vy*dt; p.life-=dt; });
    particles = particles.filter(p=>p.life>0);
    strokeTrail.forEach(p=>p.life-=dt);
    strokeTrail = strokeTrail.filter(p=>p.life>0);
    updateInkBlooms(dt);
  }

  function update(dt){
    updateFx(dt);
    if(freezeT>0){ freezeT-=dt; return; }
    castCooldown -= dt;
    ...（其余原逻辑不变，删除已移走的三段）...
  }
```

- [ ] **Step 4: 震屏应用到 render + loop 中衰减**

`loop` 里 `lastT = t;` 之后加 `shakeT = Math.max(0, shakeT-dt);`。
`render()` 开头 `clearRect` 之后加，函数**最后**加对应 `restore`：

```js
    gctx.save();
    if(shakeT>0){
      const k = (shakeT/SHAKE_DUR)*shakePow;
      gctx.translate((Math.random()-0.5)*k, (Math.random()-0.5)*k);
    } else { shakePow=0; }
    ...（全部现有绘制）...
    gctx.restore();
```

- [ ] **Step 5: 挂接命中/斩杀/越线**

`pointerMove` 中 `strokePts.push(p);` 之后加：

```js
    if(prev) lastStrokeAngle = Math.atan2(p.y-prev.y, p.x-prev.x);
```

（`lastStrokeAngle` 在 game state 区声明：`let lastStrokeAngle=0;`）

`hitEnemy` 中 `en.hitCooldown = 220;` 之后加：

```js
    shake(3); buzz(10);
    splatter(en.x, en.y, lastStrokeAngle);
```

`killEnemy` 中 `en.alive = false;` 之后加：

```js
    shake(6); buzz(25); freezeT = 50;
```

`update` 内敌人越线分支（`if(en.y > baseY()){`）里 `updateHUD();` 之前加：

```js
        shake(10); buzz(80);
```

- [ ] **Step 6: 手动验证**

浏览器打开 `mo-shou_3.html`（Chrome，另开 DevTools Console）：
- 划中敌人：屏幕轻微抖动、墨滴沿划动方向锥形喷出并下坠。
- 斩杀敌人：更强抖动 + 游戏瞬间凝滞约 0.05s（敌人停住，墨滴仍在动）。
- 故意放敌人越线：明显大幅震屏。
- Console 无任何报错；帧率无可感知下降。

- [ ] **Step 7: Commit**

```bash
git add mo-shou_3.html && git commit -m "feat: hit juice - screen shake, hit-stop, directional ink splatter, vibration"
```

---

### Task 2: 音效（Web Audio 合成 + 静音按钮）

**Files:**
- Modify: `mo-shou_3.html`

**Interfaces:**
- Produces: `ensureAudio()`, `sfx.slash() / kill() / thunder() / hurt() / clear() / chime() / combo(n)`（全部无返回值、静音/无 AudioContext 时静默）。
- Consumes: Task 1 无依赖（可独立验证）。

- [ ] **Step 1: HTML 加静音按钮**

`.btns` 内 `restartBtn` 之后加：

```html
    <button class="rbtn" id="muteBtn">音</button>
```

CSS `<style>` 内加：

```css
  .rbtn.muted{opacity:.4; text-decoration:line-through;}
```

- [ ] **Step 2: audio 模块**

IIFE 开头（`let W=0` 之前）插入：

```js
  // ---------- audio: synthesized sfx ----------
  let audioCtx=null, masterGain=null, muted=false;
  try{ muted = localStorage.getItem('inkguard_muted')==='1'; }catch(e){}
  function ensureAudio(){
    if(audioCtx){ if(audioCtx.state==='suspended') audioCtx.resume(); return; }
    try{
      audioCtx = new (window.AudioContext||window.webkitAudioContext)();
      masterGain = audioCtx.createGain();
      masterGain.gain.value = muted?0:0.9;
      masterGain.connect(audioCtx.destination);
    }catch(e){ audioCtx=null; }
  }
  function noiseBuf(dur){
    const n = Math.floor(audioCtx.sampleRate*dur);
    const buf = audioCtx.createBuffer(1,n,audioCtx.sampleRate);
    const d = buf.getChannelData(0);
    for(let i=0;i<n;i++) d[i]=Math.random()*2-1;
    return buf;
  }
  function env(gainNode,t0,peak,dur){
    gainNode.gain.setValueAtTime(0.0001,t0);
    gainNode.gain.linearRampToValueAtTime(peak,t0+0.008);
    gainNode.gain.exponentialRampToValueAtTime(0.0001,t0+dur);
  }
  function playNoise(dur,filterType,f0,f1,peak){
    const t0=audioCtx.currentTime;
    const src=audioCtx.createBufferSource(); src.buffer=noiseBuf(dur);
    const flt=audioCtx.createBiquadFilter(); flt.type=filterType;
    flt.frequency.setValueAtTime(f0,t0);
    flt.frequency.exponentialRampToValueAtTime(Math.max(40,f1),t0+dur);
    const g=audioCtx.createGain(); env(g,t0,peak,dur);
    src.connect(flt).connect(g).connect(masterGain);
    src.start(t0); src.stop(t0+dur+0.05);
  }
  function playTone(type,f0,f1,dur,peak,delay){
    const t0=audioCtx.currentTime+(delay||0);
    const o=audioCtx.createOscillator(); o.type=type;
    o.frequency.setValueAtTime(f0,t0);
    if(f1) o.frequency.exponentialRampToValueAtTime(f1,t0+dur);
    const g=audioCtx.createGain(); env(g,t0,peak,dur);
    o.connect(g).connect(masterGain);
    o.start(t0); o.stop(t0+dur+0.05);
  }
  const sfx = {
    _ok(){ return audioCtx && !muted; },
    slash(){ if(!this._ok())return; playNoise(0.08,'bandpass',3400,1500,0.25); },
    kill(){ if(!this._ok())return; playNoise(0.09,'bandpass',3000,1200,0.3); playTone('sine',130,70,0.12,0.5); },
    thunder(){ if(!this._ok())return;
      playNoise(0.03,'highpass',3000,3000,0.6);
      playNoise(1.2,'lowpass',800,90,0.55);
      playTone('sine',52,40,1.1,0.5);
    },
    hurt(){ if(!this._ok())return; playTone('triangle',220,110,0.3,0.4); },
    clear(){ if(!this._ok())return;
      [523,587,659,784,880].forEach((f,i)=>playTone('sine',f,0,0.5,0.22,i*0.09));
    },
    chime(){ if(!this._ok())return; playTone('sine',880,0,1.4,0.3); playTone('sine',1760,0,1.0,0.08); },
    combo(n){ if(!this._ok())return; playTone('sine',600*Math.pow(1.059,Math.min(n,12)),0,0.09,0.18); }
  };
```

- [ ] **Step 3: 静音按钮 + 音频解锁挂接**

按钮区（现有 `pauseBtn` handler 之后）加：

```js
  const muteBtn=document.getElementById('muteBtn');
  if(muted) muteBtn.classList.add('muted');
  muteBtn.addEventListener('click', ()=>{
    muted=!muted;
    muteBtn.classList.toggle('muted',muted);
    if(masterGain) masterGain.gain.value = muted?0:0.9;
    try{ localStorage.setItem('inkguard_muted', muted?'1':'0'); }catch(e){}
  });
```

`startBtn`、`retryBtn`、`restartBtn` 三个 click handler 的第一行都加 `ensureAudio();`；`pointerDown` 开头（`if(!running...` 之前）加 `ensureAudio();`。

- [ ] **Step 4: 音效挂接**

- `hitEnemy`：`splatter(...)` 之后加 `sfx.slash();`
- `killEnemy`：`freezeT = 50;` 之后加 `sfx.kill();`
- `resolveCast` 成功分支：`flash(...)` 之后加 `sfx.thunder();`
- 敌人越线分支：`shake(10); buzz(80);` 之后加 `sfx.hurt();`
- `beginLevelClear`：`flash(...)` 之后加 `sfx.clear();`
- `gameOver`：开头加 `sfx.chime();`
- `startBtn` handler：`ensureAudio();` 之后加 `sfx.chime();`

（`sfx.combo` 的挂接在 Task 5 combo 实现时进行。）

- [ ] **Step 5: 手动验证**

- 点开始：一声磬。划中/斩杀/越线/雷击/过关各有对应音效（配方见 spec §5 表）。
- 点「音」按钮：立即静音、按钮变灰划线；刷新页面后静音状态保持。
- Console 无报错；`file://` 下正常（无跨域问题——纯合成无外部资源）。

- [ ] **Step 6: Commit**

```bash
git add mo-shou_3.html && git commit -m "feat: synthesized Web Audio sfx with mute toggle"
```

---

### Task 3: 水墨渲染（宣纸纹理 / 飞白拖尾 / 残墨层 / 化墨死亡）

**Files:**
- Modify: `mo-shou_3.html`

**Interfaces:**
- Produces: `#stain` canvas 与 `stainCtx`、`addStain(x,y)`、`spawnShards(en,ang)`、`shards` 数组（updateFx/render 内已接管）。
- Consumes: Task 1 的 `updateFx`、`lastStrokeAngle`。

- [ ] **Step 1: 残墨 canvas**

HTML：`<canvas id="bg"></canvas>` 与 `<canvas id="game"></canvas>` 之间插入 `<canvas id="stain"></canvas>`。
JS 引用 + 纳入 `resize()`：

```js
  const stainCanvas = document.getElementById('stain');
  const stainCtx = stainCanvas.getContext('2d');
```

`resize()` 中 `[bgCanvas,gCanvas]` 改为 `[bgCanvas,stainCanvas,gCanvas]`，并加 `stainCtx.setTransform(DPR,0,0,DPR,0,0);`。

- [ ] **Step 2: addStain + 节流淡出**

fx 区加：

```js
  // ---------- stain layer ----------
  let stainFadeT=0;
  function addStain(x,y){
    const r = 14+Math.random()*10;
    const g = stainCtx.createRadialGradient(x,y,0,x,y,r);
    g.addColorStop(0,'rgba(60,52,40,0.10)');
    g.addColorStop(0.7,'rgba(60,52,40,0.05)');
    g.addColorStop(1,'rgba(60,52,40,0)');
    stainCtx.fillStyle=g;
    stainCtx.beginPath(); stainCtx.arc(x,y,r,0,Math.PI*2); stainCtx.fill();
  }
  function clearStains(){
    stainCtx.save(); stainCtx.setTransform(1,0,0,1,0,0);
    stainCtx.clearRect(0,0,stainCanvas.width,stainCanvas.height);
    stainCtx.restore();
  }
  function fadeStains(){
    stainCtx.save(); stainCtx.setTransform(1,0,0,1,0,0);
    stainCtx.globalCompositeOperation='destination-out';
    stainCtx.globalAlpha=0.12;
    stainCtx.fillStyle='#000';
    stainCtx.fillRect(0,0,stainCanvas.width,stainCanvas.height);
    stainCtx.restore();
  }
```

`update`（非 freeze 段）加：

```js
    stainFadeT += dt;
    if(stainFadeT>2000){ stainFadeT=0; fadeStains(); }
```

`killEnemy` 中 `inkBloom(en.x,en.y);` 之后加 `addStain(en.x,en.y);`。
`resetGame` 中加 `clearStains();`。

- [ ] **Step 3: 宣纸纹理（离屏，resize 时重算）**

fx 区加，并在 `drawBackground()` 末尾（setLineDash([]) 之后）加 `if(paperCanvas) bgCtx.drawImage(paperCanvas,0,0,W,H);`，`resize()` 里 `drawBackground()` 之前调 `makePaper();`：

```js
  // ---------- paper texture (offscreen) ----------
  let paperCanvas=null;
  function makePaper(){
    paperCanvas=document.createElement('canvas');
    paperCanvas.width=W; paperCanvas.height=H;
    const p=paperCanvas.getContext('2d');
    const fibers=Math.floor(W*H/900);
    for(let i=0;i<fibers;i++){
      const x=Math.random()*W, y=Math.random()*H, a=Math.random()*Math.PI, l=3+Math.random()*8;
      p.strokeStyle='rgba(120,105,80,'+(0.04+Math.random()*0.05).toFixed(3)+')';
      p.beginPath(); p.moveTo(x,y); p.lineTo(x+Math.cos(a)*l, y+Math.sin(a)*l); p.stroke();
    }
    for(let i=0;i<40;i++){
      const x=Math.random()*W,y=Math.random()*H,r=20+Math.random()*60;
      const g=p.createRadialGradient(x,y,0,x,y,r);
      g.addColorStop(0,'rgba(160,145,115,0.05)'); g.addColorStop(1,'rgba(160,145,115,0)');
      p.fillStyle=g; p.beginPath(); p.arc(x,y,r,0,Math.PI*2); p.fill();
    }
  }
```

- [ ] **Step 4: 飞白拖尾**

`pointerMove` 中 trail push 行改为（宽度随速度：快则细）：

```js
    const d = prev? Math.hypot(p.x-prev.x,p.y-prev.y) : 0;
    strokeTrail.push({x:p.x,y:p.y,life:260,maxLife:260,w:Math.max(1.5, 7-d*0.18)});
```

`render` 中拖尾绘制段整体替换为三层画法（侧淡墨 + 飞白中锋，`(i*7)%10<8` 为确定性跳段防闪烁）：

```js
    if(strokeTrail.length>1){
      gctx.lineJoin='round'; gctx.lineCap='round';
      for(let i=1;i<strokeTrail.length;i++){
        const p0=strokeTrail[i-1], p1=strokeTrail[i];
        const alpha = Math.max(0,p1.life/p1.maxLife);
        const w = ((p0.w||3)+(p1.w||3))/2*alpha + 0.5;
        gctx.strokeStyle = 'rgba(44,38,32,'+(0.16*alpha).toFixed(3)+')';
        gctx.lineWidth = w*2.1;
        gctx.beginPath(); gctx.moveTo(p0.x,p0.y); gctx.lineTo(p1.x,p1.y); gctx.stroke();
        if((i*7)%10 < 8){
          gctx.strokeStyle = 'rgba(30,26,20,'+(0.6*alpha).toFixed(3)+')';
          gctx.lineWidth = w;
          gctx.beginPath(); gctx.moveTo(p0.x,p0.y); gctx.lineTo(p1.x,p1.y); gctx.stroke();
        }
      }
    }
```

- [ ] **Step 5: 化墨死亡碎片**

fx 区加：

```js
  // ---------- ink shards (death dissolve) ----------
  let shards=[];
  function spawnShards(en, ang){
    const n = 4+Math.floor(Math.random()*3);
    for(let i=0;i<n;i++){
      const poly=[]; const pn=5+Math.floor(Math.random()*3);
      const rr = en.r*(0.35+Math.random()*0.4);
      for(let k=0;k<pn;k++){
        const a=(k/pn)*Math.PI*2;
        poly.push({x:Math.cos(a)*rr*(0.6+Math.random()*0.7), y:Math.sin(a)*rr*(0.6+Math.random()*0.7)});
      }
      const a = ang + (Math.random()-0.5)*1.2;
      shards.push({x:en.x,y:en.y,
        vx:Math.cos(a)*(0.06+Math.random()*0.12),
        vy:Math.sin(a)*(0.06+Math.random()*0.10)-0.05,
        rot:Math.random()*Math.PI*2, vr:(Math.random()-0.5)*0.01,
        poly, life:600, maxLife:600, grow:0.4+Math.random()*0.3});
    }
  }
```

`updateFx` 加：

```js
    shards.forEach(s=>{ s.vy+=0.00035*dt; s.x+=s.vx*dt; s.y+=s.vy*dt; s.rot+=s.vr*dt; s.life-=dt; });
    shards = shards.filter(s=>s.life>0);
```

`render` 中 `enemies.forEach(drawEnemy);` 之后加：

```js
    shards.forEach(s=>{
      const a = s.life/s.maxLife;
      const sc = 1 + (1-a)*s.grow;
      gctx.save();
      gctx.translate(s.x,s.y); gctx.rotate(s.rot); gctx.scale(sc,sc);
      gctx.fillStyle = 'rgba(24,21,17,'+(0.8*a).toFixed(3)+')';
      gctx.beginPath();
      s.poly.forEach((p,i)=>{ if(i===0) gctx.moveTo(p.x,p.y); else gctx.lineTo(p.x,p.y); });
      gctx.closePath(); gctx.fill();
      gctx.restore();
    });
```

`killEnemy` 中 `addStain(...)` 之后加 `spawnShards(en, lastStrokeAngle);`。
`resetGame` 中 `shards=[];`。

- [ ] **Step 6: 手动验证**

- 背景可见细微纸纤维/斑块纹理（对比改动前截图）。
- 快划时墨线变细且有断续飞白；慢划粗实；两侧有淡墨晕边。
- 斩杀：敌人碎成数片旋转飘散的墨块，约 0.6s 晕开消失；地上留淡残墨，约 20s 内渐渐消退。
- 重开游戏残墨清空。窗口缩放后纹理重新生成、无错位。Console 无报错。

- [ ] **Step 7: Commit**

```bash
git add mo-shou_3.html && git commit -m "feat: ink rendering - paper texture, feibai brush trail, stain layer, ink-shard death"
```

---

### Task 4: 全屏氛围（闪电 / 危险晕影 / 关卡墨浪转场 / 薄雾）

**Files:**
- Modify: `mo-shou_3.html`

**Interfaces:**
- Produces: `spawnLightning()`, `flashWhite`, `bolts`, `thunderQueue`, `drawInkWave(prog)`, `makeFog()`, `fogs`。
- Consumes: Task 1 `shake/buzz/updateFx`、Task 2 `sfx.thunder`、Task 3 无直接依赖。

- [ ] **Step 1: 闪电 + 白闪**

fx 区加：

```js
  // ---------- lightning ----------
  let flashWhite=0, bolts=[];
  function boltPath(x0,y0,x1,y1,depth){
    let pts=[{x:x0,y:y0},{x:x1,y:y1}];
    for(let d=0; d<depth; d++){
      const next=[pts[0]];
      for(let i=1;i<pts.length;i++){
        const a=pts[i-1], b=pts[i];
        next.push({x:(a.x+b.x)/2 + (Math.random()-0.5)*(b.y-a.y)*0.5,
                   y:(a.y+b.y)/2 + (Math.random()-0.5)*10}, b);
      }
      pts=next;
    }
    return pts;
  }
  function spawnLightning(){
    flashWhite = 120;
    const x = W*0.3+Math.random()*W*0.4;
    const main = boltPath(x,-10, x+(Math.random()-0.5)*80, baseY(), 5);
    bolts.push({pts:main, life:250, maxLife:250, w:3});
    for(let i=0;i<3;i++){
      const p = main[2+Math.floor(Math.random()*(main.length-4))];
      bolts.push({pts:boltPath(p.x,p.y, p.x+(Math.random()-0.5)*160, p.y+80+Math.random()*120, 3),
                  life:200, maxLife:200, w:1.5});
    }
  }
```

`updateFx` 加：

```js
    flashWhite = Math.max(0, flashWhite-dt);
    bolts.forEach(b=>b.life-=dt);
    bolts = bolts.filter(b=>b.life>0);
```

`render`（world 段内、particles 之后）加 bolt 绘制；`gctx.restore()` **之后**加白闪覆盖：

```js
    bolts.forEach(b=>{
      const a = b.life/b.maxLife;
      gctx.lineCap='round'; gctx.lineJoin='round';
      gctx.strokeStyle='rgba(161,58,44,'+(0.35*a).toFixed(3)+')';
      gctx.lineWidth=b.w*3.5;
      gctx.beginPath(); b.pts.forEach((p,i)=>{ i?gctx.lineTo(p.x,p.y):gctx.moveTo(p.x,p.y); }); gctx.stroke();
      gctx.strokeStyle='rgba(255,252,240,'+(0.9*a).toFixed(3)+')';
      gctx.lineWidth=b.w;
      gctx.beginPath(); b.pts.forEach((p,i)=>{ i?gctx.lineTo(p.x,p.y):gctx.moveTo(p.x,p.y); }); gctx.stroke();
    });
```

```js
    if(flashWhite>0){
      gctx.fillStyle='rgba(255,255,250,'+(flashWhite/120*0.75).toFixed(3)+')';
      gctx.fillRect(0,0,W,H);
    }
```

- [ ] **Step 2: 雷击依次化墨**

game state 区加 `let thunderQueue=[], thunderT=0;`。
`resolveCast` 成功分支替换 `enemies.forEach(en=>{ if(en.alive) killEnemy(en, 6); });` 为：

```js
      spawnLightning(); shake(9); buzz(60);
      thunderQueue = enemies.filter(e=>e.alive).sort((a,b)=>b.y-a.y);
      thunderT = 0;
```

`update`（非 freeze 段）加：

```js
    if(thunderQueue.length){
      thunderT += dt;
      while(thunderT>=60 && thunderQueue.length){
        thunderT -= 60;
        const en = thunderQueue.shift();
        if(en.alive) killEnemy(en, 6);
      }
    }
```

`resetGame` 中 `thunderQueue=[]; bolts=[]; flashWhite=0;`。
注意：queue 中敌人若先越线（alive=false）会被跳过，不重复计。

- [ ] **Step 3: 危险晕影**

`render` 的 `gctx.restore()` 之后（白闪之前）加：

```js
    if(running && hp<30){
      const breathe = 0.5+0.5*Math.sin(performance.now()*0.004);
      const va = (0.18 + 0.22*(1-hp/30)) * (0.6+0.4*breathe);
      const vg = gctx.createRadialGradient(W/2,H/2,Math.min(W,H)*0.45, W/2,H/2,Math.max(W,H)*0.72);
      vg.addColorStop(0,'rgba(161,58,44,0)');
      vg.addColorStop(1,'rgba(161,58,44,'+va.toFixed(3)+')');
      gctx.fillStyle=vg; gctx.fillRect(0,0,W,H);
    }
```

（此渐变仅在 HP<30 才创建，非常态热路径，可接受。）

- [ ] **Step 4: 关卡墨浪转场**

`beginLevelClear` 中 `levelTransition = 1400;` 改为 `levelTransition = 2200;`。
fx 区加 `drawInkWave`，`render` world 段最末（bolts 之后）加调用：

```js
  // ---------- level transition ink wave ----------
  function drawInkWave(prog){
    const bandW = W*0.7+220;
    const total = W + bandW + 440;
    const sweep = Math.min(1, prog/0.55);
    const lead = -220 + sweep*total;
    gctx.fillStyle='rgba(28,24,19,0.94)';
    gctx.beginPath();
    gctx.moveTo(lead + Math.sin(prog*14)*24, -20);
    for(let y=0;y<=H;y+=20) gctx.lineTo(lead + Math.sin(y*0.045+prog*14)*24, y);
    gctx.lineTo(lead + Math.sin(H*0.045+prog*14)*24, H+20);
    gctx.lineTo(lead-bandW + Math.sin(H*0.03+prog*10)*30, H+20);
    for(let y=H;y>=0;y-=20) gctx.lineTo(lead-bandW + Math.sin(y*0.03+prog*10)*30, y);
    gctx.closePath(); gctx.fill();
    if(prog>0.25 && prog<0.85){
      const ta = Math.sin((prog-0.25)/0.6*Math.PI);
      gctx.fillStyle='rgba(233,226,207,'+ta.toFixed(3)+')';
      gctx.font='700 44px "Songti SC","STSong","Noto Serif SC",serif';
      gctx.textAlign='center';
      gctx.fillText('第 '+(level+1)+' 關', W/2, H*0.45);
      gctx.textAlign='start';
    }
  }
```

```js
    if(levelTransition>0) drawInkWave(1 - levelTransition/2200);
```

- [ ] **Step 5: 薄雾**

fx 区加；`resize()` 中 `makePaper();` 之后加 `makeFog();`：

```js
  // ---------- fog (offscreen, drifting) ----------
  let fogCanvas=null, fogs=[];
  function makeFog(){
    fogCanvas=document.createElement('canvas');
    fogCanvas.width=420; fogCanvas.height=120;
    const f=fogCanvas.getContext('2d');
    for(let i=0;i<7;i++){
      const x=40+Math.random()*340, y=30+Math.random()*60, r=40+Math.random()*50;
      const g=f.createRadialGradient(x,y,0,x,y,r);
      g.addColorStop(0,'rgba(233,228,214,0.16)'); g.addColorStop(1,'rgba(233,228,214,0)');
      f.fillStyle=g; f.beginPath(); f.arc(x,y,r,0,Math.PI*2); f.fill();
    }
    fogs=[{x:-100,y:H*0.30,v:0.006,s:1.4},{x:W*0.5,y:H*0.42,v:-0.004,s:1.9},{x:W*0.2,y:H*0.55,v:0.003,s:2.4}];
  }
```

`updateFx` 加（回绕）：

```js
    fogs.forEach(f=>{
      f.x += f.v*dt;
      if(f.v>0 && f.x>W+40) f.x = -420*f.s-40;
      if(f.v<0 && f.x<-420*f.s-40) f.x = W+40;
    });
```

`render` world 段**最先**（拖尾之前）加：

```js
    if(fogCanvas) fogs.forEach(f=>gctx.drawImage(fogCanvas, f.x, f.y, 420*f.s, 120*f.s));
```

- [ ] **Step 6: 手动验证**

- 描线成功：全屏白闪一瞬 → 朱砂描边闪电劈至防线 → 场上敌人从下到上每 0.06s 依次化墨，雷声轰鸣。
- 故意掉血到 HP<30%：屏幕边缘朱砂晕影呼吸明灭，HP 越低越浓；回条（重开）后消失。
- 过关：浓墨横浪从左扫到右，浪中露出「第 N 關」毛笔大字，扫过后战场已清空进入下一关。
- 远山间三条薄雾极慢漂移，不喧宾夺主。Console 无报错，帧率稳定。

- [ ] **Step 7: Commit**

```bash
git add mo-shou_3.html && git commit -m "feat: atmosphere - lightning strike, danger vignette, ink-wave level transition, drifting fog"
```

---

### Task 5: HUD 重做（印章生命 / 滚动分 / 飘字 / combo / 名牌 / 条动效 / 布局）

**Files:**
- Modify: `mo-shou_3.html`

**Interfaces:**
- Produces: `combo`/`comboT`（killEnemy 内维护）、`spawnFloater(x,y,txt)`、`announceNewEnemies()`、`displayScore`。
- Consumes: Task 1 `updateFx`、Task 2 `sfx.combo`。

- [ ] **Step 1: CSS**

`<style>` 内：`.lives .dot` / `.lives .dot.lost` 两条规则替换，并新增以下全部：

```css
  .lives .dot{
    width:13px; height:13px; border-radius:2px;
    background:var(--seal); box-shadow:inset 0 0 0 1.5px rgba(233,226,207,.35);
  }
  .lives .dot.lost{
    background:transparent; border:1.5px solid var(--ink-soft); opacity:.4;
    animation:sealBreak .5s ease-out;
  }
  @keyframes sealBreak{
    0%{transform:scale(1) rotate(0); background:var(--seal); opacity:1;}
    60%{transform:scale(1.35) rotate(14deg); opacity:.7;}
    100%{transform:scale(1) rotate(0); opacity:.4;}
  }
  .score .num.bump{animation:bump .25s ease-out;}
  @keyframes bump{0%{transform:scale(1);}40%{transform:scale(1.18);}100%{transform:scale(1);}}
  .healthbar.hit{animation:hpHit .4s ease-out;}
  .healthbar.hit .healthbar-fill{background:var(--seal);}
  @keyframes hpHit{0%,50%{transform:translateX(-2px);}25%,75%{transform:translateX(2px);}100%{transform:none;}}
  .wavebar-fill{
    background:linear-gradient(90deg,var(--seal-dark),var(--seal),var(--seal-dark));
    background-size:200% 100%; animation:inkflow 3s linear infinite;
  }
  @keyframes inkflow{from{background-position:0 0;}to{background-position:200% 0;}}
  .comboTxt{
    top:26%; left:0; right:0; text-align:center; font-size:24px; font-weight:700;
    letter-spacing:4px; color:var(--seal-dark); opacity:0; transition:opacity .3s;
  }
  .comboTxt.pop{animation:comboPop .3s ease-out forwards;}
  @keyframes comboPop{0%{opacity:0;transform:scale(.6);}60%{opacity:1;transform:scale(1.15);}100%{opacity:1;transform:scale(1);}}
  .nameplate{
    top:34%; left:0; right:0; text-align:center; font-size:20px; font-weight:700;
    letter-spacing:6px; color:var(--ink); opacity:0; transition:opacity .4s;
  }
  .nameplate.show{opacity:1;}
```

布局微调：`.title` 的 `top:18px; left:22px; font-size:22px` → `top:16px; left:16px; font-size:18px`；`.lives` 的 `top:20px; right:22px` → `top:16px; right:16px`；`.btns` 的 `top:46px; right:22px` → `top:44px; right:16px`；`.score` 的 `top:14px` → `top:12px`。
（`.wavebar-fill` 原规则中的 `background:var(--seal);` 删除，其余保留。）

- [ ] **Step 2: HTML 新元素**

`#stage` 内（`.prompt` 之前）加：

```html
  <div class="hud comboTxt" id="comboTxt"></div>
  <div class="hud nameplate" id="namePlate"></div>
```

- [ ] **Step 3: 滚动分数 + bump**

game state 区加 `let displayScore=0;` 与 DOM 引用区加：

```js
  const comboTxt=document.getElementById('comboTxt');
  const namePlate=document.getElementById('namePlate');
```

`updateHUD` 中 `scoreNum.textContent = score;` 改为 `scoreNum.textContent = Math.floor(displayScore);`。
`update`（非 freeze 段，updateHUD 之前）加：

```js
    if(displayScore<score){
      displayScore = Math.min(score, displayScore + Math.max(0.05*dt, (score-displayScore)*0.012*dt));
    }
```

`killEnemy` 内（updateHUD 之前）加 bump 重触发：

```js
    scoreNum.classList.remove('bump'); void scoreNum.offsetWidth; scoreNum.classList.add('bump');
```

`resetGame` 加 `displayScore=0;`。

- [ ] **Step 4: 飘字（canvas 绘制）**

fx 区加：

```js
  // ---------- score floaters ----------
  let floaters=[];
  function spawnFloater(x,y,txt){
    floaters.push({x,y,txt,life:700,maxLife:700});
  }
```

`updateFx` 加：

```js
    floaters.forEach(f=>{ f.y -= 0.03*dt; f.life -= dt; });
    floaters = floaters.filter(f=>f.life>0);
```

`render` world 段（shards 之后）加：

```js
    if(floaters.length){
      gctx.font='600 13px "Songti SC","STSong","Noto Serif SC",serif';
      gctx.textAlign='center';
      floaters.forEach(f=>{
        gctx.fillStyle='rgba(44,38,32,'+(0.85*f.life/f.maxLife).toFixed(3)+')';
        gctx.fillText(f.txt, f.x, f.y);
      });
      gctx.textAlign='start';
    }
```

`hitEnemy` 中 `score += en.ptsPerHit;` 之后加：

```js
    spawnFloater(en.x, en.y-en.r-14, '+'+en.ptsPerHit);
```

`resetGame` 加 `floaters=[];`。

- [ ] **Step 5: combo**

game state 区加 `let combo=0, comboT=0;`。
`killEnemy` 中 `score += (bonusPts||0);` 之后加：

```js
    combo++; comboT=2000;
    const comboBonus = Math.min(20,(combo-1)*2);
    if(comboBonus>0){ score += comboBonus; spawnFloater(en.x, en.y-en.r-28, '連 +'+comboBonus); }
    if(combo>=2){
      comboTxt.textContent='連斬 ×'+combo;
      comboTxt.classList.remove('pop'); void comboTxt.offsetWidth; comboTxt.classList.add('pop');
    }
    if(combo>=3) sfx.combo(combo);
```

`update`（非 freeze 段）加：

```js
    if(comboT>0){
      comboT -= dt;
      if(comboT<=0){ combo=0; comboTxt.classList.remove('pop'); }
    }
```

`resetGame` 加 `combo=0; comboT=0; comboTxt.classList.remove('pop');`。
（雷击依次化墨会天然堆 combo——这是有意的奖励设计。）

- [ ] **Step 6: HP 受击动效**

DOM 引用区加 `const healthBar=document.querySelector('.healthbar');`。
敌人越线分支 `sfx.hurt();` 之后加：

```js
        healthBar.classList.remove('hit'); void healthBar.offsetWidth; healthBar.classList.add('hit');
```

- [ ] **Step 7: 新敌人名牌**

fx/hud 区加：

```js
  // ---------- new enemy nameplate ----------
  const ANIMAL_NAMES={tadpole:'墨蝌',octopus:'墨蛸',jellyfish:'墨蜇'};
  const CN_NUM=['零','一','二','三','四','五','六','七','八','九'];
  let seenTypes=new Set(), nameTimer=null;
  function announceNewEnemies(){
    const cfg=getLevelConfig(level);
    const news=cfg.animals.filter(a=>!seenTypes.has(a.type));
    if(!news.length) return;
    news.forEach(a=>seenTypes.add(a.type));
    namePlate.textContent = news.map(a=>ANIMAL_NAMES[a.type]+' · '+(CN_NUM[a.hits]||a.hits)+'斬').join('　');
    namePlate.classList.add('show');
    clearTimeout(nameTimer);
    nameTimer=setTimeout(()=>namePlate.classList.remove('show'), 1600);
  }
```

`startLevel` 末尾加 `announceNewEnemies();`；`startBtn` 与 `retryBtn` handler 中 `resetGame();` 之后加 `announceNewEnemies();`；`resetGame` 加 `seenTypes=new Set(); namePlate.classList.remove('show');`。

- [ ] **Step 8: 手动验证**

- 生命印章为方形朱砂印，丢命时碎裂旋转动画后变灰框。
- 得分数字滚动递增并回弹；击杀点冒 `+N` 飘字。
- 2s 内连杀出现「連斬 ×N」放大动画，combo≥3 每杀有递升音调；断 2s 后消失。
- HP 条受击红闪抖动；底部进度条有墨色流动感。
- 第 2 关开场显示「墨蛸 · 二斬」，第 3 关显示「墨蜇 · 二斬」；第 1 关开局显示「墨蝌 · 一斬」。
- HUD 四角布局更紧凑，无遮挡游戏区。Console 无报错。

- [ ] **Step 9: Commit**

```bash
git add mo-shou_3.html && git commit -m "feat: HUD overhaul - seal lives, rolling score, floaters, combo, nameplates, bar effects"
```

---

### Task 6: 最高分持久化 + 性能收尾（对象池）

**Files:**
- Modify: `mo-shou_3.html`

**Interfaces:**
- Produces: `loadBest()/saveBest(b)`；`particles`/`floaters`/`shards` 改为环形池（对外 spawn 接口不变）。
- Consumes: Task 1/3/5 的 spawn 函数与 updateFx。

- [ ] **Step 1: 最高分**

CSS 加：

```css
  .go-score .record{color:var(--seal); font-weight:700; letter-spacing:4px;}
```

persist 区（audio 模块旁）加：

```js
  // ---------- persist: best score ----------
  function loadBest(){
    try{ return JSON.parse(localStorage.getItem('inkguard_best'))||{score:0,level:1}; }
    catch(e){ return {score:0,level:1}; }
  }
  function saveBest(b){
    try{ localStorage.setItem('inkguard_best', JSON.stringify(b)); }catch(e){}
  }
```

`gameOver()` 中 `finalScoreEl.textContent = 'SCORE '+score;` 替换为：

```js
    const best = loadBest();
    if(score > best.score){
      saveBest({score:score, level:level});
      finalScoreEl.innerHTML = '<span class="record">新紀錄！</span>　SCORE '+score;
    } else {
      finalScoreEl.textContent = 'SCORE '+score+' · 最佳 '+best.score;
    }
```

- [ ] **Step 2: 环形对象池**

将 `particles`/`floaters`/`shards` 三个系统改为固定容量环形池。以 particles 为例（floaters 容量 32、shards 容量 64 同构改法）：

```js
  // ---------- pools ----------
  const P_MAX=256; const particles=[]; let pCur=0;
  for(let i=0;i<P_MAX;i++) particles.push({alive:false});
  function spawnParticle(x,y,vx,vy,life,r){
    const p = particles[pCur]; pCur=(pCur+1)%P_MAX;
    p.alive=true; p.x=x; p.y=y; p.vx=vx; p.vy=vy;
    p.life=life; p.maxLife=life; p.r=r||2;
  }
```

`burst`/`splatter` 改为调用 `spawnParticle`（burst 传 `500,2`）。
`updateFx` 中三个系统统一为标记式（**不再 filter 重建数组**）：

```js
    for(let i=0;i<P_MAX;i++){ const p=particles[i]; if(!p.alive) continue;
      p.vy+=0.0004*dt; p.x+=p.vx*dt; p.y+=p.vy*dt; p.life-=dt;
      if(p.life<=0) p.alive=false;
    }
```

render 侧同样 `if(!p.alive) continue;`。floaters/shards 的 spawn 函数字段照原样搬入池槽位（shards 的 `poly` 数组每次 spawn 重建，容量 64 可接受）。
`resetGame` 中原来的 `particles=[]; ... shards=[]; floaters=[];` 改为把三池全部 `alive=false`：

```js
    particles.forEach(p=>p.alive=false);
    floaters.forEach(f=>f.alive=false);
    shards.forEach(s=>s.alive=false);
```

注意：`particles`/`floaters`/`shards` 从 `let` 改为 `const`（不再整体重赋值），并删除 game state 区原 `let` 声明中的对应项。

- [ ] **Step 3: 手动验证（含性能）**

- 全功能回归：命中/斩杀/雷击/过关/游戏结束逐项过一遍，表现与 Task 1–5 验证一致。
- 结束界面：首局显示 `SCORE X · 最佳 0` 之后破纪录显示红字「新紀錄！」；刷新页面后最佳分保留。
- DevTools Performance 录 60s 高强度游玩：桌面 Chrome 帧率 ≥55fps；Memory 曲线锯齿正常、无持续攀升。
- Console 禁 localStorage（隐私窗口）验证游戏正常、无报错。

- [ ] **Step 4: Commit**

```bash
git add mo-shou_3.html && git commit -m "feat: best-score persistence + object pooling for particles/floaters/shards"
```

---

## 最终验收（对照 spec「测试方案」）

- [ ] 桌面 Chrome + DevTools 移动模拟（触摸、DPR=2/3）双端冒烟通过
- [ ] 每个特效触发路径人工核对（命中/斩杀/越线/雷击/过关/结束）
- [ ] 60s Performance 录制 ≥55fps、无内存持续增长
- [ ] 隐私模式（无 localStorage）与静音环境无报错
- [ ] `CLAUDE.md` 架构段补充新模块（stain 层、pools、audio）一句话说明
