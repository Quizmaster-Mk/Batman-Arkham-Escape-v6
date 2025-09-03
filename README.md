<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<title>Batman – Arkham Escape V5</title>
<style>
body { font-family: Arial, sans-serif; background:#111; color:#eee; display:flex; justify-content:center; padding:20px; }
#game { max-width:600px; width:100%; text-align:center; background:#1c1c1c; padding:20px; border-radius:10px; border:2px solid #444; }
h1 { color:#f39c12; }
button { padding:10px 20px; margin:5px; font-size:16px; cursor:pointer; border-radius:5px; border:none; }
#log { margin-top:20px; background:#222; border:1px solid #555; padding:10px; height:200px; overflow-y:auto; text-align:left; }
.avatar { font-size:50px; margin:10px; }
.stats { margin:10px 0; }
.hpbar { width:100%; height:14px; background:#333; border-radius:6px; overflow:hidden; margin-top:4px; }
.hpfill { height:100%; transition:width .3s; background:linear-gradient(90deg,#2ecc71,#1abc9c); }
.enemyfill { background:linear-gradient(90deg,#ff6b6b,#ff4d4d); }
.inventory { display:flex; flex-wrap:wrap; justify-content:center; gap:5px; margin-top:5px; }
.item { background:#222; padding:4px 8px; border-radius:5px; font-size:14px; border:1px solid #444; cursor:pointer; }
</style>
</head>
<body>
<div id="game">
<h1>🦇 Batman – Arkham Escape</h1>
<div id="scene"></div>
<div id="stats"></div>
<div id="actions"></div>
<div id="log"></div>
</div>

<script>
// --- Variables globales ---
let batman;
let difficulty = "normal";
let encounters = 0;
let currentEnemy = null;
let combatEnCours = false;
let riddles = [
  {q:"Je peux être brisé sans être touché, qui suis-je ?", answers:["Un secret","Un verre","Un cœur"], correct:0},
  {q:"Je suis toujours devant toi mais on ne peut me voir, qui suis-je ?", answers:["L'avenir","Un miroir","Le vent"], correct:0},
  {q:"Plus je prends de la place, moins on me voit, qui suis-je ?", answers:["L'obscurité","La fumée","La poussière"], correct:0}
];
let currentRiddle = 0;

// --- Utilitaires ---
function log(msg){ let l=document.getElementById('log'); l.innerHTML+="<div>"+msg+"</div>"; l.scrollTop=l.scrollHeight; }
function rnd(min,max){ return Math.floor(Math.random()*(max-min+1))+min; }
function majStats(){
  let s=document.getElementById('stats');
  let html=`<div class="avatar">🦇</div>
  <div class="stats">PV Batman: ${batman.pv}/${batman.max}<div class="hpbar"><div class="hpfill" style="width:${batman.pv/batman.max*100}%"></div></div></div>`;
  if(currentEnemy && currentEnemy.type==="combat"){
    html+=`<div class="stats">PV ${currentEnemy.nom}: ${currentEnemy.pv}/${currentEnemy.max}<div class="hpbar"><div class="hpfill enemyfill" style="width:${currentEnemy.pv/currentEnemy.max*100}%"></div></div></div>`;
  }
  if(batman.objets.length>0){
    html+=`<div>Objets:<div class="inventory">`;
    batman.objets.forEach((it,i)=>{ html+=`<div class="item" onclick="useItem(${i})">${it}</div>`; });
    html+=`</div></div>`;
  }
  s.innerHTML=html;
}
function majScene(html){ document.getElementById('scene').innerHTML = html; }
function majActions(html){ document.getElementById('actions').innerHTML = html; }

// --- Menu de difficulté ---
function menuDifficulte(){
  majScene("<p>Choisis ton niveau de difficulté :</p>");
  majActions(`
    <button onclick="startGame('facile')">😃 Facile</button>
    <button onclick="startGame('normal')">😐 Normal</button>
    <button onclick="startGame('difficile')">😈 Difficile</button>
  `);
}

// --- Début ---
function startGame(diff){
  difficulty = diff;
  if(diff==="facile"){ batman={ pv:50, max:50, objets:[], hasEvade:false, batarang:false }; }
  if(diff==="normal"){ batman={ pv:40, max:40, objets:[], hasEvade:false, batarang:false }; }
  if(diff==="difficile"){ batman={ pv:30, max:30, objets:[], hasEvade:false, batarang:false }; }
  encounters=0; combatEnCours=false; currentRiddle=0;
  document.getElementById('log').innerHTML="";
  majStats();
  log(`🦇 Batman s'éveille dans Arkham ! Difficulté : ${diff}`);
  choixPorte();
}

// --- Choix de sortie ---
function choixPorte(){
  majScene("<p>Choisis une sortie :</p>");
  majActions(`
    <button onclick="rencontre('gauche')">🚪 Porte gauche</button>
    <button onclick="rencontre('droite')">🚪 Porte droite</button>
    <button onclick="rencontre('ascenseur')">⬆️ Ascenseur</button>
  `);
}

// --- Rencontre ---
function rencontre(dir){
  if(dir==="gauche"){ 
    let pv = (difficulty==="facile"?18:(difficulty==="difficile"?25:20));
    let dmg = (difficulty==="facile"?[1,4]:(difficulty==="difficile"?[3,6]:[2,5]));
    currentEnemy={nom:"Joker", pv:pv, max:pv, dmg:dmg, emoji:"🤡", type:"combat"}; 
  }
  else if(dir==="droite"){ 
    let pv = (difficulty==="facile"?20:(difficulty==="difficile"?30:25));
    let dmg = (difficulty==="facile"?[2,5]:(difficulty==="difficile"?[4,8]:[3,7]));
    currentEnemy={nom:"Bane", pv:pv, max:pv, dmg:dmg, emoji:"💪", type:"combat"}; 
  }
  else { currentEnemy={nom:"L'Homme-Mystère", type:"riddle"}; }
  
  if(currentEnemy.type==="combat"){ debutCombat(); }
  else { enigmeRiddler(); }
}

// --- Combat ---
function debutCombat(){
  combatEnCours=true;
  majScene(`<div class="avatar">${currentEnemy.emoji}</div><p>${currentEnemy.nom} vous fait face !</p>`);
  majActions(`
    <button onclick="attaque('rapide')">⚡ Attaque rapide (3-5 PV)</button>
    <button onclick="attaque('lourde')">💥 Attaque lourde (5-9 PV)</button>
    <button onclick="esquive()">🤸 Esquive</button>
  `);
  majStats();
  log(`💥 Combat contre ${currentEnemy.nom} !`);
}

function attaque(type){
  if(!combatEnCours) return;
  let dmg=0;
  if(type==="rapide"){ dmg=rnd(3,5); }
  if(type==="lourde"){ dmg=rnd(5,9); }
  // coup critique
  if(Math.random()<0.2){ dmg=Math.round(dmg*1.5); log("💥 Coup critique !"); }
  if(batman.batarang){ dmg=Math.round(dmg*1.5); batman.batarang=false; log("🛠️ Batarang utilisé : dégâts augmentés !"); }
  currentEnemy.pv-=dmg;
  log(`🦇 Batman inflige ${dmg} PV à ${currentEnemy.nom}.`);
  majStats();
  if(currentEnemy.pv<=0){ victoire(); return; }
  setTimeout(riposte,500);
}

function esquive(){
  log("🦇 Batman tente d'esquiver...");
  if(Math.random()<0.5){ log("❌ Échec ! L'ennemi attaque."); setTimeout(riposte,500); }
  else { log("✅ Esquive réussie !"); }
}

function riposte(){
  let dmg=rnd(currentEnemy.dmg[0], currentEnemy.dmg[1]);
  if(batman.hasEvade){ dmg=0; batman.hasEvade=false; log("🛡️ Esquive/Fumigène bloque l'attaque !"); }
  batman.pv-=dmg;
  log(`${currentEnemy.emoji} ${currentEnemy.nom} inflige ${dmg} PV à Batman !`);
  majStats();
  if(batman.pv<=0){ defaite(); return; }
}

// --- Objets ---
function useItem(i){
  let item=batman.objets[i];
  if(item==="Fumigène"){ batman.hasEvade=true; log("🛡️ Fumigène prêt à bloquer la prochaine attaque !"); }
  else if(item==="Batarang"){ batman.batarang=true; log("🛠️ Batarang prêt : prochaine attaque plus puissante !"); }
  else if(item==="Trousse"){ batman.pv=Math.min(batman.max, batman.pv+10); log("💉 Trousse utilisée : +10 PV"); }
  batman.objets.splice(i,1);
  majStats();
}

// --- Énigmes ---
function enigmeRiddler(){
  if(currentRiddle>=riddles.length){ victoire(); return; }
  let r=riddles[currentRiddle];
  majScene(`<div class="avatar">❓</div><p>L’Homme-Mystère : "${r.q}"</p>`);
  let html=r.answers.map((text,i)=>`<button onclick="reponseEnigme(${i})">${text}</button>`).join("");
  majActions(html);
}

function reponseEnigme(i){
  let r=riddles[currentRiddle];
  let penalty = (difficulty==="facile"?3:(difficulty==="difficile"?8:5));
  if(i===r.correct){ log("✅ Bonne réponse !"); }
  else { batman.pv-=penalty; log(`❌ Mauvaise réponse : -${penalty} PV`); majStats(); if(batman.pv<=0){ defaite(); return; } }
  currentRiddle++;
  enigmeRiddler();
}

// --- Fin combat / rencontre ---
function victoire(){
  combatEnCours=false;
  encounters++;
  if(currentEnemy && currentEnemy.type==="combat"){
    const items=["Fumigène","Batarang","Trousse"];
    const item=items[rnd(0,items.length-1)];
    batman.objets.push(item);
    log(`🎁 Vous obtenez un objet : ${item}`);
  }
  majStats();
  if(encounters>=2){ majScene("<p>🏆 Batman atteint le toit et s'échappe d'Arkham !</p>"); majActions(`<button onclick="menuDifficulte()">🔄 Rejouer</button>`); return; }
  majScene("<p>✅ Épreuve terminée ! Choisis la prochaine sortie.</p>");
  choixPorte();
}

function defaite(){
  combatEnCours=false;
  majScene("<p>☠️ Batman est vaincu et reste enfermé dans Arkham...</p>");
  majActions(`<button onclick="menuDifficulte()">🔄 Rejouer</button>`);
}

// --- Lancement ---
menuDifficulte();
</script>
</body>
</html>
