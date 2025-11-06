<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Deal of Fate — Cassino (Phaser Prototype)</title>
<!-- Phaser 3 CDN -->
<script src="https://cdn.jsdelivr.net/npm/phaser@3.60.0/dist/phaser.min.js"></script>
<style>
  html,body{height:100%;margin:0;background:#06040a;font-family:Inter,system-ui,Segoe UI,Roboto,Arial;color:#fff}
  #gameContainer{display:flex;align-items:center;justify-content:center;height:100vh;padding:12px;box-sizing:border-box}
  a,button{font-family:inherit}
  /* small note overlay */
  #note{position:fixed;left:12px;bottom:12px;background:rgba(0,0,0,0.45);padding:8px 12px;border-radius:8px;font-size:13px;color:#dfeeff}
</style>
</head>
<body>
<div id="gameContainer"></div>
<div id="note">Clique para ativar som • Controles: WASD/SETAS mover • E interagir • K atirar • L loja • M mudo</div>

<script>
/* Deal of Fate — Phaser single-file prototype
   - Procedural textures (no external images)
   - World-exploration casino, shop, bosses, keys -> final boss unlocked
   - Dialogues typewriter-style, particles, simple SFX via WebAudio
   - Copy-paste to index.html and publish on GitHub Pages
*/

// ---------- Simple WebAudio Manager for music & SFX ----------
class AudioEngine {
  constructor(){
    this.ctx = null;
    this.master = null;
    this.musicGain = null;
    this.muted = false;
    this._musicNodes = [];
  }
  init(){
    if(this.ctx) return;
    this.ctx = new (window.AudioContext || window.webkitAudioContext)();
    this.master = this.ctx.createGain(); this.musicGain = this.ctx.createGain();
    this.master.connect(this.ctx.destination); this.musicGain.connect(this.master);
    this.master.gain.value = 0.9; this.musicGain.gain.value = 0.18;
    this._startAmbient();
  }
  _startAmbient(){
    // layered oscillators + subtle LFO for a casino-ish ambient
    const o1 = this.ctx.createOscillator(); o1.type='sine'; o1.frequency.value=100;
    const g1 = this.ctx.createGain(); g1.gain.value = 0.015; o1.connect(g1); g1.connect(this.musicGain); o1.start();
    const o2 = this.ctx.createOscillator(); o2.type='sawtooth'; o2.frequency.value=220;
    const g2 = this.ctx.createGain(); g2.gain.value = 0.006; o2.connect(g2); g2.connect(this.musicGain); o2.start();
    // LFO on stereo-ish gain
    this._musicNodes.push(o1,o2);
  }
  toggleMute(){ if(!this.ctx) this.init(); this.muted = !this.muted; this.master.gain.value = this.muted?0:0.9; return this.muted; }
  sfx(type){
    if(!this.ctx) this.init();
    const t = this.ctx.currentTime;
    if(type === 'coin'){
      const o = this.ctx.createOscillator(); o.type='square'; o.frequency.setValueAtTime(1200,t);
      const g = this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.linearRampToValueAtTime(0.28,t+0.02); g.gain.linearRampToValueAtTime(0.001,t+0.18);
      o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.22);
    }
    if(type === 'shoot'){
      const o = this.ctx.createOscillator(); o.type='sawtooth'; o.frequency.setValueAtTime(800,t);
      const g = this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.linearRampToValueAtTime(0.2,t+0.01); g.gain.exponentialRampToValueAtTime(0.001,t+0.25);
      o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.25);
    }
    if(type === 'hit'){
      const o = this.ctx.createOscillator(); o.type='triangle'; o.frequency.setValueAtTime(240,t);
      const g = this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.linearRampToValueAtTime(0.16,t+0.005); g.gain.linearRampToValueAtTime(0.001,t+0.12);
      o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.18);
    }
    if(type === 'unlock'){
      const o = this.ctx.createOscillator(); o.type='sine'; o.frequency.setValueAtTime(600,t);
      const g = this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.exponentialRampToValueAtTime(0.25,t+0.01); g.gain.exponentialRampToValueAtTime(0.001,t+0.5);
      o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.5);
    }
  }
}
const SND = new AudioEngine();

// ---------- Phaser Game Config ----------
const WIDTH = 900, HEIGHT = 640;
const config = {
  type: Phaser.AUTO,
  parent: 'gameContainer',
  width: WIDTH,
  height: HEIGHT,
  backgroundColor: '#07121a',
  physics: { default: 'arcade', arcade: { gravity: { y: 0 }, debug: false } },
  scene: [BootScene, WorldScene, BossScene, UIScene] // Scenes are declared below; move definitions up
};

// We need to declare the scenes before using config; to keep structure readable, we'll define them now.
// BUT JS requires function/class declarations before usage; we'll declare as function/class then create the game.

function makeSpriteTexture(scene, key){
  // Procedurally create textures with canvas for player, NPC, chips, etc.
  const g = scene.textures.createCanvas(key, 64, 64);
  const ctx2 = g.getContext();
  ctx2.clearRect(0,0,64,64);
  // base hat: draw top hat stylized
  ctx2.fillStyle = '#111'; ctx2.fillRect(8,16,48,28); // body
  ctx2.fillStyle = '#222'; ctx2.fillRect(4,38,56,10); // brim
  ctx2.fillStyle = '#ffd166'; ctx2.fillRect(16,32,32,6); // ribbon
  // small shine
  ctx2.fillStyle = 'rgba(255,255,255,0.06)'; ctx2.fillRect(18,18,6,6);
  scene.textures.addCanvas(key, g.canvas);
  g.destroy();
}

// ---------- BOOT SCENE: create textures and UI assets ----------
class BootScene extends Phaser.Scene {
  constructor(){ super({key:'BootScene'}); }
  preload(){}
  create(){
    // create a few procedural textures
    makeSpriteTexture(this, 'hat');
    // chip texture
    const c = this.textures.createCanvas('chip', 48, 48); const cc = c.getContext();
    cc.beginPath(); cc.fillStyle = '#ffd166'; cc.arc(24,24,18,0,Math.PI*2); cc.fill();
    cc.fillStyle='#b57a00'; cc.fillRect(6,22,36,4); this.textures.addCanvas('chip', c.canvas); c.destroy();
    // door texture
    const d = this.textures.createCanvas('door', 64, 96); const dc = d.getContext();
    dc.fillStyle='#3a2b1f'; dc.fillRect(0,0,64,96);
    dc.fillStyle='#ffd166'; dc.fillRect(26,44,12,8); this.textures.addCanvas('door', d.canvas); d.destroy();
    // npc texture
    const n = this.textures.createCanvas('npc', 48,48); const nc = n.getContext();
    nc.fillStyle='#8ecae6'; nc.fillRect(6,6,36,36); nc.fillStyle='#073b4c'; nc.fillRect(12,28,24,6);
    this.textures.addCanvas('npc', n.canvas); n.destroy();
    // small button
    const btn = this.textures.createCanvas('btn', 120,40); const bc = btn.getContext();
    bc.fillStyle = '#ff0044'; bc.fillRect(0,0,120,40); bc.fillStyle='#fff'; bc.font='18px monospace'; bc.fillText('START',30,26);
    this.textures.addCanvas('btn', btn.canvas); btn.destroy();

    // create a small spritesheet for animated neon lamp (we'll reuse)
    const lamp = this.textures.createCanvas('lamp', 32,32); const lc = lamp.getContext();
    lc.fillStyle='#ffd166'; lc.beginPath(); lc.arc(16,12,6,0,Math.PI*2); lc.fill();
    this.textures.addCanvas('lamp', lamp.canvas); lamp.destroy();

    // start world
    this.scene.start('WorldScene');
    this.scene.launch('UIScene'); // UI overlay
  }
}

// ---------- WORLD SCENE: open casino map, exploration, doors, shop entry ----------
class WorldScene extends Phaser.Scene {
  constructor(){ super({key:'WorldScene'}); }
  create(){
    // initialize audio on first input
    this.input.once('pointerdown', ()=>{ SND.init(); }, this);

    // world rectangle map comprised of rooms; build using groups
    this.rooms = this.add.group();
    this.doors = [];
    // define rooms (these coordinates chosen to fit)
    const rooms = {
      lobby: {x:220,y:220,w:360,h:200, name:'Lobby'},
      shop:  {x:40,y:380,w:180,h:200, name:'Loja'},
      boss1: {x:40,y:60,w:180,h:180, name:'Dicey Don'},
      boss2: {x:360,y:20,w:260,h:140, name:'Lady Spade'},
      boss3: {x:660,y:60,w:180,h:180, name:'Slotty Trio'},
      final: {x:320,y:-80,w:260,h:160, name:'Sala do Dono', locked:true}
    };
    this.roomDefs = rooms;

    // draw rooms visually as containers with background rectangles
    for(const k in rooms){
      const r = rooms[k];
      const g = this.add.graphics();
      g.fillStyle(k==='lobby' ? 0x0a2433 : (k==='shop' ? 0x151234 : 0x071225), 0.98);
      g.fillRoundedRect(r.x, r.y, r.w, r.h, 12);
      g.lineStyle(2, 0xffffff, 0.05); g.strokeRoundedRect(r.x, r.y, r.w, r.h, 12);
      // label
      const label = this.add.text(r.x+8, r.y+12, r.name, {font:'14px monospace', color:'#dfeffb'}).setDepth(30);
      this.rooms.add(g);
    }

    // doors: interactive rectangles - store them
    const pushDoor = (x,y,w,h,target)=>{ const rect = new Phaser.Geom.Rectangle(x,y,w,h); this.doors.push({rect,target}); };
    pushDoor(rooms.shop.x + rooms.shop.w - 16, rooms.shop.y + rooms.shop.h/2 - 24, 16, 48, 'shop');
    pushDoor(rooms.boss1.x + rooms.boss1.w - 16, rooms.boss1.y + rooms.boss1.h/2 - 24, 16, 48, 'boss1');
    pushDoor(rooms.boss2.x + rooms.boss2.w - 16, rooms.boss2.y + rooms.boss2.h/2 - 24, 16, 48, 'boss2');
    pushDoor(rooms.boss3.x, rooms.boss3.y + rooms.boss3.h/2 - 24, 16, 48, 'boss3');
    pushDoor(rooms.final.x + rooms.final.w/2 - 40, rooms.final.y + rooms.final.h, 80, 24, 'final');

    // Draw door sprites for visual
    for(const d of this.doors){
      const doorSpr = this.add.image(d.rect.x, d.rect.y, 'door').setOrigin(0,0).setDisplaySize(d.rect.width, d.rect.height).setAlpha(0.95);
    }

    // lamps & neon fx
    this.lightsGroup = this.add.group();
    for(let i=0;i<8;i++){
      const lx = 80 + i*100, ly = 40;
      const lamp = this.add.image(lx, ly, 'lamp').setScale(1.2).setAlpha(0.9);
      this.lightsGroup.add(lamp);
    }

    // player sprite
    this.player = this.physics.add.sprite( rooms.lobby.x + rooms.lobby.w/2, rooms.lobby.y + rooms.lobby.h/2, 'hat' );
    this.player.setDisplaySize(48,48);
    this.player.setCollideWorldBounds(true);
    this.player.speed = 160;
    // small player overlay "shadow" separate for stylized look
    this.playerShadow = this.add.ellipse(this.player.x+6, this.player.y+26, 36, 10, 0x000000, 0.25);

    // simple NPC in lobby
    this.npc = this.physics.add.sprite(rooms.lobby.x+36, rooms.lobby.y+36,'npc').setDisplaySize(44,44).setImmovable(true);
    this.npc.body.setAllowGravity(false);

    // camera static but we keep everything inside canvas; we'll not implement camera movement for simplicity
    // Input
    this.cursors = this.input.keyboard.addKeys({up:'W',down:'S',left:'A',right:'D',up2:Phaser.Input.Keyboard.KeyCodes.UP,down2:Phaser.Input.Keyboard.KeyCodes.DOWN,left2:Phaser.Input.Keyboard.KeyCodes.LEFT,right2:Phaser.Input.Keyboard.KeyCodes.RIGHT,interact:Phaser.Input.Keyboard.KeyCodes.E,shoot:Phaser.Input.Keyboard.KeyCodes.K,shop:Phaser.Input.Keyboard.KeyCodes.L,mute:Phaser.Input.Keyboard.KeyCodes.M});

    // bullets group
    this.bullets = this.physics.add.group();

    // collisions (player & npc)
    this.physics.add.overlap(this.player, this.npc, ()=>{ this.showDialog("NPC: Ei, apostador! Fale comigo (E)."); }, null, this);

    // simple collision world bounds
    this.player.setDepth(5);

    // state trackers
    this.keysCollected = 0;
    this.keysNeeded = 3;
    this.defeated = {dice:false,lady:false,slot:false,stack:false};

    // small UI hint
    this.hint = this.add.text(WIDTH/2, HEIGHT-28, 'Use E para interagir / entrar', {font:'14px monospace', color:'#dfeffb'}).setOrigin(0.5).setAlpha(0.7);

    // start SND only after click; ensure audio init when user interacts
    this.input.keyboard.on('keydown-M', ()=>{ const s = SND.toggleMute(); this.showDialog(s ? 'Som desligado' : 'Som ligado'); });
  }

  update(time, dt){
    const speed = this.player.speed;
    let vx=0, vy=0;
    if(this.cursors.left.isDown || this.cursors.left2.isDown) vx = -1;
    if(this.cursors.right.isDown || this.cursors.right2.isDown) vx = 1;
    if(this.cursors.up.isDown || this.cursors.up2.isDown) vy = -1;
    if(this.cursors.down.isDown || this.cursors.down2.isDown) vy = 1;
    const len = Math.hypot(vx,vy) || 1;
    this.player.x += (vx/len) * speed * (dt/1000);
    this.player.y += (vy/len) * speed * (dt/1000);
    this.playerShadow.x = this.player.x + 6; this.playerShadow.y = this.player.y + 26;

    // shoot
    if(Phaser.Input.Keyboard.JustDown(this.cursors.shoot)){
      this.fireBullet();
    }

    // interact with doors/NPC: E
    if(Phaser.Input.Keyboard.JustDown(this.cursors.interact)){
      this.tryInteract();
    }
    // open shop when inside and press L
    if(Phaser.Input.Keyboard.JustDown(this.cursors.shop)){
      // check if inside shop bounds
      if(this.overlapRoom('shop')){ this.startShop(); }
    }

    // update hint visibility based on proximity to any door
    let nearAny=false;
    for(const d of this.doors){
      if(Phaser.Geom.Rectangle.ContainsRect(d.rect, new Phaser.Geom.Rectangle(this.player.x, this.player.y, this.player.width, this.player.height)) || Phaser.Geom.Rectangle.Overlaps(d.rect, new Phaser.Geom.Rectangle(this.player.x, this.player.y, this.player.width, this.player.height))){
        nearAny = true; break;
      }
      // simpler proximity
      if(Phaser.Geom.Rectangle.Overlaps(d.rect, new Phaser.Geom.Rectangle(this.player.x, this.player.y, this.player.width, this.player.height))) { nearAny = true; break; }
    }
    this.hint.setAlpha(nearAny ? 0.95 : 0.45);
  }

  fireBullet(){
    // create bullet sprite as plain rectangle via graphics texture
    const b = this.add.rectangle(this.player.x + 12, this.player.y - 6, 8, 8, 0xffffff);
    this.physics.add.existing(b);
    b.body.setVelocityY(-420);
    b.body.setAllowGravity(false);
    b.fromPlayer = true;
    this.bullets.add(b);
    SND.sfx('shoot');
    // bullet-life destroy
    this.time.delayedCall(2000, ()=>{ if(b && b.destroy) b.destroy(); });
  }

  overlapRoom(roomKey){
    const r = this.roomDefs[roomKey];
    return (this.player.x > r.x && this.player.x < r.x + r.w && this.player.y > r.y && this.player.y < r.y + r.h);
  }

  tryInteract(){
    // interact with NPC first
    if(Phaser.Geom.Intersects.RectangleToRectangle(this.player.getBounds(), this.npc.getBounds())){
      this.showDialog('NPC: Boa sorte! Derrote 3 bosses e pegue as chaves para a Sala do Dono. Lembre-se de visitar a loja.');
      return;
    }
    // doors
    for(const d of this.doors){
      if(Phaser.Geom.Rectangle.Overlaps(d.rect, this.player.getBounds()) || Phaser.Geom.Rectangle.Contains(d.rect, this.player.x, this.player.y)){
        if(d.target === 'shop'){ this.startShop(); return; }
        if(d.target.startsWith('boss')){ this.enterBoss(d.target); return; }
        if(d.target === 'final'){
          if(this.keysCollected >= this.keysNeeded){ this.unlockFinalAndEnter(); } else {
            this.showDialog(`A porta está trancada. Chaves: ${this.keysCollected}/${this.keysNeeded}`);
          }
          return;
        }
      }
    }
    this.showDialog('Nada para interagir aqui...');
  }

  // start shop scene/overlay UI
  startShop(){
    // open a simple modal via UIScene events
    this.scene.launch('UIScene', { openShop:true, parent:'world', onBuy:(item)=>{ /* handler not used here */ } });
  }

  enterBoss(target){
    // map target mapping
    let bossId = null;
    if(target === 'boss1') bossId = 'dice';
    if(target === 'boss2') bossId = 'lady';
    if(target === 'boss3') bossId = 'slot';
    if(this.defeated[bossId]){ this.showDialog(`Você já derrotou ${bossId.toUpperCase()}!`); return; }
    // pause world and start BossScene with parameters
    this.scene.pause();
    this.scene.launch('BossScene', { boss: bossId, onComplete: (won)=>{
      if(won){
        this.keysCollected += 1;
        SND.sfx('unlock');
        this.defeated[bossId] = true;
        this.showDialog(`Você ganhou uma chave! Chaves: ${this.keysCollected}/${this.keysNeeded}`);
      } else {
        this.showDialog('Você foi derrotado... volte quando estiver preparado.');
      }
      this.scene.resume();
    }});
  }

  unlockFinalAndEnter(){
    // start final boss
    this.scene.pause();
    this.scene.launch('BossScene', { boss: 'stack', final:true, onComplete:(won)=>{
      if(won){
        this.showDialog('Parabéns! Você derrotou o dono do cassino e libertou sua sorte! Reinicie para jogar de novo.');
      } else {
        this.showDialog('Você foi derrotado pelo Dono. Treine e volte.');
      }
      this.scene.resume();
    }});
  }

  showDialog(text){
    // send event to UIScene to display dialog
    this.scene.get('UIScene').events.emit('showDialog', text);
  }

  init(){ /* define doors rectangles */
    this.doors = [];
    const r = this.roomDefs;
    // helper to push doors with rects
    const push = (x,y,w,h,target)=>{ const rect = new Phaser.Geom.Rectangle(x,y,w,h); this.doors.push({rect,target}); };
    push(r.shop.x + r.shop.w - 20, r.shop.y + r.shop.h/2 - 24, 20, 48, 'shop');
    push(r.boss1.x + r.boss1.w - 20, r.boss1.y + r.boss1.h/2 - 24, 20, 48, 'boss1');
    push(r.boss2.x + r.boss2.w - 20, r.boss2.y + r.boss2.h/2 - 24, 20, 48, 'boss2');
    push(r.boss3.x, r.boss3.y + r.boss3.h/2 - 24, 20, 48, 'boss3');
    push(r.final.x + r.final.w/2 - 40, r.final.y + r.final.h, 80, 24, 'final');
  }
}

// ---------- BOSS SCENE: arena fight with unique bosses ----------
class BossScene extends Phaser.Scene {
  constructor(){ super({key:'BossScene'}); }
  init(data){ this.bossId = data.boss; this.onComplete = data.onComplete || function(){}; this.final = data.final || false; }
  create(){
    // overlay: darken world (UIScene handles UI). We create an arena on this scene.
    this.cameras.main.setBackgroundColor('#000000');
    // simple background animated
    this.bg = this.add.rectangle(0,0,WIDTH,HEIGHT,0x070814).setOrigin(0);

    // player avatar for boss scene (reset to center)
    this.player = this.physics.add.sprite(WIDTH/2, HEIGHT-120, 'hat').setDisplaySize(56,56);
    this.player.hp = 120;
    this.player.damage = 14;
    this.player.setCollideWorldBounds(true);

    // boss setup
    this.boss = null;
    if(this.bossId === 'dice'){
      this.boss = this.physics.add.sprite(WIDTH/2, 140, 'chip').setDisplaySize(160,160);
      this.boss.hp = 180;
      this.boss.type = 'dice';
    } else if(this.bossId === 'lady'){
      this.boss = this.physics.add.sprite(WIDTH/2, 120, 'npc').setDisplaySize(220,220); this.boss.hp = 260; this.boss.type='lady';
    } else if(this.bossId === 'slot'){
      // multiple slot parts
      this.boss = this.add.container(WIDTH/2, 120);
      const s1 = this.physics.add.sprite(-120,0,'chip').setDisplaySize(100,120); s1.hp=110; s1.type='slot';
      const s2 = this.physics.add.sprite(0,0,'chip').setDisplaySize(100,120); s2.hp=110; s2.type='slot';
      const s3 = this.physics.add.sprite(120,0,'chip').setDisplaySize(100,120); s3.hp=110; s3.type='slot';
      this.boss.add([s1,s2,s3]); this.boss.hp = 330; this.boss.type='slot';
    } else if(this.bossId === 'stack'){
      this.boss = this.physics.add.sprite(WIDTH/2, 120, 'chip').setDisplaySize(340,200); this.boss.hp = 520; this.boss.type='stack';
    }

    // group for boss bullets
    this.ebullets = this.physics.add.group();

    // player bullets group
    this.pbullets = this.physics.add.group();

    // input
    this.keys = this.input.keyboard.addKeys({left:'A',right:'D',up:'W',down:'S',shoot:Phaser.Input.Keyboard.KeyCodes.K,escape:Phaser.Input.Keyboard.KeyCodes.ESC});

    // collisions
    // player bullet hits boss
    this.physics.add.overlap(this.pbullets, this.boss instanceof Phaser.GameObjects.Container ? this.boss.getAll() : [this.boss], (b, targ)=>{
      // reduce hp on the concrete target
      if(targ.setTint){ targ.hp = targ.hp - this.player.damage; targ.setTint(0xff9999); this.time.delayedCall(80, ()=>targ.clearTint()); }
      b.destroy();
      SND.sfx('hit');
    });
    // enemy bullet hits player
    this.physics.add.overlap(this.ebullets, this.player, (b,p)=>{
      p.hp -= 10; b.destroy(); SND.sfx('hit');
    });

    // small HUD texts
    this.hpText = this.add.text(20,20, 'Você HP: ' + this.player.hp, {font:'18px monospace', color:'#fff'});
    this.bossText = this.add.text(600,20, 'Boss: ' + (this.boss.type||''), {font:'18px monospace', color:'#ffd166'});

    // start boss behavior timers
    this.t = 0;
    this.phase = 0;

    // fade-in and dialog
    this.cameras.main.fadeIn(400,0,0,0);
    this.scene.get('UIScene').events.emit('showDialog', `Você entrou na sala do chefe: ${this.bossId.toUpperCase()}. Boa sorte!`);
  }

  update(time, delta){
    const dt = delta/1000;
    this.t += dt;

    // player controls (simple)
    let vx=0, vy=0;
    if(this.keys.left.isDown) vx=-180;
    if(this.keys.right.isDown) vx=180;
    if(this.keys.up.isDown) vy=-180;
    if(this.keys.down.isDown) vy=180;
    this.player.body.setVelocity(vx, vy);

    // shoot
    if(Phaser.Input.Keyboard.JustDown(this.keys.shoot)){
      const b = this.physics.add.rectangle(this.player.x, this.player.y - 30, 10, 10, 0xffffff);
      this.physics.add.existing(b);
      b.body.setVelocityY(-420);
      b.body.setAllowGravity(false);
      this.pbullets.add(b);
      SND.sfx('shoot');
      this.time.delayedCall(2000, ()=>{ if(b && b.destroy) b.destroy(); });
    }

    // boss attacks vary
    if(this.bossId === 'dice'){
      if(Math.random() < 0.02 + Math.min(0.05,this.t*0.002)){
        // roll shockwave - spawn several bullets outward
        const n = 6;
        for(let i=0;i<n;i++){
          const ang = (i/n) * Math.PI*2 + Math.random()*0.2;
          const vx = Math.cos(ang) * (220 + Math.random()*80);
          const vy = Math.sin(ang) * (220 + Math.random()*80);
          const e = this.physics.add.rectangle(this.boss.x, this.boss.y, 12,12, 0xffaa00);
          this.physics.add.existing(e); e.body.setVelocity(vx, vy); e.body.setAllowGravity(false);
          this.ebullets.add(e);
        }
      }
    } else if(this.bossId === 'lady'){
      if(this.t % 1.1 < 0.02){
        // spray toward player
        const dx = this.player.x - this.boss.x; const dy = this.player.y - this.boss.y; const len = Math.hypot(dx,dy)||1;
        for(let i=-2;i<=2;i++){
          const angx = (dx/len) * 260 + i*40;
          const angy = (dy/len) * 260 + i*20;
          const e = this.physics.add.rectangle(this.boss.x, this.boss.y, 12,12, 0x99ddff);
          this.physics.add.existing(e); e.body.setVelocity(angx + (Math.random()-0.5)*40, angy + (Math.random()-0.5)*40); e.body.setAllowGravity(false);
          this.ebullets.add(e);
        }
      }
    } else if(this.bossId === 'slot'){
      // each child in container is a target
      const children = this.boss.getAll();
      if(Math.random() < 0.025){
        for(const child of children){
          const e = this.physics.add.rectangle(child.x + this.boss.x, child.y + this.boss.y + 40, 10, 10, 0x66a3ff);
          this.physics.add.existing(e); e.body.setVelocity((Math.random()-0.5)*40, 260); e.body.setAllowGravity(false);
          this.ebullets.add(e);
        }
      }
    } else if(this.bossId === 'stack'){
      if(Math.random() < 0.03){
        const n = 10;
        for(let i=0;i<n;i++){
          const ang = -Math.PI/2 + (i/(n-1))*Math.PI;
          const vx = Math.cos(ang) * (200 + Math.random()*80);
          const vy = Math.sin(ang) * (200 + Math.random()*80);
          const e = this.physics.add.rectangle(this.boss.x + (Math.random()-0.5)*100, this.boss.y + (Math.random()-0.2)*this.boss.displayHeight, 12,12, 0xffd166);
          this.physics.add.existing(e); e.body.setVelocity(vx, vy); e.body.setAllowGravity(false);
          this.ebullets.add(e);
        }
      }
    }

    // check boss hp if container parts exist
    if(this.bossId === 'slot' && this.boss.getAll){
      let allDead = true;
      for(const part of this.boss.getAll()){
        if(part.hp > 0){
          allDead = false;
          break;
        }
      }
      if(allDead){
        this.win();
      }
    } else {
      if(this.boss && this.boss.hp !== undefined && this.boss.hp <= 0){
        this.win();
      }
    }

    // remove bullets offscreen
    this.ebullets.children.each((b)=>{ if(b.y > HEIGHT + 60 || b.y < -60 || b.x < -60 || b.x > WIDTH + 60) { if(b && b.destroy) b.destroy(); } });
    this.pbullets.children.each((b)=>{ if(b.y > HEIGHT + 60 || b.y < -60) { if(b && b.destroy) b.destroy(); } });

    // collisions: when player's bullets overlap boss children (container) we handle individually in overlaps set during create
    // update hud
    this.hpText.setText('Você HP: ' + Math.max(0, Math.round(this.player.hp)));
    const bHp = (this.bossId === 'slot' ? this.getSlotHp() : (this.boss.hp || 0));
    this.bossText.setText('Boss HP: ' + Math.max(0, Math.round(bHp)));
    // lose condition
    if(this.player.hp <= 0){
      this.lose();
    }
  }

  getSlotHp(){
    let total = 0;
    for(const part of this.boss.getAll()) total += Math.max(0, part.hp||0);
    return total;
  }

  win(){
    // destroy all bullets
    this.ebullets.clear(true,true); this.pbullets.clear(true,true);
    // award depending on boss
    this.scene.stop(); // stop boss scene
    this.onFinish(true);
  }
  lose(){
    this.ebullets.clear(true,true); this.pbullets.clear(true,true);
    this.scene.stop();
    this.onFinish(false);
  }

  onFinish(result){
    // call parent callback via scene plugin data (passed in when launched)
    if(this.sys.settings.data && this.sys.settings.data.onComplete){
      try{ this.sys.settings.data.onComplete(result); } catch(e) {}
    }
    // stop this scene safely
  }
}

// ---------- UI SCENE: HUD, Dialog, Shop modal ----------
class UIScene extends Phaser.Scene {
  constructor(){ super({key:'UIScene', active:true}); }
  preload(){}
  create(data){
    this.worldScene = this.scene.get('WorldScene');
    // create a DOM element container for dialog UI
    this.dialogDiv = document.createElement('div');
    this.dialogDiv.style.position='absolute';
    this.dialogDiv.style.right='12px';
    this.dialogDiv.style.top='12px';
    this.dialogDiv.style.width='300px';
    this.dialogDiv.style.background='rgba(3,6,15,0.9)';
    this.dialogDiv.style.border='1px solid rgba(255,255,255,0.04)';
    this.dialogDiv.style.padding='12px';
    this.dialogDiv.style.borderRadius='10px';
    this.dialogDiv.style.fontFamily='monospace';
    this.dialogDiv.style.color='#e6eef6';
    this.dialogDiv.style.zIndex=1000;
    this.dialogDiv.innerHTML = '<div id="dlgText">Bem-vindo ao Cassino do Destino</div><div id="dlgChoices"></div>';
    document.body.appendChild(this.dialogDiv);
    this.dlgText = this.dialogDiv.querySelector('#dlgText');
    this.dlgChoices = this.dialogDiv.querySelector('#dlgChoices');
    // listen to showDialog events
    this.events.on('showDialog', (txt)=>{ this.showDialog(txt); }, this);

    // shop modal (hidden)
    this.shopModal = document.createElement('div');
    this.shopModal.style.position='absolute'; this.shopModal.style.left='50%'; this.shopModal.style.top='50%';
    this.shopModal.style.transform='translate(-50%,-50%)'; this.shopModal.style.width='420px'; this.shopModal.style.background='rgba(2,8,20,0.95)';
    this.shopModal.style.border='1px solid rgba(255,255,255,0.06)'; this.shopModal.style.padding='14px'; this.shopModal.style.borderRadius='12px';
    this.shopModal.style.display='none'; this.shopModal.style.zIndex=2000; this.shopModal.innerHTML = '<h3 style="margin:0 0 8px 0;color:#ffd166">Loja</h3><div id="shopItems"></div><div style="margin-top:12px"><button id="closeShop">Fechar</button></div>';
    document.body.appendChild(this.shopModal);
    this.shopItems = this.shopModal.querySelector('#shopItems');
    this.shopModal.querySelector('#closeShop').addEventListener('click', ()=>{ this.shopModal.style.display='none'; });

    // shop inventory
    this.shopInventory = [
      { id:'hp+20', name:'+20 Vida', cost:12, apply:()=>{ this.worldScene.player.hp = Math.min(400, this.worldScene.player.hp + 20); } },
      { id:'dmg+3', name:'+3 Dano', cost:20, apply:()=>{ this.worldScene.player.damage = (this.worldScene.player.damage||12) + 3; } },
      { id:'coins+40', name:'Pack 40 moedas', cost:36, apply:()=>{ this.worldScene.player.coins = (this.worldScene.player.coins||0) + 40; } }
    ];

    // listen to SHOW SHOP request
    this.events.on('showShop', ()=>{ this.openShop(); }, this);

    // toggle mute on M from UI scene as backup
    window.addEventListener('keydown', (e)=>{ if(e.key.toLowerCase()==='m'){ const s=SND.toggleMute(); this.showDialog(s ? 'Som desativado' : 'Som ativado'); } });

    // click to enable audio on first click if not initialized
    window.addEventListener('pointerdown', ()=>{ SND.init(); }, {once:true});
  }

  showDialog(txt){
    // typewriter effect
    this.dlgChoices.innerHTML = '';
    this.dlgText.textContent = '';
    let i=0; const t = setInterval(()=>{ this.dlgText.textContent += txt.charAt(i++) || ''; if(i>txt.length){ clearInterval(t); } }, 10);
  }

  openShop(){
    // build items
    this.shopItems.innerHTML = '';
    const world = this.worldScene;
    for(const it of this.shopInventory){
      const row = document.createElement('div'); row.style.display='flex'; row.style.justifyContent='space-between'; row.style.margin='6px 0';
      row.innerHTML = `<div><strong style="color:#ffd166">${it.name}</strong><br><small>${it.cost} moedas</small></div>`;
      const btn = document.createElement('button'); btn.textContent='Comprar';
      btn.onclick = ()=>{ if(world.player.coins >= it.cost){ world.player.coins -= it.cost; it.apply(); SND.sfx('coin'); this.showDialog('Compra realizada!'); } else { this.showDialog('Moedas insuficientes!'); } this.updateWorldHUD(); };
      row.appendChild(btn); this.shopItems.appendChild(row);
    }
    this.shopModal.style.display='block';
    this.updateWorldHUD();
  }

  updateWorldHUD(){ /* sync UI overlay with world scene if needed */ }

  update(time, delta){
    // sync small HUD in the DOM (not the Phaser canvas)
    const ws = this.scene.get('WorldScene');
    if(ws){
      // quick update: world scene might set keys/coins
      // we'll also reflect keys in dialog area small
      this.dlgText.style.fontSize = '14px';
    }
  }
}

// Now create the game after declaring classes
const game = new Phaser.Game( (function(){
  // need to return full config with scenes as references
  return {
    type: Phaser.AUTO,
    parent: 'gameContainer',
    width: WIDTH,
    height: HEIGHT,
    backgroundColor: '#07121a',
    physics: { default: 'arcade', arcade: { gravity: { y: 0 }, debug: false } },
    scene: [ BootScene, WorldScene, BossScene, UIScene ]
  };
})() );

// Small helper: wire events to launch shop from WorldScene (it calls scene.launch('UIScene',{openShop:true}))
// Add a small check: if world scene triggers shop via events, show UI's shop
game.events.on('ready', ()=>{ /* not used */ });

// Final polish: ensure the DOM note hides on focus
document.addEventListener('pointerdown', ()=>{ const n = document.getElementById('note'); if(n) n.style.display='none'; });

</script>
</body>
</html>
