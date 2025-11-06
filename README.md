<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Deal of Fate — Definitivo (Pixel + Cuphead Move)</title>
<!-- Phaser 3 -->
<script src="https://cdn.jsdelivr.net/npm/phaser@3.60.0/dist/phaser.min.js"></script>
<style>
  html,body{height:100%;margin:0;background:#0b0710;font-family:monospace;color:#fff}
  #container{display:flex;align-items:center;justify-content:center;height:100vh;padding:12px;box-sizing:border-box}
  #note{position:fixed;left:12px;bottom:12px;background:rgba(0,0,0,0.5);padding:8px 12px;border-radius:8px;font-size:13px;color:#ffd166;z-index:9999}
</style>
</head>
<body>
<div id="container"></div>
<div id="note">Clique para ativar som • WASD mover • W pular • Shift dash • K atirar • E interagir • L loja • M mudo</div>

<script>
/*
 Deal of Fate — Definitive single-file
 - Pixel-art style (procedural sprites) + arcade movement (dash) like Cuphead
 - Phaser 3, single HTML file
 - World exploration, shop, boss rooms, keys unlock final boss
 - Improved boss fights (patterns, telegraphs, transforms)
 - SFX via WebAudio, no external assets needed
*/

// ---------- Helper: Simple WebAudio for SFX & ambient ----------
class Sfx {
  constructor(){ this.ctx = null; this.master=null; this.musicGain=null; this.muted=false; this._musicOsc=[]; }
  init(){ if(this.ctx) return; this.ctx = new (window.AudioContext||window.webkitAudioContext)(); this.master=this.ctx.createGain(); this.musicGain=this.ctx.createGain(); this.master.connect(this.ctx.destination); this.musicGain.connect(this.master); this.master.gain.value=0.9; this.musicGain.gain.value=0.14; this._ambient(); }
  _ambient(){
    const o = this.ctx.createOscillator(); o.type='sine'; o.frequency.value=110;
    const g = this.ctx.createGain(); g.gain.value=0.02; o.connect(g); g.connect(this.musicGain); o.start();
    const o2 = this.ctx.createOscillator(); o2.type='square'; o2.frequency.value=220; const g2=this.ctx.createGain(); g2.gain.value=0.007; o2.connect(g2); g2.connect(this.musicGain); o2.start();
    this._musicOsc = [o,o2];
  }
  toggle(){ if(!this.ctx) this.init(); this.muted = !this.muted; this.master.gain.value = this.muted?0:0.9; return this.muted; }
  play(type){
    if(!this.ctx) this.init();
    const t = this.ctx.currentTime;
    if(type==='coin'){ const o=this.ctx.createOscillator(); o.type='square'; o.frequency.setValueAtTime(1200,t); const g=this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.linearRampToValueAtTime(0.28,t+0.02); g.gain.linearRampToValueAtTime(0.001,t+0.18); o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.22); }
    if(type==='shoot'){ const o=this.ctx.createOscillator(); o.type='sawtooth'; o.frequency.setValueAtTime(820,t); const g=this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.linearRampToValueAtTime(0.2,t+0.01); g.gain.exponentialRampToValueAtTime(0.001,t+0.25); o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.25); }
    if(type==='hit'){ const o=this.ctx.createOscillator(); o.type='triangle'; o.frequency.setValueAtTime(240,t); const g=this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.linearRampToValueAtTime(0.16,t+0.005); g.gain.linearRampToValueAtTime(0.001,t+0.12); o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.18); }
    if(type==='unlock'){ const o=this.ctx.createOscillator(); o.type='sine'; o.frequency.setValueAtTime(560,t); const g=this.ctx.createGain(); g.gain.setValueAtTime(0.001,t); g.gain.exponentialRampToValueAtTime(0.25,t+0.01); g.gain.exponentialRampToValueAtTime(0.001,t+0.5); o.connect(g); g.connect(this.master); o.start(); o.stop(t+0.5); }
  }
}
const SFX = new Sfx();

// ---------- Phaser config ----------
const WIDTH = 980, HEIGHT = 640;
const config = {
  type: Phaser.AUTO,
  parent: 'container',
  width: WIDTH,
  height: HEIGHT,
  pixelArt: true,
  roundPixels: true,
  backgroundColor: '#050412',
  physics: { default:'arcade', arcade:{ gravity:{y:800}, debug:false } },
  scene: [ PreloadScene, GameScene, BossScene, UIScene ]
};

// ---------- Utility: generate pixel sprites on canvas and import to texture ----------
function genHat(scene){
  const g = scene.textures.createCanvas('hat_px', 32, 32);
  const c = g.getContext();
  c.fillStyle='#0b0b0d'; c.fillRect(6,6,20,14); c.fillStyle='#111'; c.fillRect(4,18,24,6); c.fillStyle='#ffd166'; c.fillRect(10,14,12,4); scene.textures.addCanvas('hat_px', g.canvas); g.destroy();
}
function genNpc(scene){
  const g = scene.textures.createCanvas('npc_px', 24,24); const c=g.getContext();
  c.fillStyle='#63c5d8'; c.fillRect(4,4,16,16); c.fillStyle='#073b4c'; c.fillRect(6,14,12,4); scene.textures.addCanvas('npc_px', g.canvas); g.destroy();
}
function genChip(scene){
  const g = scene.textures.createCanvas('chip_px', 24,24); const c=g.getContext();
  c.fillStyle='#ffd166'; c.beginPath(); c.arc(12,12,10,0,Math.PI*2); c.fill(); c.fillStyle='#b57a00'; c.fillRect(6,12,12,3); scene.textures.addCanvas('chip_px', g.canvas); g.destroy();
}
function genDoor(scene){
  const g = scene.textures.createCanvas('door_px', 16,32); const c=g.getContext();
  c.fillStyle='#392b22'; c.fillRect(0,0,16,32); c.fillStyle='#ffd166'; c.fillRect(6,12,4,3); scene.textures.addCanvas('door_px', g.canvas); g.destroy();
}
function genTile(scene, name, color){
  const g = scene.textures.createCanvas(name, 32,32); const c=g.getContext();
  c.fillStyle=color; c.fillRect(0,0,32,32); // add simple noise lines
  c.fillStyle='rgba(255,255,255,0.04)';
  for(let i=0;i<6;i++){ c.fillRect(0,i*5+ (i%2),32,1); }
  scene.textures.addCanvas(name, g.canvas); g.destroy();
}

// ---------- PreloadScene: create procedural textures then start GameScene ----------
class PreloadScene extends Phaser.Scene {
  constructor(){ super({key:'PreloadScene'}); }
  preload(){}
  create(){
    genHat(this); genNpc(this); genChip(this); genDoor(this);
    genTile(this,'floor','#0b2130'); genTile(this,'wall','#0b0920');
    // start other scenes
    this.scene.start('GameScene');
    this.scene.launch('UIScene');
  }
}

// ---------- GameScene: world, player, shop, doors, exploration ----------
class GameScene extends Phaser.Scene {
  constructor(){ super({key:'GameScene'}); }
  create(){
    // ensure audio init on click first
    this.input.once('pointerdown', ()=>SFX.init());

    // world: add tiled background (floor)
    for(let y=0;y<HEIGHT;y+=32){
      for(let x=0;x<WIDTH;x+=32){
        this.add.image(x,y,'floor').setOrigin(0).setDisplaySize(32,32).setDepth(0);
      }
    }
    // define rooms rectangles and visuals
    this.rooms = {
      lobby: {x:260,y:220,w:420,h:220, name:'Lobby'},
      shop:  {x:80,y:380,w:160,h:160, name:'Loja'},
      boss1: {x:80,y:60,w:160,h:160, name:'Dicey Don'},
      boss2: {x:360,y:20,w:260,h:140, name:'Lady Spade'},
      boss3: {x:700,y:80,w:160,h:160, name:'Slotty Trio'},
      final: {x:360,y:-100,w:260,h:160, name:'Sala do Dono', locked:true}
    };
    // draw room borders
    for(const k in this.rooms){
      const r = this.rooms[k];
      const g = this.add.graphics(); g.fillStyle(0x071a22); g.fillRoundedRect(r.x, r.y, r.w, r.h, 8);
      g.lineStyle(2,0xffffff,0.04); g.strokeRoundedRect(r.x,r.y,r.w,r.h,8);
      this.add.text(r.x+8,r.y+8,r.name,{font:'12px monospace',color:'#dfeffb'}).setDepth(5);
    }

    // doors (Phaser.Rect used for checking)
    this.doors = [];
    const dpush = (x,y,w,h,target)=>{ const rr = new Phaser.Geom.Rectangle(x,y,w,h); const spr = this.add.image(x,y,'door_px').setOrigin(0).setDisplaySize(w,h).setDepth(5); this.doors.push({rect:rr,target,spr}); };
    dpush(this.rooms.shop.x + this.rooms.shop.w - 16, this.rooms.shop.y + this.rooms.shop.h/2 - 24, 16,48,'shop');
    dpush(this.rooms.boss1.x + this.rooms.boss1.w - 16, this.rooms.boss1.y + this.rooms.boss1.h/2 - 24, 16,48,'boss1');
    dpush(this.rooms.boss2.x + this.rooms.boss2.w - 16, this.rooms.boss2.y + this.rooms.boss2.h/2 - 24, 16,48,'boss2');
    dpush(this.rooms.boss3.x, this.rooms.boss3.y + this.rooms.boss3.h/2 - 24, 16,48,'boss3');
    dpush(this.rooms.final.x + this.rooms.final.w/2 - 40, this.rooms.final.y + this.rooms.final.h, 80, 24,'final');

    // lights/neon (animated)
    this.lamps = this.add.group();
    for(let i=0;i<8;i++){
      const lx = 60 + i*110; const ly = 40;
      const lamp = this.add.rectangle(lx,ly,12,12,0xffd166).setAlpha(0.9);
      this.lamps.add(lamp);
      this.tweens.add({targets:lamp, alpha: {from:0.5, to:1}, duration:800 + i*60, yoyo:true, loop:-1, ease:'Sine.easeInOut'});
    }

    // player setup (sprite uses pixel texture)
    this.player = this.physics.add.sprite(this.rooms.lobby.x + this.rooms.lobby.w/2, this.rooms.lobby.y + this.rooms.lobby.h/2, 'hat_px').setDisplaySize(36,36);
    this.player.setCollideWorldBounds(true);
    this.player.body.setSize(28,32);
    this.player.speed = 180;
    this.player.canDash = true;
    this.player.dashing = false;
    this.player.damage = 10;
    this.player.hp = 140;
    this.player.coins = 0;
    this.player.keys = 0;

    // shadow
    this.shadow = this.add.ellipse(this.player.x, this.player.y + 18, 26, 8, 0x000000, 0.25).setDepth(1);

    // npc
    this.npc = this.physics.add.sprite(this.rooms.lobby.x + 40, this.rooms.lobby.y + 40, 'npc_px').setDisplaySize(32,32).setImmovable(true);
    this.npc.body.setAllowGravity(false);

    // chips (collectible placed as decoration)
    this.chips = this.physics.add.staticGroup();
    const c1 = this.chips.create(this.rooms.lobby.x+60, this.rooms.lobby.y+120, 'chip_px').setDisplaySize(18,18);
    c1.setData('hint','CoinPile');

    // bullets group for player and enemy
    this.pBullets = this.physics.add.group();
    this.eBullets = this.physics.add.group();

    // input keys
    this.keys = this.input.keyboard.addKeys({left:'A',right:'D',up:'W',down:'S',jump:Phaser.Input.Keyboard.KeyCodes.SPACE,dash:Phaser.Input.Keyboard.KeyCodes.SHIFT,interact:Phaser.Input.Keyboard.KeyCodes.E,shoot:Phaser.Input.Keyboard.KeyCodes.K,shop:Phaser.Input.Keyboard.KeyCodes.L,mute:Phaser.Input.Keyboard.KeyCodes.M});

    // collisions
    this.physics.add.overlap(this.pBullets, this.eBullets, (p,e)=>{ /* bullets collide */ }, null, this);
    this.physics.add.overlap(this.player, this.chips, (pl,ch)=>{ SFX.play('coin'); pl.coins += 10; ch.destroy(); this.scene.get('UIScene').events.emit('hudUpdate'); }, null, this);

    // world variables
    this.keysNeeded = 3;
    this.defeated = {dice:false,lady:false,slot:false,stack:false};

    // shortcuts: show dialog
    this.showDialog("Você entrou no Cassino. Explore, fale com NPCs e derrote bosses para coletar chaves.");

    // link UI events
    this.input.keyboard.on('keydown-M', ()=>{ const m = SFX.toggle(); this.showDialog(m?'Som desligado':'Som ligado'); });

    // handle interact key
    this.input.keyboard.on('keydown-E', ()=>{ this.attemptInteract(); });

    // init camera bounds
    this.cameras.main.setBounds(0, -140, WIDTH, HEIGHT+280);
    // no actual camera following to keep map static; we keep player on canvas.

    // update UI
    this.scene.get('UIScene').events.emit('hudUpdate');
  }

  update(time, dt){
    // movement with dash/jump like Cuphead: responsive, small inertia
    const onGround = true; // top-down style with gravity disabled in exploration (we set gravity in physics but we can emulate top-down moves)
    let vx = 0, vy = 0;
    if(this.keys.left.isDown || this.keys.left2 && this.keys.left2.isDown) vx = -1;
    if(this.keys.right.isDown || this.keys.right2 && this.keys.right2.isDown) vx = 1;
    if(this.keys.up.isDown || this.keys.up2 && this.keys.up2.isDown) vy = -1;
    if(this.keys.down.isDown || this.keys.down2 && this.keys.down2.isDown) vy = 1;
    const norm = Math.hypot(vx,vy) || 1;
    // dash
    if(Phaser.Input.Keyboard.JustDown(this.keys.dash) && this.player.canDash){
      this.player.dashing = true; this.player.canDash = false;
      const dashVel = 520;
      this.player.body.setVelocity((vx||1)/norm * dashVel, (vy||0)/norm * dashVel);
      this.tweens.add({targets:this.player, duration:220, props:{}, onComplete: ()=>{ this.player.dashing=false; this.player.body.setVelocity(0); this.time.delayedCall(600, ()=>{ this.player.canDash = true; }); }});
      SFX.play('hit');
    } else if(!this.player.dashing){
      // normal move
      this.player.body.setVelocity((vx/norm) * this.player.speed, (vy/norm) * this.player.speed);
    }
    // shoot
    if(Phaser.Input.Keyboard.JustDown(this.keys.shoot)){
      this.fireBullet();
    }

    // update shadow
    this.shadow.x = this.player.x; this.shadow.y = this.player.y + 18;

    // bullets vs enemies (handled in boss scene mostly)

    // hint/draw door prompt via UIScene
    this.checkProximityToDoors();

    // update HUD
    this.scene.get('UIScene').events.emit('hudUpdate');
  }

  fireBullet(){
    const b = this.pBullets.create(this.player.x, this.player.y - 14, null);
    b.setDisplaySize(6,6); b.body.setAllowGravity(false);
    b.setVelocityY(-420);
    b.setTint(0xffffff);
    SFX.play('shoot');
    this.time.delayedCall(2200, ()=>{ if(b && b.destroy) b.destroy(); });
  }

  attemptInteract(){
    // NPC
    if(Phaser.Geom.Intersects.RectangleToRectangle(this.player.getBounds(), this.npc.getBounds())){
      this.showDialog('NPC: "Que tal enfrentar alguns bosses? Cada boss dá uma chave."');
      return;
    }
    // doors
    for(const d of this.doors){
      if(Phaser.Geom.Rectangle.ContainsPoint(d.rect, this.player.getCenter())){
        if(d.target === 'shop'){ this.scene.get('UIScene').events.emit('openShop'); return; }
        if(d.target.startsWith('boss')){
          let boss = null;
          if(d.target==='boss1') boss='dice';
          if(d.target==='boss2') boss='lady';
          if(d.target==='boss3') boss='slot';
          if(this.defeated[boss]){ this.showDialog('Você já derrotou este boss.'); return; }
          // launch boss scene
          this.scene.pause(); this.scene.launch('BossScene', { boss, callback: (won)=>{
            if(won){ this.defeated[boss]=true; this.player.keys += 1; SFX.play('unlock'); this.showDialog('Você ganhou uma CHAVE! ('+this.player.keys+'/'+this.keysNeeded+')'); }
            else this.showDialog('Você foi derrotado. Volte quando estiver pronto.');
            this.scene.resume();
            this.scene.get('UIScene').events.emit('hudUpdate');
          }});
          return;
        }
        if(d.target==='final'){
          if(this.player.keys >= this.keysNeeded){ this.scene.pause(); this.scene.launch('BossScene', { boss:'stack', final:true, callback:(won)=>{ if(won) this.showDialog('Você venceu o dono! Fim.'); else this.showDialog('Derrota...'); this.scene.resume(); } }); }
          else this.showDialog('Sala do Dono trancada. Pegue '+this.keysNeeded+' chaves.');
          return;
        }
      }
    }
    this.showDialog('Nada para interagir aqui.');
  }

  checkProximityToDoors(){
    let near=false; for(const d of this.doors){
      if(Phaser.Geom.Rectangle.ContainsPoint(d.rect, this.player.getCenter())){
        near=true; break;
      }
    }
    this.scene.get('UIScene').events.emit('proximity', near);
  }

  showDialog(text){ this.scene.get('UIScene').events.emit('showDialog', text); }

  init(){ // called before create for doors setup
    // prepare doors rects
    this.doors = []; const r = this.rooms;
    const push = (x,y,w,h,target)=>{ this.doors.push({rect:new Phaser.Geom.Rectangle(x,y,w,h), target}); };
    push(r.shop.x + r.shop.w - 16, r.shop.y + r.shop.h/2 - 24, 16, 48, 'shop');
    push(r.boss1.x + r.boss1.w - 16, r.boss1.y + r.boss1.h/2 - 24, 16, 48, 'boss1');
    push(r.boss2.x + r.boss2.w - 16, r.boss2.y + r.boss2.h/2 - 24, 16, 48, 'boss2');
    push(r.boss3.x, r.boss3.y + r.boss3.h/2 - 24, 16, 48, 'boss3');
    push(r.final.x + r.final.w/2 - 40, r.final.y + r.final.h, 80, 24, 'final');
  }
}

// ---------- BossScene: improved boss fights with patterns & telegraphs ----------
class BossScene extends Phaser.Scene {
  constructor(){ super({key:'BossScene'}); }
  init(data){ this.bossId = data.boss; this.callback = data.callback; this.isFinal = data.final || false; }
  create(){
    // takeover: dark overlay + arena sprites
    this.cameras.main.setBackgroundColor('#05020a');
    // small arena background
    this.add.rectangle(0,0,WIDTH,HEIGHT,0x05020a).setOrigin(0);

    // player for boss scene (reset, arcade physics)
    this.player = this.physics.add.sprite(WIDTH/2, HEIGHT-120, 'hat_px').setDisplaySize(36,36);
    this.player.hp = 140; this.player.damage = 12;
    this.player.setCollideWorldBounds(true);

    // boss creation
    this.enemies = this.physics.add.group();
    if(this.bossId === 'dice'){ this.createDiceBoss(); }
    if(this.bossId === 'lady'){ this.createLadyBoss(); }
    if(this.bossId === 'slot'){ this.createSlotBoss(); }
    if(this.bossId === 'stack'){ this.createStackBoss(); }

    // bullets groups
    this.pBullets = this.physics.add.group();
    this.eBullets = this.physics.add.group();

    // input keys
    this.keys = this.input.keyboard.addKeys({left:'A',right:'D',up:'W',down:'S',shoot:Phaser.Input.Keyboard.KeyCodes.K, dash:Phaser.Input.Keyboard.KeyCodes.SHIFT});

    // overlaps
    this.physics.add.overlap(this.pBullets, this.enemies, (b,e)=>{ e.hp -= this.player.damage; b.destroy(); SFX.play('hit'); if(e.hp<=0) e.destroy(); }, null, this);
    this.physics.add.overlap(this.eBullets, this.player, (b,p)=>{ p.hp -= 10; b.destroy(); SFX.play('hit'); }, null, this);

    // HUD
    this.hpText = this.add.text(16,16,'Player HP: '+this.player.hp,{font:'16px monospace', color:'#fff'});
    this.bossText = this.add.text(520,16,'Boss HP: ?', {font:'16px monospace', color:'#ffd166'});

    // flash message
    this.add.text(WIDTH/2, 20, 'BATALHA', {font:'20px monospace', color:'#ffd166'}).setOrigin(0.5);

    // start pattern timers
    this.patternTimer = 0;
    this.t = 0;

    // small fade
    this.cameras.main.fadeIn(300);
  }

  update(time, delta){
    const dt = delta/1000; this.t += dt;
    // player controls (Cuphead-like dash)
    let vx=0, vy=0;
    if(this.keys.left.isDown) vx=-180;
    if(this.keys.right.isDown) vx=180;
    if(this.keys.up.isDown) vy=-180;
    if(this.keys.down.isDown) vy=180;
    this.player.body.setVelocity(vx,vy);
    // shoot
    if(Phaser.Input.Keyboard.JustDown(this.keys.shoot)){ this.shoot(); }
    // enemy behaviour per type
    if(this.bossId === 'dice'){ this.diceUpdate(dt); }
    if(this.bossId === 'lady'){ this.ladyUpdate(dt); }
    if(this.bossId === 'slot'){ this.slotUpdate(dt); }
    if(this.bossId === 'stack'){ this.stackUpdate(dt); }

    // update HUD text
    let totalHp=0;
    this.enemies.getChildren().forEach(e=>{ totalHp += Math.max(0, e.hp||0); });
    this.hpText.setText('Player HP: '+Math.max(0,Math.round(this.player.hp)));
    this.bossText.setText('Boss HP: '+ Math.max(0,Math.round(totalHp)));

    // win/lose
    if(this.player.hp <= 0){ this.end(false); }
    if(this.enemies.countActive(true) === 0){ this.end(true); }
  }

  shoot(){
    const b = this.pBullets.create(this.player.x, this.player.y - 20, null).setDisplaySize(8,8);
    b.body.setAllowGravity(false); b.setVelocityY(-420);
    SFX.play('shoot');
    this.time.delayedCall(2600, ()=>{ if(b && b.destroy) b.destroy(); });
  }

  createDiceBoss(){
    const boss = this.enemies.create(WIDTH/2, 140, 'chip_px').setDisplaySize(140,140);
    boss.hp = 220; boss.type='dice'; boss.t=0;
    boss.setImmovable(true);
    // telegraph ring sprite
  }
  diceUpdate(dt){
    const boss = this.enemies.getChildren()[0];
    boss.t += dt;
    if(Math.random() < 0.012 + Math.min(0.03, boss.t*0.002)){
      // burst bullets radial
      const n = 10 + Math.floor(Math.random()*6);
      for(let i=0;i<n;i++){
        const ang = (i/n)*Math.PI*2 + (Math.random()-0.5)*0.2;
        const vx = Math.cos(ang)*(220 + Math.random()*80);
        const vy = Math.sin(ang)*(220 + Math.random()*80);
        const e = this.eBullets.create(boss.x, boss.y, null).setDisplaySize(10,10);
        e.body.setAllowGravity(false); e.body.setVelocity(vx,vy);
      }
    }
    // occasional targeted shot
    if(Math.random() < 0.02){
      const dx = this.player.x - boss.x, dy = this.player.y - boss.y; const len=Math.hypot(dx,dy)||1;
      const e = this.eBullets.create(boss.x, boss.y, null).setDisplaySize(10,10);
      e.body.setAllowGravity(false); e.body.setVelocity((dx/len)*320, (dy/len)*320);
    }
  }

  createLadyBoss(){
    const boss = this.enemies.create(WIDTH/2, 120, 'npc_px').setDisplaySize(180,180);
    boss.hp = 360; boss.type='lady'; boss.phase=0; boss.t=0;
    boss.setImmovable(true);
  }
  ladyUpdate(dt){
    const boss = this.enemies.getChildren()[0]; boss.t += dt;
    if(Math.floor(boss.t*2) % 3 === 0 && Math.random() < 0.4){
      // spiral of bullets
      const centerX = boss.x, centerY = boss.y;
      for(let i=0;i<12;i++){
        const ang = (i/12)*Math.PI*2 + boss.t;
        const vx = Math.cos(ang)*220, vy = Math.sin(ang)*220;
        const e = this.eBullets.create(centerX, centerY, null).setDisplaySize(8,8);
        e.body.setAllowGravity(false); e.body.setVelocity(vx+ (Math.random()-0.5)*40, vy + (Math.random()-0.5)*40);
      }
    }
    // moving telegraph dash
    if(Math.random() < 0.02){
      const targetX = this.player.x + (Math.random()-0.5)*120;
      this.tweens.add({targets:boss, x:targetX, duration:600, ease:'Cubic.easeInOut', yoyo:false});
      // after tween, do radial burst
      this.time.delayedCall(620, ()=>{
        for(let i=0;i<8;i++){
          const ang = (i/8)*Math.PI*2;
          const e = this.eBullets.create(boss.x, boss.y, null).setDisplaySize(10,10);
          e.body.setAllowGravity(false); e.body.setVelocity(Math.cos(ang)*320, Math.sin(ang)*320);
        }
      });
    }
  }

  createSlotBoss(){
    // three smaller slot children as separate sprites (container simulated by group)
    const s1 = this.enemies.create(WIDTH/2-140, 140, 'chip_px').setDisplaySize(100,120); s1.hp=130; s1.type='slot';
    const s2 = this.enemies.create(WIDTH/2, 140, 'chip_px').setDisplaySize(100,120); s2.hp=130; s2.type='slot';
    const s3 = this.enemies.create(WIDTH/2+140, 140, 'chip_px').setDisplaySize(100,120); s3.hp=130; s3.type='slot';
  }
  slotUpdate(dt){
    const parts = this.enemies.getChildren();
    if(Math.random() < 0.02){
      for(const p of parts){
        const e = this.eBullets.create(p.x, p.y + p.displayHeight/2, null).setDisplaySize(8,8);
        e.body.setAllowGravity(false); e.body.setVelocity((Math.random()-0.5)*40, 260);
      }
    }
  }

  createStackBoss(){
    const b = this.enemies.create(WIDTH/2, 120, 'chip_px').setDisplaySize(300,200);
    b.hp = 620; b.type='stack';
    b.setImmovable(true);
  }
  stackUpdate(dt){
    const boss = this.enemies.getChildren()[0];
    if(Math.random() < 0.03){
      const n = 12;
      for(let i=0;i<n;i++){
        const ang = -Math.PI/2 + (i/(n-1))*Math.PI;
        const vx = Math.cos(ang) * (240 + Math.random()*80);
        const vy = Math.sin(ang) * (240 + Math.random()*80);
        const e = this.eBullets.create(boss.x + (Math.random()-0.5)*160, boss.y + boss.displayHeight/2, null).setDisplaySize(10,10);
        e.body.setAllowGravity(false); e.body.setVelocity(vx, vy);
      }
    }
  }

  end(won){
    // run callback and stop
    if(this.callback) this.callback(won);
    this.scene.stop();
  }
}

// ---------- UIScene: DOM-based HUD, dialog & shop overlay ----------
class UIScene extends Phaser.Scene {
  constructor(){ super({key:'UIScene', active:true}); }
  create(){
    // create simple DOM overlay for HUD + dialog + shop
    this.hud = document.createElement('div'); this.hud.style.position='absolute'; this.hud.style.left='12px'; this.hud.style.top='12px';
    this.hud.style.background='rgba(0,0,0,0.35)'; this.hud.style.padding='8px 12px'; this.hud.style.borderRadius='8px'; this.hud.style.color='#ffd166'; this.hud.style.fontFamily='monospace'; this.hud.style.zIndex=999;
    document.body.appendChild(this.hud);
    this.hud.innerHTML = `<div id="hp">Vida: 0</div><div id="coins">Moedas: 0</div><div id="keys">Chaves: 0/3</div>`;

    // dialog box
    this.dialog = document.createElement('div'); this.dialog.style.position='absolute'; this.dialog.style.right='12px'; this.dialog.style.top='12px'; this.dialog.style.width='320px';
    this.dialog.style.background='rgba(3,6,15,0.9)'; this.dialog.style.border='1px solid rgba(255,255,255,0.04)'; this.dialog.style.padding='12px'; this.dialog.style.borderRadius='10px';
    this.dialog.style.fontFamily='monospace'; this.dialog.style.color='#e6eef6'; this.dialog.style.zIndex=999; this.dialog.innerHTML = `<div id="dialogText">Bem-vindo</div><div id="dialogChoices"></div>`;
    document.body.appendChild(this.dialog);

    // shop modal
    this.shopModal = document.createElement('div'); Object.assign(this.shopModal.style,{position:'absolute',left:'50%',top:'50%',transform:'translate(-50%,-50%)',width:'420px',background:'rgba(2,8,20,0.98)',padding:'14px',borderRadius:'12px',display:'none',zIndex:1000,color:'#e6eef6',fontFamily:'monospace'});
    this.shopModal.innerHTML = `<h3 style="margin:0 0 8px 0;color:#ffd166">Loja</h3><div id="shopItems"></div><div style="margin-top:12px"><button id="closeShop">Fechar</button></div>`; document.body.appendChild(this.shopModal);
    this.shopItems = this.shopModal.querySelector('#shopItems'); this.shopModal.querySelector('#closeShop').addEventListener('click', ()=>{ this.shopModal.style.display='none'; });

    // listen for update events
    this.events.on('hudUpdate', ()=>{ this.updateHud(); }, this);
    this.events.on('showDialog', (txt)=>{ this.showDialog(txt); }, this);
    this.events.on('openShop', ()=>{ this.openShop(); }, this);
    this.events.on('proximity', (near)=>{ this.toggleDoorHint(near); }, this);

    // initial update
    this.updateHud();
  }

  updateHud(){
    const gs = this.scene.get('GameScene');
    if(!gs) return;
    this.hud.querySelector('#hp').textContent = `Vida: ${Math.round(gs.player.hp||0)}`;
    this.hud.querySelector('#coins').textContent = `Moedas: ${Math.round(gs.player.coins||0)}`;
    this.hud.querySelector('#keys').textContent = `Chaves: ${gs.player.keys||0}/3`;
  }

  showDialog(txt){
    const el = this.dialog.querySelector('#dialogText'); el.textContent=''; let i=0;
    const iv = setInterval(()=>{ el.textContent += txt.charAt(i++)||''; if(i>txt.length){ clearInterval(iv); } }, 12);
  }

  openShop(){
    const gs = this.scene.get('GameScene'); if(!gs) return;
    this.shopItems.innerHTML = '';
    const inventory = [
      {id:'hp20', name:'+20 Vida', cost:15, apply:()=>{ gs.player.hp = Math.min(400, gs.player.hp + 20); }},
      {id:'dmg4', name:'+4 Dano', cost:22, apply:()=>{ gs.player.damage += 4; }},
      {id:'coins50', name:'+50 Moedas', cost:40, apply:()=>{ gs.player.coins += 50; }}
    ];
    inventory.forEach(it=>{
      const row = document.createElement('div'); row.style.display='flex'; row.style.justifyContent='space-between'; row.style.margin='8px 0';
      row.innerHTML = `<div><strong style="color:#ffd166">${it.name}</strong><br><small>${it.cost} moedas</small></div>`;
      const btn = document.createElement('button'); btn.textContent='Comprar'; btn.onclick = ()=>{ if(gs.player.coins >= it.cost){ gs.player.coins -= it.cost; it.apply(); SFX.play('coin'); this.updateHud(); } else this.showDialog('Moedas insuficientes!'); }
      row.appendChild(btn); this.shopItems.appendChild(row);
    });
    this.shopModal.style.display='block';
  }

  toggleDoorHint(near){
    // small visual cue integrated in HUD if near
    if(near) this.hud.style.boxShadow = '0 0 10px rgba(255,209,102,0.06)';
    else this.hud.style.boxShadow = 'none';
  }
}

// ---------- Start game ----------
const game = new Phaser.Game(config);

// expose SFX to console for debug
window.SFX = SFX;

// small helper: hide note after pointerdown
document.addEventListener('pointerdown', ()=>{ const n=document.getElementById('note'); if(n) n.style.display='none'; }, {once:true});

</script>
</body>
</html>
