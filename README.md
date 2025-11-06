<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Deal of Fate — Cassino (Protótipo Mundo Aberto)</title>
<style>
  :root{
    --bg:#07121a; --panel:#0f1720; --accent:#ffd166; --muted:#9aa6b2;
    --ui:#0b1220;
  }
  html,body{height:100%;margin:0;font-family:Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;background:linear-gradient(180deg,#05030a,#060416);}
  #wrap{width:100%;height:100%;display:flex;align-items:center;justify-content:center;padding:12px;box-sizing:border-box}
  #gameUI{width:1100px;max-width:98vw;background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(0,0,0,0.25));border-radius:12px;overflow:hidden;display:flex;flex-direction:column;box-shadow:0 20px 60px rgba(2,6,23,0.6)}
  header{display:flex;align-items:center;padding:12px 16px;border-bottom:1px solid rgba(255,255,255,0.03)}
  header h1{color:var(--accent);font-size:18px;margin:0}
  #main{display:flex;gap:12px;padding:12px}
  #canvasWrap{width:760px;background:linear-gradient(180deg,#07121a,#001018);border-radius:8px;padding:8px;box-sizing:border-box;position:relative}
  canvas{display:block;width:100%;height:640px;background:#07121a;border-radius:6px}
  #side{width:320px;display:flex;flex-direction:column;gap:8px}
  .panel{background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(0,0,0,0.2));padding:12px;border-radius:10px}
  #dialogue{min-height:120px;white-space:pre-wrap}
  .choices{display:flex;flex-direction:column;gap:8px;margin-top:8px}
  .ui-btn{background:transparent;border:1px solid rgba(255,255,255,0.06);padding:8px;border-radius:8px;color:var(--accent);cursor:pointer}
  #hud{position:absolute;left:12px;top:12px;z-index:40;padding:8px;border-radius:8px;background:rgba(0,0,0,0.35);border:1px solid rgba(255,255,255,0.03)}
  #floatingHint{position:absolute;left:50%;transform:translateX(-50%);bottom:18px;z-index:50;background:rgba(0,0,0,0.55);padding:8px 12px;border-radius:8px;color:var(--muted);font-size:13px}
  footer{padding:10px;text-align:center;font-size:12px;color:var(--muted)}
  .shop-list{display:flex;flex-direction:column;gap:8px}
  .shop-item{display:flex;justify-content:space-between;align-items:center}
  .large{font-size:16px}
  /* simple accessibility focus */
  button:focus{outline:2px solid rgba(255,209,102,0.4)}
</style>
</head>
<body>
<div id="wrap">
  <div id="gameUI">
    <header>
      <h1>Deal of Fate — Cassino (Protótipo Mundo Aberto)</h1>
    </header>

    <div id="main">
      <div id="canvasWrap">
        <div id="hud" class="panel">
          Vida: <span id="life">100</span> &nbsp; | &nbsp;
          Moedas: <span id="coins">0</span> &nbsp; | &nbsp;
          Chaves: <span id="keys">0</span>/3
        </div>
        <canvas id="game" width="740" height="640"></canvas>
        <div id="floatingHint">Use W A S D para mover • E para interagir • K para atirar • L para abrir loja (quando dentro)</div>
      </div>

      <div id="side">
        <div class="panel">
          <div class="large">Diálogo</div>
          <div id="dialogue">Bem-vindo ao Cassino do Destino. Explore o salão para encontrar bosses e a loja.</div>
          <div id="choices" class="choices"></div>
        </div>

        <div class="panel">
          <div class="large">Loja (entre na sala e pressione E)</div>
          <div class="shop-list" id="shopList">
            <!-- dynamic -->
          </div>
        </div>

        <div class="panel">
          <div class="large">Status / Dicas</div>
          <div id="statusBox"><small>Derrote bosses para coletar chaves e abrir a Sala do Dono. Explore livremente.</small></div>
        </div>
      </div>
    </div>

    <footer>Arquivo único — cole no index.html do repositório e publique no GitHub Pages.</footer>
  </div>
</div>

<script>
/* ===== Deal of Fate — Mundo Aberto (single-file prototype) =====
   - mapa com salas (Lobby, Loja, 3 Boss Rooms, Sala do Dono fechada)
   - andar livre pelo cassino (WASD)
   - interagir com portas/loja com 'E'
   - bosses dão chaves; 3 chaves abrem sala final
   - diálogos estilo Undertale, loja funcional, HUD, sons via WebAudio
   - Autor: usuário Caio (com ajuda ChatGPT)
*/

// Canvas
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');

// UI
const lifeEl = document.getElementById('life');
const coinsEl = document.getElementById('coins');
const keysEl = document.getElementById('keys');
const dialogueEl = document.getElementById('dialogue');
const choicesEl = document.getElementById('choices');
const shopListEl = document.getElementById('shopList');
const floatingHint = document.getElementById('floatingHint');

// Audio (simple WebAudio SFX + music oscillator background)
class SoundSys {
  constructor() {
    this.ctx = null;
    this.master = null;
    this.musicGain = null;
    this.muted = false;
  }
  init(){
    if(this.ctx) return;
    this.ctx = new (window.AudioContext || window.webkitAudioContext)();
    this.master = this.ctx.createGain(); this.musicGain = this.ctx.createGain();
    this.master.connect(this.ctx.destination); this.musicGain.connect(this.master);
    this.master.gain.value = 0.9; this.musicGain.gain.value = 0.18;
    this._startMusic();
  }
  _startMusic(){
    // layered ambient: two oscillators for retro-casino mood
    const o1 = this.ctx.createOscillator(); o1.type = 'sine'; o1.frequency.value = 110;
    const g1 = this.ctx.createGain(); g1.gain.value = 0.02; o1.connect(g1); g1.connect(this.musicGain); o1.start();
    const o2 = this.ctx.createOscillator(); o2.type = 'square'; o2.frequency.value = 220;
    const g2 = this.ctx.createGain(); g2.gain.value = 0.01; o2.connect(g2); g2.connect(this.musicGain); o2.start();
    this._musicNodes = [o1,o2];
  }
  toggleMute(){ if(!this.ctx) this.init(); this.muted = !this.muted; this.master.gain.value = this.muted?0:0.9; }
  sfx(type){
    if(!this.ctx) this.init();
    const t = this.ctx.currentTime;
    if(type==='shoot'){
      const o = this.ctx.createOscillator(); o.type='sawtooth'; o.frequency.setValueAtTime(900,t);
      const g = this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.linearRampToValueAtTime(0.2,t+0.01); g.gain.exponentialRampToValueAtTime(0.001,t+0.25);
      o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.25);
    }
    if(type==='hit'){
      const o = this.ctx.createOscillator(); o.type='triangle'; o.frequency.setValueAtTime(220,t);
      const g = this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.linearRampToValueAtTime(0.16,t+0.005); g.gain.linearRampToValueAtTime(0.001,t+0.15);
      o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.18);
    }
    if(type==='coin'){
      const o = this.ctx.createOscillator(); o.type='square'; o.frequency.setValueAtTime(1200,t);
      const g = this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.linearRampToValueAtTime(0.32,t+0.02); g.gain.linearRampToValueAtTime(0.001,t+0.18);
      o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.22);
    }
    if(type==='unlock'){
      const o = this.ctx.createOscillator(); o.type='sine'; o.frequency.setValueAtTime(600,t);
      const g = this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.exponentialRampToValueAtTime(0.25,t+0.01); g.gain.exponentialRampToValueAtTime(0.001,t+0.5);
      o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.5);
    }
  }
}
const SND = new SoundSys();

// Game state
const STATE = {
  player: { x: 360, y: 420, w: 36, h: 52, speed: 180, hp: 120, coins: 0, keys: 0, damage: 12 },
  keysNeeded: 3,
  mode: 'explore', // 'explore', 'boss', 'shop', 'dialog'
  inDialog: false,
  currentDialog: null,
  bullets: [],
  particles: [],
  enemies: [],
  map: null,
  canInteract: null,
  lastTime:0
};

// Tile/map layout (simple rooms defined in world coords)
// We'll implement a flat 2D world (900x640 canvas area)
// Rooms are placed in coordinates; doors are rectangles you can interact with.
// Rooms:
// - Lobby center
// - Shop room down-left
// - Boss rooms left/top/right
// - Final boss room top-center locked until keys collected

// Define rooms with rectangles and metadata
const ROOMS = {
  lobby: { x:200,y:240,w:340,h:220, name:'Lobby' },
  shop:  { x:40, y:360, w:160,h:160, name:'Loja' , interact:'shop' },
  boss1: { x:40, y:80,  w:160,h:160, name:'Dicey Don Room', boss:'dice' },
  boss2: { x:360,y:40,  w:200,h:140, name:'Lady Spade Room', boss:'lady' },
  boss3: { x:660,y:80,  w:160,h:160, name:'Slotty Trio Room', boss:'slot' },
  final: { x:320,y: -40, w:260,h:160, name:'Sala do Dono', boss:'stack', locked:true }
};

// Doors: rectangles you can press E to enter
const DOORS = [
  {x: ROOMS.shop.x + ROOMS.shop.w - 20, y: ROOMS.shop.y + Math.floor(ROOMS.shop.h/2)-20, w: 20, h:40, to:'shop'},
  {x: ROOMS.boss1.x + ROOMS.boss1.w - 20, y: ROOMS.boss1.y + Math.floor(ROOMS.boss1.h/2)-20, w:20, h:40, to:'boss1'},
  {x: ROOMS.boss2.x + ROOMS.boss2.w - 20, y: ROOMS.boss2.y + Math.floor(ROOMS.boss2.h/2)-20, w:20, h:40, to:'boss2'},
  {x: ROOMS.boss3.x, y: ROOMS.boss3.y + Math.floor(ROOMS.boss3.h/2)-20, w:20, h:40, to:'boss3'},
  {x: ROOMS.final.x + Math.floor(ROOMS.final.w/2)-40, y: ROOMS.final.y + ROOMS.final.h, w:80, h:20, to:'final'}
];

// Draw helper: rounded rect
function roundRect(ctx,x,y,w,h,r){
  ctx.beginPath();
  ctx.moveTo(x+r,y); ctx.arcTo(x+w,y,x+w,y+h,r); ctx.arcTo(x+w,y+h,x,y+h,r); ctx.arcTo(x,y+h,x,y,r); ctx.arcTo(x,y,x+w,y,r); ctx.closePath();
  ctx.fill();
}

// Dialogue System (typewriter + choices)
function pushDialog(text, choices){
  STATE.inDialog = true; STATE.modePrev = STATE.mode; STATE.mode = 'dialog';
  STATE.currentDialog = { text, choices };
  dialogueEl.textContent = '';
  choicesEl.innerHTML = '';
  typewriter(text, dialogueEl, ()=>{ if(choices && choices.length>0) renderChoices(choices); else { /* auto close */ }});
}
function typewriter(text, element, callback){
  element.textContent = '';
  let i=0;
  const iv = setInterval(()=>{ element.textContent+= text[i++]||''; if(i>text.length){ clearInterval(iv); callback && callback(); } }, 12);
}
function renderChoices(choices){
  choicesEl.innerHTML = '';
  choices.forEach(ch=>{
    const btn = document.createElement('button'); btn.textContent = ch.text; btn.className = 'ui-btn';
    btn.onclick = ()=>{ SND.sfx('coin'); ch.onChoose(); STATE.inDialog=false; STATE.mode = STATE.modePrev || 'explore'; choicesEl.innerHTML=''; dialogueEl.textContent=''; };
    choicesEl.appendChild(btn);
  });
}

// Shop content (inside room)
const SHOP_ITEMS = [
  {id:'hp20', name:'+20 Vida', cost:15, apply:()=>{ STATE.player.hp = Math.min(300, STATE.player.hp + 20); }},
  {id:'dmg+4', name:'+4 Dano', cost:20, apply:()=>{ STATE.player.damage += 4; }},
  {id:'coinpack', name:'Pacote 50 moedas', cost:40, apply:()=>{ STATE.player.coins += 50; }}
];

function openShop(){
  // show shop panel in side area
  shopListEl.innerHTML = '';
  SHOP_ITEMS.forEach(item=>{
    const div = document.createElement('div'); div.className='shop-item';
    div.innerHTML = `<div><strong>${item.name}</strong><br><small>${item.cost} moedas</small></div>`;
    const btn = document.createElement('button'); btn.textContent='Comprar'; btn.className='ui-btn';
    btn.onclick = ()=>{ if(STATE.player.coins >= item.cost){ STATE.player.coins -= item.cost; item.apply(); SND.sfx('coin'); updateHUD(); openShop(); } else { pushDialog('Moedas insuficientes. Volte depois.'); } };
    div.appendChild(btn);
    shopListEl.appendChild(div);
  });
  pushDialog('Você entrou na Loja. Compre melhorias com suas moedas!', [{text:'Sair da loja', onChoose:()=>{ /* nothing */ }}]);
}

// HUD update
function updateHUD(){
  lifeEl.textContent = Math.max(0, Math.round(STATE.player.hp));
  coinsEl.textContent = Math.round(STATE.player.coins);
  keysEl.textContent = `${STATE.player.keys}`;
}

// Input
const keys = {};
window.addEventListener('keydown', e=>{ keys[e.key.toLowerCase()] = true; if(e.key.toLowerCase()==='m') SND.toggleMute(); if(e.key.toLowerCase()==='e') tryInteract(); if(e.key.toLowerCase()==='k') playerShoot(); if(e.key.toLowerCase()==='l' && STATE.mode==='shop') openShop(); });
window.addEventListener('keyup', e=>{ keys[e.key.toLowerCase()] = false; });

// Interaction: check if player is near a door
function tryInteract(){
  if(STATE.mode !== 'explore') return;
  for(const d of DOORS){
    if(rectOverlap(STATE.player, d)){
      if(d.to === 'shop'){ STATE.mode = 'shop'; openShop(); return; }
      if(d.to.startsWith('boss')){
        // boss doors might be locked if already defeated? We'll let player enter any time to fight (repeatable but keys only on first defeat)
        enterBossRoom(d.to);
        return;
      }
      if(d.to === 'final'){
        if(STATE.player.keys >= STATE.keysNeeded){
          // unlock and enter final
          ROOMS.final.locked = false; SND.sfx('unlock'); enterBossRoom('final'); return;
        } else {
          pushDialog(`A porta está trancada. Você precisa de ${STATE.keysNeeded} chaves (você tem ${STATE.player.keys}).`);
          return;
        }
      }
    }
  }
}

// Rect overlap
function rectOverlap(a,b){
  return !(a.x+a.w < b.x || a.x > b.x + b.w || a.y + a.h < b.y || a.y > b.y + b.h);
}

// Enter boss room (transitions to boss mode)
// If boss already defeated, show dialog and return to lobby.
const defeatedBosses = { dice:false, lady:false, slot:false, stack:false };
function enterBossRoom(roomId){
  let bossKey = null;
  if(roomId==='boss1') bossKey='dice';
  if(roomId==='boss2') bossKey='lady';
  if(roomId==='boss3') bossKey='slot';
  if(roomId==='final') bossKey='stack';
  if(bossKey && defeatedBosses[bossKey]){
    pushDialog(`A sala está vazia — você já derrotou ${bossKey.toUpperCase()}.`);
    return;
  }
  // Prepare arena: spawn boss entity
  STATE.mode = 'boss';
  STATE.enemies = [];
  if(bossKey==='dice') spawnBoss_Dice();
  if(bossKey==='lady') spawnBoss_Lady();
  if(bossKey==='slot') spawnBoss_Slot();
  if(bossKey==='stack') spawnBoss_Stack();
  pushDialog(`Você entrou na sala do chefe — prepare-se!`, [{text:'Vamos', onChoose:()=>{ /* start fight */ }}]);
}

// Boss implementations (simple behaviours)
function spawnBoss_Dice(){
  STATE.enemies.push({ id:'dice', x:420, y:120, w:120, h:120, hp:160, phase:0, t:0 });
}
function spawnBoss_Lady(){
  STATE.enemies.push({ id:'lady', x:420, y:110, w:140, h:160, hp:220, phase:0, t:0 });
}
function spawnBoss_Slot(){
  STATE.enemies.push({ id:'slot', x:360, y:120, w:80, h:120, hp:110, phase:0, t:0 });
  STATE.enemies.push({ id:'slot', x:420, y:120, w:80, h:120, hp:110, phase:0, t:0 });
  STATE.enemies.push({ id:'slot', x:480, y:120, w:80, h:120, hp:110, phase:0, t:0 });
}
function spawnBoss_Stack(){
  STATE.enemies.push({ id:'stack', x:420, y:120, w:260, h:160, hp:450, phase:0, t:0 });
}

// Player shooting
function playerShoot(){
  if(STATE.mode !== 'boss' && STATE.mode !== 'explore') return;
  // single bullet
  const b = { x: STATE.player.x + STATE.player.w/2 - 4, y: STATE.player.y, w:8, h:8, vy:-360, from:'player' };
  STATE.bullets.push(b);
  SND.sfx('shoot');
}

// Update loop
function update(dt){
  // movement only in explore or boss mode (player can move in boss)
  if(STATE.mode === 'explore' || STATE.mode === 'boss' || STATE.mode==='shop'){
    let dx=0, dy=0;
    if(keys['w']||keys['arrowup']) dy = -1;
    if(keys['s']||keys['arrowdown']) dy = 1;
    if(keys['a']||keys['arrowleft']) dx = -1;
    if(keys['d']||keys['arrowright']) dx = 1;
    const len = Math.hypot(dx,dy) || 1;
    STATE.player.x += (dx/len) * STATE.player.speed * dt;
    STATE.player.y += (dy/len) * STATE.player.speed * dt;
    // Clamp to canvas
    STATE.player.x = Math.max(8, Math.min(canvas.width - STATE.player.w - 8, STATE.player.x));
    STATE.player.y = Math.max(8, Math.min(canvas.height - STATE.player.h - 8, STATE.player.y));
  }

  // bullets movement
  for(const b of STATE.bullets){
    b.y += (b.vy * dt);
  }
  STATE.bullets = STATE.bullets.filter(b => b.y > -40 && b.y < canvas.height + 40);

  // enemies behaviour (boss attacks)
  for(const e of STATE.enemies){
    e.t += dt * 60;
    if(e.id==='dice'){
      // periodic roll shoot
      e.phase = Math.floor(e.t/60) % 6;
      if(Math.random() < 0.02){
        // shoot random bullet toward player
        const dx = STATE.player.x - e.x;
        const dy = STATE.player.y - e.y;
        const len = Math.hypot(dx,dy) || 1;
        STATE.bullets.push({ x: e.x + e.w/2, y: e.y + e.h/2, w:10, h:10, vx:(dx/len)*200 + (Math.random()-0.5)*60, vy:(dy/len)*200 + (Math.random()-0.5)*60 , from:'enemy'});
      }
    }
    if(e.id==='lady'){
      // telegraphed dashes
      if(e.t % 90 < 2 && Math.random() < 0.24){
        // dash: spawn multiple projectiles
        for(let i=0;i<6;i++){
          const angle = (i/6)*Math.PI*2 + Math.random()*0.4;
          STATE.bullets.push({ x:e.x+e.w/2, y:e.y+e.h/2, w:8, h:8, vx:Math.cos(angle)*260, vy:Math.sin(angle)*260, from:'enemy'});
        }
      }
    }
    if(e.id==='slot'){
      // each slot periodically shoots downward waves
      if(Math.random() < 0.015){
        STATE.bullets.push({ x:e.x + e.w/2, y:e.y + e.h, w:8, h:8, vx:(Math.random()-0.5)*40, vy:200, from:'enemy' });
      }
    }
    if(e.id==='stack'){
      // heavy spreads
      if(Math.random() < 0.03){
        const n = 8;
        for(let i=0;i<n;i++){
          const ang = (i/(n-1))*Math.PI - Math.PI/2;
          STATE.bullets.push({ x:e.x + Math.random()*e.w, y: e.y + e.h/2, w:10, h:10, vx:Math.cos(ang)*220 + (Math.random()-0.5)*50, vy:Math.sin(ang)*220 + (Math.random()-0.5)*50, from:'enemy' });
        }
      }
    }
  }

  // bullet collisions: player vs enemy bullets & bullets vs enemies
  for(const b of STATE.bullets.slice()){
    if(b.from === 'enemy'){
      // treat vx maybe absent
      const bb = { x: b.x, y: b.y, w:b.w, h:b.h };
      if(rectOverlap(bb, STATE.player)){
        // damage
        STATE.player.hp -= 8;
        SND.sfx('hit');
        b.y = -9999;
      }
    } else if(b.from === 'player'){
      for(const e of STATE.enemies){
        const eb = { x:e.x, y:e.y, w:e.w, h:e.h };
        if(rectOverlap(b, eb)){
          e.hp -= STATE.player.damage;
          b.y = -9999;
          SND.sfx('hit');
        }
      }
    }
  }

  // remove dead enemies and handle boss defeat
  for(let i = STATE.enemies.length -1; i>=0; i--){
    const e = STATE.enemies[i];
    if(e.hp <= 0){
      // Reward: coins, key (only first time)
      // Determine boss key by id
      let bossKey = e.id;
      // Mark defeated
      if(bossKey && !defeatedBosses[bossKey]) {
        defeatedBosses[bossKey] = true;
        STATE.player.keys += 1;
        pushDialog(`Você derrotou ${bossKey.toUpperCase()}! Você ganhou uma CHAVE. (${STATE.player.keys}/${STATE.keysNeeded})`, [{text:'Continuar', onChoose:()=>{ /* after boss */ }}]);
        SND.sfx('unlock');
      } else {
        // if already defeated (shouldn't happen often), just coin
        pushDialog(`Inimigo derrotado! Você recebeu moedas.`, [{text:'Ok', onChoose:()=>{}}]);
      }
      STATE.player.coins += 20 + Math.floor(Math.random()*30);
      updateHUD();
      STATE.enemies.splice(i,1);
      // after boss door, return to explore
      if(STATE.enemies.length === 0){
        STATE.mode = 'explore';
        // teleport player back to lobby center for clarity
        STATE.player.x = ROOMS.lobby.x + Math.floor(ROOMS.lobby.w/2) - STATE.player.w/2;
        STATE.player.y = ROOMS.lobby.y + Math.floor(ROOMS.lobby.h/2) - STATE.player.h/2;
      }
    }
  }

  // particles decay
  STATE.particles.forEach(p=>{ p.ttl -= dt*60; p.x += p.vx*dt; p.y += p.vy*dt; });
  STATE.particles = STATE.particles.filter(p=>p.ttl>0);

  // update HUD
  updateHUD();
}

// Render world: rooms, doors, player, enemies, bullets
function render(){
  // clear
  ctx.clearRect(0,0,canvas.width,canvas.height);

  // floor bg
  ctx.fillStyle = '#07121a'; ctx.fillRect(0,0,canvas.width,canvas.height);

  // draw rooms (boxes)
  for(const key in ROOMS){
    const r = ROOMS[key];
    // room appearance
    ctx.fillStyle = (key==='lobby' ? '#0a2433' : (key==='shop' ? '#1a1a2e' : '#101026'));
    ctx.globalAlpha = 0.98;
    roundRect(ctx, r.x, r.y, r.w, r.h, 8);
    // outline
    ctx.strokeStyle = 'rgba(255,255,255,0.04)'; ctx.lineWidth = 1; ctx.stroke();
    // label
    ctx.fillStyle = '#dfeffb'; ctx.font = '14px monospace'; ctx.fillText(r.name, r.x + 8, r.y + 18);
    // if final and locked, draw padlock
    if(key === 'final' && (ROOMS.final.locked || STATE.player.keys < STATE.keysNeeded)){
      ctx.fillStyle = 'rgba(0,0,0,0.6)'; ctx.fillRect(r.x, r.y, r.w, r.h);
      ctx.fillStyle = '#ffd166'; ctx.font = '16px monospace';
      ctx.fillText('Sala do Dono (Trancada)', r.x + 12, r.y + r.h/2);
    }
  }
  ctx.globalAlpha = 1;

  // draw doors
  for(const d of DOORS){
    ctx.fillStyle = '#ffd166'; ctx.fillRect(d.x, d.y, d.w, d.h);
    ctx.strokeStyle = '#000'; ctx.strokeRect(d.x, d.y, d.w, d.h);
  }

  // draw items or indicators (shop sign)
  ctx.fillStyle = '#ffd166'; ctx.font = '18px monospace';
  ctx.fillText('LOJA', ROOMS.shop.x + 8, ROOMS.shop.y + 20);

  // draw player
  ctx.fillStyle = '#ffffff';
  // player shadow
  ctx.fillStyle = 'rgba(0,0,0,0.25)'; ctx.fillRect(STATE.player.x + 6, STATE.player.y + STATE.player.h - 6, STATE.player.w - 8, 6);
  ctx.fillStyle = '#ff6b6b'; ctx.fillRect(STATE.player.x, STATE.player.y, STATE.player.w, STATE.player.h);
  ctx.fillStyle = '#ffd166'; ctx.fillRect(STATE.player.x + 6, STATE.player.y + STATE.player.h - 22, STATE.player.w - 12, 8);

  // draw bullets
  for(const b of STATE.bullets){
    if(b.from === 'player'){ ctx.fillStyle = '#fff'; ctx.fillRect(b.x, b.y, b.w, b.h); }
    else { ctx.fillStyle = '#ffb3b3'; ctx.fillRect(b.x, b.y, 8, 8); }
  }

  // draw enemies
  for(const e of STATE.enemies){
    // boss body
    ctx.fillStyle = (e.id==='dice' ? '#e9c46a' : e.id==='lady' ? '#8ecae6' : e.id==='slot' ? '#3f72af' : '#ffd166');
    ctx.fillRect(e.x, e.y, e.w, e.h);
    // hp bar
    ctx.fillStyle = 'rgba(0,0,0,0.5)'; ctx.fillRect(e.x, e.y - 10, e.w, 6);
    const hpPct = Math.max(0, e.hp)/ (function(){ if(e.id==='dice') return 160; if(e.id==='lady') return 220; if(e.id==='slot') return 110; if(e.id==='stack') return 450; return 100; })();
    ctx.fillStyle = '#ff4d4d'; ctx.fillRect(e.x, e.y - 10, e.w * hpPct, 6);
    ctx.fillStyle = '#000'; ctx.font = '14px monospace'; ctx.fillText(e.id.toUpperCase(), e.x + 6, e.y + 16);
  }

  // particles
  for(const p of STATE.particles){
    ctx.fillStyle = `rgba(255,255,255,${p.ttl/20})`; ctx.fillRect(p.x, p.y, 2, 2);
  }

  // interaction hint if near door
  let near = false;
  for(const d of DOORS){
    if(rectOverlap(STATE.player, d)){
      near = true;
      // show hint
      ctx.fillStyle = 'rgba(0,0,0,0.6)';
      ctx.fillRect(8, canvas.height - 36, 380, 28);
      ctx.fillStyle = '#ffd166'; ctx.font = '14px monospace';
      ctx.fillText(`Pressione [E] para entrar em ${d.to.toUpperCase()}`, 16, canvas.height - 16);
      break;
    }
  }
  if(!near){
    // hide nothing; floatingHint is separate
  }
}

// Game loop driver
function gameloop(ts){
  if(!STATE.lastTime) STATE.lastTime = ts;
  const dt = Math.min(0.033, (ts - STATE.lastTime)/1000); STATE.lastTime = ts;
  update(dt);
  render();
  requestAnimationFrame(gameloop);
}

// Start audio on first interaction (browser autostart policy)
window.addEventListener('click', ()=>{ SND.init(); }, { once:true });

// Initial HUD
updateHUD();
openShop(); // preload shop list (won't auto-open dialogue)

// Start the loop
requestAnimationFrame(gameloop);

/* ===========================
   Boss-specific spawn behavior
   (Note: bosses were created above; below are helper enhancements)
   =========================== */

// For clarity, create stable sizes & positions for spawned bosses
function spawnBoss_Dice(){ STATE.enemies = [{ id:'dice', x:420-60, y:120, w:120, h:120, hp:160, t:0 }]; }
function spawnBoss_Lady(){ STATE.enemies = [{ id:'lady', x:420-70, y:80, w:140, h:160, hp:220, t:0 }]; }
function spawnBoss_Slot(){ STATE.enemies = [{ id:'slot', x:360, y:120, w:80, h:120, hp:110, t:0},{ id:'slot', x:420, y:120, w:80, h:120, hp:110, t:0},{ id:'slot', x:480, y:120, w:80, h:120, hp:110, t:0 }]; }
function spawnBoss_Stack(){ STATE.enemies = [{ id:'stack', x:420-130, y:80, w:260, h:160, hp:450, t:0 }]; }

// Override earlier spawn functions to refer to these proper ones
spawnBoss_Dice = spawnBoss_Dice;
spawnBoss_Lady = spawnBoss_Lady;
spawnBoss_Slot = spawnBoss_Slot;
spawnBoss_Stack = spawnBoss_Stack;

/* ===========================
   Small safety & helper UI tweaks
   =========================== */
// Ensure player starts in lobby center
STATE.player.x = ROOMS.lobby.x + Math.floor(ROOMS.lobby.w/2) - STATE.player.w/2;
STATE.player.y = ROOMS.lobby.y + Math.floor(ROOMS.lobby.h/2) - STATE.player.h/2;

// make sure final room initially locked
ROOMS.final.locked = true;

// Expose some debugging helpers in console (optional)
window.DEALOF = { STATE, ROOMS, DOORS, SND, spawnBoss_Dice, spawnBoss_Lady, spawnBoss_Slot, spawnBoss_Stack };

/* ===========================
   Notes and polish recommendations:
   - This is a prototype: for better collisions, sprites and asset loading, use Phaser/Godot.
   - Add images for player and bosses, and more varied attacks/telegraphing.
   - Expand shop and NPC dialogues as desired.
   =========================== */
</script>
</body>
</html>
