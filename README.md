<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Deal of Fate 🎰</title>
  <style>
    * {margin: 0; padding: 0; box-sizing: border-box;}
    body {
      background: radial-gradient(circle, #200020, #000);
      color: #fff;
      font-family: 'Courier New', monospace;
      overflow: hidden;
    }
    canvas {
      display: block;
      margin: 0 auto;
      background: #1b0020;
      border: 4px solid #fff;
    }
    #menu, #dialogo, #loja {
      position: absolute;
      top: 0; left: 0;
      width: 100%; height: 100%;
      background: rgba(0, 0, 0, 0.85);
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      color: #fff;
      font-size: 20px;
      text-align: center;
      z-index: 5;
    }
    button {
      background: #ff0044;
      color: white;
      border: none;
      padding: 15px 30px;
      margin: 10px;
      border-radius: 10px;
      font-size: 18px;
      cursor: pointer;
      transition: 0.2s;
    }
    button:hover { background: #ff3366; }
    #dialogo, #loja { display: none; }
  </style>
</head>
<body>
  <div id="menu">
    <h1>🎩 Deal of Fate 🎰</h1>
    <p>Você perdeu uma aposta para o dono do cassino...</p>
    <button onclick="iniciarJogo()">🎮 Start Game</button>
    <button onclick="mostrarCreditos()">💬 Créditos</button>
  </div>

  <div id="dialogo"></div>
  <div id="loja"></div>
  <canvas id="game" width="800" height="500"></canvas>

  <audio id="bgm" loop>
    <source src="https://cdn.pixabay.com/download/audio/2023/03/01/audio_2b0dc91fcb.mp3?filename=retro-gaming-music-139255.mp3" type="audio/mpeg">
  </audio>

  <script>
    const canvas = document.getElementById("game");
    const ctx = canvas.getContext("2d");
    const menu = document.getElementById("menu");
    const dialogo = document.getElementById("dialogo");
    const loja = document.getElementById("loja");
    const bgm = document.getElementById("bgm");

    let jogoAtivo = false;
    let player = { x: 380, y: 400, vx: 0, vy: 0, w: 40, h: 40, cor: "#ff0044", moedas: 0 };
    let tiros = [];
    let inimigos = [{x: 380, y: 100, w: 60, h: 60, vida: 50, tipo: "Dicey Don"}];

    document.addEventListener("keydown", (e) => {
      if(!jogoAtivo) return;
      if(e.key === "a" || e.key === "ArrowLeft") player.vx = -5;
      if(e.key === "d" || e.key === "ArrowRight") player.vx = 5;
      if(e.key === " " || e.key === "k") atirar();
    });
    document.addEventListener("keyup", (e) => {
      if(e.key === "a" || e.key === "ArrowLeft" || e.key === "d" || e.key === "ArrowRight") player.vx = 0;
    });

    function iniciarJogo() {
      menu.style.display = "none";
      bgm.play();
      jogoAtivo = true;
      loop();
    }

    function mostrarCreditos() {
      menu.innerHTML = `
        <h2>💬 Créditos</h2>
        <p>🎩 Criado por: Caio Nascimento dos Santos</p>
        <p>🧠 Ideia e design com ajuda da IA ChatGPT (GPT-5)</p>
        <button onclick="location.reload()">Voltar</button>
      `;
    }

    function atirar() {
      tiros.push({x: player.x + player.w/2 - 5, y: player.y, w: 10, h: 10, vy: -8});
    }

    function loop() {
      if(!jogoAtivo) return;
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // Fundo
      ctx.fillStyle = "#150015";
      ctx.fillRect(0,0,canvas.width,canvas.height);
      ctx.fillStyle = "#ff0044";
      ctx.font = "20px Courier";
      ctx.fillText("Moedas: " + player.moedas, 10, 30);

      // Player
      player.x += player.vx;
      ctx.fillStyle = player.cor;
      ctx.fillRect(player.x, player.y, player.w, player.h);

      // Tiros
      for(let i=0; i<tiros.length; i++){
        let t = tiros[i];
        t.y += t.vy;
        ctx.fillStyle = "#fff";
        ctx.fillRect(t.x, t.y, t.w, t.h);
      }
      tiros = tiros.filter(t => t.y > 0);

      // Inimigos
      for(let e of inimigos){
        ctx.fillStyle = "#00ffff";
        ctx.fillRect(e.x, e.y, e.w, e.h);
        ctx.fillStyle = "#fff";
        ctx.fillText(e.tipo, e.x, e.y - 10);

        // Dano
        for(let t of tiros){
          if(t.x < e.x + e.w && t.x + t.w > e.x && t.y < e.y + e.h && t.y + t.h > e.y){
            e.vida -= 5;
            t.y = -100;
          }
        }

        // Boss derrota
        if(e.vida <= 0){
          player.moedas += 10;
          inimigos.splice(inimigos.indexOf(e),1);
          mostrarDialogo("Você derrotou " + e.tipo + "! +10 moedas", ()=>{});
        }
      }

      // Próximo boss
      if(inimigos.length === 0){
        inimigos.push({x: Math.random()*700, y: 100, w: 60, h: 60, vida: 60, tipo: "Slotty Trio"});
      }

      requestAnimationFrame(loop);
    }

    function mostrarDialogo(texto, callback){
      dialogo.style.display = "flex";
      dialogo.innerHTML = `<p id='fala'></p>`;
      let i=0;
      function digitar(){
        if(i<texto.length){
          document.getElementById("fala").textContent += texto.charAt(i);
          i++;
          setTimeout(digitar,30);
        } else {
          dialogo.innerHTML += `<br><button onclick="fecharDialogo()">OK</button>`;
          dialogo.callback = callback;
        }
      }
      digitar();
    }

    function fecharDialogo(){
      dialogo.style.display = "none";
      if(dialogo.callback) dialogo.callback();
    }
  </script>
</body>
</html>
