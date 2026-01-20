<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Max Bass Web Player</title>

<style>
body{
    margin:0;
    background:#030612;
    color:#fff;
    font-family:Arial,Helvetica,sans-serif
}
.app{max-width:420px;margin:auto;padding:16px}
.card{
    background:rgba(255,255,255,.06);
    border-radius:18px;
    padding:14px;
    margin-bottom:14px
}
button{
    width:100%;
    padding:14px;
    border:none;
    border-radius:14px;
    background:#00ffb0;
    color:#000;
    font-size:16px;
    font-weight:600
}
.controls{display:flex;gap:10px}
.controls button{width:33%}
label{font-size:12px;color:#aaa}
select,input{width:100%}
.small-btn{
    padding:10px;
    margin-top:8px;
    background:#ff6b6b;
    font-size:14px
}
</style>
</head>

<body>
<div class="app">

<div class="card" style="text-align:center">
    <h2>🎧 Max Bass Web Player</h2>
    <label>Máximo grave possível • Zero clipping</label>
</div>

<audio id="audio"></audio>

<div class="card">
    <label>Adicionar músicas ou pastas</label>
    <input type="file" id="files" webkitdirectory multiple>
    <button class="small-btn" style="background:#00c090" onclick="addFavorite()">⭐ Adicionar aos Favoritos</button>
</div>

<div class="card">
    <label>Playlist</label>
    <select id="playlist" size="6"></select>
    <button class="small-btn" onclick="removeFromPlaylist()">❌ Remover da Playlist</button>
</div>

<div class="card">
    <label>Favoritos</label>
    <select id="favorites" size="4"></select>
    <button class="small-btn" onclick="removeFavorite()">❌ Remover Favorito</button>
</div>

<div class="card controls">
    <button onclick="prev()">⏮</button>
    <button onclick="playPause()">▶ / ⏸</button>
    <button onclick="next()">⏭</button>
</div>

<div class="card">
    <label>Progresso da música</label>
    <input type="range" id="seek" min="0" max="100" value="0">
</div>

<div class="card">
    <label>Volume protegido</label>
    <input type="range" id="volume" min="0" max="1" step="0.01" value="0.75">
</div>

<div class="card">
    <label>Intensidade do Grave (Seguro)</label>
    <input type="range" id="bass" min="0" max="10" value="7">
</div>

</div>

<script>
const audio = document.getElementById("audio");
const files = document.getElementById("files");
const playlistUI = document.getElementById("playlist");
const favoritesUI = document.getElementById("favorites");
const bassCtrl = document.getElementById("bass");
const volumeCtrl = document.getElementById("volume");
const seek = document.getElementById("seek");

let ctx, src;
let headroom, lowCut, bassShelf, psychoBass;
let compA, compB, limiter, out;
let list=[], favs=[], idx=0, ready=false;

function initAudio(){
    if(ready) return;
    ctx = new (window.AudioContext||webkitAudioContext)();
    src = ctx.createMediaElementSource(audio);

    headroom = ctx.createGain(); headroom.gain.value = 0.2;
    lowCut = ctx.createBiquadFilter(); lowCut.type="highpass"; lowCut.frequency.value=28;
    bassShelf = ctx.createBiquadFilter(); bassShelf.type="lowshelf"; bassShelf.frequency.value=90;
    psychoBass = ctx.createBiquadFilter(); psychoBass.type="peaking"; psychoBass.frequency.value=150; psychoBass.Q.value=1.2;

    compA = ctx.createDynamicsCompressor(); compA.threshold.value=-30; compA.ratio.value=3.2;
    compB = ctx.createDynamicsCompressor(); compB.threshold.value=-16; compB.ratio.value=1.5;
    limiter = ctx.createDynamicsCompressor(); limiter.threshold.value=-6; limiter.ratio.value=100;

    out = ctx.createGain(); out.gain.value=0.85;

    src.connect(headroom)
       .connect(lowCut)
       .connect(compA)
       .connect(bassShelf)
       .connect(psychoBass)
       .connect(compB)
       .connect(limiter)
       .connect(out)
       .connect(ctx.destination);

    ready=true;
    updateBass();
}

function updateBass(){
    const b=+bassCtrl.value;
    bassShelf.gain.value=b*0.9;
    psychoBass.gain.value=b*0.7;
    out.gain.value=Math.max(0.6,0.9-b*0.03);
}

bassCtrl.oninput=updateBass;
volumeCtrl.oninput=e=>out.gain.value=Math.min(e.target.value,0.9);

files.onchange=()=>{
    [...files.files].filter(f=>f.type.startsWith("audio")).forEach(f=>{
        if(list.find(m=>m.name===f.name)) return;
        list.push({url:URL.createObjectURL(f),name:f.name});
        renderPlaylist();
    });
    if(!audio.src && list.length){
        idx=0; audio.src=list[0].url;
    }
    files.value="";
};

function renderPlaylist(){
    playlistUI.innerHTML="";
    list.forEach((m,i)=>{
        const o=document.createElement("option");
        o.textContent=m.name;
        o.value=i;
        playlistUI.appendChild(o);
    });
}

playlistUI.onchange=e=>{
    idx=e.target.value;
    audio.src=list[idx].url;
    playPause(true);
};

favoritesUI.onchange=e=>{
    audio.src=favs[e.target.value].url;
    playPause(true);
};

function removeFromPlaylist(){
    const i=playlistUI.selectedIndex;
    if(i<0) return;

    const removed=list.splice(i,1)[0];
    renderPlaylist();

    if(idx>=list.length) idx=list.length-1;
    if(list[idx]) audio.src=list[idx].url;
    else audio.pause();

    playlistUI.selectedIndex=idx;
}

function addFavorite(){
    if(!list[idx]) return;
    if(favs.find(f=>f.url===list[idx].url)) return;
    favs.push(list[idx]);
    const o=document.createElement("option");
    o.textContent=list[idx].name;
    o.value=favs.length-1;
    favoritesUI.appendChild(o);
}

function removeFavorite(){
    const i=favoritesUI.selectedIndex;
    if(i<0) return;
    favs.splice(i,1);
    favoritesUI.remove(i);
}

function playPause(force){
    initAudio(); ctx.resume();
    if(audio.paused||force) audio.play();
    else audio.pause();
}

function next(){
    if(!list.length) return;
    idx=(idx+1)%list.length;
    audio.src=list[idx].url;
    playPause(true);
    playlistUI.selectedIndex=idx;
}

function prev(){
    if(!list.length) return;
    idx=(idx-1+list.length)%list.length;
    audio.src=list[idx].url;
    playPause(true);
}

audio.ontimeupdate=()=>{
    seek.value=(audio.currentTime/audio.duration)*100||0;
};
seek.oninput=()=>{
    audio.currentTime=(seek.value/100)*audio.duration;
};
audio.onended=next;
</script>
</body>
</html>
