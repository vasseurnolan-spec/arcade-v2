# arcade-v2
<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover, user-scalable=no">
<title>Escadrille</title>
<style>
  :root { box-sizing: border-box; }
  html, body { height: 100%; margin: 0; background: #12293f; overflow: hidden; overscroll-behavior: none; }
  body { -webkit-user-select: none; user-select: none; -webkit-touch-callout: none; }
  #wrap { position: fixed; left: 0; right: 0; top: env(safe-area-inset-top, 0px); bottom: env(safe-area-inset-bottom, 0px); display: flex; justify-content: center; align-items: center; }
  canvas { display: block; touch-action: none; background: #7cc4ee; }
</style>
</head>
<body>
<div id="wrap"><canvas id="c"></canvas></div>
<script>
(() => {
  const wrap = document.getElementById('wrap'), cv = document.getElementById('c'), ctx = cv.getContext('2d');
  const W = 360; let H = 640, S = 1, rect;
  const FONT = '"Trebuchet MS", "Segoe UI", sans-serif';
  function resize() {
    const ch = wrap.clientHeight, cw = Math.min(wrap.clientWidth, ch * 0.62);
    const D = Math.min(window.devicePixelRatio || 1, 3);
    cv.style.width = cw + 'px'; cv.style.height = ch + 'px';
    cv.width = Math.round(cw * D); cv.height = Math.round(ch * D);
    S = cv.width / W; H = ch / cw * W; rect = cv.getBoundingClientRect();
  }
  addEventListener('resize', resize); resize();

  // ---------- données ----------
  const PUP = { P: ['#ffb020', 'P'], S: ['#2aa7e8', 'S'], H: ['#e8465f', '+'], B: ['#444c5c', 'B'], R: ['#ff6a2a', 'R'], X: ['#8a4fe0', '×2'], W: ['#22a06b', 'W'], T: ['#3fb8c9', 'T'], F: ['#d9a500', 'F'], M: ['#e8465f', 'M'], G: ['#7a86ff', 'G'], C: ['#e0a800', '◎'], Y: ['#ff7ab8', '♥'], K: ['#b8782a', '🎁'], '$': ['#ffd23f', '◎'] };
  const DROPS = 'PPSSHBRRXWTFMGC';
  const TIMERS = [['shield', 420, 'S'], ['rapid', 480, 'R'], ['x2', 600, 'X'], ['wing', 600, 'W'], ['slow', 360, 'T'], ['fort', 720, 'F'], ['mega', 480, 'M'], ['ghost', 240, 'G'], ['cbT', 1200, 'C']];
  const BOSSES = [
    { n: 'Le Faucheur', c: ['#7a1230', '#ff4d6d', '#ffd0d8'] },
    { n: 'La Perceuse', c: ['#b86a14', '#ffd23f', '#fff2c2'] },
    { n: 'La Mère', c: ['#2d6f8f', '#7ff3ff', '#d9fbff'] },
    { n: 'Iron Fortress', c: ['#5a6070', '#ff9a3d', '#e8eef5'] },
    { n: 'Storm', c: ['#1a2250', '#ffe45e', '#ffffff'] }
  ];
  const BORDER = [3, 4, 0, 1, 2];

  let state = 'hub', hi = 0, dailyMsg = '', backTo = 'menu', inRun = false, runF = 0, shopMax = 0;
  let sv = { coins: 0, owned: ['classic'], eq: 'classic' };
  try { hi = +localStorage.getItem('escadrille-hi') || 0; const r = JSON.parse(localStorage.getItem('arcade-save')); if (r && r.owned) sv = r; } catch (e) {}
  // ---------- GameModeContext (PLAY / TEST) ----------
  const GM = { mode: 'PLAY', allowRewards: true, allowProgression: true, allowDamage: true, allowEnemyDeath: true, allowWaveProgression: true, allowSaveChanges: true };
  const setMode = m => { const pl = m === 'PLAY'; Object.assign(GM, { mode: m, allowRewards: pl, allowProgression: pl, allowDamage: pl, allowEnemyDeath: pl, allowWaveProgression: pl, allowSaveChanges: pl }); };
  const isTest = () => GM.mode === 'TEST';
  let testSnap = null, testSk = 'classic', ult = 0, ultT = 0, uan = [];
  const save = () => { if (!GM.allowSaveChanges) return; try { localStorage.setItem('arcade-save', JSON.stringify(sv)); } catch (e) {} };
  const defSt = () => ({ games: 0, kills: 0, boss: 0, elites: 0, coins: 0, bestWave: 0, bestScore: 0, bestCombo: 0, time: 0, trans: 0, shots: 0, pd: 0, rush: 0, mg: 0, chest: 0 });
  sv.st = Object.assign(defSt(), sv.st); sv.ach = sv.ach || {}; sv.bs = sv.bs || {}; sv.lv = sv.lv || {}; sv.o = Object.assign({ q: 0, vib: 1, dist: 1, cam: 1, part: 0 }, sv.o);
  const MT = [['kills', 100, 150, 'Détruire 100 ennemis'], ['kills', 300, 350, 'Détruire 300 ennemis'], ['wave', 10, 200, 'Atteindre la vague 10'], ['wave', 15, 350, 'Atteindre la vague 15'], ['boss', 2, 300, 'Vaincre 2 boss'], ['combo', 10, 200, 'Faire un combo x10'], ['coins', 1000, 250, 'Gagner 1 000 ◎'], ['elite', 10, 250, 'Détruire 10 élites'], ['lvl', 3, 400, "Niveau 3 d'un skin supérieur"], ['pd', 20, 300, 'Faire 20 esquives parfaites'], ['mg', 3, 400, 'Réussir 3 mini-jeux'], ['rush', 1, 350, 'Terminer un Rush'], ['boss', 1, 300, 'Battre un boss'], ['plv', 7, 2000, 'Atteindre le niveau 7 (évolution)'], ['tests', 5, 300, 'Tester 5 skins'], ['chest', 3, 400, 'Trouver 3 coffres'], ['kills', 1000, 600, 'Détruire 1 000 ennemis']];
  const STK = { kills: 'kills', boss: 'boss', coins: 'coins', elite: 'elites', pd: 'pd', mg: 'mg', rush: 'rush', chest: 'chest' };
  const mprog = m => STK[m.k] ? sv.st[STK[m.k]] - m.b : (m.m || 0);
  function newMis() { const used = (sv.mis || []).map(m => m.t), pool = MT.map((t, i) => i).filter(i => !used.includes(i)), i = pool[(Math.random() * pool.length) | 0], t = MT[i]; return { t: i, k: t[0], g: t[1], rw: Math.round(t[2] * (1 + (sv.st.bestWave || 0) / 8)), n: t[3], b: sv.st[STK[t[0]]] || 0, m: 0 }; }
  if (!sv.mis || sv.mis.length !== 3) { sv.mis = []; for (let i = 0; i < 3; i++) sv.mis.push(newMis()); }
  { const today = new Date().toISOString().slice(0, 10);
    if (sv.day !== today) { const yd = new Date(Date.now() - 864e5).toISOString().slice(0, 10); sv.streak = sv.day === yd ? (sv.streak || 0) + 1 : 1; const dr = 50 + Math.min(sv.streak, 7) * 25; sv.coins += dr; sv.st.coins += dr; sv.day = today; dailyMsg = 'BONUS DU JOUR : +' + dr + ' ◎ (série ' + sv.streak + ')'; save(); } }
  const bkey = e => e.k === 3 ? 'b' + e.bi : e.mini ? 'm' + e.mf : e.el ? 'el' : ['sc', 'ch', 'po', 'xx', 'dr', 'cn', 'tg'][e.k];
  const BEST = [['sc', 'Éclaireur', 'Patrouille en haut et tire droit.', 1], ['ch', 'Chasseur', 'Suit le joueur et le vise.', 1], ['po', 'Gros porteur', 'Lourd, tire en éventail.', 4], ['dr', 'Drone', 'Plonge sur le joueur.', 5], ['el', 'Élite', 'Plus solide, aura orange.', 5], ['m0', 'Mini-boss Bélier', 'Charge annoncée par une zone jaune.', 8], ['m1', 'Mini-boss Essaim', 'Noyau protégé par 6 drones.', 13], ['m2', 'Mini-boss Tourelle', 'Spirale de tirs, vulnérable ouverte.', 18], ['b0', 'Boss Faucheur', 'Anneaux et éventails de balles.', 5], ['b1', 'Boss Perceuse', 'Charge avec avertissement.', 10], ['b2', 'Boss Mère', 'Drones et pluie annoncée.', 15]];
  BEST.push(['m3', 'Mini-boss Bouclier', "Bouclier tournant : attends l'ouverture.", 28], ['m4', 'Mini-boss Sniper', 'Ligne de visée, puis tir puissant.', 33], ['b3', 'Boss Iron Fortress', 'Détruis ses canons avant le noyau.', 5], ['b4', 'Boss Storm', 'Éclairs annoncés, charge rapide.', 10]);
  const SKLW = [1, 3, 6, 10, 15, 21, 28], SLN = ['ÉVEIL', 'CHARGE', 'ASCENSION', 'APOGÉE', 'FORME ULTIME', 'TRANSCENDANCE', 'APOTHÉOSE'];
  const PAGES = ['hub', 'prof', 'menu', 'shop', 'col', 'opt', 'ach', 'best', 'stat', 'chest', 'bonus'];
  const PCAP = () => [260, 150, 70][sv.o.q];
  const SKR = ['#9aa7b5', '#3fa0e8', '#a055e0', '#ffb020', '#ff3d7f', '#19e3ff', '#fff1b8', '#ff5ac8', '#ffffff'], SKN = ['COMMUN', 'RARE', 'ÉPIQUE', 'LÉGENDAIRE', 'MYTHIQUE', 'COSMIQUE', 'DIVIN', 'TRANSCENDANT', '🌌 DIVINE ABSOLUE'];
  const BM = { fire: 'flame', star: 'star', void: 'void', gold: 'gold', spark: 'geo', ghost: 'ghost', rb: 'rb' };
  const mk = (id, n, r, price, shape, body, wing, glow, flame, fx, cos) => ({ id, n, r, price, shape, body, wing, glow, flame, fx, cos, bul: cos ? 'cosmic' : BM[fx], rb: fx === 'rb', void: fx === 'void' });
  const FXC = { fire: ['#ff7a3d', '#ffd966', '#ff3d3d'], star: ['#fff', '#bde0ff', '#ffd966'], spark: ['#9be7ff', '#fff', '#7a5cff'], void: ['#b57cff', '#3a1060'], ghost: ['#bde0ff', '#fff'], demon: ['#ff2a3a', '#3a0010', '#ff7a3d'], angel: ['#ffffff', '#fff1b8', '#ffd966'], gold: ['#fff1b8', '#ffd966'], elec: ['#ffe14a', '#fff', '#ffb020'], sam: ['#ff5a4a', '#ffd966', '#fff'] };
  const SKINS = [
    mk('classic', 'Classique', 0, 0, '', '#f4f7fb', '#2f6fc4', '#2a4a7a', '#ffb347'),
    mk('a01', 'A-01', 0, 100, '', '#e9edf2', '#5a7fa8', '#2a4a7a', '#ffb347'),
    mk('scout', 'Scout', 0, 150, 'arrow', '#ffe3c2', '#ff8a3d', '#7a3a12', '#ffd36b'),
    mk('coral', 'Corail', 0, 200, '', '#fff1ec', '#ff6b57', '#7a2a22', '#ffd36b'),
    mk('forest', 'Forêt', 0, 250, '', '#e8f5e0', '#2e9b55', '#1c4a2a', '#9dff6a'),
    mk('arrow', 'Arrow', 0, 300, 'arrow', '#d8fff6', '#1fa08a', '#0f4a40', '#9dffe0'),
    mk('night', 'Nuit', 1, 500, '', '#2b3350', '#7a5cff', '#bde0ff', '#9a7bff'),
    mk('inter', 'Interceptor', 1, 700, 'inter', '#dfe8f2', '#e8465f', '#1c2a4a', '#ff9a5a'),
    mk('falcon', 'Falcon', 1, 900, 'falcon', '#8a6a4a', '#d9a566', '#fff1d0', '#ffd966'),
    mk('stealth', 'Stealth', 1, 1100, 'stealth', '#3a4150', '#5d6b82', '#9be7ff', '#7fe3ff'),
    mk('ranger', 'Ranger', 1, 1400, 'ranger', '#e8f0e0', '#6b8f3a', '#26401a', '#c9ff7a'),
    mk('gold', 'Or', 2, 2000, '', '#fff1b8', '#e0a800', '#7a5a00', '#ffffff', 'gold'),
    mk('cyber', 'Cyber', 2, 2500, 'inter', '#1d2a3a', '#19e3ff', '#e0ffff', '#19e3ff', 'spark'),
    mk('mecha', 'Mecha', 2, 3200, 'mecha', '#7a8494', '#e8a31f', '#2a3040', '#ff9a3d'),
    mk('inferno', 'Inferno', 2, 4000, 'falcon', '#3a1a14', '#ff5a1f', '#ffe0a0', '#ffb020', 'fire'),
    mk('phantom', 'Phantom', 2, 5000, 'stealth', '#2a2f45', '#8a9cff', '#e0e8ff', '#bde0ff', 'ghost'),
    mk('rainbow', 'Arc-en-ciel', 3, 7500, '', '#ffffff', '#ff55aa', '#2a4a7a', '#fff', 'rb'),
    mk('dragon', 'Dragon', 3, 9000, 'dragon', '#2e7a4a', '#1c4a2a', '#ffe45e', '#ff7a3d', 'fire'),
    mk('phoenix', 'Phoenix', 3, 11000, 'dragon', '#ffcf5a', '#ff4a1f', '#fff', '#ffb020', 'fire'),
    mk('galaxy', 'Galaxy', 3, 13000, 'falcon', '#1a1e5a', '#7a5cff', '#fff', '#bde0ff', 'star'),
    mk('titan', 'Titan', 3, 15000, 'titan', '#6a7a8f', '#e8c04a', '#2a3040', '#ffd966'),
    { ...mk('pikachu', 'Pikachu', 4, 40000, 'falcon', '#ffe14a', '#f2b705', '#fff6a8', '#ffe14a', 'elec'), bul: 'elec' },
    mk('void', 'Void', 4, 20000, 'stealth', '#14091f', '#8a3cff', '#e0b3ff', '#b57cff', 'void'),
    mk('celestial', 'Celestial', 4, 28000, 'falcon', '#ffffff', '#ffd966', '#fff6c2', '#fff', 'star'),
    mk('nebula', 'Nebula', 4, 36000, 'titan', '#2a1a5a', '#ff5ac8', '#bde0ff', '#b57cff', 'star'),
    mk('omega', 'Omega', 4, 50000, 'mecha', '#1a1a24', '#ff3d7f', '#ffe0f0', '#ff3d7f', 'spark'),
    { ...mk('samurai', 'Samouraï', 5, 85000, 'arrow', '#2a2a33', '#8a1c1c', '#ffd966', '#ff5a4a', 'sam'), bul: 'slash' },
    mk('quasar', 'Quasar', 5, 60000, 'inter', '#0a1a2a', '#19e3ff', '#ffffff', '#19e3ff', 'spark'),
    mk('supernova', 'Supernova', 5, 90000, 'titan', '#3a1a5a', '#ff9a3d', '#fff1b8', '#ffd966', 'fire'),
    mk('seraph', 'Seraph', 6, 160000, 'dragon', '#ffffff', '#fff1b8', '#ffd966', '#fff', 'gold'),
    mk('solaris', 'Solaris', 6, 250000, 'falcon', '#ffb020', '#ff5a1f', '#fff6c2', '#ffd966', 'fire'),
    mk('infinity', 'Infinity', 7, 500000, 'stealth', '#0a0a18', '#b57cff', '#ffffff', '#b57cff', 'star'),
    mk('zero', 'Zero', 7, 750000, 'arrow', '#0a0a0a', '#f0f0f0', '#ffffff', '#ffffff', 'ghost'),
    { ...mk('prism', 'Prisme', 1, 0, 'inter', '#e8f8ff', '#7fe3ff', '#ffffff', '#9be7ff', 'spark'), fp: 10 },
    { ...mk('aurora', 'Aurore', 2, 0, 'falcon', '#d8ffe8', '#ff7ac8', '#ffffff', '#7dffb0', 'star'), fp: 25 },
    { ...mk('chrono', 'Chrono', 3, 0, 'titan', '#2a3a6a', '#ffd966', '#bde0ff', '#ffd966', 'gold'), fp: 60 },
    mk('blackhole', 'Black Hole', 5, 55000, 'stealth', '#1a0f2a', '#6a2fb8', '#c9a0ff', '#8a3cff', 'void', { core: '#05010a', h: '#8a3cff', r1: '#c9a0ff', r2: '#6a2fb8', bh: 1, nm: ['SINGULARITÉ', 'ACCRÉTION', 'HORIZON', 'EFFONDREMENT', 'ABYSSE'] }),
    mk('dquasar', 'Dark Quasar', 5, 70000, 'inter', '#0a1020', '#19e3ff', '#e0ffff', '#19e3ff', 'spark', { core: '#02060f', h: '#19e3ff', r1: '#19e3ff', r2: '#7a5cff' }),
    mk('ncore', 'Nebula Core', 5, 90000, 'titan', '#2a1a5a', '#ff5ac8', '#bde0ff', '#ffcf5a', 'star', { core: '#2a1a5a', h: '#ff5ac8', r1: '#ffcf5a', r2: '#5ac8ff' }),
    mk('singod', 'Singularity God', 6, 180000, 'dragon', '#1a1020', '#ffd966', '#fff1b8', '#ffd966', 'gold', { core: '#05010a', h: '#ffd966', r1: '#ffd966', r2: '#b57cff', bh: 1 }),
    mk('eclipse', 'Celestial Eclipse', 6, 220000, 'falcon', '#e8e0c8', '#2a2a44', '#fff1b8', '#fff1b8', 'gold', { core: '#0a0a14', h: '#fff1b8', r1: '#fff1b8', r2: '#3a3a5a' }),
    mk('seravoid', 'Seraph of the Void', 6, 280000, 'dragon', '#ffffff', '#8a3cff', '#e0b3ff', '#fff', 'void', { core: '#1a0a2a', h: '#ffffff', r1: '#ffffff', r2: '#8a3cff' }),
    mk('omegabh', 'Omega Black Hole', 7, 550000, 'stealth', '#08040f', '#ff3d7f', '#ffe0f0', '#ff3d7f', 'void', { core: '#000', h: '#b57cff', r1: '#ff3d7f', r2: '#8a3cff', bh: 1, nm: ['SINGULARITÉ', 'ACCRÉTION', 'HORIZON', 'EFFONDREMENT', 'ABYSSE'] }),
    mk('infsing', 'Infinity Singularity', 7, 700000, 'arrow', '#05050f', '#9be7ff', '#fff', '#b57cff', 'star', { core: '#000', h: '#9be7ff', r1: '#fff', r2: '#b57cff' }),
    mk('collapse', 'Eternal Collapse', 7, 1000000, 'titan', '#120a05', '#ffb020', '#fff6c2', '#ffd966', 'fire', { core: '#000', h: '#ffb020', r1: '#ffd966', r2: '#19e3ff', bh: 1 }),
    mk('uchiwa', 'Uchiwa', 5, 250000, 'stealth', '#0b0b10', '#b3122a', '#ff3040', '#ff2a3a', 'demon', { core: '#9a0f22', h: '#ff2a3a', r1: '#ff4a5a', r2: '#1a0a0a', eye: 'u', nm: ['ŒIL SOMBRE', 'PREMIER TOMOE', 'DEUX TOMOE', 'TROIS TOMOE', 'ŒIL DU NÉANT'] }),
    mk('hyuga', 'Hyuga', 5, 250000, 'arrow', '#ffffff', '#bfe6ff', '#eaf6ff', '#9be7ff', 'ghost', { core: '#e8f4ff', h: '#bfe6ff', r1: '#ffffff', r2: '#7fc8ff', eye: 'h', nm: ['ŒIL CLAIR', 'ANNEAUX', 'AURA BLEUE', 'CERCLES D\'ÉNERGIE', 'VISION PANORAMIQUE'] }),
    mk('senju', 'Senju', 4, 120000, 'angel', '#e8f5d8', '#2f9d4a', '#9dff9d', '#7dff9a', 'spark', { core: '#3a7d2e', h: '#5aff8a', r1: '#9dff9d', r2: '#8a5a2b', eye: 's', nm: ['PARTICULES VERTES', 'FEUILLES', 'MOTIFS DE BOIS', 'AURA NATURELLE', 'RACINES LUMINEUSES'] }),
    mk('cosmicgod', 'Cosmic God', 8, 2000000, 'titan', '#0a0620', '#7a4cff', '#ffffff', '#19e3ff', 'star', { core: '#000', h: '#7a4cff', r1: '#19e3ff', r2: '#ff5ac8', bh: 1, nm: ['ÉVEIL COSMIQUE', 'ORBITES', 'DISTORSION', 'ANNEAUX STELLAIRES', 'COSMIC COLLAPSE'] }),
    mk('celemp', 'Celestial Emperor', 8, 2200000, 'angel', '#fff8dc', '#ffd23f', '#ffffff', '#ffd966', 'gold', { core: '#fff1b8', h: '#ffd966', r1: '#fff', r2: '#ffb020', nm: ['COURONNE', 'AILES D\'ÉNERGIE', 'TRAÎNÉE DORÉE', 'HALO ROYAL', 'CELESTIAL JUDGMENT'] }),
    mk('voidlord', 'Void Lord', 8, 2400000, 'demon', '#050508', '#3a0a60', '#b57cff', '#7a2cff', 'void', { core: '#000', h: '#8a3cff', r1: '#b57cff', r2: '#ff3d7f', bh: 1, nm: ['FISSURES', 'DISTORSION', 'FAILLE', 'ABÎME', 'VOID RIFT'] }),
    mk('solarphx', 'Solar Phoenix', 8, 2600000, 'angel', '#fff2c2', '#ff6a1a', '#ffd23f', '#ff9a3d', 'fire', { core: '#ff6a1a', h: '#ffb020', r1: '#ffd23f', r2: '#ff3d3d', nm: ['BRAISE', 'AILES DE FEU', 'TRAÎNÉE SOLAIRE', 'COURONNE DE FLAMMES', 'PHOENIX REBIRTH'] }),
    mk('dragoncel', 'Dragon Céleste', 8, 2800000, 'titan', '#12324a', '#19e3ff', '#ffffff', '#7fe3ff', 'fire', { core: '#19e3ff', h: '#19e3ff', r1: '#ffd966', r2: '#19e3ff', nm: ['ÉCAILLES', 'CORNES', 'AILES', 'QUEUE D\'ÉNERGIE', 'DRAGON NOVA'] }),
    mk('spiritsam', 'Spirit Samurai', 8, 2800000, 'arrow', '#1a1030', '#7fe3ff', '#ffffff', '#9be7ff', 'ghost', { core: '#fff', h: '#7fe3ff', r1: '#fff', r2: '#7fe3ff', nm: ['ARMURE', 'LAME', 'AURA', 'SPECTRE', 'SPIRIT SLASH'] }),
    { ...mk('demonking', 'Demon King', 7, 1200000, 'demon', '#14161e', '#5a0a18', '#ff3d4a', '#7a0a1a', 'demon'), dk: 'demon', bul: 'demon' },
    { ...mk('archangel', 'Archangel', 7, 1000000, 'angel', '#ffffff', '#ffd966', '#fff1b8', '#ffe9a0', 'angel'), dk: 'angel', bul: 'arrow' }
  ];

  // ===================== ULTIMATE UPDATE : rareté EXTRÊME ULTIME + 18 nouveaux skins =====================
  SKR.push('#ff2bd6'); SKN.push('🌌 EXTRÊME ULTIME');
  const NX = (id, n, r, price, shape, body, wing, glow, flame, fx, bul, cos) => ({ ...mk(id, n, r, price, shape, body, wing, glow, flame, fx, cos), bul, ext: id });
  SKINS.push(
    NX('infcel', 'Infernal Celestial', 3, 16000, 'angel', '#fff0d0', '#ff5a1f', '#fff6c2', '#ffb020', 'fire', 'flame', { core: '#ffb020', h: '#ff7a3d', r1: '#ffe9a0', r2: '#ff3d3d', nm: ['BRAISE CÉLESTE', 'AILES ARDENTES', 'HALO DE FEU', 'SOLEIL NAISSANT', 'ZÉNITH INFERNAL'] }),
    NX('moonreaper', 'Moon Reaper', 3, 18000, 'stealth', '#d8e0f0', '#3a4a8a', '#ffffff', '#bde0ff', 'ghost', 'slash', { core: '#e8f0ff', h: '#9bb8ff', r1: '#e8f0ff', r2: '#6a7ad8', nm: ['CROISSANT', 'LAME PÂLE', 'ÉCLIPSE PARTIELLE', 'FAUCHEUSE', 'NOUVELLE LUNE'] }),
    NX('starforged', 'Starforged', 6, 300000, 'titan', '#2a2040', '#ffb84a', '#ffffff', '#ffd966', 'star', 'star', { core: '#ffb84a', h: '#ffd966', r1: '#fff1b8', r2: '#ff7a3d', nm: ['ÉTINCELLE', 'ENCLUME STELLAIRE', 'ALLIAGE ASTRAL', 'CŒUR FORGÉ', 'ÉTOILE FORGÉE'] }),
    NX('thundersov', 'Thunder Sovereign', 6, 340000, 'falcon', '#fff8c8', '#2a6aff', '#ffffff', '#ffe14a', 'elec', 'elec', { core: '#ffffff', h: '#ffe14a', r1: '#ffe14a', r2: '#4aa3ff', nm: ['CHARGE', 'ARC ROYAL', 'COURONNE D\'ÉCLAIRS', 'TRÔNE DE TEMPÊTE', 'SOUVERAIN ORAGE'] }),
    NX('crystalnova', 'Crystal Nova', 6, 380000, 'inter', '#e8fbff', '#7ad8ff', '#ffffff', '#bdf0ff', 'spark', 'geo', { core: '#e8fbff', h: '#7ad8ff', r1: '#ffffff', r2: '#c9a0ff', nm: ['ÉCLAT', 'FACETTES', 'PRISME VIVANT', 'GÉODE STELLAIRE', 'NOVA CRISTALLINE'] }),
    NX('thunderabs', 'Thunder Absolute', 9, 3500000, 'falcon', '#e8f6ff', '#2a7aff', '#ffffff', '#7fd0ff', 'elec', 'elec', { core: '#ffffff', h: '#4aa3ff', r1: '#9be7ff', r2: '#ffffff', nm: ['NOYAU ÉLECTRIQUE', 'ARCS PERMANENTS', 'AILES DE FOUDRE', 'TEMPÊTE VIVANTE', 'TONNERRE ABSOLU'] }),
    NX('infeternal', 'Inferno Eternal', 9, 4000000, 'falcon', '#fff6e0', '#ff6a1a', '#ffffff', '#ffb020', 'fire', 'flame', { core: '#ffffff', h: '#ffb020', r1: '#ffd23f', r2: '#ff3d3d', nm: ['NOYAU INCANDESCENT', 'AILES DE FEU', 'TRAÎNÉE SOLAIRE', 'COURONNE ARDENTE', 'SOLEIL ÉTERNEL'] }),
    NX('chronoabs', 'Chrono Absolute', 9, 4500000, 'titan', '#1a2a4a', '#ffd966', '#bde0ff', '#ffe9a0', 'gold', 'gold', { core: '#bde0ff', h: '#ffd966', r1: '#ffe9a0', r2: '#7fc8ff', nm: ['SECONDE', 'ANNEAUX TEMPORELS', 'DISTORSION', 'HORLOGE COSMIQUE', 'FIN DES TEMPS'] }),
    NX('seraphabs', 'Seraph Absolute', 9, 5000000, 'angel', '#ffffff', '#fff1b8', '#ffe9a0', '#ffffff', 'angel', 'arrow', { core: '#ffffff', h: '#ffe9a0', r1: '#ffffff', r2: '#ffd966', nm: ['PREMIÈRE AILE', 'TROIS PAIRES', 'HALO GÉANT', 'ARMURE DE LUMIÈRE', 'SÉRAPHIN SUPRÊME'] }),
    NX('celabs', 'Celestial Absolute', 9, 5500000, 'angel', '#fffbe8', '#ffe9a0', '#ffffff', '#ffd966', 'gold', 'gold', { core: '#ffffff', h: '#ffd966', r1: '#ffffff', r2: '#ffd23f', nm: ['AUBE', 'COURONNE', 'HALOS', 'SYMBOLES CÉLESTES', 'ABSOLU CÉLESTE'] }),
    NX('evoid', 'Eternal Void', 9, 6000000, 'stealth', '#050308', '#2a0a4a', '#7a2cff', '#8a3cff', 'void', 'void', { core: '#000000', h: '#5a1cc0', r1: '#8a3cff', r2: '#2a0a4a', nm: ['OMBRE', 'FISSURES', 'DÉBRIS', 'TROU NOIR', 'VIDE ÉTERNEL'] }),
    NX('omnidrg', 'Omnidragon', 9, 6500000, 'dragon', '#3a0a10', '#ff3d2a', '#ffd23f', '#ff7a1a', 'fire', 'flame', { core: '#ffd23f', h: '#ff3d2a', r1: '#ffd23f', r2: '#ff3d2a', nm: ['ÉCAILLES', 'CORNES', 'AILES DE DRAGON', 'QUEUE D\'ÉNERGIE', 'OMNIDRAGON'] }),
    NX('dragonvoid', 'Dragon Void', 9, 7000000, 'dragon', '#0a0414', '#6a1cc0', '#ff3d7f', '#b57cff', 'void', 'void', { core: '#000000', h: '#6a1cc0', r1: '#ff3d7f', r2: '#b57cff', nm: ['ŒUF DU NÉANT', 'AILES D\'OMBRE', 'GUEULE SANS FOND', 'SOUFFLE NOIR', 'DRAGON DU NÉANT'] }),
    NX('abscosmos', 'Absolute Cosmos', 9, 8000000, 'titan', '#0a0b14', '#c9d3e6', '#ffffff', '#7ab8ff', 'star', 'star', { core: '#05060f', h: '#7a4cff', r1: '#9be7ff', r2: '#ff9ae6', nm: ['POUSSIÈRE D\'ÉTOILES', 'PLANÈTES', 'MINI-GALAXIES', 'ANNEAU COSMIQUE', 'COSMOS ABSOLU'] }),
    NX('astralemp', 'Astral Emperor', 9, 9000000, 'angel', '#1a1040', '#ffd966', '#ffffff', '#9be7ff', 'star', 'star', { core: '#ffffff', h: '#ffd966', r1: '#ffd966', r2: '#9be7ff', nm: ['CONSTELLATION', 'CAPE ASTRALE', 'COURONNE D\'ÉTOILES', 'EMPIRE STELLAIRE', 'MILLE ÉTOILES'] }),
    NX('dimzero', 'Dimension Zero', 9, 10000000, 'arrow', '#050505', '#e8e8ff', '#ffffff', '#ffffff', 'ghost', 'ghost', { core: '#000000', h: '#ffffff', r1: '#ffffff', r2: '#7a7aff', nm: ['ZÉRO', 'GLITCH', 'ANNEAU VIDE', 'DIMENSION NULLE', 'RUPTURE ZÉRO'] }),
    NX('dimbreaker', 'Dimension Breaker', 9, 11000000, 'stealth', '#10101a', '#ff3dd0', '#19e3ff', '#b57cff', 'spark', 'geo', { core: '#05050a', h: '#19e3ff', r1: '#ff3dd0', r2: '#19e3ff', nm: ['FISSURE', 'FRAGMENTS', 'PORTAIL', 'DÉCHIREMENT', 'BRISEUR DE MONDES'] }),
    NX('emperor', 'Emperor of Everything', 9, 12000000, 'titan', '#2a1a4a', '#ffd23f', '#ffffff', '#ffd966', 'gold', 'gold', { core: '#ffffff', h: '#ffd23f', r1: '#ffd23f', r2: '#b57cff', nm: ['SCEPTRE', 'ARMURE ROYALE', 'ANNEAUX IMPÉRIAUX', 'FRAGMENTS DE GALAXIES', 'EMPEREUR DE TOUT'] })
  );

  const skin = () => SKINS.find(k => k.id === (isTest() ? testSk : sv.eq)) || SKINS[0];
  const LVLW = [1, 3, 6, 9, 13, 18, 25], LVN = ['Standard', 'Ailes renforcées', 'Canons jumelés', 'Réacteur turbo', 'Légende', 'Prototype', 'Titan'];
  const minP = () => [0, 1, 1, 2, 2, 3, 4, 4][p.lvl];
  const TIERS = [1, 4, 7, 10, 13, 16, 21, 26, 31, 41];
  const tier = w => w >= 41 ? 10 + ((w - 41) / 10 | 0) : TIERS.filter(x => w >= x).length;
  const isMini = w => w >= 8 && w % 5 === 3;
  const CT = [3, 5, 8, 11, 15, 19, 24, 29, 34, 40], tcoin = t => t <= 10 ? CT[t - 1] : 40 + (t - 10) * 8;
  const waveBonus = w => Math.round(6 + 4.2 * w + .11 * w * w);
  const TC = ['#fff', '#9be7ff', '#7dff9a', '#ffe45e', '#ffa14a', '#ff6a6a', '#ff4df0', '#b57cff', '#7fe3ff', '#ff3d3d'];
  function coinOf(e) {
    const t = e.tr || 1;
    if (e.k >= 5) return e.k === 5 ? 10 : 0;
    if (e.k === 3) return Math.round(150 + wave * 15);
    if (e.mini) return Math.round(60 + tcoin(t) * 3);
    return Math.max(1, Math.round(tcoin(t) * [1, 2, 5, 0, .5][e.k] * (e.el ? 4 : e.ch ? 9 : e.cor ? 14 : e.beh ? 2 : 1) * (p.fort > 0 ? 2 : 1) * (e.pspec ? 3 : 1) * (e.el ? p.eliteB : 1)));
  }
  const cm = () => Math.min(20, 1 + (combo / 5 | 0));
  const MN = ['BÉLIER', 'ESSAIM', 'TOURELLE', 'BOUCLIER', 'SNIPER'], EVN = ['Chasseur de base', 'Chasseur renforcé', 'Chasseur énergétique', 'Chasseur astral', 'Chasseur Omega'];
  const pdps = () => p.dmg * p.rate * (p.power + p.multi * .7 + ((p.wing > 0 || p.drone) ? 1 : 0)) * (1 + p.crit) * (p.mega > 0 ? 2 : 1);
  const pf = () => Math.min(3.6, 1 + Math.log2(Math.max(1, pdps())) * .55);
  function capDmg(e, d) {
    if (protectd(e)) return 0;
    const fr = e.k === 3 ? .035 : e.mini ? .06 : 0; if (!fr) return d;
    if (time - (e.dw0 || 0) > 60) { e.dw0 = time; e.dw = 0; }
    const g = Math.max(0, Math.min(d, e.mhp * fr * (e.stun > 0 ? 3 : e.vul ? 2.2 : 1) - e.dw)); e.dw += g; return g;
  }
  const DNG = w => w <= 5 ? 0 : w <= 10 ? 1 : w <= 20 ? 2 : w <= 30 ? 3 : 4, DN = ['FAIBLE', 'DANGER', 'ÉLEVÉ', 'CRITIQUE', 'EXTRÊME'], DC = ['#7dff9a', '#ffe45e', '#ffa14a', '#ff5a5a', '#c27cff'];
  const ZONES = [[1, '#4aa3e0', '#bfe6fa'], [11, '#ff8a5c', '#ffd29a'], [21, '#0b1030', '#34407a'], [31, '#101a45', '#4a6aa8'], [41, '#05030f', '#241a50'], [51, '#0a0220', '#3a1060']];
  const hx = h => [1, 3, 5].map(i => parseInt(h.substr(i, 2), 16));
  const mix = (a, b, t) => { const A = hx(a), B = hx(b); return 'rgb(' + A.map((v, i) => Math.round(v + (B[i] - v) * t)).join(',') + ')'; };
  function skyCols(w) {
    let i = 0; while (i + 1 < ZONES.length && w >= ZONES[i + 1][0]) i++;
    const z = ZONES[i], n = ZONES[i + 1], t = n ? Math.max(0, (w - (n[0] - 3)) / 3) : 0;
    return t > 0 ? [mix(z[1], n[1], t), mix(z[2], n[2], t)] : [z[1], z[2]];
  }
  const STARS = Array.from({ length: 50 }, () => ({ x: Math.random() * 360, y: Math.random() * 700, s: Math.random() * 1.5 + .5 }));
  const AM = (() => {
    let ac, mg, sg, nb, nextT = 0, step = 0, want = null, cur = null, sw = 0; const st = Object.assign({ m: .5, s: .7 }, sv.opt || {}); sv.opt = st;
    const TH = {
      menu: { dt: .5, sc: [0, 3, 5, 7, 10], root: 196, lead: .035, drums: 0 }, w1: { dt: .32, sc: [0, 3, 5, 7, 10, 12, 15], root: 220, lead: .05, drums: 0 },
      w2: { dt: .27, sc: [0, 3, 5, 7, 10, 12, 15], root: 233, lead: .055, drums: 1 }, w3: { dt: .22, sc: [0, 2, 3, 5, 7, 8, 10, 12], root: 262, lead: .06, drums: 2 },
      mini: { dt: .2, sc: [0, 1, 5, 7, 8, 12], root: 247, lead: .06, drums: 2 }, boss: { dt: .17, sc: [0, 1, 3, 6, 7, 8, 12], root: 165, lead: .065, drums: 3 }, final: { dt: .15, sc: [0, 1, 4, 6, 7, 10, 12], root: 147, lead: .07, drums: 3 } };
    const tone = (fr, d, type, v, sl, w = 0, bus = sg) => { if (!ac) return; const t = ac.currentTime + w, o = ac.createOscillator(), g = ac.createGain(); o.type = type; o.frequency.setValueAtTime(fr, t); if (sl) o.frequency.exponentialRampToValueAtTime(Math.max(20, sl), t + d); g.gain.setValueAtTime(v, t); g.gain.exponentialRampToValueAtTime(.0001, t + d); o.connect(g); g.connect(bus); o.start(t); o.stop(t + d + .02); };
    const noise = (d, v) => { if (!ac) return; const t = ac.currentTime, s2 = ac.createBufferSource(), g = ac.createGain(), fl = ac.createBiquadFilter(); s2.buffer = nb; fl.type = 'lowpass'; fl.frequency.setValueAtTime(1800, t); fl.frequency.exponentialRampToValueAtTime(100, t + d); g.gain.setValueAtTime(v, t); g.gain.exponentialRampToValueAtTime(.0001, t + d); s2.connect(fl); fl.connect(g); g.connect(sg); s2.start(t); s2.stop(t + d); };
    const apply = () => { if (ac) { mg.gain.value = st.m * .3; sg.gain.value = st.s * .6; } };
    const arp = (fs, d, type, v, gap) => fs.forEach((fr, i) => tone(fr, d, type, v, 0, i * gap));
    const SFX = { shoot: () => tone(740, .05, 'square', .03, 400), hit: () => tone(200, .12, 'sawtooth', .08, 80), pop: () => noise(.08, .08), boom: () => noise(.35, .3),
      coin: () => arp([988, 1319], .1, 'sine', .08, .06), up: () => arp([523, 659, 784], .14, 'triangle', .1, .08), level: () => arp([392, 523, 659, 784, 1047], .16, 'triangle', .12, .07),
      boss: () => { tone(90, .9, 'sawtooth', .15, 45); noise(.6, .15); }, warn: () => arp([520, 520], .1, 'square', .06, .15), win: () => arp([523, 659, 784, 1047], .2, 'sine', .1, .09),
      buy: () => arp([660, 880, 1320], .12, 'triangle', .1, .06), equip: () => tone(880, .1, 'triangle', .08), rare: () => arp([784, 988, 1319, 1568], .25, 'sine', .12, .1) };
    return { st, apply,
      init() { if (!ac) { try { ac = new (window.AudioContext || window.webkitAudioContext)(); mg = ac.createGain(); sg = ac.createGain(); mg.connect(ac.destination); sg.connect(ac.destination); nb = ac.createBuffer(1, ac.sampleRate, ac.sampleRate); const d = nb.getChannelData(0); for (let i = 0; i < d.length; i++) d[i] = Math.random() * 2 - 1; apply(); nextT = ac.currentTime; } catch (e) { ac = null; } } if (ac && ac.state === 'suspended') ac.resume(); },
      sfx(n) { if (ac && SFX[n]) SFX[n](); },
      tick(th) {
        if (!ac || st.m <= 0) return; const now = ac.currentTime;
        if (th !== want) { want = th; mg.gain.cancelScheduledValues(now); mg.gain.setTargetAtTime(.0001, now, .12); sw = now + .5; }
        if (want !== cur && now >= sw) { cur = want; nextT = now + .05; step = 0; mg.gain.setTargetAtTime(st.m * .3, now, .3); }
        if (!cur) return; if (nextT < now - 1) nextT = now;
        const T = TH[cur];
        while (nextT < now + .25) {
          const s2 = step++, base = s2 % 32 < 16 ? 0 : (s2 % 32 < 24 ? -2 : 3), w = nextT - now, fr = n => T.root * Math.pow(2, (base + n) / 12);
          if (s2 % 8 === 0) [0, 7, 12].forEach(n => tone(fr(n) * .5, T.dt * 8, 'sine', .07, 0, w, mg));
          if (s2 % 2 === 0 || T.drums) tone(fr(T.sc[(s2 * 3 + (s2 >> 3)) % T.sc.length]), T.dt * 1.6, T.drums > 1 ? 'sawtooth' : 'triangle', T.lead, 0, w, mg);
          if (T.drums >= 1 && s2 % (T.drums === 3 ? 2 : 4) === 0) tone(90, .12, 'sine', .12, 40, w, mg);
          if (T.drums >= 1 && s2 % 2 === 1) tone(7000, .03, 'square', .012, 0, w, mg);
          if (T.drums >= 2 && s2 % 8 === 4) tone(190, .1, 'sawtooth', .07, 90, w, mg);
          if (T.drums === 3 && s2 % 4 === 2) tone(fr(0) * .25, T.dt * 1.8, 'sawtooth', .08, 0, w, mg);
          nextT += T.dt;
        }
      }
    };
  })();
  const earn = (n, raw) => { if (!GM.allowRewards) return; if (!raw && p && p.cg) n = Math.round(n * coinMul()); sv.coins += n; sv.st.coins += n; if (p) p.earned = (p.earned || 0) + n; save(); };
  let shopMsg = '', shopT = 0;
  let p, bullets, ebullets, enemies, pups, parts, floats = [], clouds, score, time, shake, target, keys = {};
  let wave, wq, wt, gap, banner, bombT, bossName, upT = 0, upName = '', wTot = 1, clr = 0, msg = null, msgT = 0, shopY = 0, sd = null, combo = 0, comboT = 0, rings = [], warns = [], sfxN = 0, strikes = [], mg = null, freeze = 0, hubGo = 0, shotFx = 0, shopR = 0, shopSel = null, ctier = 0, lastMg = '', mgRetry = 0, equipFx = 0, equipSk = null, shopF = 0, shopHeads = [], shopTotal = 0;


  // ===================== ULTIMATE UPDATE : systèmes méta =====================
  sv.pb = sv.pb || {}; sv.ch = Array.isArray(sv.ch) && sv.ch.length === 6 ? sv.ch : [0, 0, 0, 0, 0, 0]; sv.tk = sv.tk || 0; sv.tested = sv.tested || {}; sv.axp = sv.axp || 0;
  let evt = null, evCd = 1700, evBan = null, rush = null, lastRushW = 0, brOn = false, brQ = 0, brT = 0, pdT = 0, pdRun = 0, pdStreak = 0, pdMsg = null, dashCd = 0, xpPop = null, testedNow = {}, chestMsg = null, portalPt = null;
  const nearC = (q, c) => Math.hypot(q.x - c.x, q.y - c.y) < 40;
  const coinCurve = w => 1 + w * w / 60;
  const canHit = () => !(p.inv > 0) && !(p.ghost > 0) && !(p.dsh > 0);
  const protectd = e => !!((mg && (e === mg.boss || e.mgt)) || e.preT > 0 || ((e.k === 3 || e.brush) && !e.in));
  const passive = e => !!((mg && e === mg.boss) || e.stun > 0 || e.wake > 0 || e.preT > 0 || ((e.k === 3 || e.mini) && !e.in));
  function coinMul() {
    let b = (p.cbT > 0 ? p.cb : 1) * (evt && evt.k === 'jackpot' ? 3 : 1) * (rush ? 2 * (1 + Math.min(.5, rush.k * .01)) : 1);
    b = Math.min(16, b);                                   // plafond : jamais de x2 -> x4 -> x8 -> x16 infini
    return coinCurve(Math.max(1, wave)) * p.cg * b;
  }
  const xpMul = () => Math.min(3, (evt && evt.k === 'surcharge' ? 2 : 1) * (rush ? 1.5 : 1));

  // ---- bonus permanents (BR = rareté des bonus / coffres) ----
  const BR = ['COMMUN', 'RARE', 'ÉPIQUE', 'LÉGENDAIRE', 'DIVIN', 'EXTRÊME ULTIME'], BRC = ['#9aa7b5', '#3fa0e8', '#a055e0', '#ffb020', '#fff1b8', '#ff2bd6'];
  const PB = [
    ['pc1', 'Poche pleine', 0, '+3 % de pièces par niveau', 10, (q, l) => { q.cg *= 1 + .03 * l; }],
    ['px1', 'Apprenti', 0, "+3 % d'XP par niveau", 10, (q, l) => { q.xg *= 1 + .03 * l; }],
    ['ps1', 'Réacteurs', 0, '+2 % de vitesse par niveau', 10, (q, l) => { q.spd *= 1 + .02 * l; }],
    ['pr1', 'Gâchette', 0, '+2 % de cadence par niveau', 10, (q, l) => { q.rate *= 1 + .02 * l; }],
    ['pa1', 'Gros calibre', 0, '+2 % de taille de tir par niveau', 10, (q, l) => { q.bsz *= 1 + .02 * l; }],
    ['pc2', 'Marchand', 1, '+6 % de pièces par niveau', 5, (q, l) => { q.cg *= 1 + .06 * l; }],
    ['px2', 'Érudit', 1, "+6 % d'XP par niveau", 5, (q, l) => { q.xg *= 1 + .06 * l; }],
    ['pb2', 'Bouclier de départ', 1, 'Bouclier de 5 s par niveau au départ', 3, (q, l) => { q.shield = Math.max(q.shield, 300 * l); }],
    ['pd2', 'Maître des pouvoirs', 1, 'Recharge des pouvoirs -4 % par niveau', 5, (q, l) => { q.cdr *= 1 - .04 * l; }],
    ['pk2', 'Combo généreux', 1, 'Récompense de combo +10 % par niveau', 5, (q, l) => { q.comboB *= 1 + .1 * l; }],
    ['pe3', 'Chasseur de coffres', 2, '+3 % de chance de coffre par niveau', 5, (q, l) => { q.chestC += .03 * l; }],
    ['pg3', 'Réflexes', 2, 'Esquives parfaites +10 % par niveau', 5, (q, l) => { q.dodgeB += .1 * l; }],
    ['pp3', 'Pouvoirs amplifiés', 2, 'Dégâts des pouvoirs +8 % par niveau', 5, (q, l) => { q.pwUp *= 1 + .08 * l; }],
    ['pl3', "Prime d'élite", 2, 'Pièces des élites +10 % par niveau', 5, (q, l) => { q.eliteB *= 1 + .1 * l; }],
    ['pf4', 'Fortune', 3, '+15 % de pièces par niveau', 3, (q, l) => { q.cg *= 1 + .15 * l; }],
    ['px4', 'Prodige', 3, "+15 % d'XP par niveau", 3, (q, l) => { q.xg *= 1 + .15 * l; }],
    ['ph4', 'Deuxième cœur', 3, '+1 vie au départ par niveau', 2, (q, l) => { q.lives = Math.min(6, q.lives + l); }],
    ['pj4', 'Pouvoir jumeau', 3, 'Un second pouvoir automatique', 1, q => { q.pw2 = 1; }],
    ['pd5', 'Aura divine', 4, '+2 vies, régénération, bouclier de départ', 1, q => { q.lives = Math.min(6, q.lives + 2); q.regen = 1; q.shield = Math.max(q.shield, 600); }],
    ['pu6', 'ASCENSION', 5, '+25 % pièces et XP, second souffle, fantôme au départ', 1, q => { q.cg *= 1.25; q.xg *= 1.25; q.sec = 1; q.ghost = 300; }]
  ];
  function applyPB() { for (const b of PB) { const l = (sv.pb && sv.pb[b[0]]) || 0; if (l) b[5](p, l); } }
  function resetRun() {
    Object.assign(p, { cg: 1, xg: 1, cdr: 1, dodgeB: 0, chestC: 0, eliteB: 1, pwUp: 1, ghost: 0, sec: 0, cb: 1, cbT: 0, sx: 0, dsh: 0, pw: 0, pw2: 0, pw2t: 0, mvx: 0, mvy: 0, comboB: 1, paid: 0 });
    evt = null; evCd = 1700; evBan = null; rush = null; lastRushW = 0; brOn = false; brQ = 0; brT = 0; pdT = 0; pdRun = 0; pdStreak = 0; pdMsg = null; dashCd = 0; xpPop = null; portalPt = null;
    applyPB();
  }

  // ---- XP / niveaux visuels (plus rapides) ----
  const XTH = [0, 30, 100, 220, 400, 650, 1000];
  const skLv = () => { let l = Math.max(1, SKLW.filter(w => wave >= w).length); if (p) for (let i = 1; i < 7; i++) if (p.sx >= XTH[i]) l = Math.max(l, i + 1); return l; };
  function lvProg(L) { if (L >= 7) return 1; const a = SKLW[L - 1], b = SKLW[L], wp = (wave - a) / (b - a), xp = (p.sx - XTH[L - 1]) / (XTH[L] - XTH[L - 1]); return Math.max(0, Math.min(1, Math.max(wp, xp))); }
  function chkSl() {
    const sk = skin(), L = skLv(); if (sk.cos) sv.lv[sk.id] = Math.max(sv.lv[sk.id] || 0, L);
    if (L > p.sl) { p.sl = L; if (sk.cos) sv.st.trans++; levelUp(sk, L); earn(30 * L * L); if (L >= 5) giveChest(L >= 7 ? 2 : 1); }
  }
  function gainXp(n) {
    if (!GM.allowProgression) return;
    n = Math.max(1, Math.round(n * 1.5 * p.xg * xpMul())); p.xp += n; p.sx += n;
    if (xpPop) { xpPop.n += n; xpPop.t = 70; } else xpPop = { n, t: 70 };
    chkSl();
    if (p.xp >= p.xpN) { p.xp -= p.xpN; p.xpN = Math.round(p.xpN * 1.35); openPick(); }
  }

  // ---- esquive parfaite ----
  function perfectDodge() {
    if (pdT > 0 || !GM.allowProgression) return; pdT = 10; pdRun++; pdStreak = Math.min(pdStreak + 1, 20);
    combo += 1 + (p.dodgeB > 0 ? 1 : 0); comboT = 150;
    const coin = Math.round((2 + wave * .5) * (1 + pdStreak * .05) * (1 + p.dodgeB)); earn(coin);
    gainXp(2 + (pdStreak >> 2)); if (ult < 100) ult = Math.min(100, ult + .8); if (p.pw > 0) p.pw = Math.max(0, p.pw - 25);
    sv.st.pd = (sv.st.pd || 0) + 1; pdMsg = { t: 45, n: pdStreak };
    if (rings.length < 30) rings.push({ x: p.x, y: p.y, r: 6, max: 38, c: '#ffe45e' });
  }

  // ---- dash ----
  function doDash() {
    if (state !== 'play' || dashCd > 0) return; dashCd = isTest() ? 20 : 150;
    let dx = p.mvx, dy = p.mvy; const m = Math.hypot(dx, dy);
    if (m < .6) { dx = p.x < W / 2 ? 1 : -1; dy = 0; } else { dx /= m; dy /= m; }
    for (let i = 0; i < 6 && parts.length < PCAP(); i++) parts.push({ x: p.x - dx * i * 8, y: p.y - dy * i * 8, vx: -dx, vy: -dy, life: 18, max: 18, r: 3, c: '#bde9ff' });
    p.x = Math.max(22, Math.min(W - 22, p.x + dx * 90)); p.y = Math.max(80, Math.min(H - 30, p.y + dy * 90)); p.dsh = 16;
    AM.sfx('up'); rings.push({ x: p.x, y: p.y, r: 6, max: 50, c: '#bde9ff' });
  }

  // ---- coffres ----
  const CW = [60, 26, 10, 3.2, .7, .1];
  function rollChestR(boost = 0) {
    const w = CW.map((x, i) => i ? x * (1 + boost * i * .5) * (p ? p.luck : 1) : x); let r = Math.random() * w.reduce((a, b) => a + b, 0);
    for (let i = 0; i < 6; i++) { if ((r -= w[i]) < 0) return i; } return 0;
  }
  function giveChest(r) {
    if (!GM.allowRewards) return; sv.ch[r]++; sv.st.chest = (sv.st.chest || 0) + 1;
    floats.push({ x: p.x, y: p.y - 40, l: 80, t: '🎁 COFFRE ' + BR[r] });
    if (r >= 2) { msg = { a: '🎁 COFFRE ' + BR[r], b: 'Ajouté à tes coffres !' }; msgT = 150; }
    AM.sfx(r >= 3 ? 'rare' : 'coin'); save();
  }
  function giveBonus(br) {
    for (let b = br; b >= 0; b--) { const pool = PB.filter(x => x[2] === b && (sv.pb[x[0]] || 0) < x[4]); if (pool.length) { const x = pool[(Math.random() * pool.length) | 0]; sv.pb[x[0]] = (sv.pb[x[0]] || 0) + 1; return x; } }
    return null;
  }
  function chestSkin(r) {
    const rg2 = [null, null, null, [1, 3], [3, 6], [6, 9]][r]; if (!rg2) return null;
    if (Math.random() >= [0, 0, 0, .1, .14, .3][r]) return null;
    const pool = SKINS.filter(k => k.r >= rg2[0] && k.r <= rg2[1] && !k.fp && !k.dk && k.price > 0 && !sv.owned.includes(k.id));
    return pool.length ? pool[(Math.random() * pool.length) | 0] : null;
  }
  function openChest(r) {
    if (!(sv.ch[r] > 0)) return null; sv.ch[r]--;
    const cur = coinCurve(Math.max(5, sv.st.bestWave)), lines = [];
    const coins = Math.round([120, 400, 1200, 4500, 22000, 110000][r] * cur * (.8 + Math.random() * .4)); sv.coins += coins; sv.st.coins += coins; lines.push('+' + coins + ' ◎');
    const xp = [10, 30, 80, 200, 500, 1500][r]; sv.axp += xp; lines.push('+' + xp + " XP de compte");
    const fr = [0, 1, 2, 4, 8, 16][r]; if (fr) { sv.frag = (sv.frag || 0) + fr; lines.push('+' + fr + ' ◆ fragments'); }
    const tk = [1, 1, 2, 3, 5, 8][r]; sv.tk += tk; lines.push('+' + tk + ' 🎫 tickets');
    if (Math.random() < [.1, .3, .55, .8, 1, 1][r]) { const b = giveBonus(Math.min(5, r)); if (b) lines.push('BONUS ' + BR[b[2]] + ' : ' + b[1] + ' niv ' + sv.pb[b[0]]); }
    const sk = chestSkin(r); if (sk) { sv.owned.push(sk.id); lines.push('!SKIN : ' + sk.n + ' (' + SKN[sk.r] + ')'); }
    save(); return lines;
  }

  // ---- événements aléatoires ----
  const EVS = {
    surcharge: ['⚡ SURCHARGE', 'XP ×2 pendant 15 s', 900, '#b57cff'], jackpot: ['💰 JACKPOT', 'Pièces ×3 pendant 15 s', 900, '#ffd966'],
    invasion: ['👾 INVASION', 'Beaucoup de petits ennemis !', 1000, '#ff6a6a'], treasure: ['🏆 CHASSE AU TRÉSOR', 'Attrape le coffre !', 900, '#ffb020'],
    frenzy: ['🔥 FRÉNÉSIE', 'Les ennemis arrivent vite !', 600, '#ff7a3d'], portal: ['🌌 PORTAIL', 'Des ennemis spéciaux sortent du portail', 900, '#7a4cff'],
    coinrain: ['🪙 PLUIE DE PIÈCES', 'Ramasse les pièces !', 700, '#ffe45e'], boost: ['✖ BOOST DE PIÈCES', 'Pièces multipliées !', 60, '#ffb020'],
    meteor: ['☄ PLUIE DE MÉTÉORES', 'Évite les zones annoncées !', 800, '#ff5a3a'], blessing: ['✨ BÉNÉDICTION', '+1 vie et bouclier', 60, '#9dffb0']
  };
  function pickEvent() {
    const L = ['surcharge', 'jackpot', 'boost', 'coinrain', 'treasure', 'frenzy', 'blessing'];
    if (wave >= 2) L.push('invasion'); if (wave >= 3) L.push('meteor'); if (wave >= 4) L.push('portal', 'portal');
    return L[(Math.random() * L.length) | 0];
  }
  function startEvent(k) {
    const d = EVS[k]; evt = { k, t: d[2], max: d[2] }; evBan = { a: d[0], b: d[1], c: d[3], t: 170 }; AM.sfx('rare');
    if (k === 'boost') { const r = Math.random(), m = r < .55 ? 2 : r < .88 ? 4 : 8; p.cb = m; p.cbT = 1200; evBan.b = 'Pièces ×' + m + ' pendant 20 s !'; }
    else if (k === 'treasure') pups.push({ x: rnd(60, W - 60), y: -20, t: 'K', ph: 0 });
    else if (k === 'portal') portalPt = { x: rnd(80, W - 80), y: 150, t: 0, n: 0 };
    else if (k === 'blessing') { p.lives = Math.min(6, p.lives + 1); p.shield = Math.max(p.shield, 360); boom(p.x, p.y, 20, ['#9dffb0', '#fff']); }
  }
  function endEvent(timeout) {
    if (!evt) return; const k = evt.k;
    if (k === 'treasure' && timeout) { msg = { a: 'COFFRE PERDU', b: 'Il est reparti…' }; msgT = 100; }
    if (k === 'meteor') { const r = Math.round(40 + wave * 10); earn(r); msg = { a: '☄ MÉTÉORES ÉVITÉES', b: '+' + Math.round(r * coinMul()) + ' ◎' }; msgT = 120; }
    evt = null; portalPt = null; evCd = Math.round(rnd(1500, 2800));
  }
  function lastEnemy(n0) { return enemies.length > n0 ? enemies[enemies.length - 1] : null; }
  function evUpdate() {
    if (!evt || mg) return; evt.t--; const k = evt.k, n0 = enemies.length;
    if (k === 'invasion' && evt.t % 14 === 0 && enemies.length < 16) { spawn({ k: 0 }); const e = lastEnemy(n0); if (e) { e.hp = e.mhp = Math.max(1, Math.ceil(e.hp * .35)); } }
    else if (k === 'frenzy' && evt.t % 18 === 0 && enemies.length < 14) { spawn({ k: Math.random() < .7 ? 0 : 1 }); }
    else if (k === 'portal' && portalPt) {
      portalPt.t++;
      if (evt.t % 70 === 0 && portalPt.n < 4 && enemies.length < 12) { portalPt.n++; spawn({ k: 1 }); const e = lastEnemy(n0); if (e) { e.x = portalPt.x; e.y = portalPt.y; e.el = 1; e.hp = e.mhp = Math.ceil(e.hp * 3.5); e.r *= 1.1; e.pspec = 1; boom(e.x, e.y, 10, ['#7a4cff', '#fff']); } }
    }
    else if (k === 'coinrain' && evt.t % 9 === 0 && pups.length < 40) pups.push({ x: rnd(20, W - 20), y: -10, t: '$', ph: 0 });
    else if (k === 'jackpot' && evt.t % 40 === 0 && pups.length < 40) pups.push({ x: rnd(20, W - 20), y: -10, t: '$', ph: 0 });
    else if (k === 'meteor' && evt.t % 55 === 0) { for (let i = 0; i < 2; i++) strikes.push({ x: rnd(30, W - 30), t: 75, w: 30 }); AM.sfx('warn'); }
    if (evt.t <= 0 || (k === 'treasure' && evt.done)) endEvent(!evt.done);
  }

  // ---- Rush ----
  function startRush() { rush = { t: 1800, max: 1800, k: 0 }; lastRushW = wave; evBan = { a: '⚡ RUSH ⚡', b: '30 secondes · détruis-les tous !', c: '#ff7a3d', t: 170 }; AM.sfx('rare'); }
  function rushSpawn() {
    const n0 = enemies.length; spawn({ k: Math.random() < .65 ? 0 : 1 }); const e = lastEnemy(n0);
    if (e && !e.el && e.k < 2 && Math.random() < .22) { e.el = 1; e.hp = e.mhp = Math.ceil(e.hp * 3.5); e.r *= 1.1; }
  }
  function endRush(forced) {
    if (!rush) return; const r = rush; rush = null; const bonus = Math.round(120 + wave * 30 + r.k * 18);
    if (!forced || r.k > 0) {
      earn(bonus); gainXp(20 + r.k * 2);
      if (GM.allowProgression) { sv.st.rush = (sv.st.rush || 0) + 1; if (r.k >= 12 && Math.random() < .5 + p.chestC) giveChest(rollChestR(1)); }
      msg = { a: '⚡ RUSH TERMINÉ !', b: r.k + ' ennemis · +' + Math.round(bonus * coinMul()) + ' ◎' }; msgT = 170; AM.sfx('win');
    }
    checkAll();
  }
  function startBossRush() { brOn = true; brQ = 3; brT = 40; evBan = { a: '👑 BOSS RUSH', b: '3 mini-boss · une épreuve avant chacun', c: '#ff3d7f', t: 190 }; AM.sfx('boss'); }
  function spawnBR() {
    const n0 = enemies.length; spawn({ k: 2, mini: 1, t: 0 }); const e = lastEnemy(n0); if (!e || !e.mini) return;
    const H0 = [4, 3, 4.5, 4, 3.5], mf = ((3 - brQ) * 2 + ((wave / 5) | 0)) % 5; e.hp = e.mhp = Math.ceil(e.mhp * H0[mf] / H0[e.mf]); e.mf = mf; e.r = [36, 24, 30, 32, 28][mf]; e.x = W / 2; e.brush = 1;
  }
  function endBossRush() {
    brOn = false; const r = Math.round(500 + wave * 60); earn(r); gainXp(100); giveChest(rollChestR(3));
    msg = { a: '👑 BOSS RUSH TERMINÉ !', b: '+' + Math.round(r * coinMul()) + ' ◎' }; msgT = 190; AM.sfx('win');
  }
  function waveHook(isBoss) {
    if (isBoss) { if (rush) endRush(true); if (evt) { evt = null; portalPt = null; evCd = 1800; } return; }
    if (!isMini(wave) && !rush && !brOn && !evt) {
      if (wave >= 3 && wave - lastRushW >= 5 && Math.random() < .5) startRush();
      else if (wave >= 12 && Math.random() < .07) startBossRush();
    }
  }
  function metaUpdate() {
    if (pdT > 0) pdT--; if (dashCd > 0) dashCd--; if (p.dsh > 0) p.dsh--;
    if (p.pw2t > 0 && --p.pw2t === 0) usePower(true);
    if (evBan && --evBan.t <= 0) evBan = null; if (xpPop && --xpPop.t <= 0) xpPop = null; if (pdMsg && --pdMsg.t <= 0) pdMsg = null;
    if (mg && 'cmhe'.includes(mg.kind)) for (const x of enemies) if (x.mgt && x.hp > 0 && Math.hypot(x.x - p.x, x.y - p.y) < x.r + 14) x.hp = 0;   // mini-jeux tactiles : on touche les cibles
    if (!GM.allowWaveProgression) return;
    if (rush && !mg) { rush.t--; if (time % 22 === 0 && enemies.length < ENCAP() + 6) rushSpawn(); if (rush.t <= 0) endRush(); }
    if (evt) evUpdate(); else if (!mg && !rush && !brOn && wave >= 2 && !enemies.some(e => e.k === 3 || e.mini) && --evCd <= 0) startEvent(pickEvent());
    if (brOn && !mg) {
      if (brQ > 0 && !enemies.some(e => e.mini || e.k === 3)) { if (--brT <= 0) { spawnBR(); brQ--; brT = 60; } }
      else if (brQ === 0 && !enemies.some(e => e.brush)) endBossRush();
    }
  }

  // ---- zone de survie : si le couloir d'esquive se referme, on retire les tirs lointains qui le bouchent ----
  function relief() {
    if (time % 4 !== 0 || ebullets.length < 6) return;
    let live = ebullets.filter(b => !b.dead), g = gapOf(live), k = 0;
    while (g < 44 && k++ < 3) {
      let best = null, bd = 1e9;
      for (const b of live) { if (b.dead || b.vy <= .1 || b.y > p.y - 100) continue; const t = (p.y - b.y) / b.vy; if (t > 100 || t < 16) continue; const d = Math.abs(b.x + b.vx * t - p.x); if (d < bd) { bd = d; best = b; } }
      if (!best) break; best.dead = true; live = live.filter(b => !b.dead); g = gapOf(live);
    }
  }

  // ===================== ULTIMATE UPDATE : ultimes uniques par skin =====================
  const GUL = [
    ['classic', 'ÉTINCELLE CLASSIQUE', 'nova'], ['a01', 'RAYON A-01', 'beam'], ['scout', 'PIQUÉ ÉCLAIREUR', 'comet'], ['coral', 'MARÉE DE CORAIL', 'wave'], ['forest', 'FEUILLES ÉMERAUDE', 'leaf'],
    ['arrow', 'PLUIE DE FLÈCHES', 'lances'], ['night', 'NUIT ÉTOILÉE', 'stellar'], ['inter', 'SALVE INTERCEPTEUR', 'missile'], ['falcon', 'PIQUÉ DU FAUCON', 'dive'], ['stealth', 'FRAPPE FURTIVE', 'blades'],
    ['ranger', 'BARRAGE DU RANGER', 'laser'], ['gold', "PLUIE D'OR", 'rain'], ['cyber', 'ESSAIM DE DRONES', 'drones'], ['mecha', 'ARCS MÉCANIQUES', 'chain'], ['inferno', 'ÉRUPTION INFERNO', 'flame'],
    ['phantom', 'SPECTRES ORBITAUX', 'orbit'], ['rainbow', 'ARC TRIOMPHAL', 'prism'], ['dragon', 'TORNADE DU DRAGON', 'tornado'], ['phoenix', 'CENDRES DU PHOENIX', 'feather'], ['galaxy', 'SPIRALE GALACTIQUE', 'spiral'],
    ['titan', 'IMPACT TITAN', 'meteor'], ['void', 'ONDE DU VIDE', 'hole'], ['celestial', 'PILIERS CÉLESTES', 'pillar'], ['nebula', 'TEMPÊTE DE NÉBULEUSE', 'spiral'], ['omega', 'RAYON OMEGA', 'beam'],
    ['quasar', 'EXPLOSION QUASAR', 'stellar'], ['supernova', 'SUPERNOVA', 'nova'], ['seraph', 'PLUMES DU SÉRAPH', 'feather'], ['solaris', 'ÉCLAT SOLARIS', 'flame'], ['infinity', 'BOUCLE INFINIE', 'portal'],
    ['zero', 'POINT ZÉRO', 'hole'], ['prism', 'EXPLOSION PRISMATIQUE', 'prism'], ['aurora', "RIDEAU D'AURORE", 'wave'], ['chrono', 'FLUX CHRONO', 'clock'], ['blackhole', 'HORIZON DES ÉVÉNEMENTS', 'hole'],
    ['dquasar', 'TROU NOIR ÉLECTRIQUE', 'hole', 1], ['ncore', 'CŒUR DE NÉBULEUSE', 'orbit'], ['singod', 'DIEU DE LA SINGULARITÉ', 'rift'], ['eclipse', 'ÉCLIPSE CÉLESTE', 'moon'], ['seravoid', 'SÉRAPH DU NÉANT', 'pillar'],
    ['omegabh', 'OMEGA EFFONDREMENT', 'laser'], ['infsing', 'SINGULARITÉ INFINIE', 'portal'], ['collapse', 'EFFONDREMENT ÉTERNEL', 'meteor'],
    ['infcel', 'ÉCLAT DU SOLEIL', 'flame'], ['moonreaper', 'ÉCLIPSE LUNAIRE', 'moon'], ['starforged', 'SUPERNOVA FORGÉE', 'nova'], ['thundersov', 'ORAGE ABSOLU', 'bolts'], ['crystalnova', 'PRISON CRISTALLINE', 'crystal'],
    ['abscosmos', 'BIG BANG ABSOLU', 'x_cosmos'], ['evoid', "FIN DE L'EXISTENCE", 'x_void'], ['celabs', 'DESCENTE DIVINE', 'x_cel'], ['omnidrg', 'APOCALYPSE DU DRAGON', 'x_drg'], ['thunderabs', 'JUGEMENT DU TONNERRE', 'x_thun'],
    ['infeternal', 'SOLEIL APOCALYPTIQUE', 'x_inf'], ['chronoabs', 'FIN DES TEMPS', 'x_chr'], ['seraphabs', 'APOCALYPSE CÉLESTE', 'x_ser'], ['emperor', 'AUTORITÉ ABSOLUE', 'x_emp'], ['dimbreaker', 'FRACTURE INFINIE', 'x_dim'],
    ['dragonvoid', 'APOCALYPSE DU NÉANT', 'x_dvoid'], ['astralemp', 'COURONNE DES MILLE ÉTOILES', 'x_astral'], ['dimzero', 'RUPTURE DIMENSIONNELLE', 'x_zero']
  ];
  const GU = {}; GUL.forEach((g, i) => { GU[g[0]] = { n: g[1], st: g[2], v: i + 1, el: g[3] || 0 }; });
  const gInfo = sk => GU[sk.id] || { n: sk.n.toUpperCase(), st: 'nova', v: SKINS.indexOf(sk) + 100, el: 0 };
  const ULTCOL = sk => [(sk.cos && sk.cos.r1) || sk.flame || '#fff', (sk.cos && sk.cos.r2) || sk.wing || '#9be7ff', sk.glow || '#fff'];

  // --- anciens points d'entrée : l'ultime Chidori générique est supprimé ---
  const ukey = sk => UK[sk.id] || 'g';
  function ultOf(sk) { if (UK[sk.id]) return { n: NM[sk.id], f: [] }; return { n: gInfo(sk).n, f: [] }; }

  function startUan(k, sk, c) {
    if (k !== 'g') return startUanOld(k, sk, c);
    if (uan.length > 3) uan.shift(); const g = gInfo(sk), ex = g.st[0] === 'x';
    uan.push({ k: 'g', st: g.st, v: g.v, el: g.el, sd: g.v % 2 ? 1 : -1, seed: g.v * 7.13 + 1, T: ex ? 170 : 112, hf: ex ? 92 : 56, m: (.75 + sk.r * .13) * (ex ? 1.15 : 1), t: 0, c, cc: ULTCOL(sk), r: sk.r, x: p.x, y: p.y, zs: [], ex });
  }
  function uImpact(a) { if (a.k === 'g') gImpact(a); else uImpactOld(a); }
  function uHit2(a, test, m) {
    const d = (8 + wave * 1.4) * (1 + a.r * .12) * m, bd = (20 + wave) * (1 + a.r * .05) * m; let n = 0;
    for (const e of enemies) { if (protectd(e) || !test(e)) continue; const v = e.k === 3 ? bd : d; e.hp -= v; e.flash = 3; if (n++ < 6) floats.push({ x: e.x, y: e.y, l: 30, t: '-' + Math.round(v) }); }
  }
  function gArea(a) {
    const x = a.x, v = a.v, band = (cx, w) => e => Math.abs(e.x - cx) < w, circ = (cx, cy, R) => e => Math.hypot(e.x - cx, e.y - cy) < R, all = () => true;
    switch (a.st) {
      case 'nova': return [circ(x, a.y - 100, 210), 1.05];
      case 'beam': return [band(x, 52), 1.5];
      case 'lances': return [e => [-1, 0, 1].some(k => Math.abs(e.x - (x + k * (60 + (v % 3) * 10))) < 34), 1.25];
      case 'wave': return [circ(x, a.y, 230), 1.15];
      case 'meteor': return [circ(W / 2, H * .34, 200), 1.3];
      case 'hole': return [circ(x, Math.max(130, a.y - 170), 230), 1.3];
      case 'dive': return [e => [-1, 0, 1].some(k => Math.abs(e.x - (x + k * 70)) < 44), 1.3];
      case 'pillar': return [e => [0, 1, 2, 3].some(i => Math.abs(e.x - (W * (i + .5) / 4 + (v % 2 ? 20 : -20))) < 42), 1.2];
      case 'tornado': return [circ(x, a.y - 120, 180), 1.3];
      case 'laser': return [e => { const dy = a.y - 14 - e.y; return dy > 0 && Math.abs(Math.atan2(e.x - x, dy)) < 1.2; }, 1];
      case 'orbit': return [circ(x, a.y - 20, 200), 1.1];
      case 'crystal': return [e => e.y > H * .22, 1.1];
      default: return [all, a.ex ? 1.2 : 1];
    }
  }
  function gImpact(a) {
    const [test, m] = gArea(a); shake = a.ex ? 16 : 11; AM.sfx('boom'); if (a.r >= 5 || a.ex) ebullets.length = 0;
    uHit2(a, test, a.m * m);
    if (a.st === 'clock' || a.st === 'x_chr') freeze = 4;
    if (a.st === 'x_emp') a.tg = enemies.filter(e => !protectd(e)).slice(0, 8).map(e => [e.x, e.y]);
    if (a.st === 'wave' || a.st === 'tornado') for (const e of enemies) if (e.k < 3 && !e.mini && !e.tst && !protectd(e) && test(e)) e.y -= 10;
    boom(W / 2, H * .34, 14 + a.r * 2, [a.cc[0], a.cc[1], '#fff']); rings.push({ x: W / 2, y: H * .34, r: 8, max: 150 + a.r * 10, c: a.cc[0] });
    if (a.ex) a.xh = [a.hf + 10, a.hf + 20, a.hf + 32];
  }
  function gTick(a) {
    const t = a.t, st = a.st;
    if ((st === 'clock' || st === 'x_chr') && t === 2) p.slow = Math.max(p.slow, 80);
    if ((st === 'hole' || st === 'x_void' || st === 'x_dvoid') && t < a.hf) for (const e of enemies) if (e.k < 3 && !e.mini && !e.tst && !protectd(e)) { e.x += (W / 2 - e.x) * .015; e.y += (H * .34 - e.y) * .01; }
    if (a.xh && a.xh.includes(t)) { uHit2(a, () => true, a.m * .35); shake = 7; if (rings.length < 30) rings.push({ x: W / 2, y: H * .34, r: 6, max: 120, c: a.cc[1] }); }
  }

  // --- rendu : une fonction par style, variations via v / sd / couleurs ---
  const glw = (x, y, r, col, al) => { if (!(r > 1)) return; const g = ctx.createRadialGradient(x, y, 0, x, y, r); g.addColorStop(0, col); g.addColorStop(1, 'rgba(0,0,0,0)'); ctx.globalAlpha = Math.max(0, Math.min(1, al)); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(x, y, r, 0, 7); ctx.fill(); };
  const lnx = (x1, y1, x2, y2, w, col, al) => { ctx.globalAlpha = Math.max(0, Math.min(1, al)); ctx.strokeStyle = col; ctx.lineWidth = Math.max(.5, w); ctx.beginPath(); ctx.moveTo(x1, y1); ctx.lineTo(x2, y2); ctx.stroke(); };
  const rgx = (x, y, r, col, al, w) => { if (!(r > 0.5)) return; ctx.globalAlpha = Math.max(0, Math.min(1, al)); ctx.strokeStyle = col; ctx.lineWidth = Math.max(.5, w || 2); ctx.beginPath(); ctx.arc(x, y, r, 0, 7); ctx.stroke(); };
  const DR = {};
  DR.nova = e => { const cx = e.x, cy = e.y - 80, r = 6 + e.ch * e.ch * 46 + e.post * 6, n = 6 + e.v % 5; glw(cx, cy, r * 1.7, e.c1, .7 * e.fd); glw(cx, cy, r * .6, '#fff', .9 * e.fd); for (let i = 0; i < n; i++) { const q = i * 6.283 / n + e.t * .04 * e.sd; lnx(cx + Math.cos(q) * r * .5, cy + Math.sin(q) * r * .5, cx + Math.cos(q) * r * 1.5, cy + Math.sin(q) * r * 1.5, 2, e.c2, e.fd * .9); } if (e.post) rgx(cx, cy, e.post * 9, e.c3, e.fd, 3); };
  DR.beam = e => { const twin = e.v % 3 === 0, w = e.post ? (20 + 14 * Math.sin(e.t * .5)) * e.fd + 4 : 0; glw(e.x, e.y - 14, 8 + e.ch * 26, e.c1, .9); if (e.post) { for (const o of twin ? [-26, 26] : [0]) { ctx.globalAlpha = .55 * e.fd; ctx.fillStyle = e.c1; ctx.fillRect(e.x + o - w, 0, w * 2, Math.max(0, e.y - 20)); ctx.globalAlpha = e.fd; ctx.fillStyle = '#fff'; ctx.fillRect(e.x + o - w * .35, 0, w * .7, Math.max(0, e.y - 20)); } } else for (let i = 0; i < 4; i++) { const q = e.t * .2 + i * 1.57; lnx(e.x + Math.cos(q) * 22, e.y + Math.sin(q) * 22, e.x, e.y - 14, 1.5, e.c2, .6); } };
  DR.comet = e => { const n = 4 + e.v % 4; for (let i = 0; i < n; i++) { const s = (e.t - i * 6) / 50; if (s < 0 || s > 1.2) continue; const x0 = W * ((i + .5) / n) - e.sd * 90, x1 = x0 + e.sd * 160 * s, y1 = -20 + H * .8 * s; lnx(x1 - e.sd * 60, y1 - 90, x1, y1, 3, e.c1, .7); glw(x1, y1, 12, '#fff', .9); glw(x1, y1, 24, e.c2, .5); } if (e.post && e.post < 14) glw(W / 2, e.cy0, 20 + e.post * 12, e.c1, .5 * e.fd); };
  DR.wave = e => { for (let i = 0; i < 5; i++) { const s = (e.t - i * 9) / 60; if (s < 0) continue; const r = s * 230; ctx.globalAlpha = Math.max(0, 1 - s) * .8; ctx.strokeStyle = i % 2 ? e.c2 : e.c1; ctx.lineWidth = 5 - i * .6; ctx.beginPath(); ctx.ellipse(e.x, e.y, r, r * (.55 + (e.v % 3) * .12), 0, 0, 7); ctx.stroke(); } glw(e.x, e.y, 30 + e.ch * 30, e.c3, .4 * e.fd); };
  DR.leaf = e => { for (let i = 0; i < 26; i++) { const s = Math.min(1, e.t / (e.hf + 40)), px = e.x + Math.sin(i * 2.3 + e.t * .06 * e.sd) * (30 + s * 150), py = e.y - 20 - s * (H * .5) * (e.R(i) + .2) + Math.sin(e.t * .1 + i) * 10; ctx.save(); ctx.translate(px, py); ctx.rotate(e.t * .12 * e.sd + i); ctx.globalAlpha = .85 * e.fd; ctx.fillStyle = i % 2 ? e.c1 : e.c2; ctx.beginPath(); ctx.ellipse(0, 0, 7, 3, 0, 0, 7); ctx.fill(); ctx.restore(); } if (e.post && e.post < 12) glw(W / 2, e.cy0, 60 + e.post * 10, e.c1, .35 * e.fd); };
  DR.lances = e => { const lanes = [-1, 0, 1].map(k => e.x + k * (60 + (e.v % 3) * 10)); for (const lx of lanes) { if (!e.post) { ctx.setLineDash([6, 8]); lnx(lx, 0, lx, e.y - 30, 1.5, e.c1, .3 + .5 * e.ch); ctx.setLineDash([]); glw(lx, 20, 8 + e.ch * 10, e.c2, .6); } else { const hy = Math.min(H, e.post * 60), w = 10 * e.fd + 2; ctx.globalAlpha = .6 * e.fd; ctx.fillStyle = e.c1; ctx.fillRect(lx - w, 0, w * 2, hy); ctx.fillStyle = '#fff'; ctx.globalAlpha = e.fd; ctx.fillRect(lx - w * .3, 0, w * .6, hy); } } };
  DR.stellar = e => { const cx = e.x, cy = e.y - 90, n = 28, r = e.post ? 20 + e.post * 6 : 60 * (1 - e.ch); glw(cx, cy, 14 + e.ch * 24 + e.post, e.c1, .6 * e.fd); for (let i = 0; i < n; i++) { const q = i * 6.283 / n + e.sd * (e.post ? 0 : e.t * .12) + e.R(i) * .3, rr = r * (.6 + e.R(i + 9) * .6), px = cx + Math.cos(q) * rr, py = cy + Math.sin(q) * rr; ctx.globalAlpha = e.post ? e.fd : e.ch; ctx.fillStyle = i % 3 ? e.c2 : '#fff'; ctx.fillRect(px - 1.5, py - 1.5, 3, 3); if (i % 4 === 0) { ctx.fillRect(px - 5, py - .5, 10, 1); ctx.fillRect(px - .5, py - 5, 1, 10); } } };
  DR.missile = e => { const n = 6 + e.v % 4; for (let i = 0; i < n; i++) { const s = (e.t - i * 4) / 46; if (s < 0 || s > 1) continue; const sx = e.x + (i - (n - 1) / 2) * 8, tx = W * ((i + .5) / n), tx2 = sx + (tx - sx) * s, ty = e.y - s * (e.y - 80 - e.R(i) * 120); lnx(sx + (tx2 - sx) * .7, ty + 26 * (1 + s), tx2, ty, 2.5, e.c2, .5); glw(tx2, ty, 7, '#fff', .9); if (s > .9) glw(tx2, ty, 22, e.c1, .7); } };
  DR.dive = e => { for (let k = 0; k < 3; k++) { const s = (e.t - k * 8) / 40; if (s < 0 || s > 1.1) continue; const dx = e.x + (k - 1) * 70, yy = -30 + s * (H + 20); ctx.globalAlpha = .9 * Math.min(1, e.fd + .2); ctx.fillStyle = e.c1; ctx.beginPath(); ctx.moveTo(dx, yy + 26); ctx.lineTo(dx - 16, yy - 14); ctx.lineTo(dx, yy - 4); ctx.lineTo(dx + 16, yy - 14); ctx.closePath(); ctx.fill(); lnx(dx, yy - 10, dx, yy - 120, 5, e.c2, .35); glw(dx, yy, 20, '#fff', .35); } };
  DR.blades = e => { const n = 2 + e.v % 3; for (let i = 0; i < n; i++) { const s = Math.min(1, Math.max(0, (e.t - 18 - i * 6) / 8)); if (s <= 0) continue; const a0 = (-.6 + i * (1.2 / Math.max(1, n - 1))) * e.sd - .3, cx = W / 2, cy = e.cy0 + 40, L = 380, x1 = cx - Math.cos(a0) * L, y1 = cy - Math.sin(a0) * L, x2 = x1 + Math.cos(a0) * 2 * L * s, y2 = y1 + Math.sin(a0) * 2 * L * s, al = e.t > e.hf ? e.fd : 1; lnx(x1, y1, x2, y2, 10 * al + 2, e.c1, .5 * al); lnx(x1, y1, x2, y2, 3, '#fff', al); } if (e.t < 20) glw(e.x, e.y, 10 + e.t, e.c2, .6); if (e.t >= e.hf - 1 && e.t <= e.hf + 1) { ctx.globalAlpha = .3; ctx.fillStyle = '#fff'; ctx.fillRect(0, 0, W, H); } };
  DR.drones = e => { for (let r = 0; r < 3; r++) for (let i = 0; i < 5; i++) { const k = r * 5 + i, s = Math.min(1, Math.max(0, (e.t - k * 2) / 36)); if (s <= 0) continue; const px = 30 + (W - 60) * (i + .5) / 5 + Math.sin(e.t * .1 + k) * 4, py = -10 + s * (130 + r * 85); ctx.globalAlpha = e.t > e.hf + 10 ? Math.max(0, e.fd) : 1; ctx.fillStyle = e.c1; ctx.beginPath(); ctx.moveTo(px, py - 7); ctx.lineTo(px + 6, py); ctx.lineTo(px, py + 7); ctx.lineTo(px - 6, py); ctx.closePath(); ctx.fill(); lnx(px, py - 7, px, py - 24, 1.5, e.c2, .5); if (e.post && e.post < 10) rgx(px, py, e.post * 3, '#fff', 1 - e.post / 10, 2); } };
  DR.flame = e => { for (let i = 0; i < 14; i++) { const bx = 12 + i * (W - 24) / 13, s = e.post ? Math.min(1, e.post / 4) : e.ch * .35, hh = (60 + e.R(i) * 70) * s, by = e.post ? H - e.post * 14 : H; ctx.globalAlpha = e.post ? e.fd : .6 * e.ch; const g = ctx.createLinearGradient(0, by - hh, 0, by + 20); g.addColorStop(0, 'rgba(255,255,255,0)'); g.addColorStop(.5, i % 2 ? e.c1 : e.c2); g.addColorStop(1, 'rgba(255,60,20,0)'); ctx.fillStyle = g; ctx.beginPath(); ctx.ellipse(bx, by, 22, Math.max(1, hh), 0, 0, 7); ctx.fill(); } glw(e.x, e.y - 20, 12 + e.ch * 20, e.c3, .6 * e.fd); };
  DR.prism = e => { const cx = e.x, cy = e.y - 70, n = 8 + e.v % 4, len = 40 + (e.post ? 300 : e.ch * 120); for (let i = 0; i < n; i++) { const q = i * 6.283 / n + e.t * .05 * e.sd; ctx.globalAlpha = .55 * e.fd; ctx.strokeStyle = `hsl(${(i * 360 / n + e.t * 4) % 360},100%,62%)`; ctx.lineWidth = e.post ? 5 : 9; ctx.beginPath(); ctx.moveTo(cx, cy); ctx.lineTo(cx + Math.cos(q) * len, cy + Math.sin(q) * len); ctx.stroke(); } if (e.post) for (let k = 0; k < 3; k++) rgx(cx, cy, e.post * (6 + k * 2), `hsl(${(k * 120 + e.t * 6) % 360},100%,65%)`, e.fd, 3); glw(cx, cy, 20 + e.ch * 20, '#fff', .8 * e.fd); };
  DR.feather = e => { const wsp = 12 + 56 * Math.min(1, e.ch * 1.3); ctx.globalAlpha = .55 * e.fd; for (const d of [-1, 1]) for (let j = 0; j < 4; j++) { ctx.fillStyle = j % 2 ? e.c1 : '#fff'; ctx.beginPath(); ctx.moveTo(e.x, e.y); ctx.quadraticCurveTo(e.x + d * wsp * .8, e.y - 40 - j * 10, e.x + d * (wsp + 20 + j * 8), e.y - 6 + j * 8); ctx.quadraticCurveTo(e.x + d * wsp * .5, e.y - 10, e.x, e.y + 8); ctx.fill(); } if (e.post) for (let i = 0; i < 20; i++) { const fx = e.R(i) * W, fy = -10 + ((e.t * (1.5 + e.R(i + 3)) + i * 40) % (H + 20)); ctx.globalAlpha = e.fd * .8; ctx.save(); ctx.translate(fx + Math.sin(e.t * .1 + i) * 8, fy); ctx.rotate(Math.sin(e.t * .1 + i)); ctx.fillStyle = i % 2 ? e.c2 : '#fff'; ctx.beginPath(); ctx.ellipse(0, 0, 2.5, 8, 0, 0, 7); ctx.fill(); ctx.restore(); } };
  DR.meteor = e => { for (let i = 0; i < 5; i++) { const s = (e.t - i * 7) / 46; if (s < 0 || s > 1) continue; const tx = W / 2 + (i - 2) * 46, ty = e.cy0 + (i % 2) * 40, sx = tx - e.sd * 140, sy = -30, px = sx + (tx - sx) * s, py = sy + (ty - sy) * s; lnx(px - e.sd * 40, py - 70, px, py, 6, e.c1, .5); glw(px, py, 16, '#fff', .9); glw(px, py, 30, e.c2, .6); if (s > .97) glw(tx, ty, 50, e.c1, .7); } if (e.post) { rgx(W / 2, e.cy0 + 20, e.post * 10, e.c1, e.fd, 4); glw(W / 2, e.cy0 + 20, 60 * e.fd + 10, e.c2, .6 * e.fd); } };
  DR.hole = e => { const zx = e.x, zy = Math.max(130, e.y - 170), vf = Math.min(e.ch, e.fd), R = 8 + 62 * e.ch + e.post * 1.2; ctx.globalCompositeOperation = 'source-over'; ctx.globalAlpha = .9 * e.fd; ctx.fillStyle = '#04010a'; ctx.beginPath(); ctx.arc(zx, zy, Math.max(1, R * Math.min(1, e.fd + .1)), 0, 7); ctx.fill(); ctx.globalCompositeOperation = 'lighter'; rgx(zx, zy, R * 1.05, e.c1, vf, 3); for (let i = 0; i < 12; i++) { const q = e.t * .08 * e.sd + i * .52, rr = R + 8 + (1 - ((e.t * .02 + i * .13) % 1)) * 60; ctx.globalAlpha = .7 * vf; ctx.fillStyle = i % 2 ? e.c1 : e.c2; ctx.fillRect(zx + Math.cos(q) * rr - 1, zy + Math.sin(q) * rr * .7 - 1, 2.5, 2.5); } if (e.el) { ctx.globalAlpha = Math.max(0, vf); for (let i = 0; i < 4; i++) { const q = e.R(i + e.t % 7) * 6.283; bolt(zx + Math.cos(q) * R, zy + Math.sin(q) * R, zx + Math.cos(q) * (R + 60), zy + Math.sin(q) * (R + 60), .8, e.c1); } } if (e.post) rgx(zx, zy, R + e.post * 5, e.c3, e.fd, 3); };
  DR.clock = e => { const cx = e.x, cy = e.cy0, r = 50 + e.ch * 40; [[cx, cy, r], [cx - 100, cy + 70, r * .45], [cx + 100, cy + 70, r * .45]].forEach(([fx, fy, fr], k) => { const al = (e.post ? e.fd : e.ch) * .9; rgx(fx, fy, fr, e.c1, al, 3); rgx(fx, fy, fr * .86, e.c2, al * .5, 1.5); for (let m = 0; m < 12; m++) { const q = m * .5236; lnx(fx + Math.cos(q) * fr * .8, fy + Math.sin(q) * fr * .8, fx + Math.cos(q) * fr * .92, fy + Math.sin(q) * fr * .92, 2, e.c3, al); } const sp = e.t * (.15 + e.ch * .5) * e.sd * (k + 1); lnx(fx, fy, fx + Math.cos(sp) * fr * .7, fy + Math.sin(sp) * fr * .7, 3, '#fff', al); lnx(fx, fy, fx + Math.cos(sp / 12) * fr * .45, fy + Math.sin(sp / 12) * fr * .45, 4, e.c1, al); }); if (e.post) rgx(cx, cy, e.post * 12, e.c1, e.fd, 3); };
  DR.moon = e => { const cx = e.x + e.sd * (1 - e.ch) * 160, cy = e.cy0 - 20, r = 56, al = e.post ? e.fd : Math.min(1, e.ch * 1.5); glw(cx, cy, r * 2.2, e.c1, .35 * al); ctx.globalAlpha = al; ctx.fillStyle = e.c1; ctx.beginPath(); ctx.arc(cx, cy, r, 0, 7); ctx.fill(); ctx.globalCompositeOperation = 'source-over'; ctx.fillStyle = '#05040f'; ctx.globalAlpha = al * .95; ctx.beginPath(); ctx.arc(cx + e.sd * r * .45, cy, r * .94, 0, 7); ctx.fill(); ctx.globalCompositeOperation = 'lighter'; if (e.post) rgx(cx, cy, r + e.post * 9, e.c2, e.fd, 4); };
  DR.portal = e => { [[W * .22, e.cy0 - 30], [W * .78, e.cy0 - 30], [W / 2, e.cy0 + 60]].forEach(([px, py], k) => { const open = Math.min(1, Math.max(0, (e.t - k * 8) / 20)) * (e.post ? e.fd : 1), rx = 10 + 26 * open; if (open <= 0) return; for (let m = 0; m < 3; m++) { const rr = Math.max(1, rx - m * 5); ctx.globalAlpha = (.8 - m * .2) * open; ctx.strokeStyle = m % 2 ? e.c2 : e.c1; ctx.lineWidth = 3; ctx.beginPath(); ctx.ellipse(px, py, rr, rr * 1.5, e.t * .05 * e.sd, 0, 7); ctx.stroke(); } if (e.post) lnx(px, py, px + (k - 1) * 40, H, 6 * e.fd + 1, e.c1, .5 * e.fd); }); };
  DR.rift = e => { const s = Math.min(1, e.t / 30), pts = []; for (let i = 0; i <= 10; i++) pts.push([W * (.1 + .8 * i / 10) + (e.R(i) - .5) * 60, e.cy0 - 80 + i * 24 * e.sd + (e.R(i + 5) - .5) * 50]); const upto = Math.floor(10 * s), w = (e.post ? 12 * e.fd : 3) + 1; for (const [lw, col, al] of [[w * 2.5, e.c1, .35], [w, '#fff', .9]]) { ctx.lineWidth = lw; ctx.strokeStyle = col; ctx.globalAlpha = al * (e.post ? e.fd : 1); ctx.beginPath(); pts.slice(0, upto + 1).forEach((q, i) => i ? ctx.lineTo(q[0], q[1]) : ctx.moveTo(q[0], q[1])); ctx.stroke(); } if (e.post) glw(W / 2, e.cy0, 60 + e.post * 12, e.c2, .4 * e.fd); };
  DR.crystal = e => { for (let i = 0; i < 9; i++) { const s = Math.min(1, Math.max(0, (e.t - i * 3) / 30)), bx = 20 + (W - 40) * i / 8, hh = (90 + e.R(i) * 140) * s; ctx.globalAlpha = (e.post ? e.fd : 1) * .75; ctx.fillStyle = i % 2 ? e.c1 : e.c2; ctx.beginPath(); ctx.moveTo(bx - 14, H); ctx.lineTo(bx, H - hh); ctx.lineTo(bx + 14, H); ctx.closePath(); ctx.fill(); lnx(bx, H - hh, bx, H, 1.5, '#fff', .6 * (e.post ? e.fd : 1)); } if (e.post) glw(W / 2, e.cy0 + 60, 40 + e.post * 10, e.c3, .4 * e.fd); };
  DR.bolts = e => { ctx.lineCap = 'round'; for (let i = 0; i < 6; i++) { const bx = 30 + e.R(i) * (W - 60), by = 120 + e.R(i + 4) * 330, at = 18 + i * 6; if (e.t < at) { rgx(bx, by, 14 * (e.t / at), e.c1, .6, 2); continue; } if (e.t < at + 8) { ctx.globalAlpha = 1; bolt(bx + (e.R(i + e.t) - .5) * 14, 0, bx, by, 2, e.c1); glw(bx, by, 34, e.c1, .4); } } if (e.post > 0 && e.post < 14) { ctx.globalAlpha = 1 - e.post / 14; bolt(W / 2, 0, W / 2, e.cy0, 6, e.c1); glw(W / 2, e.cy0, 56, e.c1, .5 * (1 - e.post / 14)); } };
  DR.spiral = e => { const cx = e.x, cy = e.y - 100 - 30 * e.ch, arms = 2 + e.v % 3, sc = .4 + 1.5 * e.ch; for (let m = 0; m < arms; m++) for (let i = 0; i < 18; i++) { const q = e.t * (.05 + .09 * e.ch) * e.sd + m * 6.283 / arms + i * .38, r = (4 + i * 3.6) * sc; ctx.globalAlpha = (1 - i / 20) * e.fd; ctx.fillStyle = m % 2 ? e.c2 : e.c1; ctx.beginPath(); ctx.arc(cx + Math.cos(q) * r, cy + Math.sin(q) * r * .6, 1.8, 0, 7); ctx.fill(); } glw(cx, cy, 14 + 12 * e.ch, '#fff', .5 * e.fd); if (e.post && e.post < 14) glw(cx, cy, 30 + e.post * 8, e.c1, .5 * (1 - e.post / 14)); };
  DR.orbit = e => { const n = 5 + e.v % 3; for (let i = 0; i < n; i++) { const q = e.t * .08 * e.sd + i * 6.283 / n, r = e.post ? 40 + e.post * 8 : 90 - 50 * e.ch, px = e.x + Math.cos(q) * r, py = e.y - 20 + Math.sin(q) * r * .6; glw(px, py, 10, i % 2 ? e.c1 : e.c2, .9 * e.fd); lnx(px, py, e.x, e.y - 20, 1, e.c3, .25 * e.fd); } if (e.post) rgx(e.x, e.y - 20, e.post * 9, e.c1, e.fd, 3); };
  DR.chain = e => { let px = e.x, py = e.y - 20; [[W * .2, 150], [W * .45, 210], [W * .7, 150], [W * .85, 260], [W * .5, 100]].forEach(([tx, ty], i) => { const s = Math.min(1, Math.max(0, (e.t - 14 - i * 8) / 8)); if (s > 0) { ctx.globalAlpha = 1; bolt(px, py, px + (tx - px) * s, py + (ty - py) * s, 1.4, e.c1); if (s >= 1) glw(tx, ty, 22, e.c2, .6 * e.fd); } px = tx; py = ty; }); glw(e.x, e.y - 10, 10 + e.ch * 18, e.c1, .6); };
  DR.pillar = e => { for (let i = 0; i < 4; i++) { const px = W * (i + .5) / 4 + (e.v % 2 ? 20 : -20), w = e.post ? 26 * e.fd + 3 : 4; if (e.t < i * 6) continue; ctx.globalAlpha = .55 * (e.post ? e.fd : e.ch); const g = ctx.createLinearGradient(px, 0, px, H); g.addColorStop(0, 'rgba(255,255,255,0)'); g.addColorStop(.5, i % 2 ? e.c1 : e.c2); g.addColorStop(1, 'rgba(255,255,255,0)'); ctx.fillStyle = g; ctx.fillRect(px - w, 0, w * 2, H); rgx(px, H * .7, 10 + w, e.c3, .6 * (e.post ? e.fd : 1), 2); } };
  DR.tornado = e => { const h = 280 * Math.min(1, e.ch * 1.4); for (let i = 0; i < 22; i++) { const f = i / 21, yy = e.y - f * h, q = e.t * .25 * e.sd + i * 1.1, r = 8 + f * 52 * (e.post ? 1 + e.post * .03 : 1); ctx.globalAlpha = (1 - f * .5) * e.fd * .8; ctx.fillStyle = i % 2 ? e.c1 : e.c2; ctx.beginPath(); ctx.ellipse(e.x + Math.cos(q) * r, yy, 6, 3, 0, 0, 7); ctx.fill(); } if (e.post && e.post < 12) glw(e.x, e.y - h, 40 + e.post * 8, e.c1, .4 * (1 - e.post / 12)); };
  DR.laser = e => { const n = 3 + e.v % 3; for (let i = 0; i < n; i++) { const a0 = -1.1 + 2.2 * i / (n - 1), sw = e.post ? Math.sin(e.post * .12 * e.sd) * .35 : 0, a1 = a0 + sw - 1.57; ctx.globalAlpha = (e.post ? e.fd : e.ch) * .8; ctx.strokeStyle = i % 2 ? e.c2 : e.c1; ctx.lineWidth = e.post ? 5 : 1.5; ctx.beginPath(); ctx.moveTo(e.x, e.y - 14); ctx.lineTo(e.x + Math.cos(a1) * 700, e.y - 14 + Math.sin(a1) * 700); ctx.stroke(); } glw(e.x, e.y - 14, 10 + e.ch * 14, '#fff', .8); };
  DR.rain = e => { for (let i = 0; i < 34; i++) { if (e.t < 10 + (i % 8)) continue; const sx = e.R(i) * W, spd = 7 + e.R(i + 8) * 7, yy = ((e.t * spd * .7) + e.R(i + 3) * H) % (H + 40) - 20; ctx.globalAlpha = (e.post ? e.fd : 1) * .8; ctx.fillStyle = i % 2 ? e.c1 : '#fff'; ctx.beginPath(); ctx.arc(sx, yy, 2.6, 0, 7); ctx.fill(); lnx(sx, yy - 14, sx, yy, 1.5, e.c2, .5 * (e.post ? e.fd : 1)); } glw(e.x, e.y - 30, 12 + e.ch * 28, e.c1, .5 * e.fd); };

  // --- EXTRÊME ULTIME : 13 animations dédiées, plusieurs phases ---
  DR.x_cosmos = e => { const cx = e.x, cy = e.y - 130;
    if (e.t < e.hf) { const c = e.ch, d0 = 300 * (1 - Math.min(1, c * 1.6)) + 10;
      for (let i = 0; i < 30; i++) { const q = e.R(i) * 6.283; ctx.globalAlpha = .8; ctx.fillStyle = i % 3 ? e.c1 : e.c2; ctx.fillRect(cx + Math.cos(q) * d0 - 1, cy + Math.sin(q) * d0 * .8 - 1, 2.5, 2.5); }
      for (let k = 0; k < 4; k++) { const q = e.t * (.06 + c * .25) + k * 1.57, r = 56 - 20 * c; glw(cx + Math.cos(q) * r, cy + Math.sin(q) * r * .6, 6, ['#ffd966', '#ff9ae6', '#9be7ff', '#fff'][k], .9); }
      const gr = c < .45 ? 0 : (c - .45) / .55, R0 = 70 * (1 - gr) + 3;
      for (let m = 0; m < 2; m++) for (let i = 0; i < 16; i++) { const q = e.t * .12 + m * 3.14 + i * .42, r = (4 + i * 4) * (R0 / 70); ctx.globalAlpha = (1 - i / 18) * Math.min(1, c * 2); ctx.fillStyle = m ? e.c2 : e.c1; ctx.beginPath(); ctx.arc(cx + Math.cos(q) * r, cy + Math.sin(q) * r * .6, 1.6, 0, 7); ctx.fill(); }
      glw(cx, cy, 8 + 14 * gr, '#fff', .7 * Math.min(1, c * 2));
    } else { const p2 = e.post;
      if (p2 < 3) { ctx.globalAlpha = .5; ctx.fillStyle = '#fff'; ctx.fillRect(0, 0, W, H); }
      for (let k = 0; k < 3; k++) rgx(cx, cy, p2 * (9 + k * 3), [e.c1, e.c2, '#fff'][k], e.fd, 4 - k);
      for (let i = 0; i < 36; i++) { const sx = e.R(i) * W, sy = (p2 * 14 * (.6 + e.R(i + 2)) + e.R(i + 5) * H) % H; lnx(sx, sy - 30, sx, sy, 1.5, i % 2 ? '#fff' : e.c1, e.fd); }
      for (let g = 0; g < 3; g++) { const a = Math.sin(Math.max(0, p2 - g * 12) / 30 * 3.14); if (a <= 0) continue; const gx = 70 + g * 110, gy = e.cy0 + 90 - g * 40; for (let i = 0; i < 12; i++) { const q = p2 * .15 + i * .55, r = 3 + i * 2.4; ctx.globalAlpha = a * e.fd; ctx.fillStyle = i % 2 ? e.c2 : e.c1; ctx.fillRect(gx + Math.cos(q) * r, gy + Math.sin(q) * r * .6, 2, 2); } }
    } };
  DR.x_void = e => { const cx = W / 2, cy = e.cy0 + 20, ph1 = Math.min(1, e.t / (e.hf * .55)), R = e.post ? 150 + e.post * 2 : 10 + 140 * ph1 * ph1, dk = e.post ? e.fd : Math.min(1, e.t / 40);
    ctx.globalCompositeOperation = 'source-over'; ctx.globalAlpha = .38 * dk; ctx.fillStyle = '#05010f'; ctx.fillRect(0, 0, W, H);
    ctx.globalAlpha = .93 * Math.max(0, e.post ? e.fd : 1); ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(cx, cy, Math.max(1, R * (e.post ? e.fd * .6 + .4 : 1)), 0, 7); ctx.fill();
    ctx.globalCompositeOperation = 'lighter'; rgx(cx, cy, R * 1.04, e.c1, dk, 3);
    if (e.t > e.hf * .4) for (let i = 0; i < 8; i++) { const a = e.R(i) * 6.283; lnx(cx + Math.cos(a) * R, cy + Math.sin(a) * R, cx + Math.cos(a) * (R + R * .5 * ph1), cy + Math.sin(a) * (R + R * .5 * ph1), 1.5, e.c2, .7 * dk); }
    if (e.post) { rgx(cx, cy, e.post * 11, '#7a2cff', e.fd, 8); rgx(cx, cy, e.post * 8, e.c1, e.fd, 3); } };
  DR.x_cel = e => { const ctop = 110, open = Math.min(1, e.t / 36), vis = e.post ? e.fd : open;
    ctx.globalAlpha = .22 * (e.post ? e.fd : Math.min(1, e.t / 30)); ctx.fillStyle = '#fff6c8'; ctx.fillRect(0, 0, W, H);
    for (let k = 0; k < 3; k++) { ctx.globalAlpha = vis * (.9 - k * .2); ctx.strokeStyle = k % 2 ? e.c2 : e.c1; ctx.lineWidth = 2.5; ctx.beginPath(); ctx.ellipse(W / 2, ctop, Math.max(1, (60 + k * 34) * open), Math.max(1, (16 + k * 9) * open), 0, 0, 7); ctx.stroke(); }
    for (let i = 0; i < 10; i++) { const q = i * .628 + e.t * .03; ctx.globalAlpha = vis; ctx.fillStyle = e.c3; ctx.fillRect(W / 2 + Math.cos(q) * 80 * open - 2, ctop + Math.sin(q) * 22 * open - 2, 4, 4); }
    if (e.t > 24 && e.t < e.hf) for (let i = 0; i < 7; i++) { ctx.globalAlpha = .3 * Math.min(1, (e.t - 24) / 20); ctx.fillStyle = e.c1; const rx = W / 2 + (i - 3) * 38; ctx.beginPath(); ctx.moveTo(rx - 4, ctop); ctx.lineTo(rx + 4, ctop); ctx.lineTo(rx + (i - 3) * 14 + 10, H); ctx.lineTo(rx + (i - 3) * 14 - 10, H); ctx.fill(); }
    if (e.t >= e.hf - 14) { const s = Math.min(1, (e.t - (e.hf - 14)) / 8), w = e.post ? 18 * e.fd + 4 : 5 + 10 * s; ctx.globalAlpha = e.post ? e.fd : s; ctx.fillStyle = e.c1; ctx.fillRect(W / 2 - w, ctop, w * 2, (H - ctop) * s); ctx.fillStyle = '#fff'; ctx.fillRect(W / 2 - w * .35, ctop, w * .7, (H - ctop) * s); }
    if (e.post) { rgx(W / 2, e.cy0 + 60, e.post * 10, e.c1, e.fd, 5); for (let i = 0; i < 14; i++) { const fx = e.R(i) * W, fy = e.cy0 - 40 + ((e.post * (1.2 + e.R(i + 2)) + i * 30) % 260); ctx.globalAlpha = e.fd; ctx.fillStyle = i % 2 ? e.c1 : '#fff'; ctx.beginPath(); ctx.ellipse(fx, fy, 2.2, 7, Math.sin(e.t * .1 + i), 0, 7); ctx.fill(); } } };
  const dragonHead = (e, hx, hy, open, body, tooth) => { ctx.fillStyle = body; ctx.beginPath(); ctx.moveTo(hx - 24, hy - 18); ctx.lineTo(hx + 24, hy - 18); ctx.lineTo(hx + 8, hy + 10); ctx.lineTo(hx - 8, hy + 10); ctx.closePath(); ctx.fill(); ctx.beginPath(); ctx.moveTo(hx - 16, hy + 14 + open); ctx.lineTo(hx + 16, hy + 14 + open); ctx.lineTo(hx, hy + 32 + open); ctx.closePath(); ctx.fill(); ctx.fillStyle = tooth; for (const d of [-1, 1]) { ctx.beginPath(); ctx.moveTo(hx + d * 18, hy - 18); ctx.lineTo(hx + d * 32, hy - 40); ctx.lineTo(hx + d * 24, hy - 14); ctx.fill(); } };
  DR.x_drg = e => { const cx = e.x, N = 18, c = e.ch, op = e.t > e.hf * .55, hy = 86, vis = Math.min(1, e.t / 40) * (e.post ? e.fd : 1);
    for (let i = 0; i < N; i++) { const f = i / (N - 1), q = e.t * .1 * e.sd + f * 9, r = (110 - f * 70) * (1 - (e.post ? 0 : c * .2)), bx = cx + Math.cos(q) * r, by = e.y - 30 + Math.sin(q) * r * .55 - f * 60; ctx.globalAlpha = vis * (1 - f * .3); ctx.fillStyle = i % 2 ? e.c1 : e.c2; ctx.beginPath(); ctx.arc(bx, by, 9 - f * 4, 0, 7); ctx.fill(); }
    ctx.globalAlpha = vis; dragonHead(e, cx, hy, op ? 12 : 3, e.c1, e.c3);
    if (op && !e.post) glw(cx, hy + 26, 10 + 30 * Math.min(1, (e.t - e.hf * .55) / (e.hf * .45)), '#fff', .9);
    if (e.post) { const w = 24 * e.fd + 6; ctx.globalAlpha = .6 * e.fd; ctx.fillStyle = e.c1; ctx.fillRect(cx - w, hy + 30, w * 2, H); ctx.globalAlpha = e.fd; ctx.fillStyle = '#fff'; ctx.fillRect(cx - w * .35, hy + 30, w * .7, H); rgx(cx, e.cy0 + 80, e.post * 10, e.c2, e.fd, 5); for (let i = 0; i < 16; i++) { const sx = cx + (e.R(i) - .5) * 160, sy = e.cy0 + ((e.post * (1 + e.R(i + 3)) * 3 + i * 20) % 300); ctx.fillStyle = i % 2 ? e.c1 : '#fff'; ctx.fillRect(sx, sy, 2.5, 2.5); } } };
  DR.x_thun = e => { ctx.lineCap = 'round';
    for (let i = 0; i < 22; i++) { const at = 6 + i * 2.2, bx = 20 + e.R(i) * (W - 40), by = 100 + e.R(i + 9) * 380; if (e.t < at) continue; if (e.t < at + 5) { ctx.globalAlpha = 1; bolt(bx + (e.R(i + e.t) - .5) * 10, 0, bx, by, 1.2, e.c1); glw(bx, by, 20, e.c1, .5); } else if (e.t < e.hf) glw(bx, by, 6, e.c1, .3); }
    if (e.t < e.hf && e.t > e.hf - 20) rgx(W / 2, e.cy0, 40 * (1 - (e.hf - e.t) / 20) + 6, '#fff', .8, 3);
    if (e.post > 0 && e.post < 18) { ctx.globalAlpha = 1; bolt(W / 2, 0, W / 2, e.cy0, 9, e.c1); glw(W / 2, e.cy0, 70, '#fff', .6 * (1 - e.post / 18)); }
    if (e.post > 8 && e.post < 28) for (let k = 0; k < 6; k++) { if (e.post < 8 + k * 2) continue; const q = k * 1.047 + .3, tx = W / 2 + Math.cos(q) * 120, ty = e.cy0 + Math.sin(q) * 120; ctx.globalAlpha = 1; bolt(tx + (e.R(k) - .5) * 20, 0, tx, ty, 3, e.c1); glw(tx, ty, 26, e.c1, .5); }
    if (e.post > 20) rgx(W / 2, e.cy0, (e.post - 20) * 12, e.c2, e.fd, 5); };
  DR.x_inf = e => { const cx = W / 2, cy = e.cy0 - 20, R = e.post ? 70 + e.post : 4 + 60 * e.ch * e.ch;
    glw(cx, cy, R * 2.4, e.c2, .55 * e.fd); glw(cx, cy, R * 1.3, e.c1, .85 * e.fd); glw(cx, cy, R * .7, '#fff', .95 * e.fd);
    for (let i = 0; i < 12; i++) { const q = i * .5236 + e.t * .03 * e.sd, l = R * (1.2 + .5 * Math.sin(e.t * .2 + i)); lnx(cx + Math.cos(q) * R, cy + Math.sin(q) * R, cx + Math.cos(q) * (R + l * .5), cy + Math.sin(q) * (R + l * .5), 3, e.c1, .6 * e.fd); }
    if (e.post) { const yy = H - e.post * 16; ctx.globalAlpha = .75 * e.fd; const g = ctx.createLinearGradient(0, yy - 60, 0, yy + 30); g.addColorStop(0, 'rgba(255,200,60,0)'); g.addColorStop(.6, e.c1); g.addColorStop(1, 'rgba(255,60,20,0)'); ctx.fillStyle = g; ctx.fillRect(0, yy - 60, W, 90); rgx(cx, cy, e.post * 9, '#fff', e.fd, 4); } };
  DR.x_chr = e => { const cx = W / 2, cy = e.cy0;
    ctx.globalCompositeOperation = 'source-over'; ctx.globalAlpha = .22 * (e.post ? e.fd : Math.min(1, e.t / 20)); ctx.fillStyle = '#0a1a3a'; ctx.fillRect(0, 0, W, H); ctx.globalCompositeOperation = 'lighter';
    [[cx, cy - 30, 54], [cx - 105, cy + 40, 30], [cx + 105, cy + 40, 30], [cx - 60, cy + 120, 24], [cx + 60, cy + 120, 24]].forEach(([fx, fy, fr], k) => { const op = Math.min(1, Math.max(0, (e.t - k * 7) / 14)) * (e.post ? e.fd : 1); if (op <= 0) return; rgx(fx, fy, fr, e.c1, op, 3); const sp = e.t * (.3 + k * .12) * (e.t < e.hf ? 1 : 2); lnx(fx, fy, fx + Math.cos(sp) * fr * .8, fy + Math.sin(sp) * fr * .8, 2.5, '#fff', op); lnx(fx, fy, fx + Math.cos(sp * .083) * fr * .5, fy + Math.sin(sp * .083) * fr * .5, 3.5, e.c2, op); });
    if (e.t > e.hf * .4 && e.t < e.hf) for (let i = 0; i < 3; i++) rgx(cx, cy, (e.t * 3 + i * 60) % 200, e.c3, .35, 2);
    if (e.post) { rgx(cx, cy, e.post * 14, e.c1, e.fd, 5); if (e.post < 5) { ctx.globalAlpha = .4 * (1 - e.post / 5); ctx.fillStyle = '#fff'; ctx.fillRect(0, 0, W, H); } for (let i = 0; i < 16; i++) { const q = i * .3927; lnx(cx + Math.cos(q) * e.post * 6, cy + Math.sin(q) * e.post * 6, cx + Math.cos(q) * (e.post * 6 + 30), cy + Math.sin(q) * (e.post * 6 + 30), 2, e.c2, e.fd); } } };
  DR.x_ser = e => {
    for (let i = 0; i < 46; i++) { const s = Math.min(1, e.t / (e.hf * .7)), tx = e.x + (e.R(i) - .5) * 220, ty = e.y - 60 - e.R(i + 4) * 260, px = e.x + (tx - e.x) * s, py = e.y + (ty - e.y) * s + Math.sin(e.t * .1 + i) * 3; ctx.globalAlpha = (e.post ? e.fd : Math.min(1, e.t / 10)) * .85; ctx.fillStyle = i % 3 ? '#fff' : e.c1; ctx.fillRect(px - 1.2, py - 1.2, 2.4, 2.4); }
    if (e.t >= e.hf * .6) for (let k = 0; k < 7; k++) { const lx = 30 + (W - 60) * k / 6, at = e.hf * .6 + k * 4; if (e.t < at + 10) { ctx.setLineDash([5, 7]); lnx(lx, 0, lx, H, 1.2, e.c1, .35); ctx.setLineDash([]); } if (e.t >= at && e.t < at + 26) { const s = Math.min(1, (e.t - at) / 6), w = 5 * (1 - (e.t - at) / 30); ctx.globalAlpha = .9; ctx.fillStyle = e.c1; ctx.fillRect(lx - w, 0, w * 2, H * s); ctx.fillStyle = '#fff'; ctx.fillRect(lx - w * .3, 0, w * .6, H * s); } }
    if (e.post) { rgx(W / 2, e.cy0 + 40, e.post * 11, e.c1, e.fd, 5); glw(W / 2, e.cy0 + 40, 60 + e.post * 4, '#fff', .5 * e.fd); } };
  DR.x_emp = e => { const z = Math.min(1, e.t / 30), zc = e.cy0 + 80;
    ctx.globalAlpha = .3 * (e.post ? e.fd : z); ctx.fillStyle = e.c1; ctx.beginPath(); ctx.ellipse(W / 2, zc, Math.max(1, 150 * z), Math.max(1, 90 * z), 0, 0, 7); ctx.fill();
    for (let i = 0; i < 10; i++) { const q = i * .628 + e.t * .02 * e.sd, sx = W / 2 + Math.cos(q) * 160 * z, sy = zc + Math.sin(q) * 100 * z; ctx.globalAlpha = (e.post ? e.fd : z); ctx.fillStyle = e.c2; ctx.beginPath(); ctx.moveTo(sx, sy - 7); ctx.lineTo(sx + 5, sy); ctx.lineTo(sx, sy + 7); ctx.lineTo(sx - 5, sy); ctx.fill(); }
    ctx.globalAlpha = z * (e.post ? e.fd : 1); ctx.fillStyle = e.c1; ctx.beginPath(); for (let i = 0; i < 5; i++) { const px = W / 2 + (i - 2) * 12; ctx.moveTo(px - 5, zc - 150); ctx.lineTo(px, zc - 150 - (i % 2 ? 10 : 20)); ctx.lineTo(px + 5, zc - 150); } ctx.fill();
    if (e.t > e.hf * .4 && e.t < e.hf + 10) { const s = (e.t - e.hf * .4) / (e.hf * .6 + 10), yy = H - s * H; ctx.globalAlpha = .45; ctx.fillStyle = e.c1; ctx.fillRect(0, yy - 14, W, 28); ctx.fillStyle = '#fff'; ctx.fillRect(0, yy - 2, W, 4); }
    if (e.post && e.a.tg) e.a.tg.forEach(([tx, ty], i) => { const at = 4 + i * 4; if (e.post < at || e.post > at + 14) return; const s = (e.post - at) / 14; rgx(tx, ty, 8 + s * 36, e.c1, 1 - s, 3); glw(tx, ty, 14 + s * 20, '#fff', .8 * (1 - s)); lnx(tx, ty - 120, tx, ty, 3, e.c2, 1 - s); }); };
  DR.x_dim = e => { const s = Math.min(1, e.t / 26), pts = []; for (let i = 0; i <= 12; i++) pts.push([W * i / 12, e.cy0 - 100 + i * 20 + (e.R(i) - .5) * 70]); const upto = Math.floor(12 * s), w = e.post > 30 ? Math.max(1, 6 * e.fd) : 6;
    for (const [lw, col, al] of [[w * 2.5, e.c1, .4], [w * .7, '#fff', .95]]) { ctx.lineWidth = lw; ctx.strokeStyle = col; ctx.globalAlpha = al; ctx.beginPath(); pts.slice(0, upto + 1).forEach((q, i) => i ? ctx.lineTo(q[0], q[1]) : ctx.moveTo(q[0], q[1])); ctx.stroke(); }
    const P = [[W * .15, e.cy0 + 10], [W * .85, e.cy0 + 10], [W * .35, e.cy0 + 130], [W * .65, e.cy0 + 130]];
    P.forEach(([px, py], k) => { const open = Math.min(1, Math.max(0, (e.t - 26 - k * 6) / 16)) * (e.post > 20 ? Math.max(0, 1 - (e.post - 20) / 16) : 1); if (open <= 0) return; const rr = 8 + 22 * open; ctx.globalAlpha = open; ctx.strokeStyle = k % 2 ? e.c2 : e.c1; ctx.lineWidth = 3; ctx.beginPath(); ctx.ellipse(px, py, rr, rr * 1.4, e.t * .05, 0, 7); ctx.stroke();
      const f = Math.max(0, e.t - e.hf * .6);
      if (f > 0 && e.post < 20) { if (k === 0) lnx(px, py, px + 200, py + 40 + f * 2, 4, e.c1, open); else if (k === 1) rgx(px, py, f * 3, e.c2, open, 3); else if (k === 2) { for (let i = 0; i < 8; i++) { const q = f * .2 + i * .78; lnx(px, py, px + Math.cos(q) * (20 + f * 3), py + Math.sin(q) * (20 + f * 3), 2, e.c1, open); } } else { ctx.globalAlpha = 1; bolt(px, py, px + (e.R(f) - .5) * 100, 0, 2, e.c2); } }
      if (e.post >= 20 && e.post < 36) rgx(px, py, (e.post - 20) * 5, '#fff', 1 - (e.post - 20) / 16, 3); }); };
  DR.x_dvoid = e => { const cx = e.x, hy = 90, op = e.t > e.hf * .5, ph = Math.min(1, e.t / (e.hf * .5)), vis = e.post ? e.fd : Math.min(1, e.t / 30);
    ctx.globalCompositeOperation = 'source-over'; ctx.globalAlpha = .3 * vis; ctx.fillStyle = '#05010f'; ctx.fillRect(0, 0, W, H);
    ctx.globalAlpha = vis * .95; dragonHead(e, cx, hy, op ? 12 : 3, '#0b0414', '#b57cff'); ctx.globalCompositeOperation = 'lighter';
    rgx(cx, hy + 20, 26 + (op ? 6 : 0), e.c1, vis, 2);
    const orb = op ? 6 + 38 * Math.min(1, (e.t - e.hf * .5) / (e.hf * .5)) : 4 * ph; if (!e.post) { ctx.globalCompositeOperation = 'source-over'; ctx.globalAlpha = vis; ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(cx, hy + 36, Math.max(1, orb), 0, 7); ctx.fill(); ctx.globalCompositeOperation = 'lighter'; rgx(cx, hy + 36, orb + 3, e.c1, vis, 2); }
    if (e.post) { const w = 22 * e.fd + 6; ctx.globalAlpha = .6 * e.fd; ctx.fillStyle = '#6a1cc0'; ctx.fillRect(cx - w, hy + 40, w * 2, H); ctx.globalAlpha = e.fd; ctx.fillStyle = e.c1; ctx.fillRect(cx - w * .3, hy + 40, w * .6, H); ctx.globalCompositeOperation = 'source-over'; ctx.globalAlpha = .9 * e.fd; ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(cx, e.cy0 + 120, Math.max(1, 70 * e.fd), 0, 7); ctx.fill(); ctx.globalCompositeOperation = 'lighter'; rgx(cx, e.cy0 + 120, 70 * e.fd + e.post * 3, e.c2, e.fd, 4); } };
  DR.x_astral = e => { const cx = e.x, cy = e.y - 100, s = Math.min(1, e.t / (e.hf * .6)), vis = e.post ? e.fd : 1;
    for (let i = 0; i < 9; i++) { const tx = cx + (i - 4) * 14, ty = cy - (i % 2 ? 10 : 30) - (i === 4 ? 14 : 0), sx = e.R(i) * W, sy = e.R(i + 3) * H * .6, px = sx + (tx - sx) * s, py = sy + (ty - sy) * s; ctx.globalAlpha = vis; ctx.fillStyle = e.c1; ctx.fillRect(px - 3, py - .6, 6, 1.2); ctx.fillRect(px - .6, py - 3, 1.2, 6); glw(px, py, 8, e.c2, .6 * vis); }
    if (s >= 1) { lnx(cx - 56, cy + 2, cx + 56, cy + 2, 2, e.c1, .8 * vis); glw(cx, cy - 20, 40, e.c1, .35 * vis); }
    if (e.t > e.hf * .5 && !e.post) for (let i = 0; i < 20; i++) { const sx = e.R(i + 7) * W, sy = ((e.t - e.hf * .5) * 8 * (.6 + e.R(i)) + e.R(i + 2) * 200) % H; lnx(sx, sy - 26, sx, sy, 1.4, '#fff', .6); }
    if (e.post) { for (let i = 0; i < 70; i++) { const sx = e.R(i + 7) * W, sy = (e.post * 22 * (.5 + e.R(i)) + e.R(i + 2) * H) % H; lnx(sx, sy - 34, sx, sy, 1.4, i % 3 ? '#fff' : e.c2, e.fd); } const w = 16 * e.fd + 3; ctx.globalAlpha = .6 * e.fd; ctx.fillStyle = e.c1; ctx.fillRect(cx - w, 0, w * 2, cy); rgx(W / 2, e.cy0 + 60, e.post * 10, e.c1, e.fd, 4); } };
  DR.x_zero = e => { const cx = W / 2, cy = e.cy0 + 20;
    if (e.t < e.hf) { const k = e.t / e.hf; for (let i = 0; i < 6; i++) rgx(cx, cy, Math.max(2, (220 - i * 28) * (1 - k * .95)), i % 2 ? e.c2 : e.c1, .3 + k * .6, 2);
      for (let i = 0; i < 5; i++) { const r = e.R(i + Math.floor(e.t / 3)); ctx.globalAlpha = .18 * k; ctx.fillStyle = '#fff'; ctx.fillRect((r - .5) * 30, 40 + r * (H - 120), W, 6 + r * 8); }
      glw(cx, cy, 6 + 10 * k, '#fff', .9);
    } else { const p2 = e.post; if (p2 < 4) { ctx.globalAlpha = .55; ctx.fillStyle = '#fff'; ctx.fillRect(0, 0, W, H); }
      for (let i = 0; i < 24; i++) { const q = i * .2618; lnx(cx + Math.cos(q) * p2 * 5, cy + Math.sin(q) * p2 * 5, cx + Math.cos(q) * (p2 * 9 + 20), cy + Math.sin(q) * (p2 * 9 + 20), 2, i % 2 ? e.c2 : '#fff', e.fd); }
      rgx(cx, cy, p2 * 12, '#fff', e.fd, 5); rgx(cx, cy, p2 * 8, e.c2, e.fd, 3); } };

  function drawG(a) {
    const t = a.t, hf = a.hf, T = a.T, ch = Math.min(1, t / hf), post = Math.max(0, t - hf), fd = t > hf ? Math.max(0, 1 - post / (T - hf)) : 1;
    const R = i => { const s = Math.sin(i * 12.9898 + a.seed) * 43758.5453; return s - Math.floor(s); };
    const env = { a, t, hf, T, x: a.x, y: a.y, ch, post, fd, c1: a.cc[0], c2: a.cc[1], c3: a.cc[2], v: a.v, sd: a.sd, el: a.el, cy0: H * .34, R };
    ctx.save(); ctx.globalCompositeOperation = 'lighter'; ctx.lineCap = 'round'; ctx.lineJoin = 'round';
    const f = DR[a.st] || DR.nova; f(env); ctx.restore();
  }

  // --- pouvoirs spéciaux (automatiques ~7 s) : visuel selon la famille du skin ---
  const PWKIND = { nova: 'burst', beam: 'beam', comet: 'dart', wave: 'ring', leaf: 'leaf', lances: 'spike', stellar: 'stars', missile: 'dart', dive: 'dart', blades: 'slash', drones: 'bits', flame: 'fire', prism: 'rays', feather: 'feather', meteor: 'burst', hole: 'pull', clock: 'tick', moon: 'crescent', portal: 'ringx', rift: 'crack', crystal: 'spike', bolts: 'bolt', spiral: 'stars', orbit: 'bits', chain: 'bolt', pillar: 'beam', tornado: 'leaf', laser: 'beam', rain: 'stars', x_cosmos: 'stars', x_void: 'pull', x_cel: 'rays', x_drg: 'bite', x_thun: 'bolt', x_inf: 'fire', x_chr: 'tick', x_ser: 'feather', x_emp: 'ringx', x_dim: 'crack', x_dvoid: 'bite', x_astral: 'stars', x_zero: 'ringx' };
  const UKP = { celemp: 'rays', cosmic: 'stars', void: 'pull', phx: 'fire', drg: 'bite', spirit: 'slash', sam: 'slash', pika: 'bolt', arch: 'feather', demonk: 'crack', forest: 'leaf', palm: 'ring', katon: 'fire' };
  const PWNAME = { burst: 'ÉCLAT', beam: 'RAYON COURT', dart: 'SALVE', ring: 'ONDE', leaf: 'TOURBILLON DE FEUILLES', spike: 'PIQUES', stars: "POUSSIÈRE D'ÉTOILES", slash: 'SLASH COURT', bits: 'ESSAIM', fire: 'FLAMME', rays: 'RAYONS', feather: 'PLUMES', pull: 'ATTRACTION', tick: 'DISTORSION TEMPORELLE', crescent: 'CROISSANT', ringx: 'PORTAIL', crack: 'FISSURE', bite: "MORSURE D'ÉNERGIE", bolt: 'DÉCHARGE ÉLECTRIQUE' };
  const pwKind = sk => UKP[UK[sk.id]] || PWKIND[gInfo(sk).st] || 'burst';
  const powerName = sk => (PWNAME[pwKind(sk)] || 'ÉCLAT') + ' · ' + sk.n.toUpperCase();
  function usePower(second) {
    const sk = skin(), k = pwKind(sk), cc = ULTCOL(sk), t = isTest(); if (state !== 'play' || mg) return; if (!second && p.pw > 0 && !t) return;
    if (!second) { p.pw = t ? 20 : Math.round(420 * p.cdr); if (p.pw2 && !t) p.pw2t = Math.round(p.pw / 2); }
    const d = (4 + wave * .5) * (1 + sk.r * .12) * p.pwUp; let n = 0;
    for (const e of enemies) if ((t || Math.hypot(e.x - p.x, e.y - p.y) < 115) && !protectd(e)) {
      e.hp -= d; e.flash = 3; if (n++ < 5) floats.push({ x: e.x, y: e.y, l: 24, t: '-' + Math.round(d) });
      if (k === 'pull' && e.k < 3 && !e.mini && !e.tst) { e.x += (p.x - e.x) * .12; e.y += (p.y - e.y) * .06; }
    }
    if (k === 'tick' && !t) p.slow = Math.max(p.slow, 90);
    AM.sfx('pop'); const i = SKINS.indexOf(sk); if (rings.length < 30) rings.push({ x: p.x, y: p.y, r: 6, max: 115, c: cc[0] }); boom(p.x, p.y, 8 + sk.r, [cc[0], '#fff']);
    if (uan.length > 6) uan.shift(); uan.push({ k: 'pw', pk: k, T: 22, t: 0, hf: 999, c: cc[0], c2: cc[1], r: sk.r, x: p.x, y: p.y, sd: i % 2 ? 1 : -1, nv: 5 + i % 4, zs: [] });
  }
  function drawPW(a) {
    const t = a.t, al = Math.max(0, 1 - t / a.T), x = a.x, y = a.y, c = a.c, c2 = a.c2, k = a.pk, r = 16 + t * 5.5, n = a.nv, sd = a.sd;
    ctx.save(); ctx.globalCompositeOperation = 'lighter'; ctx.lineCap = 'round'; ctx.globalAlpha = al;
    const spokes = (inner, outer, w, col) => { for (let i = 0; i < n; i++) { const q = i * 6.283 / n + t * .05 * sd; lnx(x + Math.cos(q) * inner, y + Math.sin(q) * inner, x + Math.cos(q) * outer, y + Math.sin(q) * outer, w, col, al); } };
    switch (k) {
      case 'bolt': ctx.globalAlpha = al; for (let i = 0; i < n; i++) { const q = i * 6.283 / n, r2 = r + 18; bolt(x + Math.cos(q) * 14, y + Math.sin(q) * 14, x + Math.cos(q) * r2, y + Math.sin(q) * r2, .9, c); } break;
      case 'slash': ctx.strokeStyle = c; ctx.lineWidth = 5 * al + 1; ctx.beginPath(); ctx.arc(x, y, r * .8, -1.2 * sd, -1.2 * sd + 2.4 * sd * Math.min(1, t / 8), sd < 0); ctx.stroke(); ctx.strokeStyle = '#fff'; ctx.lineWidth = 1.5; ctx.stroke(); break;
      case 'ring': ctx.strokeStyle = c; ctx.lineWidth = 4 * al + 1; ctx.beginPath(); ctx.ellipse(x, y, r, r * .6, 0, 0, 7); ctx.stroke(); break;
      case 'leaf': for (let i = 0; i < n + 3; i++) { const q = t * .2 * sd + i * 6.283 / (n + 3), rr = 12 + t * 3.4; ctx.fillStyle = i % 2 ? c : c2; ctx.beginPath(); ctx.ellipse(x + Math.cos(q) * rr, y + Math.sin(q) * rr, 6, 2.6, q, 0, 7); ctx.fill(); } break;
      case 'spike': for (let i = 0; i < n; i++) { const q = i * 6.283 / n + .3, rr = Math.min(r, 70); ctx.fillStyle = i % 2 ? c : c2; ctx.beginPath(); ctx.moveTo(x + Math.cos(q) * 12, y + Math.sin(q) * 12); ctx.lineTo(x + Math.cos(q) * rr, y + Math.sin(q) * rr); ctx.lineTo(x + Math.cos(q + .2) * 14, y + Math.sin(q + .2) * 14); ctx.fill(); } break;
      case 'stars': for (let i = 0; i < n * 2; i++) { const q = i * 6.283 / (n * 2) + t * .04 * sd, rr = 12 + t * 4; ctx.fillStyle = i % 2 ? c : '#fff'; ctx.fillRect(x + Math.cos(q) * rr - 1.5, y + Math.sin(q) * rr - 1.5, 3, 3); } break;
      case 'bits': for (let i = 0; i < n; i++) { const q = t * .2 * sd + i * 6.283 / n, rr = 30; ctx.fillStyle = i % 2 ? c : c2; ctx.fillRect(x + Math.cos(q) * rr - 3, y + Math.sin(q) * rr - 3, 6, 6); lnx(x, y, x + Math.cos(q) * rr, y + Math.sin(q) * rr, 1, c, al * .4); } rgx(x, y, 30 + t, c, al * .5, 1.5); break;
      case 'fire': for (let i = 0; i < n; i++) { const q = i * 6.283 / n, rr = 14 + t * 3; glw(x + Math.cos(q) * rr, y + Math.sin(q) * rr - t * 1.2, 11, i % 2 ? c : c2, al); } break;
      case 'rays': spokes(14, r + 16, 2.5, c); break;
      case 'feather': for (let i = 0; i < n; i++) { const q = i * 6.283 / n + t * .05, rr = 14 + t * 3; ctx.fillStyle = i % 2 ? '#fff' : c; ctx.beginPath(); ctx.ellipse(x + Math.cos(q) * rr, y + Math.sin(q) * rr - t, 2.4, 7, q, 0, 7); ctx.fill(); } break;
      case 'pull': ctx.strokeStyle = c; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(x, y, Math.max(2, 70 - t * 3), 0, 7); ctx.stroke(); spokes(Math.max(2, 70 - t * 3), Math.max(2, 70 - t * 3) + 14, 2, c2); break;
      case 'tick': rgx(x, y, 34, c, al, 2.5); lnx(x, y, x + Math.cos(t * .5 * sd) * 28, y + Math.sin(t * .5 * sd) * 28, 3, '#fff', al); lnx(x, y, x + Math.cos(t * .04) * 18, y + Math.sin(t * .04) * 18, 4, c, al); rgx(x, y, 34 + t * 2, c2, al * .6, 2); break;
      case 'crescent': ctx.strokeStyle = c; ctx.lineWidth = 6 * al + 1; ctx.beginPath(); ctx.arc(x, y, Math.min(r, 90), -2.2, -.9); ctx.stroke(); ctx.strokeStyle = '#fff'; ctx.lineWidth = 1.5; ctx.stroke(); break;
      case 'ringx': for (let m = 0; m < 2; m++) { ctx.strokeStyle = m ? c2 : c; ctx.lineWidth = 3; ctx.beginPath(); ctx.ellipse(x, y, Math.max(1, r - m * 12), Math.max(1, (r - m * 12) * 1.3), t * .1 * sd, 0, 7); ctx.stroke(); } break;
      case 'crack': ctx.strokeStyle = c; ctx.lineWidth = 2.5; for (let i = 0; i < 3; i++) { ctx.beginPath(); ctx.moveTo(x, y); const q = i * 2.1 + a.nv; for (let j = 1; j < 5; j++) ctx.lineTo(x + Math.cos(q + (j % 2 ? .2 : -.2)) * j * (r / 4), y + Math.sin(q + (j % 2 ? .2 : -.2)) * j * (r / 4)); ctx.stroke(); } break;
      case 'bite': ctx.fillStyle = c; for (const d of [-1, 1]) { ctx.beginPath(); ctx.moveTo(x, y - 26 + d * (1 - Math.min(1, t / 8)) * -40); ctx.lineTo(x - 18, y - 4 * d); ctx.lineTo(x + 18, y - 4 * d); ctx.fill(); } glw(x, y, 24 + t * 2, c2, al * .7); break;
      case 'beam': ctx.fillStyle = c; ctx.fillRect(x - 6 * al - 2, y - 150, 12 * al + 4, 150); ctx.fillStyle = '#fff'; ctx.fillRect(x - 1.5, y - 150, 3, 150); break;
      case 'dart': for (let i = 0; i < n; i++) { const dx = (i - (n - 1) / 2) * 12; lnx(x + dx, y - 10 - t * 6, x + dx * 1.6, y - 40 - t * 9, 2.5, i % 2 ? c : c2, al); } break;
      default: glw(x, y, r * 1.4, c, al * .8); spokes(8, r, 2, c2);
    }
    ctx.restore();
  }

  // --- effets de destruction par skin / rareté ---
  const FXS = { elec: 'l', fire: 'f', star: 's', void: 'q', spark: 'q', ghost: 'g', angel: 'e', gold: 's', demon: 'f', sam: 'l' };
  function killFx(e) {
    const sk = skin(); killLv(e, sk); if (sk.r < 1 && !sk.ext) return;
    const fx = sk.dk || sk.fx, cos = sk.cos, base = FXC[fx], cols = (sk.ext || !base) ? [(cos && cos.r1) || sk.flame || '#fff', (cos && cos.r2) || sk.wing || '#fff', '#fff'] : base, sh = FXS[fx] || 'q';
    const n = Math.max(3, Math.min(sk.r >= 9 ? 24 : 16, 3 + sk.r * 2) >> sv.o.part);
    for (let i = 0; i < n && parts.length < PCAP(); i++) { const q = Math.random() * 6.283, v = rnd(1, sk.r >= 6 ? 4.4 : 3.2); parts.push({ x: e.x, y: e.y, vx: Math.cos(q) * v, vy: Math.sin(q) * v - (fx === 'angel' || fx === 'fire' ? .8 : 0), life: rnd(22, 40), max: 40, r: rnd(1.5, 3.5) + (sk.r >= 6 ? 1 : 0), c: cols[(Math.random() * cols.length) | 0], s: sh }); }
    if (rings.length < 30) { rings.push({ x: e.x, y: e.y, r: 4, max: 30 + sk.r * 5, c: cols[0] }); if (sk.r >= 6) rings.push({ x: e.x, y: e.y, r: 2, max: 50 + sk.r * 4, c: cols[1] }); }
    if (sk.r >= 8) for (let i = 0; i < 8 && parts.length < PCAP(); i++) { const q = i * .785; parts.push({ x: e.x, y: e.y, vx: Math.cos(q) * 5, vy: Math.sin(q) * 5, life: 16, max: 16, r: 2, c: '#fff', s: 'l' }); }
  }
  function drawPart(a) {
    const k = Math.max(0, a.life / a.max); ctx.globalAlpha = k; ctx.fillStyle = a.c; ctx.strokeStyle = a.c;
    if (a.s === 'l') { ctx.lineWidth = 1.6; ctx.beginPath(); ctx.moveTo(a.x, a.y); ctx.lineTo(a.x - a.vx * 3, a.y - a.vy * 3); ctx.stroke(); }
    else if (a.s === 'q') ctx.fillRect(a.x - a.r * k, a.y - a.r * k, a.r * 2 * k + 1, a.r * 2 * k + 1);
    else if (a.s === 's') { ctx.fillRect(a.x - a.r, a.y - .5, a.r * 2, 1); ctx.fillRect(a.x - .5, a.y - a.r, 1, a.r * 2); }
    else if (a.s === 'e') { ctx.beginPath(); ctx.ellipse(a.x, a.y, 1.6, 4.5, a.vx, 0, 7); ctx.fill(); }
    else { ctx.beginPath(); ctx.arc(a.x, a.y, a.r * (k + .3), 0, 7); ctx.fill(); }
  }

  // ===================== ULTIMATE UPDATE : interface =====================
  const pwPos = () => isTest() ? { x: W - 44, y: H - 290 } : { x: W - 48, y: H - 140 };
  const dashPos = () => isTest() ? { x: 44, y: H - 226 } : { x: 48, y: H - 72 };
  function drawBtns() {
    const btn = (c, label, prog, col) => {
      ctx.save(); ctx.fillStyle = 'rgba(18,41,63,.62)'; ctx.beginPath(); ctx.arc(c.x, c.y, 29, 0, 7); ctx.fill();
      ctx.strokeStyle = prog >= 1 ? col : '#7fe3ff'; ctx.lineWidth = 5; ctx.beginPath(); ctx.arc(c.x, c.y, 25, -1.57, -1.57 + 6.283 * Math.max(.001, Math.min(1, prog))); ctx.stroke();
      ctx.fillStyle = '#fff'; ctx.font = `bold 10px ${FONT}`; ctx.textAlign = 'center'; ctx.textBaseline = 'middle'; ctx.fillText(label, c.x, c.y + 1); ctx.restore();
    };
    btn(pwPos(), 'POUV.', p.pw > 0 ? 1 - p.pw / 420 : 1, '#ffd966'); btn(dashPos(), 'DASH', dashCd > 0 ? 1 - dashCd / 150 : 1, '#bde9ff');
  }
  function drawWorldMeta() {
    if (portalPt) { ctx.save(); for (let m = 0; m < 3; m++) { ctx.globalAlpha = .8 - m * .2; ctx.strokeStyle = m % 2 ? '#b57cff' : '#7a4cff'; ctx.lineWidth = 3; ctx.beginPath(); ctx.ellipse(portalPt.x, portalPt.y, Math.max(1, 30 - m * 7 + Math.sin(time * .1) * 2), Math.max(1, 44 - m * 9), time * .03 * (m % 2 ? 1 : -1), 0, 7); ctx.stroke(); } ctx.restore(); }
  }
  function hudExtra() {
    ctx.save(); ctx.textAlign = 'center'; ctx.textBaseline = 'alphabetic';
    const bar = (txt, f, col) => { ctx.fillStyle = 'rgba(18,41,63,.65)'; ctx.beginPath(); ctx.roundRect(W / 2 - 78, 48, 156, 16, 8); ctx.fill(); ctx.fillStyle = col; ctx.beginPath(); ctx.roundRect(W / 2 - 76, 50, Math.max(4, 152 * Math.max(0, Math.min(1, f))), 12, 6); ctx.fill(); ctx.fillStyle = '#fff'; ctx.font = `bold 10px ${FONT}`; ctx.fillText(txt, W / 2, 60); };
    if (rush) bar('⚡ RUSH ' + Math.ceil(rush.t / 60) + 's · ' + rush.k + ' kills · ×' + (2 * (1 + Math.min(.5, rush.k * .01))).toFixed(2), rush.t / rush.max, '#ff7a3d');
    else if (evt && EVS[evt.k]) bar(EVS[evt.k][0] + ' ' + Math.ceil(evt.t / 60) + 's', evt.t / evt.max, EVS[evt.k][3]);
    else if (brOn) bar('👑 BOSS RUSH · ' + (brQ + enemies.filter(e => e.brush).length) + ' restants', .5, '#ff3d7f');
    if (p.cbT > 0) { ctx.textAlign = 'right'; ctx.fillStyle = '#ffd966'; ctx.font = `bold 13px ${FONT}`; ctx.fillText('◎ ×' + p.cb + ' · ' + Math.ceil(p.cbT / 60) + 's', W - 14, 72); }
    if (xpPop) { ctx.textAlign = 'center'; ctx.globalAlpha = Math.min(1, xpPop.t / 25); ctx.fillStyle = '#d6b8ff'; ctx.font = `bold 14px ${FONT}`; ctx.fillText('⭐ XP +' + xpPop.n, W / 2, H - 92 - (70 - xpPop.t) * .3); ctx.globalAlpha = 1; }
    if (pdMsg) { ctx.globalAlpha = Math.min(1, pdMsg.t / 15); ctx.fillStyle = '#ffe45e'; ctx.font = `bold 16px ${FONT}`; ctx.textAlign = 'center'; ctx.fillText('⚡ ESQUIVE PARFAITE' + (pdMsg.n > 1 ? '  ×' + pdMsg.n : ''), W / 2, H * .55 - (45 - pdMsg.t) * .5); ctx.globalAlpha = 1; }
    if (evBan) { const a = Math.min(1, evBan.t / 30, (170 - evBan.t) / 12 + .2); ctx.globalAlpha = Math.max(0, a); ctx.fillStyle = evBan.c; ctx.font = `bold 24px ${FONT}`; ctx.fillText(evBan.a, W / 2, H * .2); ctx.fillStyle = '#fff'; ctx.font = `bold 13px ${FONT}`; ctx.fillText(evBan.b, W / 2, H * .2 + 22); ctx.globalAlpha = 1; }
    ctx.restore();
  }

  // ---- pages Coffres / Bonus ----
  function drawChest() {
    const rows = BR.map((n, r) => ({ t: 'Coffre ' + n, s: 'Possédés : ' + sv.ch[r] + (sv.ch[r] ? '  ·  touche pour ouvrir' : ''), r: sv.ch[r] ? 'OUVRIR' : '—', c: BRC[r], dim: !sv.ch[r] }));
    rows.push({ t: '🎫 Tickets : ' + sv.tk, s: '5 tickets → 1 coffre RARE', r: '5 🎫', c: '#2f6fc4', dim: sv.tk < 5 }, { t: 'Échange épique', s: '15 tickets → 1 coffre ÉPIQUE', r: '15 🎫', c: '#a055e0', dim: sv.tk < 15 });
    drawList('COFFRES', 'Boss, Rush, événements, niveaux…', rows);
    if (chestMsg) {
      ctx.fillStyle = 'rgba(8,18,32,.86)'; ctx.fillRect(0, 0, W, H); ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
      ctx.fillStyle = BRC[chestMsg.r]; ctx.font = `bold 22px ${FONT}`; ctx.fillText('COFFRE ' + BR[chestMsg.r], W / 2, 150);
      chestMsg.l.forEach((s, i) => { const sp = s[0] === '!'; ctx.fillStyle = sp ? '#ff9ae6' : '#fff'; ctx.font = `bold ${sp ? 16 : 14}px ${FONT}`; ctx.fillText(sp ? s.slice(1) : s, W / 2, 200 + i * 30); });
      ctx.fillStyle = '#9fb4c8'; ctx.font = `12px ${FONT}`; ctx.fillText('Touche pour continuer', W / 2, 220 + chestMsg.l.length * 30); ctx.textBaseline = 'alphabetic';
    }
  }
  function chestTap(q) {
    if (chestMsg) { chestMsg = null; return; }
    const off = q.y - 112 - shopY, i = Math.floor(off / 60); if (off < 0 || off % 60 > 52) return;
    if (i >= 0 && i < 6) { const l = openChest(i); if (l) { chestMsg = { r: i, l }; AM.sfx(i >= 3 ? 'rare' : 'win'); } else AM.sfx('hit'); }
    else if (i === 6 && sv.tk >= 5) { sv.tk -= 5; sv.ch[1]++; save(); AM.sfx('coin'); }
    else if (i === 7 && sv.tk >= 15) { sv.tk -= 15; sv.ch[2]++; save(); AM.sfx('coin'); }
    else AM.sfx('hit');
  }
  function drawBonus() {
    const rows = PB.map(b => { const l = sv.pb[b[0]] || 0; return { t: b[1] + '  ' + (l ? 'niv ' + l + '/' + b[4] : 'verrouillé'), s: BR[b[2]] + ' · ' + b[3], r: l ? '✓' : '—', c: BRC[b[2]], dim: !l }; });
    drawList('BONUS PERMANENTS', 'Gagnés dans les coffres · actifs à chaque partie', rows);
  }

  // ---- Mode Test complet ----
  const markT = () => { if (isTest()) testedNow[testSk] = 1; };
  function jumpR(d) { const cur = skin().r, N = SKN.length; for (let s = 1; s <= N; s++) { const r = ((cur + d * s) % N + N) % N, i = SKINS.findIndex(k => k.r === r); if (i >= 0) { testSk = SKINS[i].id; AM.sfx('equip'); p.sl = 1; uan = []; return; } } }
  const TBT = [
    ['◂ SKIN', () => cycSkin(-1)], ['SKIN ▸', () => cycSkin(1)], ['ÉVOL ', () => { p.lvl = p.lvl % 7 + 1; p.power = Math.max(p.power, minP()); p.xl = p.lvl; AM.sfx('up'); rings.push({ x: p.x, y: p.y, r: 8, max: 120, c: '#ffd966' }); }],
    ['◂◂ RARETÉ', () => jumpR(-1)], ['RARETÉ ▸▸', () => jumpR(1)], ['NIVEAU ', () => { p.sl = p.sl % 7 + 1; AM.sfx('rare'); rings.push({ x: p.x, y: p.y, r: 8, max: 120, c: '#b57cff' }); }],
    ['ATTAQUE', () => { markT(); for (let i = 0; i < 3; i++) shoot(); }], ['POUVOIR', () => { markT(); usePower(); }], ['ULTIME', () => { markT(); ult = 100; ultT = 0; fireUlt(); }],
    ['EFFETS', () => { for (const e of enemies) if (e.tst) { killFx(e); boom(e.x, e.y, 14, ['#ff7a3d', '#ffd966', '#fff']); } }],
    ['ÉVÉNEMENT', () => { const ks = Object.keys(EVS), d = EVS[ks[(Math.random() * ks.length) | 0]]; evBan = { a: d[0], b: d[1], c: d[3], t: 150 }; }], ['QUITTER', () => { inRun = false; toMenu(); }]];
  const tbRect = i => ({ x: 8 + (i % 3) * 116, y: H - 140 + ((i / 3) | 0) * 32, w: 112, h: 28 });
  function cycSkin(d) { const i = SKINS.findIndex(k => k.id === testSk); testSk = SKINS[(i + d + SKINS.length) % SKINS.length].id; AM.sfx('equip'); p.sl = 1; uan = []; if (skin().dk) { equipFx = 60; equipSk = skin(); } }
  function testTap(q) { for (let i = 0; i < TBT.length; i++) if (inRect(q, tbRect(i))) { TBT[i][1](); return true; } return false; }
  function drawTestPanel() {
    const sk = skin(), idx = SKINS.findIndex(k => k.id === sk.id) + 1;
    ctx.fillStyle = 'rgba(18,41,63,.72)'; ctx.fillRect(0, H - 196, W, 196);
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    ctx.fillStyle = '#fff'; ctx.font = `bold 12px ${FONT}`; ctx.fillText('SKIN : ' + sk.n.toUpperCase() + ' · ' + SKN[sk.r] + ' (' + idx + '/' + SKINS.length + ')', W / 2, H - 182);
    ctx.fillStyle = '#e8d9ff'; ctx.fillText('ULTIME : ' + ultOf(sk).n, W / 2, H - 167);
    ctx.fillStyle = '#ffe9a0'; ctx.fillText('POUVOIR : ' + powerName(sk), W / 2, H - 152);
    TBT.forEach((b, i) => pill(tbRect(i), b[0] + (i === 2 ? p.lvl : i === 5 ? p.sl : ''), i === 11 ? '#c0392b' : i === 8 ? '#7a4fe0' : '#2f6fc4'));
    ctx.textBaseline = 'alphabetic';
  }

  function init() {
    p = { x: W / 2, y: H - 110, lives: 3, power: 1, shield: 0, rapid: 0, x2: 0, wing: 0, slow: 0, inv: 0, cd: 0, lvl: 1, earned: 0, kills: 0, bc: 0, sl: 1, luck: 1, fort: 0, mega: 0, got: {}, myth: 0, exo: 0, xl: 1, stars: 0, varm: 0, vcd: 0, nova: 0, frac: 0, clone: 0, xp: 0, xpN: 25, dmg: 1, rate: 1, spd: 1, multi: 0, pierce: 0, crit: 0, bsz: 1, drone: 0, regen: 0, rg: 0, mag: 0, phx: 0, storm: 0, tilt: 0 };
    bullets = []; ebullets = []; enemies = []; pups = []; parts = []; floats = []; rings = []; warns = []; strikes = []; mg = null; freeze = 0; ctier = 0; mgRetry = 0;
    uan = []; resetRun(); score = 0; time = 0; shake = 0; wave = 0; wq = []; wt = 0; gap = 60; banner = 0; bombT = 0; bossName = ''; upT = 0; clr = 0; msgT = 0; combo = 0; comboT = 0;
  }
  clouds = [[.45, .15, .35, 5], [.8, .4, .55, 4], [1.25, .8, .7, 2]].flatMap(([s, v, a, n]) => Array.from({ length: n }, () => ({ x: Math.random() * W, y: Math.random() * 700, s, v, a })));
  init();

  // ---------- entrée ----------
  function pt(e) { return { x: (e.clientX - rect.left) / rect.width * W, y: (e.clientY - rect.top) / rect.height * H }; }
  cv.addEventListener('pointerdown', e => {
    cv.setPointerCapture(e.pointerId); const q = pt(e); AM.init();
    if (state === 'play') { if (inRect(q, PILL)) { state = 'pause'; target = null; } else if (isTest() && testTap(q)) { target = null; } else if (nearC(q, ultPos())) { fireUlt(); } else if (nearC(q, pwPos())) { usePower(); } else if (nearC(q, dashPos())) { doDash(); } else target = q; return; }
    if (['shop', 'col', 'ach', 'best', 'stat', 'prof', 'chest', 'bonus'].includes(state)) { sd = { y: q.y, sy: shopY, mv: 0 }; return; }
    tapUI(q);
  });
  cv.addEventListener('pointermove', e => {
    if (target) target = pt(e);
    if (sd) { const q = pt(e); if (Math.abs(q.y - sd.y) > 8) sd.mv = 1; if (sd.mv) shopY = Math.max(-shopMax, Math.min(0, sd.sy + q.y - sd.y)); }
  });
  cv.addEventListener('wheel', e => { if (['shop', 'col', 'ach', 'best', 'stat', 'prof', 'chest', 'bonus'].includes(state)) { shopY = Math.max(-shopMax, Math.min(0, shopY - e.deltaY)); e.preventDefault(); } }, { passive: false });
  const up = e => {
    target = null;
    if (sd && !sd.mv && e && e.clientX !== undefined) { const q = pt(e); if (inRect(q, backBtn())) state = backTo; else if (state === 'shop') shopTap(q); else if (state === 'chest') chestTap(q); }
    sd = null;
  };
  cv.addEventListener('pointerup', up); cv.addEventListener('pointercancel', up);
  document.addEventListener('touchmove', e => e.preventDefault(), { passive: false });
  addEventListener('keydown', e => { keys[e.key] = true; if (e.key === ' ' || e.key === 'Enter') { if (state === 'menu' || state === 'over') start(); } if (e.key === 'Escape' && state === 'play') { state = 'pause'; target = null; } else if (e.key === 'Escape' && state === 'pause') state = 'play'; });
  addEventListener('keyup', e => { keys[e.key] = false; });
  function start() { if (isTest()) exitTest(); setMode('PLAY'); init(); state = 'play'; inRun = true; sv.st.games++; runF = 0; nextWave(); }
  function toMenu() { if (isTest()) exitTest(); target = null; init(); state = 'menu'; }
  const PILL = { x: W - 44, y: 76, w: 36, h: 24 };
  const inRect = (q, r) => q.x >= r.x && q.x <= r.x + r.w && q.y >= r.y && q.y <= r.y + r.h;
  const menuBtn = () => ({ x: W / 2 - 60, y: H - 64, w: 120, h: 38 });
  if (!ctx.roundRect) ctx.roundRect = function (x, y, w, h) { this.rect(x, y, w, h); };
  const GAMES = [
    { n: 'Escadrille', d: 'Tir aérien · vagues et mini-boss', ok: 1 },
    { n: 'Briqueboum', d: 'Casse-briques explosif' },
    { n: 'Serpent Néon', d: 'Snake lumineux et rapide' },
    { n: 'Ver de Sable', d: 'Bac à sable : creuse et effondre' }
  ];
  function menuCards() {
    const h = Math.min(84, (H - 230) / GAMES.length - 12);
    return GAMES.map((g, i) => ({ g, x: 24, y: 130 + i * (h + 12), w: W - 48, h }));
  }
  function pill(r, label, col) {
    ctx.fillStyle = col || 'rgba(18,41,63,.72)'; ctx.beginPath(); ctx.roundRect(r.x, r.y, r.w, r.h, r.h / 2); ctx.fill();
    ctx.fillStyle = '#fff'; ctx.font = `bold 14px ${FONT}`; ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    ctx.fillText(label, r.x + r.w / 2, r.y + r.h / 2 + 1);
  }
  const HUBG = [{ id: 'air', n: 'AIR ARCADE', d: "Shoot'em up : vagues, boss, mini-jeux", go: () => { state = 'menu'; }, stat: () => 'Vague max ' + sv.st.bestWave + '  ·  Record ' + hi }];
  const hubCard = i => ({ x: 20, y: 120 + i * 170, w: W - 40, h: 150 });
  const hubProf = () => ({ x: 20, y: H - 90, w: 150, h: 46 }), hubOpt = () => ({ x: 190, y: H - 90, w: 150, h: 46 });
  const accLvl = () => { const t = sv.st, x = t.kills + t.games * 30 + t.bestWave * 40, l = 1 + Math.floor(Math.sqrt(x / 25)), a = (l - 1) ** 2 * 25, b = l * l * 25; return [l, (x - a) / (b - a)]; };
  function drawHub() {
    ctx.textBaseline = 'alphabetic'; ctx.textAlign = 'center'; ctx.fillStyle = '#fff'; ctx.font = `bold 30px ${FONT}`; ctx.fillText('ARCADE-BACKUP 4', W / 2, 66);
    ctx.font = `bold 14px ${FONT}`; ctx.fillStyle = '#ffd966'; ctx.fillText('◎ ' + sv.coins + '   ◆ ' + (sv.frag || 0) + '   ·   Niveau ' + accLvl()[0], W / 2, 92);
    HUBG.forEach((g, i) => {
      const c = hubCard(i), sc = 1 + (hubGo > 0 ? (15 - hubGo) * .004 : Math.sin(time * .05) * .004), L = 1 + ((time / 100) | 0) % 7;
      ctx.save(); ctx.translate(c.x + c.w / 2, c.y + c.h / 2); ctx.scale(sc, sc); ctx.translate(-c.x - c.w / 2, -c.y - c.h / 2);
      ctx.fillStyle = 'rgba(255,255,255,.94)'; ctx.beginPath(); ctx.roundRect(c.x, c.y, c.w, c.h, 18); ctx.fill();
      ship(c.x + 62, c.y + 78, 1.5, skin(), 4, Math.sin(time * .03) * .4, 4, L);
      ctx.textAlign = 'left'; ctx.fillStyle = '#12293f'; ctx.font = `bold 22px ${FONT}`; ctx.fillText(g.n, c.x + 122, c.y + 40);
      ctx.font = `12px ${FONT}`; ctx.fillStyle = '#3a4a5c'; ctx.fillText(g.d, c.x + 122, c.y + 60); ctx.fillText(g.stat(), c.x + 122, c.y + 78);
      pill({ x: c.x + 122, y: c.y + 92, w: 140, h: 40 }, 'JOUER', '#2f6fc4'); ctx.restore();
    });
    ctx.textAlign = 'center'; ctx.fillStyle = 'rgba(18,41,63,.6)'; ctx.font = `italic 13px ${FONT}`; ctx.fillText("D'autres jeux arriveront ici.", W / 2, 320);
    pill(hubProf(), 'PROFIL'); pill(hubOpt(), 'OPTIONS');
  }
  function drawProf() {
    const [l, pr] = accLvl(), t = sv.st;
    drawList('Profil', 'Niveau ' + l, [{ t: 'Niveau ' + l, s: 'Progression vers le niveau ' + (l + 1), r: Math.round(pr * 100) + ' %', p: pr }].concat([['Meilleure vague', t.bestWave], ['Meilleur score', t.bestScore], ['Ennemis détruits', t.kills], ['Temps joué', fmtT(t.time)], ['Skins', sv.owned.length + ' / ' + SKINS.length], ['Succès', Object.keys(sv.ach).length + ' / ' + ACH.length], ['◎ Monnaie', sv.coins], ['◆ Fragments', sv.frag || 0], ['🍀 Luck (max)', (t.luck || 1).toFixed(2)]].map(r => ({ t: r[0], r: String(r[1]) }))));
  }
  const jouerBtn = () => ({ x: 20, y: 300, w: 150, h: 56 }), testBtn = () => ({ x: 190, y: 300, w: 150, h: 56 });
  const MB = [['BOUTIQUE', 'shop'], ['COLLECTION', 'col'], ['SUCCÈS', 'ach'], ['BESTIAIRE', 'best'], ['STATISTIQUES', 'stat'], ['OPTIONS', 'opt'], ['COFFRES', 'chest'], ['BONUS', 'bonus']];
  const mbRect = i => ({ x: 16 + (i % 2) * 168, y: 364 + ((i / 2) | 0) * 50, w: 160, h: 42 });
  const stk = (i, y0) => ({ x: 70, y: H * y0 + i * 62, w: 220, h: 48 });
  function drawMenu() {
    const sk = skin(), L = 1 + ((time / 100) | 0) % 7;
    ctx.textBaseline = 'alphabetic'; ctx.textAlign = 'center'; ctx.fillStyle = '#fff'; ctx.font = `bold 36px ${FONT}`; ctx.fillText('ESCADRILLE', W / 2, 70);
    ship(W / 2, 165, 1.8, sk, 3, Math.sin(time * .03) * .4, 5, L);
    ctx.font = `bold 15px ${FONT}`; ctx.fillText(sk.n + (sk.cos ? ' — NIVEAU ' + L + '/7' : ''), W / 2, 238);
    ctx.fillStyle = '#ffd966'; ctx.fillText('◎ ' + sv.coins + '  ·  Record ' + hi + '  ·  Vague max ' + sv.st.bestWave, W / 2, 262);
    if (dailyMsg) { ctx.fillStyle = '#12293f'; ctx.font = `bold 13px ${FONT}`; ctx.fillText(dailyMsg, W / 2, 284); }
    pill(backBtn(), '◂ HUB'); pill(jouerBtn(), inRun ? 'REPRENDRE' : 'JOUER', '#2f6fc4'); pill(testBtn(), 'MODE TEST', '#7a4fe0'); MB.forEach((m, i) => pill(mbRect(i), m[0]));
  }
  function tapUI(q) {
    const hitI = (n, y0) => { for (let i = 0; i < n; i++) if (inRect(q, stk(i, y0))) return i; return -1; };
    if (state === 'hub') {
      HUBG.forEach((g, i) => { if (hubGo === 0 && inRect(q, hubCard(i))) { hubGo = 15; AM.sfx('equip'); } });
      if (inRect(q, hubProf())) { backTo = 'hub'; state = 'prof'; shopY = 0; } else if (inRect(q, hubOpt())) { backTo = 'hub'; state = 'opt'; }
    } else if (state === 'menu') {
      if (inRect(q, backBtn())) { state = 'hub'; return; }
      if (inRect(q, jouerBtn())) { if (inRun) state = 'resume'; else start(); return; }
      if (inRect(q, testBtn())) { startTest(); return; }
      MB.forEach((m, i) => { if (inRect(q, mbRect(i))) { backTo = 'menu'; state = m[1]; shopY = 0; if (m[1] === 'shop') { shopSel = sv.eq; shopR = skin().r; } } });
    } else if (state === 'resume') { const i = hitI(3, .3); if (i === 0) state = 'play'; else if (i === 1) start(); else if (i === 2) state = 'menu'; }
    else if (state === 'pause') {
      const i = hitI(6, .26);
      if (isTest() && (i === 3 || i === 4 || i === 5)) { const h = i === 5; inRun = false; toMenu(); if (h) state = 'hub'; return; }
      if (i === 0) state = 'play'; else if (i === 1) { backTo = 'pause'; state = 'opt'; } else if (i === 2) { backTo = 'pause'; state = 'col'; shopY = 0; }
      else if (i === 3) state = 'menu'; else if (i === 4) { endRun(); inRun = false; toMenu(); } else if (i === 5) state = 'hub';
    } else if (state === 'over') { const i = hitI(3, .58); if (i === 0) start(); else if (i === 1) toMenu(); else if (i === 2) { toMenu(); state = 'hub'; } }
    else if (state === 'conf') { const i = hitI(2, .3); if (i === 0) state = 'opt'; else if (i === 1) resetAll(); }
    else if (state === 'opt') optTap(q);
    else if (state === 'pick') { for (const c of pickCards()) if (inRect(q, c)) choose(c.u); }
  }
  function resetAll() {
    try { localStorage.removeItem('arcade-save'); localStorage.removeItem('escadrille-hi'); } catch (e) {}
    hi = 0; sv.coins = 0; sv.owned = ['classic']; sv.eq = 'classic'; sv.st = defSt(); sv.ach = {}; sv.bs = {}; sv.lv = {}; sv.mis = []; for (let i = 0; i < 3; i++) sv.mis.push(newMis());
    sv.pb = {}; sv.ch = [0, 0, 0, 0, 0, 0]; sv.tk = 0; sv.tested = {}; sv.axp = 0;
    inRun = false; save(); toMenu();
  }
  function modal(title, lines, labels, y0) {
    ctx.fillStyle = 'rgba(18,41,63,.75)'; ctx.fillRect(0, 0, W, H); ctx.textAlign = 'center'; ctx.textBaseline = 'alphabetic';
    ctx.fillStyle = '#fff'; ctx.font = `bold 32px ${FONT}`; ctx.fillText(title, W / 2, H * .13); ctx.font = `15px ${FONT}`;
    lines.forEach((l, i) => { const g2 = l[0] === '!'; ctx.fillStyle = g2 ? '#ffd966' : '#fff'; ctx.fillText(g2 ? l.slice(1) : l, W / 2, H * .13 + 32 + i * 24); });
    labels.forEach((l, i) => pill(stk(i, y0), l, i === 0 ? '#2f6fc4' : null));
  }
  function drawList(title, sub, rows) {
    ctx.textBaseline = 'alphabetic'; ctx.textAlign = 'center'; ctx.fillStyle = '#12293f';
    ctx.font = `bold 30px ${FONT}`; ctx.fillText(title, W / 2, 66); ctx.font = `bold 14px ${FONT}`; ctx.fillText(sub, W / 2, 92);
    shopMax = Math.max(0, rows.length * 60 - (H - 150));
    ctx.save(); ctx.beginPath(); ctx.rect(0, 104, W, H - 104); ctx.clip();
    rows.forEach((r, i) => {
      const y = 112 + shopY + i * 60; if (y < 50 || y > H) return;
      ctx.fillStyle = r.dim ? 'rgba(255,255,255,.45)' : 'rgba(255,255,255,.92)'; ctx.beginPath(); ctx.roundRect(16, y, W - 32, 52, 12); ctx.fill();
      if (r.p !== undefined) { ctx.fillStyle = 'rgba(18,41,63,.15)'; ctx.fillRect(24, y + 45, W - 48, 4); ctx.fillStyle = '#2f9b55'; ctx.fillRect(24, y + 45, (W - 48) * Math.min(1, r.p), 4); }
      ctx.textAlign = 'left'; ctx.fillStyle = '#12293f'; ctx.font = `bold 14px ${FONT}`; ctx.fillText(r.t, 26, y + 21);
      ctx.font = `12px ${FONT}`; ctx.fillStyle = '#3a4a5c'; ctx.fillText(r.s || '', 26, y + 39);
      ctx.textAlign = 'right'; ctx.font = `bold 14px ${FONT}`; ctx.fillStyle = r.c || '#12293f'; ctx.fillText(r.r || '', W - 26, y + 24);
    });
    ctx.restore(); pill(backBtn(), 'Retour');
  }
  const fmtT = t => Math.floor(t / 3600) + ' h ' + Math.floor(t % 3600 / 60) + ' min';
  function drawAch() {
    const ms = sv.mis.map(m => ({ t: 'Mission : ' + m.n, s: Math.min(mprog(m), m.g) + ' / ' + m.g, r: '+' + m.rw + ' ◎', c: '#1d8a4a', p: mprog(m) / m.g }));
    drawList('Succès', Object.keys(sv.ach).length + ' / ' + ACH.length + ' obtenus', ms.concat(ACH.map(a => ({ t: a[1], s: a[2] + ' · +' + a[3] + ' ◎', r: sv.ach[a[0]] ? '✓' : '🔒', c: sv.ach[a[0]] ? '#1d8a4a' : '#6a7a8c', dim: !sv.ach[a[0]] }))));
  }
  function drawBest() {
    drawList('Bestiaire', BEST.filter(b => b[0] in sv.bs).length + ' / ' + BEST.length + ' découverts', BEST.map(b => { const seen = b[0] in sv.bs; return { t: seen ? b[1] : '???', s: seen ? b[2] + ' (dès la vague ' + b[3] + ')' : 'Non découvert', r: seen ? '× ' + sv.bs[b[0]] : '', dim: !seen }; }));
  }
  function drawStat() {
    const t = sv.st, lv = Object.keys(sv.lv).reduce((a, k) => a + sv.lv[k], 0);
    drawList('Statistiques', 'Mis à jour en direct', [['Meilleure vague', t.bestWave], ['Meilleur score', t.bestScore], ['Ennemis détruits', t.kills], ['Boss vaincus', t.boss], ['Élites vaincus', t.elites], ['Parties jouées', t.games], ['Temps de jeu', fmtT(t.time)], ['◎ gagnés', t.coins], ['Meilleur combo', 'x' + t.bestCombo], ['Skins débloqués', sv.owned.length + ' / ' + SKINS.length], ['Succès obtenus', Object.keys(sv.ach).length + ' / ' + ACH.length], ['Niveaux de skins sup.', lv], ['Transformations', t.trans], ['Tirs cosmiques', t.shots], ['Mini-jeux réussis', t.mg || 0], ['◆ Fragments', sv.frag || 0], ['🍀 Luck (max)', (t.luck || 1).toFixed(2)]].map(r => ({ t: r[0], r: String(r[1]) })));
  }
  const backBtn = () => ({ x: 14, y: 14, w: 76, h: 30 });
  const FILT = { x: W - 138, y: 14, w: 126, h: 30 };
  function shopCards() {
    const w = (W - 60) / 2, out = []; shopHeads = []; let y = 118;
    for (let r = 0; r < SKN.length; r++) {
      const list = SKINS.filter(k => k.r === r && (shopF === 0 || (shopF === 1) === sv.owned.includes(k.id)));
      if (!list.length) continue;
      shopHeads.push({ r, y: y + shopY }); y += 28;
      list.forEach((k, i) => out.push({ k, x: 24 + (i % 2) * (w + 12), y: y + shopY + ((i / 2) | 0) * 128, w, h: 118 }));
      y += Math.ceil(list.length / 2) * 128 + 8;
    }
    shopTotal = y; return out;
  }
  function shopDeco() {
    shopMax = Math.max(0, shopTotal - H + 24); shopY = Math.max(-shopMax, shopY);
    ctx.textAlign = 'center'; ctx.font = `bold 13px ${FONT}`;
    for (const h of shopHeads) { ctx.fillStyle = SKR[h.r]; ctx.fillText('──  ' + SKN[h.r] + '  ──', W / 2, h.y + 18); }
    if (shopMax > 0) { const th = Math.max(30, (H - 120) * (H - 120) / shopTotal); ctx.fillStyle = 'rgba(18,41,63,.4)'; ctx.fillRect(W - 5, 118 + (-shopY / shopMax) * (H - 120 - th), 3, th); }
  }
  function drawCol() {
    ctx.textBaseline = 'alphabetic'; ctx.textAlign = 'center'; ctx.fillStyle = '#12293f';
    ctx.font = `bold 30px ${FONT}`; ctx.fillText('Collection', W / 2, 70);
    ctx.font = `bold 15px ${FONT}`; ctx.fillText(sv.owned.length + ' / ' + SKINS.length + ' skins obtenus', W / 2, 98);
    ctx.save(); ctx.beginPath(); ctx.rect(0, 108, W, H - 108); ctx.clip();
    for (const c of shopCards()) {
      const k = c.k, own = sv.owned.includes(k.id), dk = '#33425a';
      ctx.fillStyle = 'rgba(255,255,255,.88)'; ctx.beginPath(); ctx.roundRect(c.x, c.y, c.w, c.h, 14); ctx.fill();
      ctx.strokeStyle = SKR[k.r]; ctx.lineWidth = 1.5; ctx.stroke();
      ctx.fillStyle = SKR[k.r]; ctx.font = `bold 10px ${FONT}`; ctx.fillText(SKN[k.r], c.x + c.w / 2, c.y + 14);
      ship(c.x + c.w / 2, c.y + c.h * .45, 1.05, own ? k : { shape: k.shape, body: dk, wing: dk, glow: dk, flame: dk }, own ? 3 : 1, 0, 1, 1 + ((time / 100 | 0) % 5));
      ctx.fillStyle = '#12293f'; ctx.font = `bold 15px ${FONT}`; ctx.fillText(own ? k.n : '???', c.x + c.w / 2, c.y + c.h - 28);
      ctx.font = `13px ${FONT}`; ctx.fillText(own ? 'Obtenu' : 'À découvrir', c.x + c.w / 2, c.y + c.h - 10);
    }
    shopDeco();
    ctx.restore(); pill(backBtn(), 'Retour');
  }
  function shopTap(q) {
    if (inRect(q, FILT)) { shopF = (shopF + 1) % 3; shopY = 0; AM.sfx('equip'); return; }
    if (inRect(q, backBtn())) { state = backTo; return; }
    for (const c of shopCards()) if (q.y > 112 && inRect(q, c)) {
      const k = c.k;
      if (sv.owned.includes(k.id)) { sv.eq = k.id; AM.sfx('equip'); if (k.dk) { equipFx = 60; equipSk = k; } shopMsg = 'Skin équipé !'; shopT = 90; }
      else if (k.fp ? (sv.frag || 0) >= k.fp : sv.coins >= k.price) { if (k.fp) sv.frag -= k.fp; else sv.coins -= k.price; sv.owned.push(k.id); sv.eq = k.id; AM.sfx('buy'); if (k.dk) { equipFx = 60; equipSk = k; } shopMsg = 'Skin obtenu !'; shopT = 90; }
      else { shopMsg = k.fp ? 'Pas assez de ◆' : 'Pas assez de ◎'; shopT = 90; }
      save();
    }
  }
  function drawShop() {
    ctx.textBaseline = 'alphabetic'; ctx.textAlign = 'center'; ctx.fillStyle = '#12293f';
    ctx.font = `bold 34px ${FONT}`; ctx.fillText('Boutique', W / 2, 74);
    ctx.font = `bold 17px ${FONT}`; ctx.fillText('◎ ' + sv.coins + '   ◆ ' + (sv.frag || 0), W / 2, 100);
    ctx.save(); ctx.beginPath(); ctx.rect(0, 108, W, H - 108); ctx.clip();
    for (const c of shopCards()) {
      const k = c.k, own = sv.owned.includes(k.id), eq = sv.eq === k.id;
      ctx.fillStyle = 'rgba(255,255,255,.92)'; ctx.beginPath(); ctx.roundRect(c.x, c.y, c.w, c.h, 14); ctx.fill();
      ctx.strokeStyle = eq ? '#2f6fc4' : SKR[k.r]; ctx.lineWidth = eq ? 3 : 1.5; ctx.stroke();
      ctx.fillStyle = SKR[k.r]; ctx.font = `bold 10px ${FONT}`; ctx.fillText(SKN[k.r], c.x + c.w / 2, c.y + 14); ship(c.x + c.w / 2, c.y + c.h * .45, Math.min(1.1, c.h / 110), k, 3, 0, 1, 1 + ((time / 100 | 0) % 5));
      ctx.fillStyle = '#12293f'; ctx.font = `bold 15px ${FONT}`; ctx.fillText(k.n, c.x + c.w / 2, c.y + c.h - 28);
      ctx.font = `13px ${FONT}`; ctx.fillStyle = eq ? '#2f6fc4' : own ? '#12293f' : (k.fp ? (sv.frag || 0) >= k.fp : sv.coins >= k.price) ? '#1d8a4a' : '#9a3b3b';
      ctx.fillText(eq ? 'Équipé' : own ? 'Choisir' : k.fp ? '◆ ' + k.fp : '◎ ' + k.price, c.x + c.w / 2, c.y + c.h - 10);
    }
    shopDeco(); ctx.restore(); pill(backBtn(), 'Retour'); pill(FILT, ['TOUS', 'POSSÉDÉS', 'NON POSSÉDÉS'][shopF]);
    if (equipFx > 0) { const t = 1 - equipFx / 60, dm = equipSk.dk === 'demon'; ctx.fillStyle = `rgba(0,0,0,${.55 * Math.sin(t * 3.14)})`; ctx.fillRect(0, 0, W, H); ctx.globalAlpha = Math.min(1, t * 2.2); ship(W / 2, H / 2, 2.2 + t * .6, equipSk, 3, 0, 5, 3); ctx.globalAlpha = 1; ctx.strokeStyle = dm ? '#ff2a3a' : '#ffd966'; ctx.lineWidth = 4; ctx.beginPath(); ctx.arc(W / 2, H / 2, t * 200, 0, 7); ctx.stroke(); for (let i = 0; i < 16; i++) { const a = i * .393 + t * 2, r = t * 150 + 20; ctx.fillStyle = dm ? '#ff5a3a' : '#fff1b8'; ctx.fillRect(W / 2 + Math.cos(a) * r, H / 2 + Math.sin(a) * r, 3, 3); } }
    if (shopT > 0) { ctx.globalAlpha = Math.min(1, shopT / 20); ctx.fillStyle = '#12293f'; ctx.font = `bold 18px ${FONT}`; ctx.fillText(shopMsg, W / 2, H - 30); ctx.globalAlpha = 1; }
  }

  // ---------- utilitaires ----------
  const rnd = (a, b) => a + Math.random() * (b - a);
  const hit = (a, b, r) => (a.x - b.x) ** 2 + (a.y - b.y) ** 2 < r * r;
  const add = n => { if (!GM.allowRewards) return; score += n * (p.x2 > 0 ? 2 : 1); };
  function boom(x, y, n, cols) {
    for (let i = 0; i < n && parts.length < PCAP(); i++) {
      const a = Math.random() * 7, v = rnd(1, 5);
      parts.push({ x, y, vx: Math.cos(a) * v, vy: Math.sin(a) * v, life: rnd(20, 40), max: 40, r: rnd(2, 5), c: cols[(Math.random() * cols.length) | 0] });
    }
  }
  function shoot() {
    if ((sfxN++ & 3) === 0) AM.sfx('shoot'); const SK0 = skin(); if (SK0.cos) sv.st.shots++;
    if (SK0.dk) { shotFx = 8; for (let i = 0; i < 2 && parts.length < PCAP(); i++) parts.push({ x: p.x + rnd(-4, 4), y: p.y - 22, vx: rnd(-.6, .6), vy: -rnd(.5, 1.5), life: 14, max: 14, r: 2.5, c: SK0.dk === 'demon' ? '#ff4a5a' : '#ffe9a0' }); }
    const x = p.x, y = p.y - 18;
    const b = (dx, vx, yy = y) => bullets.push({ x: x + dx, y: yy, vx, vy: -10, d: p.dmg * (p.mega > 0 ? 2 : 1) * (Math.random() < p.crit ? 2 : 1), sz: p.bsz, pr: p.pierce, st: SK0.bul, hc: SK0.cos && SK0.cos.h });
    const n = p.power;
    if (n === 1) b(0, 0); else if (n === 2) { b(-10, 0); b(10, 0); } else if (n === 3) { b(0, 0); b(-9, -1.6); b(9, 1.6); } else { b(-13, 0); b(-4, 0); b(4, 0); b(13, 0); }
    for (let i = 0; i < p.multi; i++) { const sd = i % 2 ? 1 : -1, k = 1 + (i >> 1); b(sd * (12 + k * 5), sd * .9 * k); }
    if (p.wing > 0 || p.drone) { b(-38, 0, p.y + 8); b(38, 0, p.y + 8); }
    if (p.clone) bullets.push({ x: W - p.x, y, vx: 0, vy: -10, d: p.dmg * .5, sz: p.bsz, pr: p.pierce, st: SK0.bul, hc: SK0.cos && SK0.cos.h });
  }
  function aim(e, v, off = 0) {
    v *= 1 + Math.min(.4, ((e.tr || 1) - 1) * .03);
    const a = Math.atan2(p.y - e.y, p.x - e.x) + off;
    ebullets.push({ x: e.x, y: e.y + 10, vx: Math.cos(a) * v, vy: Math.sin(a) * v });
  }
  function ring(e, n, v, o = 0) {
    n = Math.min(n, 12 + (wave >> 3)); const g0 = (Math.random() * n) | 0;
    for (let i = 0; i < n; i++) { if ((i - g0 + n) % n < 2) continue; const a = o + i / n * 6.283; ebullets.push({ x: e.x, y: e.y, vx: Math.cos(a) * v, vy: Math.sin(a) * v }); }
  }
  function dropPup(x, y) { const t = Math.random() < .02 ? 'Y' : DROPS[(Math.random() * DROPS.length) | 0]; pups.push({ x, y, t, ph: 0 }); }

  // ---------- Mode Test ----------
  function startTest() {
    if (inRun) { endRun(); inRun = false; }
    testSnap = JSON.stringify(sv); testSk = sv.eq; setMode('TEST'); init();
    p.lives = 99; p.lvl = 1; ult = 100; ultT = 0; state = 'play'; inRun = true; runF = 0;
  }
  function exitTest() {
    const td = Object.assign({}, sv.tested, testedNow);
    if (testSnap) { const o = sv.o; sv = JSON.parse(testSnap); sv.o = o; sv.tested = Object.assign({}, sv.tested, td); }
    testSnap = null; setMode('PLAY'); ult = 0; ultT = 0; uan = []; testedNow = {}; if (Object.keys(sv.tested).length) { checkAll(); save(); }
  }
  function testEnv() {
    let n = 0; for (const e of enemies) if (e.tst) n++;
    for (; n < 4; n++) enemies.push({ k: 0, tst: 1, x: 50 + n * 85, y: 120 + (n % 2) * 40, hy: 120 + (n % 2) * 40, hp: 1e9, mhp: 1e9, vy: 2.4, ph: n, r: 14, cd: 1e9, dx: 0, tr: 1 });
    for (const e of enemies) { e.hp = 1e9; e.cd = 1e9; }
    ebullets.length = 0;
  }
  // ---------- Évolution visuelle (niveaux 1-7) ----------
  function levelUp(sk, L) {
    const c = (sk.cos && sk.cos.r1) || sk.flame || '#ffd966', nm = L >= 6 ? SLN[L - 1] : ((sk.cos && sk.cos.nm) || SLN)[L - 1];
    AM.sfx(L >= 6 ? 'win' : 'rare'); msg = { a: L >= 7 ? '✦ APOTHÉOSE ✦' : 'NIVEAU SUPÉRIEUR !', b: L >= 7 ? 'NIVEAU 7/7 · MAXIMUM' : nm + ' · NIVEAU ' + L + '/7' }; msgT = L >= 6 ? 210 : 150;
    const n = L >= 7 ? 5 : L >= 5 ? 3 : 2; for (let i = 0; i < n; i++) rings.push({ x: p.x, y: p.y, r: 6 + i * 8, max: 110 + i * 40, c: i % 2 ? '#fff' : c });
    boom(p.x, p.y, 12 + L * 4, [c, '#fff', '#ffd966']);
  }
  function drawEvo(sk, L) {
    if (!(L >= 2)) return; const t = time, c1 = sk.flame || '#ffd966';
    ctx.save(); ctx.globalCompositeOperation = 'lighter'; ctx.lineWidth = 1.6;
    if (L >= 6) { const R = 36 + (L - 6) * 8, g = ctx.createRadialGradient(0, 0, 6, 0, 0, R); g.addColorStop(0, 'rgba(255,240,200,.14)'); g.addColorStop(1, 'rgba(255,200,100,0)'); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(0, 0, R, 0, 7); ctx.fill(); }
    ctx.fillStyle = c1; if (L === 2) for (const d of [-1, 1]) { ctx.globalAlpha = .6 + .3 * Math.sin(t * .1); ctx.beginPath(); ctx.arc(d * 30, 14, 2, 0, 7); ctx.fill(); }
    if (L >= 3) { ctx.globalAlpha = .75; ctx.strokeStyle = c1; ctx.beginPath(); ctx.ellipse(0, 0, 32, 12, t * .03, 0, 7); ctx.stroke(); }
    if (L >= 4) { ctx.fillStyle = '#fff'; for (let i = 0; i < 3; i++) { const ph = (t * .05 + i / 3) % 1; ctx.globalAlpha = 1 - ph; ctx.fillRect(-10 + i * 10 + Math.sin(t * .1 + i) * 3, 22 + ph * 24, 1.8, 6); } ctx.globalAlpha = .8; for (const d of [-1, 1]) { ctx.beginPath(); ctx.arc(d * 10, -17, 2.2 + Math.sin(t * .2) * .6, 0, 7); ctx.fill(); } }
    if (L >= 5) { ctx.globalAlpha = .7; ctx.strokeStyle = '#bde9ff'; ctx.beginPath(); ctx.ellipse(0, 0, 38, 15, -t * .045 + 1, 0, 7); ctx.stroke(); }
    if (L >= 6) { ctx.globalAlpha = .95; ctx.fillStyle = c1; for (let i = 0; i < 4; i++) { const a = t * .05 + i * 1.571; ctx.beginPath(); ctx.arc(Math.cos(a) * 44, Math.sin(a) * 34, 2.4, 0, 7); ctx.fill(); } }
    if (L >= 7) { ctx.fillStyle = '#fff'; for (let i = 0; i < 4; i++) { const a = -t * .07 + i * 1.571, gx = Math.cos(a) * 52, gy = Math.sin(a) * 40, q = 2 + 2 * Math.abs(Math.sin(t * .12 + i * 2)); ctx.globalAlpha = .9; ctx.fillRect(gx - q, gy - .6, q * 2, 1.2); ctx.fillRect(gx - .6, gy - q, 1.2, q * 2); }
      ctx.globalAlpha = .45; ctx.strokeStyle = '#fff'; ctx.setLineDash([4, 6]); ctx.beginPath(); ctx.arc(0, 0, 56, t * .02, t * .02 + 6.283); ctx.stroke(); ctx.setLineDash([]); }
    ctx.restore();
  }
  function drawPika() {
    const t = time; poly([[-6, -17], [-15, -42], [-2, -23]], '#ffe14a'); poly([[6, -17], [15, -42], [2, -23]], '#ffe14a'); poly([[-12, -36], [-15, -42], [-9, -39]], '#222'); poly([[12, -36], [15, -42], [9, -39]], '#222');
    ell(-8, -2, 2.6, 2.6, '#e8322e'); ell(8, -2, 2.6, 2.6, '#e8322e'); ell(-3, -12, 1.4, 1.8, '#222'); ell(3, -12, 1.4, 1.8, '#222');
    poly([[4, 16], [20, 24], [12, 26], [26, 40], [8, 31], [14, 30]], '#ffd23f');
    ctx.save(); ctx.globalCompositeOperation = 'lighter'; ctx.strokeStyle = 'rgba(255,240,120,.9)'; ctx.lineWidth = 1.2;
    for (let i = 0; i < 2; i++) { const a = rnd(0, 6.283); ctx.beginPath(); ctx.moveTo(Math.cos(a) * 24, Math.sin(a) * 24); ctx.lineTo(Math.cos(a) * 31 + rnd(-3, 3), Math.sin(a) * 31 + rnd(-3, 3)); ctx.stroke(); }
    ctx.restore();
  }
  function drawSam() {
    const t = time; for (const d of [-1, 1]) { poly([[d * 8, -6], [d * 24, -10], [d * 26, 5], [d * 8, 7]], '#8a1c1c'); ctx.strokeStyle = '#ffd966'; ctx.lineWidth = 1.2; ctx.beginPath(); ctx.moveTo(d * 24, -10); ctx.lineTo(d * 26, 5); ctx.stroke(); }
    poly([[0, -34], [-7, -22], [7, -22]], '#ffd966'); poly([[-9, -22], [0, -26], [9, -22], [6, -16], [-6, -16]], '#8a1c1c');
    ctx.strokeStyle = '#e8eef5'; ctx.lineWidth = 2; ctx.beginPath(); ctx.moveTo(17, 20); ctx.lineTo(17, -34); ctx.stroke(); ctx.fillStyle = '#ffd966'; ctx.fillRect(13, 16, 8, 2.5);
    ctx.save(); ctx.globalCompositeOperation = 'lighter'; const g = ctx.createRadialGradient(0, 0, 4, 0, 0, 38); g.addColorStop(0, `rgba(255,90,74,${.18 + .05 * Math.sin(t * .1)})`); g.addColorStop(1, 'rgba(255,90,74,0)'); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(0, 0, 38, 0, 7); ctx.fill(); ctx.restore();
  }
  // ---------- Ultimes animés (temporaires, nettoyés automatiquement) ----------
  // ===== TABLEAU DES ULTIMES : chaque skin -> son pouvoir. Chidori = secours uniquement (skins sans thème) =====
  const UK = { celemp: 'celemp', cosmicgod: 'cosmic', voidlord: 'void', solarphx: 'phx', dragoncel: 'drg', spiritsam: 'spirit', samurai: 'sam', pikachu: 'pika', archangel: 'arch', demonking: 'demonk', senju: 'forest', hyuga: 'palm', uchiwa: 'katon' };
  const NM = { celemp: 'JUGEMENT CÉLESTE', cosmicgod: 'GALAXIE ABSOLUE', voidlord: 'ÉCLAT DU VIDE', solarphx: 'RENAISSANCE DU PHÉNIX', dragoncel: 'RUGISSEMENT DU DRAGON CÉLESTE', spiritsam: 'LAME SPIRITUELLE', samurai: 'COUP DU SAMOURAÏ', pikachu: 'MÉGA ÉCLAIR', archangel: "JUGEMENT DE L'ARCHANGE", demonking: 'RAVAGE DÉMONIAQUE', senju: 'FORÊT DU SENJU', hyuga: 'GRANDE PAUME', uchiwa: 'FLAMMES DU CLAN' };
  const FK = { elec: 'pika', spark: 'pika', fire: 'phx', star: 'cosmic', void: 'void', gold: 'celemp', ghost: 'palm', angel: 'arch', demon: 'demonk', rb: 'celemp', sam: 'sam' };
  const SK2 = { arrow: 'sam', ranger: 'forest', dragon: 'drg' }, IDK = { coral: 'phx', forest: 'forest', night: 'cosmic', classic: 'chidori' };
  const OLD = { a01: ['RAYON A-01', 'ray', 'focus'], mecha: ['SALVE MÉCANIQUE', 'missiles', 'drones'], titan: ['IMPACT TITAN', 'meteors', 'blast'], inter: ['SALVE INTERCEPTEUR', 'missiles', 'focus'], falcon: ['PIQUÉ DU FAUCON', 'rain', 'shock'], stealth: ['FRAPPE FURTIVE', 'ray', 'focus'] };
  const FN = { pika: 'ÉCLAIR', phx: 'FLAMME', cosmic: 'ASTRAL', void: 'ABÎME', celemp: 'LUMIÈRE', palm: 'ONDE', arch: 'JUGEMENT', demonk: 'RAVAGE', sam: 'LAME', forest: 'NATURE', drg: 'DRAGON', chidori: 'CHIDORI GÉANT (secours)' };
  const CC = { arch: '#fff1b8', demonk: '#ff2a3a', forest: '#5aff8a', katon: '#ff6a2a', palm: '#bfe6ff' };
  const UC = { celemp: [100, 55, 1.3], cosmic: [110, 72, 1.3], void: [100, 60, 1.3], phx: [110, 55, 1.2], drg: [100, 48, 1.3], spirit: [80, 36, 1.6], sam: [64, 24, 1.3], pika: [120, 76, 1.3], chidori: [96, 50, 1], arch: [96, 50, 1.3], demonk: [96, 52, 1.3], forest: [92, 46, 1.2], katon: [92, 44, 1.3], palm: [84, 40, 1.3] };
  const oldOf = sk => OLD[sk.id] || (sk.fx || UK[sk.id] || IDK[sk.id] || SK2[sk.shape] ? null : OLD[sk.shape]);
  function startUanOld(k, sk, c) {
    if (uan.length > 2) uan.shift(); const q = UC[k], a = { k, T: q[0], hf: q[1], m: q[2] * (UK[sk.id] ? 1 : .6 + .07 * sk.r), t: 0, c: CC[k] || (k === 'chidori' ? SKR[sk.r] : c), r: sk.r, x: p.x, y: p.y, zs: [] };
    if (k === 'pika') for (let i = 0; i < 5; i++) a.zs.push({ x: rnd(35, W - 35), y: rnd(110, H * .62), at: 28 + i * 8 });
    uan.push(a);
  }
  function uHit(a, cx, cy, R, m) {
    const d = (8 + wave * 1.4) * (1 + a.r * .12) * m, bd = (20 + wave) * (1 + a.r * .05) * m; let n = 0;
    for (const e of enemies) { if (protectd(e)) continue; if (R && Math.hypot(e.x - cx, e.y - cy) > R) continue; const v = e.k === 3 ? bd : d; e.hp -= v; e.flash = 3; if (n++ < 6) floats.push({ x: e.x, y: e.y, l: 30, t: '-' + Math.round(v) }); }
  }
  function uImpactOld(a) {
    const k = a.k, c = a.c; let cx = W / 2, cy = H * .34; shake = 12; AM.sfx('boom'); if (a.r >= 6) ebullets.length = 0;
    if (k === 'sam' || k === 'spirit') freeze = 5;
    if (k === 'void') { cx = a.x; cy = Math.max(130, a.y - 170); uHit(a, cx, cy, 200, a.m); boom(cx, cy, 24, ['#b57cff', '#1a0430', '#ff3d7f']); rings.push({ x: cx, y: cy, r: 8, max: 200, c: '#b57cff' }); return; }
    if (k === 'chidori') { uHit(a, cx, cy, 190, 1); uHit(a, cx, cy, 0, .35); boom(cx, cy, 14 + a.r * 2, [c, '#fff']); rings.push({ x: cx, y: cy, r: 8, max: 170, c }, { x: cx, y: cy, r: 4, max: 110, c: '#fff' }); return; }
    if (k === 'palm') { uHit(a, a.x, a.y, 190, a.m * 1.3); uHit(a, 0, 0, 0, a.m * .3); for (const e of enemies) if (e.k < 3 && !e.mini && !e.tst) e.y -= 14; boom(a.x, a.y, 26, ['#fff', '#bfe6ff']); for (let i = 0; i < 3; i++) rings.push({ x: a.x, y: a.y, r: 6 + i * 8, max: 190 + i * 25, c: i % 2 ? '#fff' : '#7fc8ff' }); return; }
    uHit(a, cx, cy, 0, a.m);
    if (k === 'pika') { boom(cx, cy, 24, ['#ffe14a', '#fff']); rings.push({ x: cx, y: cy, r: 6, max: 150, c: '#ffe14a' }); return; }
    boom(a.x, a.y - 80, 28, [c, '#fff']); rings.push({ x: a.x, y: a.y - 60, r: 8, max: 180, c }, { x: a.x, y: a.y - 60, r: 4, max: 120, c: '#fff' });
  }
  function uanUpd() {
    if (!uan.length) return;
    for (const a of uan) {
      a.t++; a.x = p.x; a.y = p.y; const t = a.t;
      if (a.k === 'cosmic' && t < a.hf) for (const e of enemies) if (e.k < 3 && !e.mini && !e.tst) { e.x += (a.x - e.x) * .02; e.y += (a.y - 150 - e.y) * .012; }
      if (t === a.hf) uImpact(a);
      if (a.k === 'g') gTick(a);
      if (a.k === 'pika') for (const z of a.zs) if (t === z.at) { uHit(a, z.x, z.y, 60, .6); boom(z.x, z.y, 8, ['#ffe14a', '#fff']); rings.push({ x: z.x, y: z.y, r: 4, max: 50, c: '#ffe14a' }); shake = 5; }
      if (a.k === 'phx' && t === a.hf + 25) boom(a.x, a.y - 150, 24, ['#ff6a1a', '#ffd23f', '#ff3d3d']);
      if (a.k === 'drg' && t === a.hf + 28) { boom(a.x, 70, 30, ['#19e3ff', '#ffd966', '#fff']); rings.push({ x: a.x, y: 70, r: 6, max: 150, c: '#19e3ff' }); }
    }
    uan = uan.filter(a => a.t < a.T); if (rings.length > 40) rings.splice(0, rings.length - 40); if (floats.length > 30) floats.splice(0, floats.length - 30);
  }
  function bolt(x1, y1, x2, y2, w, col) {
    const pts = [[x1, y1]]; for (let i = 1; i < 7; i++) { const f = i / 7; pts.push([x1 + (x2 - x1) * f + rnd(-11, 11), y1 + (y2 - y1) * f + rnd(-6, 6)]); } pts.push([x2, y2]); ctx.lineJoin = 'round';
    for (const [lw, cl] of [[w * 2.2, col], [w * .8, '#fff']]) { ctx.strokeStyle = cl; ctx.lineWidth = lw; ctx.beginPath(); pts.forEach((q, i) => i ? ctx.lineTo(q[0], q[1]) : ctx.moveTo(q[0], q[1])); ctx.stroke(); }
  }
  function drawUan(pre) {
    if (!uan.length) return;
    for (const a of uan) {
      if ((a.k === 'g') !== !!pre) continue; if (a.k === 'g') { drawG(a); continue; } if (a.k === 'pw') { drawPW(a); continue; }
      const t = a.t, hf = a.hf, T = a.T, k = a.k, c = a.c, x = a.x, y = a.y, ch = Math.min(1, t / hf), fd = t > hf ? Math.max(0, 1 - (t - hf) / (T - hf)) : 1, post = Math.max(0, t - hf), cy0 = H * .34;
      ctx.save(); ctx.globalCompositeOperation = 'lighter'; ctx.lineCap = 'round';
      if (k === 'celemp') {
        ctx.fillStyle = '#ffd966'; ctx.globalAlpha = .3 * ch * fd; for (let i = -3; i <= 3; i++) { const ex = x + i * 30 * (.5 + ch); ctx.beginPath(); ctx.moveTo(x + i * 4, y + 8); ctx.lineTo(ex - 5, y - 170 - 100 * ch); ctx.lineTo(ex + 5, y - 170 - 100 * ch); ctx.fill(); }
        const g = ctx.createRadialGradient(x, y, 8, x, y, 30 + 50 * ch); g.addColorStop(0, 'rgba(255,240,170,.35)'); g.addColorStop(1, 'rgba(255,200,60,0)'); ctx.globalAlpha = fd; ctx.fillStyle = g; ctx.beginPath(); ctx.arc(x, y, 80, 0, 7); ctx.fill();
        ctx.strokeStyle = '#fff'; ctx.lineWidth = 2.5; ctx.globalAlpha = (.3 + .7 * ch) * fd; ctx.beginPath(); ctx.ellipse(x, y - 46, 18 + 10 * ch, 5, 0, 0, 7); ctx.stroke();
        if (post) { const yy = y - post * 16; ctx.globalAlpha = fd; ctx.fillStyle = '#ffd966'; ctx.fillRect(0, yy - 8, W, 16); ctx.fillStyle = '#fff'; ctx.fillRect(0, yy - 2, W, 4); }
      } else if (k === 'cosmic') {
        const gy = y - 150 * ch * ch, sc = .4 + 1.6 * ch, sp = t * (.06 + .08 * ch);
        for (let m = 0; m < 3; m++) for (let i = 0; i < 20; i++) { const q = sp + m * 2.094 + i * .4, r = (5 + i * 3.4) * sc; ctx.globalAlpha = (1 - i / 22) * fd; ctx.fillStyle = m === 1 ? '#19e3ff' : m ? '#ff5ac8' : '#fff'; ctx.beginPath(); ctx.arc(x + Math.cos(q) * r, gy + Math.sin(q) * r * .6, 1.8, 0, 7); ctx.fill(); }
        ctx.globalAlpha = fd; for (let i = 0; i < 2; i++) { const q = t * (.08 + .1 * ch) + i * 3.14; ctx.fillStyle = i ? '#ffd966' : '#7a4cff'; ctx.beginPath(); ctx.arc(x + Math.cos(q) * 40, y + Math.sin(q) * 18, 4, 0, 7); ctx.fill(); }
        if (post < 14) { ctx.globalAlpha = .5 * (1 - post / 14); ctx.fillStyle = '#fff'; ctx.beginPath(); ctx.arc(x, gy, 40 + post * 6, 0, 7); ctx.fill(); }
      } else if (k === 'void') {
        const zx = x, zy = Math.max(130, y - 170), vf = Math.min(ch, fd); ctx.globalCompositeOperation = 'source-over';
        const vg = ctx.createRadialGradient(W / 2, H / 2, H * .25, W / 2, H / 2, H * .75); vg.addColorStop(0, 'rgba(10,0,25,0)'); vg.addColorStop(1, `rgba(10,0,25,${.4 * vf})`); ctx.globalAlpha = 1; ctx.fillStyle = vg; ctx.fillRect(0, 0, W, H);
        ctx.globalCompositeOperation = 'lighter'; ctx.strokeStyle = '#b57cff'; ctx.lineWidth = 1.5; ctx.globalAlpha = (.4 + .6 * ch) * fd;
        for (let i = 0; i < 7; i++) { const a0 = i * .9 + .2; ctx.beginPath(); ctx.moveTo(x, y - 10); for (let j = 1; j < 6; j++) { const r = j * (8 + 22 * ch), an = a0 + Math.sin(j * 5 + i) * .22; ctx.lineTo(x + Math.cos(an) * r, y - 10 + Math.sin(an) * r); } ctx.stroke(); }
        const R = 10 + 60 * ch; ctx.globalCompositeOperation = 'source-over'; ctx.globalAlpha = .88 * fd; ctx.fillStyle = '#05010a'; ctx.beginPath(); ctx.arc(zx, zy, R, 0, 7); ctx.fill(); ctx.strokeStyle = '#b57cff'; ctx.lineWidth = 2; ctx.stroke();
        ctx.globalAlpha = fd; for (let i = 0; i < 6; i++) { const q = -t * .08 + i * 1.047, r = R + 14 + post * 5; ctx.fillStyle = '#12041f'; ctx.strokeStyle = '#b57cff'; ctx.lineWidth = 1; ctx.beginPath(); ctx.moveTo(zx + Math.cos(q) * r, zy + Math.sin(q) * r - 6); ctx.lineTo(zx + Math.cos(q) * r + 3, zy + Math.sin(q) * r); ctx.lineTo(zx + Math.cos(q) * r, zy + Math.sin(q) * r + 6); ctx.lineTo(zx + Math.cos(q) * r - 3, zy + Math.sin(q) * r); ctx.closePath(); ctx.fill(); ctx.stroke(); }
        if (post) { ctx.globalCompositeOperation = 'lighter'; ctx.globalAlpha = fd; ctx.strokeStyle = '#ff3d7f'; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(zx, zy, R + post * 5, 0, 7); ctx.stroke(); }
      } else if (k === 'phx') {
        const sp = (20 + 80 * Math.min(1, ch * 1.3)), py = y + 20 - post * 9; ctx.globalAlpha = .55 * fd;
        for (const d of [-1, 1]) { const gr = ctx.createLinearGradient(x, py - sp, x, py + 30); gr.addColorStop(0, '#ffd23f'); gr.addColorStop(1, '#ff4a1f'); ctx.fillStyle = gr; ctx.beginPath(); ctx.moveTo(x, py); ctx.bezierCurveTo(x + d * sp * .4, py - sp * .9, x + d * sp * 1.1, py - sp * .5, x + d * sp * 1.2, py + sp * .3); ctx.bezierCurveTo(x + d * sp * .7, py + sp * .1, x + d * sp * .3, py + sp * .3, x, py + 26); ctx.fill(); }
        ctx.fillStyle = '#ffe9a0'; ctx.beginPath(); ctx.moveTo(x, py - 28); ctx.lineTo(x + 6, py - 8); ctx.lineTo(x - 6, py - 8); ctx.fill();
        if (post) { const yy = y - post * 14; ctx.globalAlpha = fd * .8; const fg = ctx.createLinearGradient(0, yy - 30, 0, yy + 10); fg.addColorStop(0, 'rgba(255,200,60,0)'); fg.addColorStop(.6, '#ff6a1a'); fg.addColorStop(1, 'rgba(255,60,20,0)'); ctx.fillStyle = fg; ctx.fillRect(0, yy - 30, W, 40); }
      } else if (k === 'drg') {
        const off = post * 24; ctx.globalAlpha = Math.min(1, ch * 1.5) * Math.max(0, 1 - post / 34);
        for (let i = 0; i < 15; i++) { const f = i / 14, bx = x + Math.sin(i * .55 + t * .18) * (8 + f * 14), by = y + 28 - i * 10 - off, r = 7 - f * 2 + (i === 14 ? 3 : 0); ctx.fillStyle = i % 2 ? '#ffd966' : '#19e3ff'; ctx.beginPath(); ctx.arc(bx, by, r, 0, 7); ctx.fill();
          if (i === 7) { ctx.strokeStyle = '#7fe3ff'; ctx.lineWidth = 3; for (const d of [-1, 1]) { ctx.beginPath(); ctx.moveTo(bx, by); ctx.quadraticCurveTo(bx + d * 44, by - 24 * ch, bx + d * 52, by + 14); ctx.stroke(); } }
          if (i === 14) { ctx.fillStyle = '#fff'; for (const d of [-1, 1]) { ctx.beginPath(); ctx.moveTo(bx + d * 3, by - 6); ctx.lineTo(bx + d * 8, by - 18); ctx.lineTo(bx + d * 6, by - 4); ctx.fill(); } if (!post) { ctx.globalAlpha *= .6; ctx.beginPath(); ctx.arc(bx, by, 6 + 14 * ch, 0, 7); ctx.fill(); } } }
        if (post) { ctx.globalAlpha = fd * .5; ctx.fillStyle = '#7fe3ff'; ctx.fillRect(x - 30, 0, 60, y - off + 30); }
      } else if (k === 'sam' || k === 'spirit') {
        ctx.globalAlpha = ch * (post ? 0 : 1); ctx.strokeStyle = '#fff'; ctx.lineWidth = 2; ctx.beginPath(); ctx.moveTo(x + 14, y + 10); ctx.lineTo(x + 14, y - 40 - 40 * ch); ctx.stroke(); ctx.strokeStyle = c; ctx.globalAlpha *= .4; ctx.lineWidth = 8; ctx.stroke();
        if (t === hf - 1 || t === hf) { ctx.globalAlpha = .35; ctx.fillStyle = '#fff'; ctx.fillRect(0, 0, W, H); }
        if (post) { const rv = Math.min(1, post / 5), w = (k === 'spirit' ? 30 : 20) * fd, x0 = -30, y0 = H * .12, x1 = x0 + (W + 60) * rv, y1 = y0 + H * .62 * rv;
          ctx.globalAlpha = fd; ctx.fillStyle = c; ctx.beginPath(); ctx.moveTo(x0, y0); ctx.lineTo(x1, y1 - w); ctx.lineTo(x1, y1 + w); ctx.closePath(); ctx.fill(); ctx.fillStyle = '#fff'; ctx.beginPath(); ctx.moveTo(x0, y0); ctx.lineTo(x1, y1 - w * .35); ctx.lineTo(x1, y1 + w * .35); ctx.closePath(); ctx.fill(); }
      } else if (k === 'pika') {
        ctx.strokeStyle = c; ctx.globalAlpha = 1; if (t < hf) for (let i = 0; i < 4 + ch * 8; i++) { const q = rnd(0, 6.283), r = rnd(14, 30 + 14 * ch); bolt(x + Math.cos(q) * 14, y + Math.sin(q) * 14, x + Math.cos(q) * r, y + Math.sin(q) * r, .8, c); }
        for (const z of a.zs) if (t >= z.at && t < z.at + 10) { ctx.globalAlpha = 1 - (t - z.at) / 10; bolt(z.x + rnd(-8, 8), 0, z.x, z.y, 2, c); ctx.fillStyle = 'rgba(255,225,60,.45)'; ctx.beginPath(); ctx.arc(z.x, z.y, 28, 0, 7); ctx.fill(); }
        if (post > 0 && post < 16) { ctx.globalAlpha = 1 - post / 16; bolt(W / 2, 0, W / 2, cy0, 5, c); ctx.fillStyle = 'rgba(255,225,60,.5)'; ctx.beginPath(); ctx.arc(W / 2, cy0, 44, 0, 7); ctx.fill(); for (let i = 0; i < 5; i++) { const q = i * 1.26 + .4; bolt(W / 2, cy0, W / 2 + Math.cos(q) * 110, cy0 + Math.sin(q) * 110, 1.5, c); } }
      } else if (k === 'arch') {
        const wsp = 14 + 48 * Math.min(1, ch * 1.4); ctx.globalAlpha = .6 * fd;
        for (const d of [-1, 1]) for (let j = 0; j < 4; j++) { ctx.fillStyle = j % 2 ? '#fff1b8' : '#fff'; ctx.beginPath(); ctx.moveTo(x, y + 4); ctx.bezierCurveTo(x + d * wsp * .5, y - 30 - j * 8, x + d * wsp * 1.2, y - 22 - j * 10, x + d * (wsp + 16 + j * 5), y + 6 - j * 6); ctx.bezierCurveTo(x + d * wsp * .8, y - 8 - j * 4, x + d * wsp * .4, y - 6, x, y + 10); ctx.fill(); }
        ctx.globalAlpha = (.2 + .25 * ch) * fd; ctx.fillStyle = '#fff'; ctx.fillRect(x - 8 - 14 * ch, 0, 16 + 28 * ch, Math.max(0, y - 20));
        ctx.globalAlpha = fd; ctx.fillStyle = '#fff'; for (let i = 0; i < 8; i++) { const q = t * .07 + i * .785; ctx.beginPath(); ctx.ellipse(x + Math.cos(q) * 38, y + Math.sin(q) * 22, 1.6, 4.5, q, 0, 7); ctx.fill(); }
        if (post) { const ly = y - 30 - post * 22; ctx.globalAlpha = fd; ctx.fillStyle = '#fff1b8'; ctx.beginPath(); ctx.moveTo(x, ly - 50); ctx.lineTo(x + 14, ly + 10); ctx.lineTo(x - 14, ly + 10); ctx.fill(); ctx.fillStyle = '#fff'; ctx.fillRect(x - 4, ly, 8, Math.max(0, y - ly)); }
      } else if (k === 'demonk') {
        const g = ctx.createRadialGradient(x, y, 6, x, y, 30 + 40 * ch); g.addColorStop(0, 'rgba(255,30,50,.4)'); g.addColorStop(1, 'rgba(40,0,10,0)'); ctx.globalAlpha = fd; ctx.fillStyle = g; ctx.beginPath(); ctx.arc(x, y, 70, 0, 7); ctx.fill();
        ctx.strokeStyle = '#ff2a3a'; ctx.lineWidth = 2; ctx.globalAlpha = Math.min(1, ch * 2) * fd * (t % 10 < 7 ? 1 : .4);
        for (let i = 0; i < 4; i++) { const mx = x + Math.sin(i * 7.3) * 110, my = y - 60 - i * 55; ctx.beginPath(); ctx.moveTo(mx - 8, my - 8); ctx.lineTo(mx + 8, my + 8); ctx.moveTo(mx + 8, my - 8); ctx.lineTo(mx - 8, my + 8); ctx.stroke(); ctx.beginPath(); ctx.arc(mx, my, 12, 0, 7); ctx.stroke(); }
        if (post) { const yy = y - post * 15; ctx.globalAlpha = fd * .85; ctx.fillStyle = '#b3122a'; ctx.beginPath(); ctx.moveTo(0, yy + 30); for (let i = 0; i <= 12; i++) ctx.lineTo(i * 30, yy - (i % 2 ? 26 : 4)); ctx.lineTo(W, yy + 30); ctx.fill(); if (post < 14) { let n = 0; for (const e of enemies) if (n++ < 8) bolt(e.x - 20, e.y - 40, e.x, e.y, 1.3, '#ff2a3a'); } }
      } else if (k === 'forest') {
        ctx.lineWidth = 5; const gh = (post ? 1 : ch) * H * .6; ctx.globalAlpha = (post ? fd : 1) * .85;
        for (let i = 0; i < 9; i++) { const bx = 20 + i * 40, hh = gh * (.6 + .4 * Math.sin(i * 2.1) ** 2); ctx.strokeStyle = i % 2 ? '#2f9d4a' : '#8a5a2b'; ctx.beginPath(); ctx.moveTo(bx, H); ctx.bezierCurveTo(bx + Math.sin(t * .1 + i) * 20, H - hh * .4, bx - 18, H - hh * .7, bx + Math.sin(t * .08 + i) * 14, H - hh); ctx.stroke(); ctx.fillStyle = '#5aff8a'; ctx.beginPath(); ctx.ellipse(bx, H - hh, 7, 3.5, i, 0, 7); ctx.fill(); }
      } else if (k === 'katon') {
        const R = 8 + 34 * ch + (post ? 20 : 0), by = post ? Math.max(70, y - 50 - post * 24) : y - 50, gk = ctx.createRadialGradient(x, by, 2, x, by, R * 1.5); gk.addColorStop(0, '#fff'); gk.addColorStop(.35, '#ffb020'); gk.addColorStop(.7, '#ff4a1f'); gk.addColorStop(1, 'rgba(255,60,20,0)'); ctx.globalAlpha = fd; ctx.fillStyle = gk; ctx.beginPath(); ctx.arc(x, by, R * 1.5, 0, 7); ctx.fill();
        if (post) { ctx.fillStyle = '#ff6a2a'; ctx.globalAlpha = fd * .8; for (let i = 0; i < 6; i++) { ctx.beginPath(); ctx.arc(x + Math.sin(i * 3 + post) * 60, by + 30 + i * 20 + post * 6, 6, 0, 7); ctx.fill(); } }
      } else if (k === 'palm') {
        const R0 = post ? 20 + post * 10 : 46 - 30 * ch; ctx.strokeStyle = '#fff'; ctx.lineWidth = 2; ctx.globalAlpha = fd; ctx.beginPath(); ctx.arc(x, y, R0, 0, 7); ctx.stroke(); ctx.strokeStyle = '#7fc8ff';
        for (let i = 0; i < 3; i++) { const sg = (i % 2 ? -1 : 1) * t * .15; ctx.beginPath(); ctx.arc(x, y, Math.max(1, post ? R0 - i * 14 : 18 + i * 8), sg, sg + 3); ctx.stroke(); }
        ctx.globalAlpha = .25 * fd; ctx.fillStyle = '#e8f4ff'; ctx.beginPath(); ctx.arc(x, y, R0, 0, 7); ctx.fill();
      } else if (k === 'pw') {
        ctx.globalAlpha = 1 - t / T; ctx.strokeStyle = c; ctx.lineWidth = 2; const nn = 5 + (a.r >> 1);
        for (let i = 0; i < nn; i++) { const q = i * 6.283 / nn, r1 = 14 + t * 3, r2 = r1 + 18 + a.r * 2; if (a.fam === 'pika') bolt(x + Math.cos(q) * 14, y + Math.sin(q) * 14, x + Math.cos(q) * r2, y + Math.sin(q) * r2, .9, c); else { ctx.beginPath(); ctx.moveTo(x + Math.cos(q) * r1, y + Math.sin(q) * r1); ctx.lineTo(x + Math.cos(q) * r2, y + Math.sin(q) * r2); ctx.stroke(); } }
      } else {   // chidori géant
        const ex = t < hf ? ch * ch : 1, R = (8 + 62 * ex) * (t > hf ? fd + .25 : 1); ctx.globalAlpha = 1;
        if (t < 26) for (let i = 0; i < 6; i++) { const q = rnd(0, 6.283); bolt(x + Math.cos(q) * 12, y + Math.sin(q) * 12, x + Math.cos(q) * 30, y + Math.sin(q) * 30, .8, c); }
        const g = ctx.createRadialGradient(W / 2, cy0, 2, W / 2, cy0, R * 1.6); g.addColorStop(0, '#fff'); g.addColorStop(.35, c); g.addColorStop(1, 'rgba(0,0,0,0)'); ctx.globalAlpha = Math.min(1, .3 + ch) * (t > hf ? fd : 1); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(W / 2, cy0, R * 1.6, 0, 7); ctx.fill();
        for (let i = 0; i < 6 + a.r; i++) { const q = i * 6.283 / (6 + a.r) + t * .1; bolt(W / 2, cy0, W / 2 + Math.cos(q) * R * 1.8, cy0 + Math.sin(q) * R * 1.8, 1.2, c); }
        if (post && post < 16) { ctx.globalAlpha = 1 - post / 16; let n = 0; for (const e of enemies) if (n++ < 8) bolt(W / 2, cy0, e.x, e.y, 1.3, c); }
      }
      ctx.restore();
    }
  }

  // ---------- Ultime ----------
  const ultPos = () => isTest() ? { x: W - 44, y: H - 226 } : { x: W - 48, y: H - 72 };
  const UT = ['ray', 'blast', 'drones', 'missiles', 'rain', 'shield', 'slow', 'focus', 'vortex', 'meteors', 'clone', 'shock'];
  const UNM = { uchiwa: ['SHADOW FLAME', 'blast', 'rain'], senju: ['NATURE AWAKENING', 'vortex', 'slow'], hyuga: ['CELESTIAL PALM', 'shock', 'focus'], cosmicgod: ['COSMIC COLLAPSE', 'vortex', 'meteors'], celemp: ['CELESTIAL JUDGMENT', 'rain', 'ray'], voidlord: ['VOID RIFT', 'vortex', 'slow'], solarphx: ['PHOENIX REBIRTH', 'blast', 'shield'], dragoncel: ['DRAGON NOVA', 'focus', 'blast'], spiritsam: ['SPIRIT SLASH', 'ray', 'missiles'], demonking: ['DEMON WRATH', 'meteors', 'blast'], archangel: ['HOLY JUDGMENT', 'ray', 'shield'] };
  const UNN = ['NOVA', 'ECLIPSE', 'TEMPEST', 'AURORA', 'ZENITH', 'RIFT', 'SURGE', 'VERTEX'];
  function ultOf0(sk) { const i = Math.max(0, SKINS.indexOf(sk)), u = UNM[sk.id]; return u ? { n: u[0], f: [u[1], u[2]] } : { n: UNN[i % 8] + ' · ' + sk.n.toUpperCase(), f: [UT[i % 12], UT[(i * 5 + 3) % 12]] }; }
  function ultFx(f, sk, c) {
    const B = (dx, vx, vy, d, sz, pr) => bullets.push({ x: p.x + dx, y: p.y - 18, vx, vy, d: p.dmg * d, sz, pr, st: sk.bul });
    const hurtAll = (d, bd) => { for (const e of enemies) if (!e.tst) { e.hp -= e.k === 3 ? bd + wave : d + wave; e.flash = 3; } };
    if (f === 'ray') for (let i = -2; i <= 2; i++) B(i * 8, 0, -14, 5, 1.6, 99);
    else if (f === 'focus') for (let i = 0; i < 7; i++) B(0, 0, -12 - i, 3, 1.4, 5);
    else if (f === 'drones') for (const d of [-60, -40, 40, 60]) for (let j = 0; j < 3; j++) B(d, 0, -11 - j, 2, 1, 2);
    else if (f === 'missiles') for (let i = 0; i < 8; i++) B((i - 3.5) * 6, (i - 3.5) * .9, -10, 3, 1.2, 1);
    else if (f === 'meteors') for (let i = 0; i < 6; i++) B((i - 2.5) * 14, (i - 2.5) * 1.6, -9, 6, 2.2, 3);
    else if (f === 'rain') for (let i = 0; i < 18; i++) bullets.push({ x: 20 + i * 18.5, y: H - 30, vx: 0, vy: -12 - (i % 3), d: p.dmg * 2.5, sz: 1.2, pr: 2, st: sk.bul });
    else if (f === 'blast') { ebullets.length = 0; hurtAll(8, 20); }
    else if (f === 'shock') { ebullets.length = 0; hurtAll(5, 12); for (let i = 0; i < 3; i++) rings.push({ x: p.x, y: p.y, r: 4 + i * 6, max: 180 + i * 40, c }); }
    else if (f === 'vortex') { ebullets.length = 0; hurtAll(6, 12); for (const e of enemies) if (e.k < 3 && !e.mini && !e.tst) { e.x += (p.x - e.x) * .4; e.y += (p.y - 140 - e.y) * .2; } }
    else if (f === 'shield') p.shield = Math.max(p.shield, 300);
    else if (f === 'slow') p.slow = 300;
    else if (f === 'clone') { p.clone = 1; setTimeout(() => { if (p) p.clone = 0; }, 5000); }
  }
  function fireUlt() {
    const t = isTest(); if (state !== 'play' || mg || (!t && (ult < 100 || ultT > 0))) return;
    ult = 0; ultT = t ? 20 : 120;
    const sk = skin(), c = (sk.cos && sk.cos.r1) || sk.flame || sk.glow || '#9be7ff', u = ultOf(sk), key = ukey(sk);
    AM.sfx('rare'); shake = 14; p.inv = Math.max(p.inv, 120); msg = { a: '⚡ ' + u.n, b: '' }; msgT = 90;
    for (let i = 0; i < 3; i++) rings.push({ x: p.x, y: p.y, r: 8 + i * 10, max: 120 + i * 60, c });
    boom(p.x, p.y, 30, [c, '#fff']);
    if (key) startUan(key, sk, c); else u.f.forEach(f => ultFx(f, sk, c));
    if (sk.r >= 8 && !key) { ebullets.length = 0; p.shield = Math.max(p.shield, 240); for (const e of enemies) if (!e.tst) { e.hp -= 6 + wave; e.flash = 3; } for (let i = 0; i < 5; i++) rings.push({ x: p.x, y: p.y, r: 4 + i * 8, max: 160 + i * 50, c: i % 2 ? '#fff' : c }); }
  }
  function drawUlt() {
    const u = ultPos(), rdy = ult >= 100 || isTest();
    ctx.save(); ctx.fillStyle = 'rgba(18,41,63,.6)'; ctx.beginPath(); ctx.arc(u.x, u.y, 26 + (rdy ? 2 * Math.sin(time * .2) : 0), 0, 7); ctx.fill(); if (!rdy) { ctx.fillStyle = 'rgba(0,0,0,.25)'; ctx.fill(); }
    ctx.strokeStyle = rdy ? `rgba(255,217,102,${.7 + .3 * Math.sin(time * .2)})` : '#7fe3ff'; ctx.lineWidth = 5; ctx.beginPath(); ctx.arc(u.x, u.y, 22, -1.57, -1.57 + 6.283 * (isTest() ? 1 : ult / 100)); ctx.stroke();
    ctx.fillStyle = '#fff'; ctx.font = `bold 11px ${FONT}`; ctx.textAlign = 'center'; ctx.textBaseline = 'middle'; ctx.fillText(rdy ? 'ULTI' : Math.floor(ult) + '%', u.x, u.y + 1); ctx.restore();
  }
  // ---------- Budget de projectiles ennemis ----------
  const EBCAP = () => Math.min(40, 16 + wave), ENCAP = () => Math.min(14, 6 + (wave >> 1));
  const NEED = () => Math.min(64, W * .18);
  function gapOf(list) {  // plus grande ouverture à la hauteur du joueur (≤ 100 frames)
    const iv = [];
    for (const b of list) { if (b.vy <= 0.1 || b.y > p.y + 10) continue; const t = (p.y - b.y) / b.vy; if (t > 100) continue; iv.push(b.x + b.vx * t); }
    iv.sort((a, c) => a - c); let g = 0, last = 22; const m = 14;
    for (const x of iv) { g = Math.max(g, x - m - last); last = Math.max(last, x + m); } return Math.max(g, W - 22 - last);
  }
  function budget() {  // DodgeSafetySystem : plafond + vitesse max + ouverture garantie
    let old = 0; const nw = [];
    for (const b of ebullets) if (b.ag) old++; else nw.push(b);
    if (!nw.length) return;
    const room = Math.max(0, EBCAP() - old);
    nw.forEach((b, i) => { b.ag = 1; const sp = Math.hypot(b.vx, b.vy); if (sp > 4.8) { b.vx *= 4.8 / sp; b.vy *= 4.8 / sp; } if (i >= room || (!b.sn && Math.hypot(b.x - p.x, b.y - p.y) < 70)) b.dead = true; });
    for (let k = 0; k < 40 && gapOf(ebullets.filter(b => !b.dead)) < NEED(); k++) {
      let best = null, bd = 1e9; for (const b of nw) if (!b.dead && Math.abs(b.x - p.x) < bd) { bd = Math.abs(b.x - p.x); best = b; }
      if (!best) break; best.dead = true;
    }
  }

  // ---------- vagues ----------
  function nextWave() {
    wave++; banner = 100; wTot = 1; wt = 0; wq = [];
    sv.st.bestWave = Math.max(sv.st.bestWave, wave);
    chkSl();
    checkAll();
    const lv = LVLW.filter(w => wave >= w).length;
    if (lv > p.lvl) {
      p.lvl = lv; p.power = Math.max(p.power, minP());
      if (lv > 1) { p.lives = Math.min(6, p.lives + 1); p.shield = 300; upT = 220; upName = LVN[lv - 1]; boom(p.x, p.y, 40, ['#ffd966', '#fff', '#ffb020']); }
    }
    const isBoss = wave % 5 === 0;
    const n = isBoss ? 6 : Math.min(36, 5 + wave * 2 + Math.max(0, wave - 5)), step = Math.max(12, 56 - wave * 2.2);
    for (let i = 0; i < n; i++) {
      const r = Math.random();
      const k = wave < 3 ? (r < .75 ? 0 : 1) : (r < .55 ? 0 : (r < .88 || wave < 4) ? 1 : 2);
      wq.push({ k, t: i * step });
    }
    if (isBoss) { AM.sfx('boss'); wq.push({ k: 3, t: 70 }); bossName = BOSSES[BORDER[((wave / 5 - 1) | 0) % 5]].n; } else { bossName = ''; if (isMini(wave)) wq.push({ k: 2, mini: 1, t: 90 }); }
    if (wave > 1) pups.push({ x: rnd(40, W - 40), y: -10, t: DROPS[(Math.random() * DROPS.length) | 0], ph: 0 });
    waveHook(isBoss);
  }
  const ACH = [
    ['first', 'PREMIER VOL', 'Terminer la vague 1', 50, () => sv.st.bestWave >= 2], ['w10', 'SURVIVANT', 'Atteindre la vague 10', 150, () => sv.st.bestWave >= 10],
    ['w25', 'VÉTÉRAN', 'Atteindre la vague 25', 500, () => sv.st.bestWave >= 25], ['w50', 'LÉGENDE', 'Atteindre la vague 50', 2000, () => sv.st.bestWave >= 50],
    ['b10', 'CHASSEUR DE BOSS', 'Vaincre 10 boss', 600, () => sv.st.boss >= 10], ['k5k', 'DESTRUCTEUR', 'Détruire 5 000 ennemis', 1500, () => sv.st.kills >= 5000],
    ['c20', 'COLLECTIONNEUR', 'Posséder 20 skins', 2000, () => sv.owned.length >= 20], ['tr', 'TRANSCENDANT', 'Posséder un skin Transcendant', 3000, () => SKINS.some(k => k.r === 7 && sv.owned.includes(k.id))],
    ['bh', 'MAÎTRE DE LA SINGULARITÉ', 'Niveau 5 avec un skin trou noir', 2500, () => SKINS.some(k => k.cos && k.cos.bh && (sv.lv[k.id] || 0) >= 5)],
    ['void', 'SEIGNEUR DU VIDE', 'Niveau 5 avec un skin supérieur', 3000, () => SKINS.some(k => k.cos && (sv.lv[k.id] || 0) >= 5)]];
  function fragGain(n) { if (!GM.allowRewards) return; sv.frag = (sv.frag || 0) + n; if (p) p.frag = (p.frag || 0) + n; save(); }
  ACH.push(['dem', 'SEIGNEUR DES DÉMONS', 'Atteindre la vague 40 : débloque Demon King', 0, () => sv.st.bestWave >= 40, 'demonking'], ['ang', 'ÉLU CÉLESTE', 'Vaincre 25 boss : débloque Archangel', 0, () => sv.st.boss >= 25, 'archangel']);
  function checkAll() {
    if (!GM.allowProgression) return;
    ACH.forEach(a => { if (!sv.ach[a[0]] && a[4]()) { sv.ach[a[0]] = 1; earn(a[3], true); if (a[5] && !sv.owned.includes(a[5])) sv.owned.push(a[5]); msg = { a: 'SUCCÈS : ' + a[1], b: '+' + a[3] + ' ◎' }; msgT = 160; AM.sfx('win'); } });
    sv.mis.forEach((m, i) => {
      if (m.k === 'wave') m.m = Math.max(m.m || 0, wave); if (m.k === 'combo') m.m = Math.max(m.m || 0, cm()); if (m.k === 'lvl') m.m = Math.max(m.m || 0, skin().cos ? p.sl : 0); if (m.k === 'plv') m.m = Math.max(m.m || 0, p.sl); if (m.k === 'tests') m.m = Object.keys(sv.tested || {}).length;
      if (mprog(m) >= m.g) { earn(m.rw, true); if (m.rw >= 400) giveChest(1); msg = { a: 'MISSION ACCOMPLIE', b: '+' + m.rw + ' ◎' }; msgT = 160; AM.sfx('win'); sv.mis[i] = newMis(); }
    });
  }
  function endRun() { if (!GM.allowProgression) { runF = 0; return; } const t = sv.st; if (runF > 600 && wave >= 3 && p && !p.paid) { p.paid = 1; earn(Math.round(wave * wave * 6 * (1 + wave / 40)), true); } t.bestWave = Math.max(t.bestWave, wave); t.bestScore = Math.max(t.bestScore, score); t.time += Math.round(runF / 60); runF = 0; checkAll(); save(); }
  function spawn(q) {
    const n0 = enemies.length; spawn0(q); const e = enemies[n0]; if (!e) return;
    const tr = tier(wave); e.tr = tr;
    if (e.k === 3) { e.hp = e.mhp = Math.round(e.hp * (1 + (tr - 1) * .1) * (1 + (pf() - 1) * .7)); sv.bs[bkey(e)] = sv.bs[bkey(e)] || 0; return; }
    const hm = (1 + (tr - 1) * .3 + Math.max(0, wave - 10) * .01 + Math.max(0, wave - 5) * .09) * pf();
    e.hp = Math.ceil(e.hp * hm);
    if (e.k < 2) e.vy *= Math.min(1.5, 1 + (tr - 1) * .04);
    if (!q.mini && q.k < 3 && Math.random() < Math.min(.3, Math.max(0, wave - 4) * .012 * p.luck)) { e.el = 1; e.hp = Math.ceil(e.hp * 3.5); e.r *= 1.1; e.vy *= 1.2; }
    if (!q.mini && e.k < 2 && !e.el) {
      const bh = []; if (wave >= 6) bh.push('split'); if (wave >= 11) bh.push('ghost'); if (wave >= 15) bh.push('heal'); if (wave >= 18) bh.push('hunt'); if (wave >= 20) bh.push('jug');
      if (bh.length && Math.random() < Math.min(.4, (wave - 4) * .03)) { e.beh = bh[(Math.random() * bh.length) | 0]; if (e.beh === 'jug') { e.hp *= 5; e.vy *= .5; e.r *= 1.4; } if (e.beh === 'hunt') e.vy *= 1.4; }
      else if (wave >= 25 && Math.random() < .05) { e.cor = 1; e.hp *= 8; }
      else if (wave >= 12 && Math.random() < Math.min(.12, (wave - 10) * .006)) { e.ch = 1; e.hp *= 6; e.r *= 1.15; }
    }
    if (q.mini) { e.mini = 1; e.mf = (((wave - 8) / 5) | 0) % 5; e.r = [36, 24, 30, 32, 28][e.mf]; e.hp = Math.ceil((20 + wave) * [4, 3, 4.5, 4, 3.5][e.mf] * hm); }
    e.mhp = e.hp; sv.bs[bkey(e)] = sv.bs[bkey(e)] || 0;
  }
  function spawn0(q) {
    const k = q.k, x = rnd(34, W - 34);
    if (k === 0) enemies.push({ k, x, y: -30, hp: 1, vy: 2.4, ph: rnd(0, 6), r: 15, cd: rnd(60, 140), hy: rnd(80, 210), dx: Math.random() < .5 ? -1 : 1 });
    else if (k === 1) enemies.push({ k, x, y: -30, hp: 3, vy: rnd(1, 1.5), ph: 0, r: 18, cd: rnd(40, 90) });
    else if (k === 2) enemies.push({ k, x: rnd(70, W - 70), y: -50, hp: 14 + wave, vy: .7, ph: 0, bx: 1, r: 28, cd: 60 });
    else { const hp = 130 + wave * 14; enemies.push({ k, bi: BORDER[((wave / 5 - 1) | 0) % 5], x: W / 2, y: -70, hp, mhp: hp, ph: 0, r: 42, cd: 0, t: 0 }); }
  }

  function miniAI(e, sl) {
    if (!e.in) { e.y += 1.2 * sl; if (e.y >= 90) e.in = 1; return; }
    if (e.brush) {
      if (!e.pre) { e.pre = 1; e.preT = 70; msg = { a: '⚠ MINI-BOSS', b: 'PRÉPAREZ-VOUS…' }; msgT = 70; }
      if (e.preT > 0) { e.preT--; e.flash = e.preT % 14 < 7 ? 1 : 0; if (e.preT === 0) { e.flash = 0; mgStart(e); } return; }
      if (mg && mg.boss === e) return;
      if (e.stun > 0) { if (--e.stun === 0) { e.vul = 0; e.wake = 40; } else { e.vul = 1; e.sh = 0; e.flash = e.stun % 12 < 6 ? 1 : 0; return; } }
      if (e.wake > 0) { e.wake--; return; }
    }
    e.t = (e.t || 0) + sl; const tick = n => e.t % n < sl;
    if (wave >= 30 && tick(140)) ring(e, 8, 1.8, e.t);
    if (e.mf === 0) {
      e.sc = (e.sc || 0) + sl; e.vul = e.st === 3;
      if (!e.st) { e.x += (p.x - e.x) * .02 * sl; if (tick(70)) { aim(e, 2.8, .2); aim(e, 2.8, -.2); } if (e.sc > 170) { e.st = 1; e.sc = 0; AM.sfx('warn'); } }
      else if (e.st === 1) { e.flash = e.sc % 10 < 5 ? 2 : 0; if (e.sc > 55) e.st = 2; }
      else if (e.st === 2) { e.y += 11 * sl; if (e.y > H - 80) { e.st = 3; e.sc = 0; shake = 10; rings.push({ x: e.x, y: e.y, r: 5, max: 90, c: '#ffcf6a' }); } }
      else if (e.sc > 100) { e.y -= 3 * sl; if (e.y <= 90) { e.y = 90; e.st = 0; e.sc = 0; } }
    } else if (e.mf === 1) {
      if (!e.sp) { e.sp = 1; for (let i = 0; i < 6; i++) enemies.push({ k: 4, orb: e, a: i * 1.047, x: e.x, y: e.y, hp: 2 + ((e.tr || 1) >> 1), vx: 0, vy: 0, r: 12, tr: e.tr, cd: 0, ph: 0 }); }
      e.sh = enemies.some(d => d.orb === e); e.x = W / 2 + Math.sin(e.t * .012) * 110;
      if (tick(e.sh ? 90 : 50)) aim(e, 3);
    } else if (e.mf === 3) {
      const c = e.t % 360; e.sh = c < 230; e.x += (p.x - e.x) * .008 * sl; if (c > 200 && c < 230) e.flash = e.t % 10 < 5 ? 2 : 0;
      if (e.sh && tick(80)) { aim(e, 2.6, .15); aim(e, 2.6, -.15); } if (!e.sh && tick(30)) aim(e, 3);
    } else if (e.mf === 4) {
      e.x += Math.sin(e.t * .015) * 1.2 * sl; const c = e.t % 240;
      if (c < 100) { e.lock = { x: p.x, y: p.y }; e.fired = 0; }
      else if (c >= 160 && !e.fired && e.lock) { e.fired = 1; const a = Math.atan2(e.lock.y - e.y, e.lock.x - e.x); ebullets.push({ x: e.x, y: e.y, vx: Math.cos(a) * 7, vy: Math.sin(a) * 7 }); e.lock = null; AM.sfx('boom'); }
    } else {
      const c = e.t % 390; e.sh = c < 240; e.vul = !e.sh; e.x += (W / 2 - e.x) * .02;
      if (e.sh && e.t % 9 < sl) for (let k = 0; k < 3; k++) { const a = e.t * .03 + k * 2.094; ebullets.push({ x: e.x, y: e.y, vx: Math.cos(a) * 2.1, vy: Math.sin(a) * 2.1 }); }
      if (c > 215 && c < 240) e.flash = 2;
    }
  }
  function miniShape(e, f) {
    const c = f ? ['#fff', '#fff', '#fff'] : e.mf === 0 ? ['#a8442a', '#ffcf6a', '#fff'] : e.mf === 1 ? ['#3a8f6a', '#9dff9d', '#eaffea'] : e.mf === 2 ? ['#4a5470', '#ff6a6a', '#ffd0d0'] : e.mf === 3 ? ['#2a5a8a', '#7fe3ff', '#e0f8ff'] : ['#3a3a3a', '#ff3d3d', '#ffffff'];
    if (e.mf === 0) { ctx.scale(1.5, 1.5); poly([[0, 30], [14, 10], [34, 16], [26, -8], [10, -20], [-10, -20], [-26, -8], [-34, 16], [-14, 10]], c[0]); poly([[-24, 6], [-30, 32], [-16, 12]], c[1]); poly([[24, 6], [30, 32], [16, 12]], c[1]); ell(0, -4, 6, 8, c[2]); }
    else if (e.mf === 1) { const g = 10 + Math.sin(time * .2) * 2; ell(0, 0, g + 8, g + 8, c[0]); ell(0, 0, g, g, e.sh ? c[0] : c[1]); ell(0, 0, g / 2, g / 2, c[2]); }
    else if (e.mf === 3) { ell(0, 0, 16, 16, c[0]); ell(0, 0, 7, 7, c[2]); if (e.sh) { ctx.strokeStyle = c[1]; ctx.lineWidth = 5; ctx.beginPath(); ctx.arc(0, 0, 28, time * .05, time * .05 + 4.7); ctx.stroke(); } }
    else if (e.mf === 4) { poly([[0, 34], [8, -6], [4, -26], [-4, -26], [-8, -6]], c[0]); ell(0, -4, 3, 6, c[1]); ctx.fillStyle = c[1]; ctx.fillRect(-1.5, 20, 3, 18); }
    else { ctx.rotate(time * .01); ctx.fillStyle = c[0]; ctx.fillRect(-22, -22, 44, 44); ctx.fillStyle = c[1]; for (let i = 0; i < 4; i++) { ctx.rotate(Math.PI / 2); ctx.fillRect(-4, 22, 8, 16); } ell(0, 0, e.sh ? 8 : 12, e.sh ? 8 : 12, e.sh ? c[0] : c[2]); }
  }
  function spawnCannons(e) { for (const d of [-1, 1]) enemies.push({ k: 5, own: e, dx: d * 48, x: e.x + d * 48, y: e.y + 24, hp: 14 + wave, r: 14, tr: e.tr, cd: rnd(30, 60), ph: 0 }); }
  function cannonAI(e, sl) {
    if (!enemies.includes(e.own)) { e.hp = 0; return; }
    if (mg && mg.boss === e.own) return;
    if (e.own.stun > 0 || e.own.wake > 0 || e.own.preT > 0 || !e.own.pre) return;
    e.x = e.own.x + e.dx; e.y = e.own.y + 24; e.cd -= sl;
    if (e.cd <= 0) { [-.25, 0, .25].forEach(o => aim(e, 2.6, o)); e.cd = 110 - (e.own.php || 1) * 8; }
  }
  function targetAI(e, sl) { e.x += (e.vx || 0) * sl; e.y += (e.vy || 0) * sl; if (e.x < 30 || e.x > W - 30) e.vx = -e.vx; if (e.y < 120 || e.y > 330) e.vy = -(e.vy || 0); if (e.life !== undefined && (e.life -= sl) <= 0) { e.expired = 1; e.hp = 0; } }
  function strikeSet(n, w) { const safe = rnd(70, W - 70); for (let i = 0; i < n; i++) { let x, g = 0; do { x = rnd(25, W - 25); } while (Math.abs(x - safe) < w + 30 && ++g < 20); strikes.push({ x, t: 75, w }); } AM.sfx('warn'); }
  function zoneSet() { const C = [62, 180, 298], sf = (Math.random() * 3) | 0; mg.safeX = C[sf]; C.forEach((c, i) => { if (i !== sf) strikes.push({ x: c, t: 90, w: 112 }); }); AM.sfx('warn'); }
  function mgStart(e) {
    if (mg) return;
    ebullets = []; strikes = []; warns = []; bullets = []; e.stun = 0;
    let k; do { k = 'cszehmr'[(Math.random() * 7) | 0]; } while (k === lastMg); lastMg = k;
    mg = { kind: k, boss: e, got: 0, need: 0, sp: 0, tot: 12, fk: 0, hit: false, pen: 0 };
    if (k === 'c') { mg.m = mg.t = 720; mg.need = 6; msg = { a: 'BOUCLIER ACTIVÉ', b: 'Touche 6 cibles avec ton avion (rouges = pièges)' }; }
    else if (k === 'm') { mg.m = mg.t = 720; mg.next = 1; mg.need = 4; mg.show = 150; const o = [0, 1, 2, 3].sort(() => Math.random() - .5), cl = ['#4aa3ff', '#ff5a5a', '#5aff8a', '#ffd966']; o.forEach((n, i) => enemies.push({ k: 6, x: 50 + i * 87, y: 180 + (i % 2) * 90, hp: 3, r: 17, tr: e.tr, ph: i, mgt: 1, num: n + 1, col: cl[i], vx: 0, vy: 0 })); msg = { a: '', b: 'Retiens l\'ordre 1→4, puis touche' }; }
    else if (k === 'r') { mg.m = mg.t = 600; mg.i = 0; mg.cps = [0, 1, 2, 3].map(i => ({ x: i % 2 ? rnd(220, 320) : rnd(40, 140), y: 170 + i * 85 })); msg = { a: '', b: 'COURSE : passe par A → B → C → D' }; }
    else if (k === 'h') { mg.m = mg.t = 540; mg.next = 1; mg.need = 5; const o = [0, 1, 2, 3, 4].sort(() => Math.random() - .5); o.forEach((n, i) => enemies.push({ k: 6, x: 50 + i * 65, y: 170 + (i % 2) * 90, hp: 3, r: 16, tr: e.tr, ph: i, mgt: 1, num: n + 1, col: '#ffd966', vx: 0, vy: 0 })); msg = { a: 'BOUCLIER ACTIVÉ', b: 'CHAÎNE : touche 1 → 2 → 3 → 4 → 5' }; }
    else if (k === 's') { mg.m = mg.t = 420; msg = { a: 'BOUCLIER ACTIVÉ', b: 'Survis aux éclairs pendant 7 s' }; }
    else if (k === 'z') { mg.m = mg.t = 480; msg = { a: 'BOUCLIER ACTIVÉ', b: 'Reste dans la zone sûre (blanche)' }; }
    else {
      mg.m = mg.t = 540; mg.want = (Math.random() * 4) | 0; const cl = ['#4aa3ff', '#ff5a5a', '#5aff8a', '#ffd966'];
      cl.forEach((c, i) => enemies.push({ k: 6, x: 60 + i * 80, y: 170 + (i % 2) * 70, hp: 3, r: 15, tr: e.tr, ph: i, mgt: 1, col: c, good: i === mg.want, vx: (i % 2 ? 1 : -1) * .9, vy: 0 }));
      msg = { a: 'BOUCLIER ACTIVÉ', b: 'Touche le noyau ' + ['BLEU', 'ROUGE', 'VERT', 'JAUNE'][mg.want] };
    }
    msg.a = '⚠ ÉPREUVE · ' + { c: '🎯 CIBLES', s: '⚡ SURVIE', z: '🟢 ZONE', e: '💠 NOYAU', h: '🔗 CHAÎNE', m: '🧠 MÉMOIRE', r: '🏁 COURSE' }[k]; msgT = 170; AM.sfx('up');
  }
  function bossNew(e, sl, ph, tick) {
    if (e.bi === 3) {                                   // Iron Fortress
      e.sh = enemies.some(c => c.k === 5 && c.own === e); e.x = W / 2 + Math.sin(e.t * .008) * 90;
      if (tick(110 - ph * 8)) { const n = 3 + ph; for (let i = 0; i < n; i++) aim(e, 2.9, (i - (n - 1) / 2) * .3); }
      if (ph >= 2 && tick(150)) aim(e, 2.2, 0);
      if (tick(240) && enemies.filter(a => a.k === 4).length < 4) for (const d of [-30, 30]) enemies.push({ k: 4, x: e.x + d, y: e.y + 20, hp: 1, vx: (p.x - e.x) / 90, vy: 2.6, r: 12, ph: 0, cd: 0 });
      if (ph === 4 && tick(170)) ring(e, 10, 1.9, e.t);
    } else {                                            // Storm
      e.sc = (e.sc || 0) + sl;
      if (!e.st) e.x = W / 2 + Math.sin(e.t * (.02 + ph * .004)) * (W / 2 - 60);
      if (tick(Math.round(200 - ph * 20))) { const safe = rnd(70, W - 70); for (let i = 0; i < 2 + ph; i++) { let x, g = 0; do { x = rnd(25, W - 25); } while (Math.abs(x - safe) < 60 && ++g < 20); strikes.push({ x, t: 75, w: 34 }); } AM.sfx('warn'); }
      if (ph >= 3 && tick(70)) aim(e, 3);
      if (ph >= 2) {
        if (!e.st) { if (e.sc > 280) { e.st = 1; e.sc = 0; e.tx = p.x; AM.sfx('warn'); } }
        else if (e.st === 1) { e.flash = e.sc % 10 < 5 ? 2 : 0; e.x += (e.tx - e.x) * .1 * sl; if (e.sc > 50) e.st = 2; }
        else if (e.st === 2) { e.y += 11 * sl; if (e.y > H - 90) e.st = 3; }
        else { e.y -= 4 * sl; if (e.y <= 100) { e.y = 100; e.st = 0; e.sc = 0; } }
      }
    }
  }
  function bossAI(e, sl) {
    if (!e.in) { e.y += 1.3 * sl; if (e.y >= 100) e.in = 1; return; }
    if (!e.pre) { e.pre = 1; e.preT = 110; msg = { a: '⚠ BOSS ⚠', b: 'PRÉPAREZ-VOUS…' }; msgT = 110; AM.sfx('boss'); }
    if (e.preT > 0) { e.preT--; e.flash = e.preT % 14 < 7 ? 1 : 0; if (e.preT === 0) { e.flash = 0; mgStart(e); } return; }
    e.t += sl;
    const bm = Math.min(2, 1 + (wave / 5 - 1) * .1), tick = n => e.t % Math.round(n / bm) < sl, f = e.hp / e.mhp, p2 = f < .66, p3 = f < .33;
    const fan = (n, v) => { for (let i = 0; i < n; i++) aim(e, v, (i - (n - 1) / 2) * .25); };
    const ph = f < .25 ? 4 : f < .5 ? 3 : f < .75 ? 2 : 1, first = !e.php;
    if (e.stun > 0 && --e.stun === 0) { e.vul = 0; msg = { a: 'BOSS RÉCUPÈRE', b: BOSSES[e.bi].n }; msgT = 90; AM.sfx('boss'); e.wake = 50; }
    if (e.stun > 0) { e.sh = 0; e.vul = 1; e.flash = e.stun % 12 < 6 ? 1 : 0; return; }
    if (e.wake > 0) { e.wake--; e.flash = e.wake % 10 < 5 ? 1 : 0; return; }
    if (ph !== e.php) {
      if (!first) { freeze = 16; shake = 8; rings.push({ x: e.x, y: e.y, r: 8, max: 150, c: BOSSES[e.bi].c[1] }); msg = { a: 'PHASE ' + ph, b: BOSSES[e.bi].n }; msgT = 100; AM.sfx('boss'); if (ph !== 3 || Math.random() < .5) mgStart(e); }
      e.php = ph; if (e.bi === 3 && (first || ph === 2 || ph === 4)) spawnCannons(e);
    }
    if (mg && mg.boss === e) { e.flash = 0; return; }
    if (e.bi >= 3) { bossNew(e, sl, ph, tick); return; }
    if (e.bi === 0) {                                   // Faucheur : anneaux, éventails, spirale
      e.x = W / 2 + Math.sin(e.t * (p3 ? .024 : .015)) * (W / 2 - 70);
      if (tick(p3 ? 70 : 100)) ring(e, 10 + Math.min(6, wave / 10 | 0), p3 ? 2.1 : 1.8, e.t);
      if (p2 && tick(95)) fan(3, 2.8);
      if (p3 && tick(45)) aim(e, 3);
    } else if (e.bi === 1) {                            // Perceuse : charges + onde de choc
      e.sc = (e.sc || 0) + sl;
      if (!e.st) { e.x += (p.x - e.x) * (p2 ? .05 : .03) * sl; if (e.sc % (p3 ? 35 : 50) < sl) fan(p2 ? 5 : 3, 3.1); if (e.sc > (p3 ? 110 : 190)) { e.st = 1; e.sc = 0; AM.sfx('warn'); } }
      else if (e.st === 1) { e.flash = 2; e.x += (p.x - e.x) * .1 * sl; if (e.sc > (p3 ? 28 : 40)) e.st = 2; }
      else if (e.st === 2) { e.y += (p3 ? 12 : 9) * sl; if (e.y > H - 90) { e.st = 3; ring(e, p2 ? 18 : 10, 2.6); shake = 10; } }
      else { e.y -= 3.5 * sl; if (e.y <= 100) { e.y = 100; e.st = 0; e.sc = 0; } }
    } else {                                            // Mère : drones, pluie, anneaux
      e.x = W / 2 + Math.sin(e.t * .015) * (W / 2 - 80);
      if (tick(p2 ? 110 : 150) && enemies.filter(a => a.k === 4).length < (p3 ? 8 : 6) + Math.min(4, wave / 10 | 0))
        for (const d of [-30, 30]) enemies.push({ k: 4, x: e.x + d, y: e.y + 20, hp: 1, vx: (p.x - e.x) / 90, vy: 2.6, r: 12, ph: 0, cd: 0 });
      if (tick(p2 ? 40 : 55)) { for (let i = 0; i < (p2 ? 4 : 3); i++) warns.push({ x: rnd(20, W - 20), t: 50 }); AM.sfx('warn'); }
      if (p2 && tick(90)) fan(3, 3);
      if (p3 && tick(70)) ring(e, 14, 2.2, e.t);
    }
  }

  // ---------- mise à jour ----------
  function update() {
    for (const c of clouds) { c.y += c.v * 2; if (c.y > H + 60) { c.y = -60; c.x = Math.random() * W; } }
    if (banner) banner--; if (bombT) bombT--; if (upT) upT--; if (msgT) msgT--; if (shotFx > 0) shotFx--; if (comboT > 0) comboT--; else if (combo > 0 && time % 30 === 0) combo--; if (shopT) shopT--;
    if (state !== 'play') { time++; if (equipFx > 0) equipFx--; if (hubGo > 0 && --hubGo === 0) HUBG[0].go(); parts.forEach(a => { a.x += a.vx; a.y += a.vy; a.life--; }); parts = parts.filter(a => a.life > 0); return; }
    time++; runF++; if (freeze > 0) { freeze--; return; }
    if (ultT > 0) ultT--; else if (ult < 100) ult = Math.min(100, ult + .012); uanUpd(); if (p.pw > 0) p.pw--; else if (enemies.some(e => !e.tst && Math.hypot(e.x - p.x, e.y - p.y) < 130)) usePower();
    const ox = p.x, oy = p.y; if (target) { p.x += (target.x - p.x) * .28 * p.spd; p.y += (target.y - 60 - p.y) * .28 * p.spd; }
    if (keys.ArrowLeft) p.x -= 5; if (keys.ArrowRight) p.x += 5; if (keys.ArrowUp) p.y -= 5; if (keys.ArrowDown) p.y += 5;
    p.x = Math.max(22, Math.min(W - 22, p.x)); p.y = Math.max(80, Math.min(H - 30, p.y));
    p.tilt += (Math.max(-1, Math.min(1, (p.x - ox) / 6)) - p.tilt) * .2; p.mvx = p.mvx * .7 + (p.x - ox) * .3; p.mvy = p.mvy * .7 + (p.y - oy) * .3;
    { const sk = skin(); if (sk.fx && time % [2, 3, 6][sv.o.part] === 0 && parts.length < PCAP() * .75) { const fc = FXC[sk.fx] || [0]; parts.push({ x: p.x + rnd(-8, 8), y: p.y + 18, vx: rnd(-.6, .6), vy: rnd(1, 2.5), life: 25, max: 25, r: rnd(1.5, 3), c: sk.rb ? `hsl(${Math.random() * 360 | 0},90%,60%)` : fc[(Math.random() * fc.length) | 0] }); } }
    if (p.inv) p.inv--; if (shake) shake--;
    for (const t of TIMERS) if (p[t[0]] > 0) p[t[0]]--;
    if (p.regen && p.lives < 6 && ++p.rg >= 1500) { p.rg = 0; p.lives++; }
    if (p.vcd > 0) p.vcd--;
    if (p.nova && time % 240 === 0) { rings.push({ x: p.x, y: p.y, r: 5, max: 110, c: '#ffd966' }); enemies.forEach(e => { if (!protectd(e) && (e.x - p.x) ** 2 + (e.y - p.y) ** 2 < 12000) { e.hp -= 4 + (e.tr || 1); e.flash = 3; } }); }
    if (p.storm && time % 100 === 0) enemies.filter(e => e.y > 20 && !protectd(e)).slice(0, 2).forEach(e => { e.hp -= 3; e.flash = 3; boom(e.x, e.y, 6, ['#fff', '#9be7ff']); });
    if (--p.cd <= 0) { if (!mg) shoot(); p.cd = Math.max(3, Math.round(((p.rapid > 0 ? 6 : p.power >= 3 ? 9 : 11) - (p.lvl >= 4 ? 2 : 0)) / p.rate)); }
    const sl = p.slow > 0 ? .5 : 1; metaUpdate();

    // vagues
    if (!GM.allowWaveProgression) testEnv(); else {
    wt++;
    while (wq.length && wq[0].t <= wt && enemies.length < ENCAP() && !mg) spawn(wq.shift());
    if (!wq.length && !enemies.length) { if (!clr) { clr = 1; const bn = Math.round(waveBonus(wave) * (GM.allowRewards ? coinMul() : 1)); earn(bn, true); if (wave % 5) { AM.sfx('win'); msg = { a: 'VAGUE TERMINÉE !', b: '+' + bn + ' ◎' }; msgT = 110; } } if (--gap <= 0) { nextWave(); gap = 100; } } else { gap = 100; clr = 0; }
    }

    for (const b of bullets) { b.x += b.vx; b.y += b.vy; }
    bullets = bullets.filter(b => b.y > -20);

    for (const e of enemies) {
      e.ph += .05 * sl; if (mg && e !== mg.boss && !e.mgt) { e.flash = 0; continue; } const rf = Math.min(2.4, 1 + ((e.tr || 1) - 1) * .09) * (e.el ? 1.3 : 1);
      if (e.beh === 'ghost') e.sh = time % 180 < 70; else if (e.beh === 'heal' && time % 60 === 0) enemies.forEach(a => { if (a !== e && a.k < 3 && a.hp < a.mhp && (a.x - e.x) ** 2 + (a.y - e.y) ** 2 < 8100) a.hp = Math.min(a.mhp, a.hp + 1 + ((e.tr || 1) >> 1)); }); else if (e.beh === 'hunt') e.x += (p.x - e.x) * .02 * sl;
      if (e.k === 0) {                                      // éclaireur : reste en haut et patrouille
        if (e.y < e.hy) e.y += e.vy * sl;
        else {
          e.y = e.hy + (e.tst ? 0 : Math.sin(e.ph * 2) * 8); e.x += (e.tst ? 0 : e.dx * 1.5) * sl;
          if (e.x < 24 || e.x > W - 24) e.dx *= -1;
          e.cd -= sl * rf; if (e.cd <= 0) { ebullets.push({ x: e.x, y: e.y + 10, vx: 0, vy: 2.8 }); e.cd = rnd(110, 200); }
        }
      }
      else if (e.k === 1) {
        e.y += e.vy * sl; e.x += (p.x - e.x) * .006 * sl; e.cd -= sl * rf;
        if (e.cd <= 0 && e.y > 0 && e.y < H - 200) { aim(e, 3.2); if ((e.tr || 1) >= 5) { aim(e, 3.2, .18); aim(e, 3.2, -.18); } e.cd = rnd(80, 130); }
      } else if (e.k === 2 && e.mini) { miniAI(e, sl);
      } else if (e.k === 2) {
        if (e.y < 80) e.y += .7 * sl; else e.x += e.bx * 1.1 * sl;
        if (e.x > W - 55 || e.x < 55) e.bx *= -1;
        e.cd -= sl * rf; if (e.cd <= 0) { [-.3, 0, .3].forEach(o => aim(e, 2.7, o)); e.cd = 85; }
      } else if (e.k === 4) { if (e.orb) { if (enemies.includes(e.orb)) { e.a += .03 * sl; e.x = e.orb.x + Math.cos(e.a) * 62; e.y = e.orb.y + Math.sin(e.a) * 48; } else { e.orb = null; e.vx = (p.x - e.x) / 90; e.vy = 2.6; } } else { e.x += e.vx * sl; e.y += e.vy * sl; } }
      else if (e.k === 5) cannonAI(e, sl); else if (e.k === 6) targetAI(e, sl); else bossAI(e, sl);

      for (const b of bullets) if (!b.dead && !(b.h && b.h.includes(e)) && hit(e, b, e.r + 4 * b.sz)) { (b.h = b.h || []).push(e); if (b.pr > 0) b.pr--; else b.dead = true; e.hp -= e.sh ? 0 : capDmg(e, b.d * (e.vul ? 2 : 1)); e.flash = 3; if (p.frac && !b.fr && bullets.length < 260) for (const s2 of [-1, 1]) bullets.push({ x: b.x, y: b.y, vx: s2 * 2, vy: -8, d: b.d * .5, sz: b.sz, pr: 0, fr: 1, st: b.st }); boom(b.x, b.y, 2, ['#fff', '#ffd966']); }
      if (e.hp <= 0) kill(e);
      else if (canHit() && e.k !== 6 && !passive(e) && hit(e, p, e.r + 4)) { hurt(); if (GM.allowEnemyDeath && (e.k < 2 || e.k === 4)) { e.hp = 0; kill(e, true); } }
      if (e.flash) e.flash--;
    }
    bullets = bullets.filter(b => !b.dead);
    enemies = enemies.filter(e => e.hp > 0 && e.y < H + 50 && e.x > -60 && e.x < W + 60);

    if (mg) { ebullets.length = 0; warns.length = 0; }
    budget(); relief();
    for (const b of ebullets) {
      b.x += b.vx * sl; b.y += b.vy * sl;
      const ddx = b.x - p.x, ddy = b.y - p.y, d2 = ddx * ddx + ddy * ddy;
      if (d2 < 81 && canHit()) { b.dead = true; hurt(); }
      else if (!b.gz) { if (d2 < 576 && d2 < (b.nd || 1e9) && p.inv <= 0 && !(p.ghost > 0)) b.nd = d2; else if (b.nd && d2 > b.nd + 6) { b.gz = 1; perfectDodge(); } }
    }
    ebullets = ebullets.filter(b => !b.dead && b.y < H + 20 && b.y > -20 && b.x > -20 && b.x < W + 20);

    for (const u of pups) {
      u.y += u.t === 'K' ? .8 : 1.5; u.ph += .08; if (p.mag) u.x += (p.x - u.x) * .06;
      if (hit(u, p, 28)) { u.dead = true; add(100); pickup(u.t); boom(u.x, u.y, 12, ['#fff', '#9be7ff']); }
    }
    pups = pups.filter(u => !u.dead && u.y < H + 30);
    floats.forEach(q => { q.y -= .6; q.l--; });
    rings.forEach(r => { r.r += (r.max - r.r) * .12 + 1; }); rings = rings.filter(r => r.r < r.max - 1);
    strikes.forEach(st => { st.t--; if (st.t <= 0 && canHit() && Math.abs(p.x - st.x) < st.w / 2 + 3) hurt(); }); strikes = strikes.filter(st => st.t > -14);
    if (mg) {
      mg.t--; const bs = mg.boss, alive = enemies.includes(bs);
      const end = (win, why) => {
        enemies.forEach(x => { if (x.mgt) { x.expired = 1; x.hp = 0; } });
        if (win) { if (alive) { bs.vul = 1; bs.sh = 0; bs.stun = 420; ebullets = []; strikes = []; warns = []; AM.sfx('boom'); boom(bs.x, bs.y, 40, ['#7fe3ff', '#fff', '#ffd966']); shake = 10; rings.push({ x: bs.x, y: bs.y, r: 8, max: 160, c: '#7fe3ff' }); } combo += 10; earn(60 + wave * 5); gainXp(25 + wave); sv.st.mg = (sv.st.mg || 0) + 1; msg = { a: '🛡 BOUCLIER DÉTRUIT !', b: '😵 ÉTOURDI · VULNÉRABLE 7 s' }; msgT = 150; AM.sfx('win'); }
        else { if (alive) bs.wake = 70; msg = { a: 'DÉFI RATÉ', b: why + ' — le combat commence !' }; msgT = 140; }
        mg = null;
      };
      if (!alive) end(false, '');
      else if (mg.kind === 'c') {
        if (mg.sp < mg.tot && ((mg.m - mg.t) % 40 === 0 || (mg.t % 10 === 0 && enemies.filter(x => x.mgt && !x.fake).length < 3))) { mg.sp++; enemies.push({ k: 6, x: rnd(40, W - 40), y: rnd(130, 300), vx: (Math.random() < .5 ? -1 : 1) * rnd(.6, 1.6), vy: rnd(-.8, .8), life: 240, fake: mg.fk < 2 && mg.sp > 3 && Math.random() < .25 && ++mg.fk > 0, hp: 2 + (wave / 15 | 0), r: 14, tr: bs.tr, ph: 0, mgt: 1 }); }
        if (mg.got >= mg.need) end(true); else if ((mg.sp >= mg.tot && !enemies.some(x => x.mgt)) || mg.t <= 0) end(false, 'Seulement ' + mg.got + ' / ' + mg.need + ' cibles');
      } else if (mg.kind === 'r') {
        const cp = mg.cps[mg.i]; if (cp && Math.hypot(p.x - cp.x, p.y - cp.y) < 28) { mg.i++; AM.sfx('coin'); rings.push({ x: cp.x, y: cp.y, r: 5, max: 50, c: '#ffd966' }); }
        if (mg.i >= 4) end(true); else if (mg.t <= 0) end(false, 'Checkpoint ' + 'ABCD'[mg.i] + ' manqué');
      } else if (mg.kind === 'h' || mg.kind === 'm') {
        if (mg.show > 0) mg.show--;
        if (mg.got >= mg.need) end(true); else if (mg.pen >= 99 || mg.t <= 0) end(false, mg.pen ? 'Mauvais ordre' : 'Temps écoulé');
      } else if (mg.kind === 's' || mg.kind === 'z') {
        if (mg.kind === 's' && mg.t % 70 === 0 && mg.t > 80) strikeSet(2, 56);
        if (mg.kind === 'z' && mg.t % 120 === 0 && mg.t > 100) zoneSet();
        if (mg.t <= 0) { if (!mg.hit) end(true); else end(false, 'Tu as été touché'); }
      } else { if (mg.got >= 99) end(true); else if (mg.pen >= 99 || mg.t <= 0) end(false, mg.pen ? 'Mauvais noyau' : 'Temps écoulé'); }
    }
    if (mgRetry > 0 && --mgRetry === 0) { const b = enemies.find(x => x.k === 3); if (b) mgStart(b); }
    warns.forEach(w => { if (--w.t === 0) ebullets.push({ x: w.x, y: 90, vx: 0, vy: 3.4 }); }); warns = warns.filter(w => w.t > 0); floats = floats.filter(q => q.l > 0);
    parts.forEach(a => { a.x += a.vx; a.y += a.vy; a.vx *= .96; a.vy *= .96; a.life--; });
    parts = parts.filter(a => a.life > 0);
  }

  function pickup(t) {
    if (t === 'P') { if (p.power < 3) p.power++; else add(300); }
    else if (t === 'S') p.shield = 420;
    else if (t === 'H') p.lives = Math.min(5, p.lives + 1);
    else if (t === 'R') p.rapid = 480;
    else if (t === 'X') p.x2 = 600;
    else if (t === 'W') p.wing = 600;
    else if (t === 'T') p.slow = 360; else if (t === 'F') p.fort = 720; else if (t === 'M') p.mega = 480;
    else if (t === 'G') { p.ghost = 240; floats.push({ x: p.x, y: p.y - 30, l: 60, t: 'PHASE FANTÔME' }); }
    else if (t === 'C') { const r = Math.random(), m = r < .62 ? 2 : r < .9 ? 4 : 8; p.cb = Math.max(p.cbT > 0 ? p.cb : 1, m); p.cbT = Math.max(p.cbT, m === 2 ? 1500 : m === 4 ? 1200 : 900); floats.push({ x: p.x, y: p.y - 30, l: 70, t: '◎ ×' + p.cb }); AM.sfx('rare'); }
    else if (t === 'Y') { p.sec = 1; floats.push({ x: p.x, y: p.y - 30, l: 70, t: '♥ SECOND SOUFFLE' }); }
    else if (t === 'K') { giveChest(rollChestR(1)); if (evt && evt.k === 'treasure') evt.done = 1; }
    else if (t === '$') { const v = Math.round((12 + wave * 3) * coinMul()); earn(v, true); floats.push({ x: p.x, y: p.y - 30, l: 45, t: '+' + v + ' ◎' }); AM.sfx('coin'); }
    else if (t === 'B') {                                 // bombe
      bombT = 22; shake = 14; ebullets = [];
      for (const e of enemies) { if (protectd(e)) continue; e.hp -= e.k === 3 ? 15 : 8; e.flash = 3; }
    }
  }
  function kill(e, ram) {
    if (ult < 100) ult = Math.min(100, ult + (e.k === 3 ? 35 : e.mini ? 20 : e.el ? 8 : e.k === 2 ? 10 : e.k < 2 ? 2 : 0));
    AM.sfx(e.k >= 2 ? 'boom' : 'pop'); if (e.k < 6) killFx(e);
    if (e.k === 6 && mg) { if (e.expired) mg.miss = (mg.miss || 0) + 1; else if (mg.kind === 'h' || mg.kind === 'm') { if (e.num === mg.next) { mg.next++; mg.got++; } else { mg.pen++; shake = 4; AM.sfx('hit'); if (mg.pen >= 3) mg.pen = 99; } } else if (mg.kind === 'c') { if (e.fake) { mg.pen++; shake = 4; AM.sfx('hit'); } else mg.got++; } else if (mg.kind === 'e') { if (e.good) mg.got = 99; else { mg.pen++; shake = 4; AM.sfx('hit'); if (mg.pen >= 2) mg.pen = 99; } } }
    if (e.beh === 'split') for (const d of [-14, 14]) enemies.push({ k: 0, x: e.x + d, y: e.y, hp: 1, vy: 2.4, ph: 0, r: 12, cd: 90, hy: Math.min(e.y + 30, 220), dx: d < 0 ? -1 : 1, tr: e.tr });
    sv.st.kills++; if (e.el) sv.st.elites++; if (e.k === 3) sv.st.boss++; sv.bs[bkey(e)] = (sv.bs[bkey(e)] || 0) + 1; combo += (e.el || e.mini) ? 4 : 1; sv.st.bestCombo = Math.max(sv.st.bestCombo, cm());
    { const ti = cm() >= 50 ? 4 : cm() >= 20 ? 3 : cm() >= 10 ? 2 : cm() >= 5 ? 1 : 0; if (ti > ctier) { rings.push({ x: p.x, y: p.y, r: 6, max: 50 + ti * 35, c: ['#fff', '#ffd966', '#ff9a3d', '#ff3d7f'][ti - 1] }); floats.push({ x: p.x, y: p.y - 36, l: 60, t: 'COMBO x' + cm() }); if (ti >= 3) shake = 6; AM.sfx('rare'); } ctier = ti; } comboT = 150; p.kills++; if (p.stars && !mg && p.kills % 8 === 0) for (let i = 0; i < 5; i++) bullets.push({ x: p.x, y: p.y - 10, vx: (i - 2) * 1.4, vy: -9, d: p.dmg * 2, sz: 1.3, pr: 2, st: 'star' }); p.bc = Math.max(p.bc, combo); add(([10, 30, 120, 1000, 5, 40, 0][e.k] || 0) * cm()); const cn = Math.round(coinOf(e) * (GM.allowRewards ? coinMul() * (1 + Math.min(.6, (cm() - 1) * .02 * p.comboB)) : 1)); earn(cn, true); if (e.mini && Math.random() < .5) giveChest(rollChestR(1)); else if (e.k === 3) giveChest(rollChestR(2)); else if (e.el && Math.random() < .012 + p.chestC) giveChest(rollChestR(0)); if (rush && e.k < 6) rush.k++; if (cn) floats.push({ x: e.x, y: e.y, l: 45, t: '+' + cn + ' ◎' }); checkAll(); gainXp(Math.round(([2, 5, 15, 80, 1, 3, 0][e.k] || 0) * (1 + ((e.tr || 1) - 1) * .2) * (e.el ? 3 : 1)));
    boom(e.x, e.y, e.k >= 2 && e.k < 4 ? 55 : 16, skin().dk === 'demon' ? ['#ff2a3a', '#3a0010', '#ff7a3d', '#111'] : skin().dk === 'angel' ? ['#fff1b8', '#fff', '#ffd966'] : ['#ff7a3d', '#ffd966', '#fff', '#444']);
    if (e.k >= 2 && e.k < 4) { shake = 18; rings.push({ x: e.x, y: e.y, r: 5, max: 80, c: '#ffd966' }); }
    if (e.mini) { fragGain(1 + (Math.random() < .3 * p.luck ? 1 : 0)); p.mb = (p.mb || 0) + 1; const cb = cm() >= 10 ? .25 : 0, mult = [1.1, 1.25, 1.5, 2][Math.min(3, p.mb - 1)] + cb, base = coinOf(e), ex = Math.round(base * (mult - 1)); earn(ex); msg = { a: 'MINI-BOSS VAINCU ! +' + base + ' ◎', b: 'BOOST ×' + mult.toFixed(2) + (cb ? ' (combo)' : '') + ' = +' + (base + ex) + ' ◎' }; msgT = 170; }
    if (e.k === 3) { const fg = 3 + (wave / 10 | 0); fragGain(fg); msg = { a: 'BOSS VAINCU !', b: '+' + cn + ' ◎   +' + fg + ' ◆' }; msgT = 170; boom(e.x, e.y, 50, ['#ffd966', '#ffb020', '#fff']); shake = 20; AM.sfx('win'); dropPup(e.x - 20, e.y); dropPup(e.x + 20, e.y); dropPup(e.x, e.y + 20); return; }
    if (!ram && (e.el || e.mini || Math.random() < ([.07, .2, .6, 0, 0, 0, 0][e.k] || 0) * p.luck)) dropPup(e.x, e.y);
  }
  function hurt() {
    if (!GM.allowDamage) { p.inv = Math.max(p.inv, 20); return; }
    pdStreak = 0; if (evt && evt.k === 'meteor') evt.hit = 1; if (mg && mg.kind === 's') mg.hit = true; AM.sfx('hit'); try { if (sv.o.vib && navigator.vibrate) navigator.vibrate(25); } catch (e) {} if (p.varm && p.vcd <= 0) { p.vcd = 720; p.inv = 45; rings.push({ x: p.x, y: p.y, r: 5, max: 70, c: '#b57cff' }); return; }
    if (p.shield > 0) { p.shield = 0; p.inv = 60; boom(p.x, p.y, 14, ['#9be7ff', '#fff']); return; }
    p.lives--; p.inv = 120; p.power = Math.max(minP(), p.power - 1); shake = 12;
    boom(p.x, p.y, 25, ['#ff7a3d', '#ffd966', '#fff']);
    if (p.lives <= 0 && p.phx) { p.phx = 0; p.lives = 3; p.shield = 300; p.inv = 120; boom(p.x, p.y, 40, ['#ff7a3d', '#ffd966', '#fff']); }
    else if (p.lives <= 0 && p.sec) { p.sec = 0; p.lives = 1; p.shield = 240; p.inv = 180; msg = { a: '🌟 SECOND SOUFFLE', b: 'Tu survis avec 1 PV !' }; msgT = 140; boom(p.x, p.y, 40, ['#ff9ad0', '#fff']); }
    else if (p.lives <= 0) {
      state = 'over'; time = 0; inRun = false; endRun(); boom(p.x, p.y, 60, ['#ff7a3d', '#ffd966', '#444']);
      if (score > hi) { hi = score; try { localStorage.setItem('escadrille-hi', hi); } catch (e) {} }
    }
  }

  // ---------- dessin ----------
  function plane(x, y, sc, body, wing, glow) {
    ctx.save(); ctx.translate(x, y); ctx.scale(sc, sc);
    ctx.fillStyle = wing;
    ctx.beginPath(); ctx.moveTo(0, -4); ctx.lineTo(26, 12); ctx.lineTo(26, 17); ctx.lineTo(0, 10); ctx.lineTo(-26, 17); ctx.lineTo(-26, 12); ctx.closePath(); ctx.fill();
    ctx.beginPath(); ctx.moveTo(0, 14); ctx.lineTo(11, 25); ctx.lineTo(0, 22); ctx.lineTo(-11, 25); ctx.closePath(); ctx.fill();
    ctx.fillStyle = body;
    ctx.beginPath(); ctx.moveTo(0, -24); ctx.quadraticCurveTo(8, -8, 6, 20); ctx.lineTo(-6, 20); ctx.quadraticCurveTo(-8, -8, 0, -24); ctx.fill();
    ctx.fillStyle = glow; ctx.beginPath(); ctx.ellipse(0, -6, 3.2, 6, 0, 0, 7); ctx.fill();
    ctx.restore();
  }
  function ship(x, y, sc, sk, lv, tilt = 0, xl = 1, sl = 1) {
    const wing = sk.rb ? `hsl(${(time * 4) % 360},85%,58%)` : sk.wing, body = sk.rb ? '#fff' : sk.body;
    ctx.save(); ctx.translate(x, y + (state === 'play' ? Math.sin(time * .08) * 1.2 : 0)); ctx.rotate(tilt * .22); ctx.scale(sc, sc); if (sk.fx === 'ghost') ctx.globalAlpha = .65;
    if (lv >= 5) { ctx.strokeStyle = `rgba(255,215,90,${.3 + .15 * Math.sin(time * .1)})`; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(0, 0, 36, 0, 7); ctx.stroke(); }
    if (sk.void) { ctx.strokeStyle = `rgba(160,80,255,${.3 + .2 * Math.sin(time * .15)})`; ctx.lineWidth = 2; ctx.beginPath(); ctx.ellipse(0, 0, 34 + Math.sin(time * .2) * 2, 30, 0, 0, 7); ctx.stroke(); }
    if (lv >= 6) { ctx.strokeStyle = 'rgba(160,230,255,.85)'; ctx.lineWidth = 1.5; for (const d of [-1, 1]) { ctx.beginPath(); ctx.moveTo(d * 26, 14); for (let i = 1; i < 4; i++) ctx.lineTo(d * (26 + i * 3 + Math.random() * 4), 14 + i * 6 - Math.random() * 5); ctx.stroke(); } }
    if (lv >= 4) { poly([[-26, 16], [-36, 30], [-20, 20]], wing); poly([[26, 16], [36, 30], [20, 20]], wing); }
    const sh = sk.shape;
    if (sh === 'inter') {
      for (const d of [-1, 1]) { poly([[d * 4, 0], [d * 32, 26], [d * 32, 30], [d * 4, 20]], wing); ell(d * 9, 22, 2.8, 6, sk.glow); }
      poly([[0, -30], [5, -10], [4, 22], [-4, 22], [-5, -10]], body); ell(0, -8, 2.5, 6, sk.glow);
    } else if (sh === 'falcon') {
      for (const d of [-1, 1]) poly([[d * 3, -10], [d * 36, 0], [d * 28, 10], [d * 3, 6]], wing);
      poly([[-9, 25], [0, 15], [9, 25]], wing); ell(0, 0, 6.5, 23, body); ell(0, -8, 3, 6, sk.glow);
    } else if (sh === 'stealth') {
      poly([[0, -28], [32, 24], [0, 13], [-32, 24]], body); poly([[0, -10], [15, 15], [0, 10], [-15, 15]], wing); ell(0, 0, 3, 7, sk.glow);
    } else if (sh === 'demon') {
      const wg = ctx.createLinearGradient(0, -30, 0, 24); wg.addColorStop(0, '#5a0a18'); wg.addColorStop(1, '#0a0206');
      for (const d of [-1, 1]) {
        ctx.fillStyle = wg; ctx.beginPath(); ctx.moveTo(d * 4, -8); ctx.bezierCurveTo(d * 20, -26, d * 40, -16, d * 46, 8); ctx.lineTo(d * 37, 5); ctx.lineTo(d * 35, 15); ctx.lineTo(d * 27, 9); ctx.lineTo(d * 22, 21); ctx.lineTo(d * 12, 11); ctx.lineTo(d * 4, 15); ctx.closePath(); ctx.fill();
        ctx.strokeStyle = '#7a8296'; ctx.lineWidth = 1; ctx.stroke();
        ctx.strokeStyle = `rgba(255,50,60,${.45 + .3 * Math.sin(time * .15 + d)})`; ctx.beginPath(); ctx.moveTo(d * 8, -4); ctx.lineTo(d * 20, -8); ctx.lineTo(d * 28, 2); ctx.lineTo(d * 36, 0); ctx.stroke();
      }
      const bg = ctx.createLinearGradient(-7, 0, 7, 0); bg.addColorStop(0, '#1a1c26'); bg.addColorStop(.5, '#4a5060'); bg.addColorStop(1, '#14161e');
      ctx.fillStyle = bg; ctx.beginPath(); ctx.moveTo(0, -26); ctx.quadraticCurveTo(9, -8, 6, 22); ctx.lineTo(-6, 22); ctx.quadraticCurveTo(-9, -8, 0, -26); ctx.fill();
    } else if (sh === 'angel') {
      for (const d of [-1, 1]) for (let k = 0; k < 3; k++) { const s2 = 1 - k * .22, ag = ctx.createLinearGradient(d * 4, -10, d * 50, 10); ag.addColorStop(0, '#fff'); ag.addColorStop(1, k ? '#ffe9a0' : '#ffd966'); ctx.fillStyle = ag; ctx.beginPath(); ctx.moveTo(d * 4, -6 + k * 3); ctx.bezierCurveTo(d * 22 * s2, -26 + k * 6, d * 46 * s2, -16 + k * 6, d * 50 * s2, 8 + k * 6); ctx.bezierCurveTo(d * 36 * s2, 2 + k * 5, d * 26 * s2, 8 + k * 4, d * 4, 10 + k * 3); ctx.fill(); }
      const bg = ctx.createLinearGradient(-7, 0, 7, 0); bg.addColorStop(0, '#e8e8f0'); bg.addColorStop(.5, '#fff'); bg.addColorStop(1, '#e8e0c8');
      ctx.fillStyle = bg; ctx.beginPath(); ctx.moveTo(0, -26); ctx.quadraticCurveTo(9, -8, 6, 22); ctx.lineTo(-6, 22); ctx.quadraticCurveTo(-9, -8, 0, -26); ctx.fill(); ell(0, -8, 2.5, 6, '#ffd966');
    } else if (sh === 'arrow') {
      poly([[0, -30], [6, 6], [30, 22], [6, 14], [4, 24], [-4, 24], [-6, 14], [-30, 22], [-6, 6]], body); poly([[0, -18], [4, 4], [-4, 4]], wing); ell(0, -4, 2, 5, sk.glow);
    } else if (sh === 'ranger') {
      poly([[-34, 8], [34, 8], [34, 15], [-34, 15]], wing); ell(0, 0, 6, 22, body); ell(0, -8, 2.5, 5, sk.glow);
      for (const d of [-1, 1]) for (const xx of [14, 26]) ell(d * xx, 19, 2.5, 5, sk.glow);
    } else if (sh === 'mecha') {
      ctx.fillStyle = wing; ctx.fillRect(-30, -4, 60, 12); ctx.fillStyle = body; ctx.fillRect(-8, -22, 16, 44);
      ctx.fillStyle = sk.glow; ctx.fillRect(-15, 22, 8, 7); ctx.fillRect(7, 22, 8, 7); ell(0, -10, 3, 6, wing);
    } else if (sh === 'dragon') {
      for (const d of [-1, 1]) poly([[d * 3, -10], [d * 20, -22], [d * 26, -6], [d * 38, -8], [d * 30, 10], [d * 18, 6], [d * 10, 18], [d * 3, 8]], wing);
      ell(0, 0, 6, 24, body); poly([[0, -32], [4, -20], [-4, -20]], wing); ell(0, -12, 2, 3, sk.glow);
    } else if (sh === 'titan') {
      poly([[0, -26], [14, -6], [36, 12], [36, 22], [14, 18], [0, 26], [-14, 18], [-36, 22], [-36, 12], [-14, -6]], body);
      poly([[0, -14], [8, 0], [0, 12], [-8, 0]], wing); ell(-20, 22, 5, 5, sk.glow); ell(20, 22, 5, 5, sk.glow);
    } else plane(0, 0, 1, body, wing, sk.glow);
    if (lv >= 2) { ell(-17, 15, 3.2, 8, body); ell(17, 15, 3.2, 8, body); }
    if (lv >= 3) for (const d of [-1, 1]) { ctx.fillStyle = '#cfd8e3'; ctx.fillRect(d * 10 - 1.6, -12, 3.2, 13); poly([[d * 10 - 1.6, -12], [d * 10 + 1.6, -12], [d * 10, -18]], '#ffd966'); }
    if (lv >= 4) ell(0, 21, 4, 3, sk.rb ? wing : sk.flame);
    if (xl >= 2) { poly([[-14, 18], [-26, 30], [-10, 26]], wing); poly([[14, 18], [26, 30], [10, 26]], wing); }
    if (xl >= 3) ell(0, -1, 3.5 + Math.sin(time * .2), 5, '#bdf3ff');
    if (xl >= 4) { const no = Math.min(8, xl - 2); ctx.strokeStyle = 'rgba(140,220,255,.6)'; ctx.lineWidth = 1.5; ctx.beginPath(); ctx.ellipse(0, 0, 22, 9, time * .03, 0, 7); ctx.stroke(); for (let i = 0; i < no; i++) { const a = time * .06 + i * 6.283 / no; ell(Math.cos(a) * 24, Math.sin(a) * 10, 2, 2, '#fff'); } }
    if (xl >= 5) for (const d of [-1, 1]) { ell(d * 7, 25, 3, 5, sk.flame); poly([[d * 4, -6], [d * 24, -12], [d * 18, 2]], wing); }
    if (xl >= 7) { ctx.strokeStyle = `hsla(${(xl * 40) % 360},90%,70%,.5)`; ctx.lineWidth = 2; ctx.beginPath(); ctx.arc(0, 0, 30 + Math.sin(time * .1) * 2, 0, 7); ctx.stroke(); }
    if (!sk.cos) drawEvo(sk, sl); if (sk.id === 'pikachu') drawPika(); if (sk.id === 'samurai') drawSam();
    if (sk.dk) drawDeco(sk, tilt);
    if (sk.cos) drawCos(sk.cos, sl);
    if (sk.r >= 8) drawDiv(sk, sl);
    ctx.restore();
  }
  const poly = (pts, fill) => { ctx.fillStyle = fill; ctx.beginPath(); pts.forEach((q, i) => i ? ctx.lineTo(q[0], q[1]) : ctx.moveTo(q[0], q[1])); ctx.closePath(); ctx.fill(); };
  const ell = (x, y, rx, ry, fill) => { ctx.fillStyle = fill; ctx.beginPath(); ctx.ellipse(x, y, rx, ry, 0, 0, 7); ctx.fill(); };

  // ennemis : nez vers le bas (+y)
  function enemy(e) {
    const f = e.flash > 0, k = e.k;
    let c = f ? ['#fff', '#fff', '#fff'] : null;
    ctx.save(); ctx.translate(e.x, e.y);
    const tr = e.tr || 1;
    if (e.sh && e.beh === 'ghost') ctx.globalAlpha = .35;
    if (e.el || e.mini || e.ch || e.cor) { ctx.strokeStyle = `rgba(${e.cor ? '255,50,50' : e.ch ? '181,124,255' : '255,170,40'},${.5 + .3 * Math.sin(time * .2)})`; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(0, 0, e.r + 8, 0, 7); ctx.stroke(); }
    else if (tr >= 4) { ctx.strokeStyle = `rgba(255,255,255,${.25 + .1 * Math.sin(time * .15)})`; ctx.lineWidth = 2; ctx.beginPath(); ctx.arc(0, 0, e.r + 5, 0, 7); ctx.stroke(); }
    if (k === 0 || k === 4) {                           // éclaireur / drone : aile delta
      c = c || (k === 0 ? ['#6a2fb8', '#ff3df0', '#fff'] : ['#0f8fa8', '#7ff3ff', '#fff']);
      const s = k === 0 ? .9 : .65; ctx.scale(s, s);
      poly([[0, 22], [22, -14], [8, -8], [0, -16], [-8, -8], [-22, -14]], c[0]);
      poly([[0, 10], [5, -6], [-5, -6]], c[1]);
    } else if (k === 5) {
      ctx.fillStyle = f ? '#fff' : '#3a4150'; ctx.fillRect(-12, -10, 24, 20); ctx.fillStyle = f ? '#fff' : '#ff9a3d'; ctx.fillRect(-4, 6, 8, 14); ell(0, -2, 5, 5, '#ffd0a0');
    } else if (k === 6) {
      ctx.strokeStyle = f ? '#fff' : e.fake ? '#ff3d3d' : e.col || '#ffe45e'; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(0, 0, 12 + Math.sin(time * .2) * 2, 0, 7); ctx.stroke(); ell(0, 0, e.col ? 9 : 5, e.col ? 9 : 5, e.fake ? '#ff3d3d' : e.col || '#ff4d6d'); if (e.fake) { ctx.beginPath(); ctx.moveTo(-6, -6); ctx.lineTo(6, 6); ctx.moveTo(6, -6); ctx.lineTo(-6, 6); ctx.stroke(); }
    } else if (k === 1) {                               // chasseur : double fuselage
      c = c || ['#1f7a5a', '#9dff6a', '#d9fff0'];
      ctx.scale(1.05, 1.05);
      ctx.fillStyle = c[1]; ctx.fillRect(-26, -8, 52, 9);
      for (const s of [-1, 1]) { ell(s * 13, 0, 5, 18, c[0]); poly([[s * 13 - 5, 14], [s * 13 + 5, 14], [s * 13, 24]], c[0]); }
      ell(0, 3, 5, 6, c[2]);
    } else if (k === 2 && e.mini) { miniShape(e, f);
    } else if (k === 2) {                               // gros porteur : aile volante
      c = c || ['#2a2f3a', '#ff8a1f', '#ffdc8a'];
      ctx.scale(e.mini ? 1.8 : 1.3, e.mini ? 1.8 : 1.3);
      poly([[0, 28], [36, -6], [30, -20], [12, -12], [0, -22], [-12, -12], [-30, -20], [-36, -6]], c[0]);
      ctx.fillStyle = c[1]; ctx.fillRect(-4, -10, 8, 26);
      ell(-20, -14, 5, 5, c[2]); ell(20, -14, 5, 5, c[2]);
    } else {                                            // mini-boss
      const bc = f ? c : BOSSES[e.bi].c; ctx.scale(2.2, 2.2);
      if (e.bi === 3) {
        poly([[-34, -14], [34, -14], [40, 10], [20, 24], [-20, 24], [-40, 10]], bc[0]); ctx.fillStyle = bc[1]; ctx.fillRect(-30, -10, 60, 4); ctx.fillRect(-30, 10, 60, 4);
        ell(0, 4, 8, 8, e.sh ? '#555' : bc[1]); ell(0, 4, 4, 4, bc[2]);
      } else if (e.bi === 4) {
        poly([[0, 26], [30, -4], [18, -18], [0, -8], [-18, -18], [-30, -4]], bc[0]); ctx.strokeStyle = bc[1]; ctx.lineWidth = 1.5; ctx.beginPath();
        ctx.moveTo(-30, -4); for (let i = 1; i < 5; i++) ctx.lineTo(-30 + i * 15, -4 + (i % 2 ? -8 : 6) * (.6 + .4 * Math.sin(time * .4 + i))); ctx.stroke(); ell(0, -2, 6, 8, bc[2]);
      } else if (e.bi === 0) {
        poly([[0, 28], [36, -6], [30, -20], [12, -12], [0, -22], [-12, -12], [-30, -20], [-36, -6]], bc[0]);
        poly([[36, -6], [48, 8], [30, -2]], bc[1]); poly([[-36, -6], [-48, 8], [-30, -2]], bc[1]);
        ell(0, 0, 6, 9, bc[2]);
      } else if (e.bi === 1) {
        poly([[0, 40], [22, 0], [26, -24], [-26, -24], [-22, 0]], bc[0]);
        poly([[0, 40], [10, 12], [-10, 12]], bc[1]);
        ctx.strokeStyle = bc[1]; ctx.lineWidth = 3;
        for (let i = 0; i < 3; i++) { ctx.beginPath(); ctx.moveTo(-18 + i * 2, -2 + i * -8); ctx.lineTo(18 - i * 2, -8 + i * -8); ctx.stroke(); }
        poly([[26, -24], [40, -30], [26, -10]], bc[1]); poly([[-26, -24], [-40, -30], [-26, -10]], bc[1]);
      } else {
        ell(0, 0, 40, 14, bc[0]); ell(0, 6, 40, 8, bc[1]); ell(0, -6, 16, 12, bc[2]);
        for (let i = -3; i <= 3; i++) ell(i * 10, 4, 2.5, 2.5, f ? '#fff' : (Math.floor(e.ph * 3 + i) & 1 ? '#fff' : bc[1]));
      }
    }
    ctx.restore();
  }

  function decor(e) {
    const tr = e.tr || 1;
    if (mg && mg.boss === e) { ctx.strokeStyle = 'rgba(127,227,255,.85)'; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(e.x, e.y, e.r + 12 + Math.sin(time * .2) * 2, 0, 7); ctx.stroke(); ctx.fillStyle = 'rgba(127,227,255,.12)'; ctx.fill(); }
    if (e.num && mg && (mg.kind === 'h' || mg.show > 0)) { ctx.font = `bold 16px ${FONT}`; ctx.textAlign = 'center'; ctx.fillStyle = '#12293f'; ctx.fillText(e.num, e.x, e.y + 6); }
    if (e.k === 3 && e.stun > 0) { ctx.font = `bold 12px ${FONT}`; ctx.textAlign = 'center'; ctx.fillStyle = '#ffe45e'; ctx.fillText('VULNÉRABLE — ' + Math.ceil(e.stun / 60) + 's', e.x, e.y - e.r - 16); }
    if ((e.mf === 0 || e.bi === 1 || e.bi === 4) && e.st >= 1 && e.st <= 2) { ctx.fillStyle = `rgba(${e.st === 1 ? '255,210,60' : '255,60,60'},${.14 + .12 * Math.sin(time * .5)})`; ctx.fillRect(e.x - 28, e.y, 56, H); }
    if (tr >= 2 && e.k < 3 || tr >= 2 && e.k === 3 && false) { const n = Math.max(1, Math.min(5, tr >> 1)); ctx.fillStyle = TC[Math.min(tr, 10) - 1]; for (let i = 0; i < n; i++) ctx.fillRect(e.x - n * 3 + i * 6, e.y - e.r - 6, 4, 3); }
    if (e.mf === 4 && e.lock) { const c2 = e.t % 240, a = Math.atan2(e.lock.y - e.y, e.lock.x - e.x); ctx.strokeStyle = c2 < 100 ? 'rgba(255,255,255,.35)' : 'rgba(255,60,60,.85)'; ctx.lineWidth = c2 < 100 ? 1 : 3; ctx.beginPath(); ctx.moveTo(e.x, e.y); ctx.lineTo(e.x + Math.cos(a) * 900, e.y + Math.sin(a) * 900); ctx.stroke(); }
    if (e.beh) { ctx.font = `bold 12px ${FONT}`; ctx.textAlign = 'center'; ctx.fillStyle = '#fff'; ctx.fillText({ split: '÷', ghost: '◌', heal: '✚', hunt: '➤', jug: '■' }[e.beh], e.x, e.y - e.r - 14); }
    if (e.el || e.mini || e.ch || e.cor) { ctx.font = `bold 9px ${FONT}`; ctx.textAlign = 'center'; ctx.fillStyle = '#ffd966'; ctx.fillText(e.mini ? 'MINI-BOSS · ' + MN[e.mf] : e.cor ? 'CORROMPU' : e.ch ? 'CHAMPION' : 'Niv.' + tr + ' ÉLITE', e.x, e.y - e.r - 12); }
  }
  function drawDeco(sk, tilt) {
    const t = time, boost = p && inRun && (p.rapid > 0 || cm() >= 10), cb = inRun ? Math.min(.25, cm() / 80) : 0;
    if (sk.dk === 'demon') {
      ctx.save(); ctx.globalCompositeOperation = 'lighter'; const g = ctx.createRadialGradient(0, 0, 4, 0, 0, 44); g.addColorStop(0, `rgba(255,30,50,${.28 + .08 * Math.sin(t * .1)})`); g.addColorStop(1, 'rgba(0,0,0,0)'); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(0, 0, 44, 0, 7); ctx.fill(); ctx.restore();
      const cg = ctx.createRadialGradient(0, 2, 0, 0, 2, 7); cg.addColorStop(0, '#ff5a5a'); cg.addColorStop(1, '#5a0010'); ctx.fillStyle = cg; ctx.beginPath(); ctx.arc(0, 2, 5 + Math.sin(t * .2), 0, 7); ctx.fill();
      for (const d of [-1, 1]) {
        const hg = ctx.createLinearGradient(d * 4, -16, d * 7, -46); hg.addColorStop(0, '#2a2e3a'); hg.addColorStop(.7, '#8a90a4'); hg.addColorStop(1, boost ? '#ff6a6a' : '#c03040');
        ctx.fillStyle = hg; ctx.beginPath(); ctx.moveTo(d * 3, -16); ctx.bezierCurveTo(d * 17, -26, d * 17, -40, d * 7, -47); ctx.bezierCurveTo(d * 10, -37, d * 9, -27, d * 1, -19); ctx.fill();
        if (boost) { ctx.save(); ctx.globalCompositeOperation = 'lighter'; ctx.fillStyle = 'rgba(255,60,60,.6)'; ctx.beginPath(); ctx.arc(d * 7, -45, 6, 0, 7); ctx.fill(); ctx.restore(); }
      }
      const n = sv.o.part === 2 ? 2 : 4; for (let i = 0; i < n; i++) { const a = t * .05 + i * 6.283 / n; ctx.fillStyle = `rgba(255,${60 + i * 30},60,.8)`; ctx.fillRect(Math.cos(a) * 30 - 1, Math.sin(a) * 14 - 1, 2.5, 2.5); }
      if (shotFx > 0) { ctx.save(); ctx.globalCompositeOperation = 'lighter'; ctx.fillStyle = 'rgba(255,60,80,.6)'; ctx.beginPath(); ctx.arc(0, -26, 5 + shotFx, 0, 7); ctx.fill(); ctx.restore(); }
    } else {
      ctx.save(); ctx.globalCompositeOperation = 'lighter'; ctx.strokeStyle = `rgba(255,225,120,${Math.min(1, .75 + cb + .2 * Math.sin(t * .1))})`; ctx.lineWidth = 2.5; ctx.beginPath(); ctx.ellipse(0, -52, 12 + Math.sin(t * .03) * 1.5, 4, 0, 0, 7); ctx.stroke();
      const g = ctx.createRadialGradient(0, 0, 4, 0, 0, 46); g.addColorStop(0, 'rgba(255,230,150,.22)'); g.addColorStop(1, 'rgba(0,0,0,0)'); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(0, 0, 46, 0, 7); ctx.fill(); ctx.restore();
      ctx.save(); ctx.translate(-tilt * 6, -34 + Math.sin(t * .06) * 1.5); ctx.strokeStyle = '#ffd966'; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(0, 8, 20, 3.6, 5.82); ctx.stroke();
      const pull = shotFx > 0 ? 6 : 0; ctx.strokeStyle = 'rgba(255,255,255,.9)'; ctx.lineWidth = 1; ctx.beginPath(); ctx.moveTo(-17.9, -.8); ctx.lineTo(0, -1 + pull); ctx.lineTo(17.8, -1); ctx.stroke();
      if (shotFx > 0) { ctx.fillStyle = '#ffe9a0'; ctx.fillRect(-.8, -14 + pull, 1.6, 14); }
      ctx.restore();
      for (let i = 0; i < 3; i++) ell(Math.sin(t * .04 + i * 2) * 38, ((t * .5 + i * 30) % 50) - 10, 1.6, 3.2, 'rgba(255,255,255,.7)');
    }
  }
  function drawDiv(sk, L) {
    const t = time, id = sk.id; ctx.save(); ctx.globalCompositeOperation = 'lighter';
    const ray = (a, len, w, col, al, ox = 0, oy = 0) => { ctx.save(); ctx.translate(ox, oy); ctx.rotate(a); ctx.globalAlpha = al; ctx.fillStyle = col; ctx.beginPath(); ctx.moveTo(-w, 0); ctx.lineTo(0, -len); ctx.lineTo(w, 0); ctx.fill(); ctx.restore(); };
    if (id === 'celemp') { const n = 6 + L; for (let i = 0; i < n; i++) ray((i - (n - 1) / 2) * .26 + Math.sin(t * .05 + i) * .05, 38 + L * 4, 3, '#ffd966', .4); ctx.globalAlpha = .95; ctx.fillStyle = '#ffe9a0'; ctx.beginPath(); for (let i = 0; i < 5; i++) { const x = (i - 2) * 7; ctx.moveTo(x - 3.5, -34); ctx.lineTo(x, -34 - (i % 2 ? 8 : 14)); ctx.lineTo(x + 3.5, -34); } ctx.fill(); ctx.strokeStyle = '#fff'; ctx.lineWidth = 1.5; ctx.beginPath(); ctx.ellipse(0, -46, 18, 4.5, 0, 0, 7); ctx.stroke(); }
    else if (id === 'cosmicgod') { for (let a = 0; a < 3; a++) for (let k = 0; k < 14; k++) { const q = t * .03 + a * 2.094 + k * .36, r = 8 + k * 3.2; ctx.globalAlpha = 1 - k / 16; ctx.fillStyle = a % 2 ? '#19e3ff' : '#ff5ac8'; ctx.beginPath(); ctx.arc(Math.cos(q) * r, Math.sin(q) * r * .55, 1.8, 0, 7); ctx.fill(); } for (let i = 0; i < 2; i++) { const q = t * .04 + i * 3.14; ctx.globalAlpha = 1; ctx.fillStyle = i ? '#ffd966' : '#fff'; ctx.beginPath(); ctx.arc(Math.cos(q) * 40, Math.sin(q) * 18, 3.5, 0, 7); ctx.fill(); } }
    else if (id === 'voidlord') { ctx.strokeStyle = '#b57cff'; ctx.lineWidth = 1.4; ctx.globalAlpha = .6 + .3 * Math.sin(t * .15); for (let i = 0; i < 6; i++) { const a = i * 1.047 + .3; ctx.beginPath(); ctx.moveTo(0, 0); for (let k = 1; k < 5; k++) ctx.lineTo(Math.cos(a + (k % 2 ? .12 : -.1)) * k * 9, Math.sin(a + (k % 2 ? .12 : -.1)) * k * 9); ctx.stroke(); } ctx.globalCompositeOperation = 'source-over'; ctx.globalAlpha = 1; for (let i = 0; i < 4; i++) { const q = -t * .03 + i * 1.571, x = Math.cos(q) * 38, y = Math.sin(q) * 30; ctx.fillStyle = '#12041f'; ctx.strokeStyle = '#b57cff'; ctx.beginPath(); ctx.moveTo(x, y - 6); ctx.lineTo(x + 3, y); ctx.lineTo(x, y + 6); ctx.lineTo(x - 3, y); ctx.closePath(); ctx.fill(); ctx.stroke(); } }
    else if (id === 'solarphx') { const g = ctx.createRadialGradient(0, 0, 2, 0, 0, 30); g.addColorStop(0, 'rgba(255,230,120,.22)'); g.addColorStop(1, 'rgba(255,100,20,0)'); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(0, 0, 30, 0, 7); ctx.fill(); for (const d of [-1, 1]) for (let i = 0; i < 5; i++) ray(d * (.55 + i * .3) + Math.sin(t * .12 + i) * .08, 26 + (4 - i) * 5 + L * 2, 4.5, i % 2 ? '#ffd23f' : '#ff6a1a', .5, d * 8, 2); for (let i = 0; i < 3; i++) ray(3.14 + (i - 1) * .25, 20 + Math.sin(t * .3 + i) * 5, 3, '#ff9a3d', .8, 0, 18); }
    else if (id === 'dragoncel') { ctx.lineWidth = 2.5; for (const d of [-1, 1]) for (let i = 0; i < 3; i++) { ctx.strokeStyle = i % 2 ? '#ffd966' : '#19e3ff'; ctx.globalAlpha = .8; ctx.beginPath(); ctx.moveTo(d * 8, -4); ctx.quadraticCurveTo(d * (40 + i * 6), -26 + i * 6 + Math.sin(t * .1) * 4, d * (46 + i * 4), 14 + i * 5); ctx.stroke(); } ctx.fillStyle = '#ffd966'; ctx.globalAlpha = 1; for (const d of [-1, 1]) { ctx.beginPath(); ctx.moveTo(d * 3, -22); ctx.lineTo(d * 9, -34); ctx.lineTo(d * 6, -20); ctx.fill(); } ctx.fillStyle = '#19e3ff'; for (let k = 0; k < 8; k++) { ctx.globalAlpha = 1 - k / 9; ctx.beginPath(); ctx.arc(Math.sin(t * .15 - k * .6) * (4 + k), 24 + k * 4, 2.2, 0, 7); ctx.fill(); } }
    else if (id === 'spiritsam') { ctx.globalAlpha = .9; ctx.strokeStyle = '#fff'; ctx.lineWidth = 2; ctx.beginPath(); ctx.moveTo(14, 22); ctx.lineTo(14, -34 - L * 2); ctx.stroke(); ctx.strokeStyle = '#7fe3ff'; ctx.lineWidth = 6; ctx.globalAlpha = .3; ctx.beginPath(); ctx.moveTo(14, 22); ctx.lineTo(14, -34 - L * 2); ctx.stroke(); ctx.globalAlpha = .8; ctx.fillStyle = '#7fe3ff'; for (const d of [-1, 1]) { ctx.beginPath(); ctx.moveTo(d * 8, -6); ctx.lineTo(d * 24, -10); ctx.lineTo(d * 20, 2); ctx.lineTo(d * 8, 4); ctx.fill(); } ctx.strokeStyle = '#9be7ff'; ctx.lineWidth = 1.2; ctx.beginPath(); ctx.arc(0, 0, 36 + Math.sin(t * .1) * 2, 0, 7); ctx.stroke(); }
    ctx.restore();
  }
  function drawCos(c, L) {
    const t = time, R = 16 + L * 7;
    ctx.save(); ctx.globalCompositeOperation = 'lighter'; ctx.globalAlpha = .22 + L * .06;
    const h = ctx.createRadialGradient(0, 0, 4, 0, 0, R); h.addColorStop(0, c.h); h.addColorStop(1, 'rgba(0,0,0,0)'); ctx.fillStyle = h; ctx.beginPath(); ctx.arc(0, 0, R, 0, 7); ctx.fill(); ctx.restore();
    for (let i = 0; i < L; i++) { ctx.strokeStyle = i % 2 ? c.r2 : c.r1; ctx.lineWidth = 1.5; ctx.beginPath(); ctx.ellipse(0, 0, 12 + i * 4, 4 + i * 1.6, t * .04 * (i % 2 ? -1 : 1) + i, 0, 7); ctx.stroke(); }
    ctx.fillStyle = c.core; ctx.strokeStyle = c.r1; ctx.lineWidth = 1; ctx.beginPath(); ctx.arc(0, -2, 4 + L * 1.2, 0, 7); ctx.fill(); ctx.stroke();
    if (c.eye) {  // œil évolutif animé
      const r = 4 + L * 1.2, pu = 1 + .08 * Math.sin(t * .12);
      ctx.save(); ctx.translate(0, -2); ctx.scale(pu, pu);
      if (c.eye === 'u') { ctx.fillStyle = L > 1 ? '#c0182b' : '#222'; ctx.beginPath(); ctx.arc(0, 0, r, 0, 7); ctx.fill(); ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(0, 0, r * .3, 0, 7); ctx.fill();
        for (let i = 0; i < Math.min(3, L - 1); i++) { const a = t * .03 + i * 2.094; ctx.beginPath(); ctx.arc(Math.cos(a) * r * .62, Math.sin(a) * r * .62, r * .2, 0, 7); ctx.fill(); }
        if (L >= 5) { ctx.strokeStyle = '#b57cff'; ctx.lineWidth = 1.2; ctx.beginPath(); ctx.moveTo(0, -r); ctx.lineTo(r * .25, 0); ctx.lineTo(0, r); ctx.lineTo(-r * .25, 0); ctx.closePath(); ctx.stroke(); } }
      else if (c.eye === 'h') { ctx.strokeStyle = '#7fc8ff'; ctx.lineWidth = 1; for (let i = 1; i <= L; i++) { ctx.beginPath(); ctx.arc(0, 0, r * i / L, 0, 7); ctx.stroke(); } ctx.fillStyle = '#cfe9ff'; ctx.beginPath(); ctx.arc(0, 0, r * .25, 0, 7); ctx.fill(); }
      else { ctx.fillStyle = '#5aff8a'; for (let i = 0; i < L + 1; i++) { const a = t * .02 + i * 6.283 / (L + 1); ctx.save(); ctx.rotate(a); ctx.beginPath(); ctx.ellipse(r + 3, 0, 4, 1.8, 0, 0, 7); ctx.fill(); ctx.restore(); } }
      ctx.restore();
    }
    if (sv.o.dist) { const n = Math.ceil((2 + L * 2) / (1 + sv.o.q)); ctx.fillStyle = c.r1; for (let i = 0; i < n; i++) { const ph = (t * .02 + i / n) % 1, r = 36 * (1 - ph), a = i * 2.4 + ph * 6; ctx.globalAlpha = ph; ctx.fillRect(Math.cos(a) * r - 1, Math.sin(a) * r * .6 - 1, 2, 2); } ctx.globalAlpha = 1; }
    if (L >= 6) { ctx.strokeStyle = c.r1; ctx.lineWidth = 1.5; ctx.beginPath(); ctx.arc(0, 0, 42 + Math.sin(t * .13) * 2, t * .05, t * .05 + 4.4); ctx.stroke(); }
    if (L >= 7) { ctx.fillStyle = c.r1; for (let i = 0; i < 4; i++) { const a = t * .06 + i * 1.571; ctx.beginPath(); ctx.arc(Math.cos(a) * 48, Math.sin(a) * 48, 2.5, 0, 7); ctx.fill(); } }
    if (L >= 5) { ctx.strokeStyle = c.r2; ctx.lineWidth = 2; ctx.beginPath(); ctx.arc(0, 0, 32 + Math.sin(t * .1) * 2, 0, 7); ctx.stroke(); }
  }
  function drawBullet(b) {
    const z = b.sz || 1, st = b.st;
    if (st === 'flame') { ctx.fillStyle = 'rgba(255,120,40,.35)'; ctx.beginPath(); ctx.ellipse(b.x, b.y + 6, 5 * z, 12 * z, 0, 0, 7); ctx.fill(); ctx.fillStyle = '#ffd966'; ctx.beginPath(); ctx.moveTo(b.x, b.y - 10 * z); ctx.lineTo(b.x + 4 * z, b.y + 4); ctx.lineTo(b.x - 4 * z, b.y + 4); ctx.fill(); }
    else if (st === 'star') { ctx.fillStyle = 'rgba(160,190,255,.3)'; ctx.fillRect(b.x - 2, b.y, 4, 16); ctx.fillStyle = '#fff'; ctx.beginPath(); for (let i = 0; i < 8; i++) { const a = i * .785 + time * .1, r = (i % 2 ? 2.5 : 6) * z; ctx.lineTo(b.x + Math.cos(a) * r, b.y + Math.sin(a) * r); } ctx.fill(); }
    else if (st === 'void') { ctx.fillStyle = '#14091f'; ctx.strokeStyle = '#b57cff'; ctx.lineWidth = 2; ctx.beginPath(); ctx.ellipse(b.x, b.y, 3.5 * z, 8 * z, 0, 0, 7); ctx.fill(); ctx.stroke(); }
    else if (st === 'gold') { ctx.fillStyle = 'rgba(255,215,90,.3)'; ctx.beginPath(); ctx.arc(b.x, b.y, 8 * z, 0, 7); ctx.fill(); ctx.fillStyle = '#fff1b8'; ctx.beginPath(); ctx.arc(b.x, b.y, 3.5 * z, 0, 7); ctx.fill(); }
    else if (st === 'geo') { ctx.fillStyle = 'rgba(155,231,255,.25)'; ctx.fillRect(b.x - 1.5, b.y, 3, 18); ctx.fillStyle = '#9be7ff'; ctx.save(); ctx.translate(b.x, b.y); ctx.rotate(.785); ctx.fillRect(-3 * z, -3 * z, 6 * z, 6 * z); ctx.restore(); }
    else if (st === 'cosmic') { const c1 = b.hc || '#8a3cff', R = 7 * z, g = ctx.createRadialGradient(b.x, b.y, 1, b.x, b.y, R * 1.8); g.addColorStop(0, '#000'); g.addColorStop(.5, c1); g.addColorStop(1, 'rgba(0,0,0,0)'); ctx.fillStyle = g; ctx.beginPath(); ctx.arc(b.x, b.y, R * 1.8, 0, 7); ctx.fill(); ctx.fillStyle = '#05010a'; ctx.beginPath(); ctx.arc(b.x, b.y, R * .55, 0, 7); ctx.fill(); ctx.globalAlpha = .3; ctx.fillStyle = c1; ctx.fillRect(b.x - 1.5, b.y, 3, 22); ctx.globalAlpha = 1; }
    else if (st === 'demon') { ctx.fillStyle = 'rgba(255,30,50,.35)'; ctx.beginPath(); ctx.arc(b.x, b.y, 8 * z, 0, 7); ctx.fill(); ctx.fillStyle = '#3a0010'; ctx.strokeStyle = '#ff4a5a'; ctx.lineWidth = 1.5; ctx.beginPath(); ctx.ellipse(b.x, b.y, 3.5 * z, 7 * z, 0, 0, 7); ctx.fill(); ctx.stroke(); }
    else if (st === 'arrow') { ctx.fillStyle = 'rgba(255,225,120,.3)'; ctx.fillRect(b.x - 3, b.y - 6, 6, 24); ctx.fillStyle = '#ffe9a0'; ctx.beginPath(); ctx.moveTo(b.x, b.y - 11 * z); ctx.lineTo(b.x + 3.5 * z, b.y - 3); ctx.lineTo(b.x + 1, b.y - 3); ctx.lineTo(b.x + 1, b.y + 9); ctx.lineTo(b.x - 1, b.y + 9); ctx.lineTo(b.x - 1, b.y - 3); ctx.lineTo(b.x - 3.5 * z, b.y - 3); ctx.fill(); }
    else if (st === 'elec') { ctx.fillStyle = 'rgba(255,225,60,.35)'; ctx.beginPath(); ctx.arc(b.x, b.y, 7 * z, 0, 7); ctx.fill(); ctx.fillStyle = '#ffe14a'; ctx.beginPath(); ctx.arc(b.x, b.y, 3.5 * z, 0, 7); ctx.fill(); ctx.strokeStyle = '#fff'; ctx.lineWidth = 1.5; ctx.beginPath(); ctx.moveTo(b.x - 2, b.y + 4); ctx.lineTo(b.x + 2, b.y + 9); ctx.lineTo(b.x - 2, b.y + 13); ctx.stroke(); }
    else if (st === 'slash') { ctx.strokeStyle = 'rgba(255,90,74,.5)'; ctx.lineWidth = 6 * z; ctx.beginPath(); ctx.arc(b.x, b.y + 10, 9 * z, 3.6, 5.8); ctx.stroke(); ctx.strokeStyle = '#fff'; ctx.lineWidth = 2.2 * z; ctx.beginPath(); ctx.arc(b.x, b.y + 10, 9 * z, 3.6, 5.8); ctx.stroke(); }
    else if (st === 'ghost') { ctx.fillStyle = 'rgba(200,220,255,.5)'; ctx.fillRect(b.x - 1.5 * z, b.y - 8, 3 * z, 18); }
    else if (st === 'rb') { ctx.fillStyle = `hsl(${(time * 6 + b.y) % 360},90%,65%)`; ctx.beginPath(); ctx.arc(b.x, b.y, 4 * z, 0, 7); ctx.fill(); }
    else { ctx.fillStyle = 'rgba(255,220,90,.28)'; ctx.fillRect(b.x - 4 * z, b.y - 6, 8 * z, 26); ctx.fillStyle = '#fff6a8'; ctx.fillRect(b.x - 1.8 * z, b.y - 8, 3.6 * z, 13); ctx.fillStyle = '#fff'; ctx.fillRect(b.x - .8 * z, b.y - 7, 1.6 * z, 9); }
  }
  const OR = i => ({ x: 30, y: 100 + i * 50, w: 300, h: 42 });
  const QN = ['ÉLEVÉE', 'NORMALE', 'ÉCONOMIE'], PN = ['ÉLEVÉES', 'NORMALES', 'RÉDUITES'], oo = v => v ? 'ON' : 'OFF';
  const optLabels = () => { const o = sv.o; return ['Musique  ' + Math.round(AM.st.m * 100) + '%', 'Sons  ' + Math.round(AM.st.s * 100) + '%', 'Qualité : ' + QN[o.q], 'Vibration : ' + oo(o.vib), 'Distorsion : ' + oo(o.dist), 'Caméra (secousses) : ' + oo(o.cam), 'Particules skins : ' + PN[o.part], 'Réinitialiser la sauvegarde']; };
  function optTap(q) {
    if (inRect(q, backBtn())) { state = backTo; return; }
    for (let i = 0; i < 8; i++) if (inRect(q, OR(i))) {
      const o = sv.o;
      if (i < 2) { const k = i ? 's' : 'm', d = q.x < 120 ? -.1 : q.x > 240 ? .1 : 0; AM.st[k] = Math.min(1, Math.max(0, +(AM.st[k] + d).toFixed(1))); AM.apply(); }
      else if (i === 2) o.q = (o.q + 1) % 3; else if (i === 3) o.vib = +!o.vib; else if (i === 4) o.dist = +!o.dist; else if (i === 5) o.cam = +!o.cam; else if (i === 6) o.part = (o.part + 1) % 3;
      else { state = 'conf'; return; }
      save(); AM.sfx('equip');
    }
  }
  function drawOpt() {
    ctx.textBaseline = 'alphabetic'; ctx.textAlign = 'center'; ctx.fillStyle = '#12293f'; ctx.font = `bold 30px ${FONT}`; ctx.fillText('Options', W / 2, 66);
    optLabels().forEach((l, i) => { pill(OR(i), l, i === 7 ? '#9a3b3b' : null); if (i < 2) { ctx.fillStyle = '#fff'; ctx.font = `bold 22px ${FONT}`; ctx.textBaseline = 'middle'; ctx.fillText('−', 52, OR(i).y + 21); ctx.fillText('+', 308, OR(i).y + 21); ctx.textBaseline = 'alphabetic'; } });
    pill(backBtn(), 'Retour');
  }
  function killLv(e, sk) {
    const L = p.sl, c1 = sk.cos ? sk.cos.r1 : (sk.flame || sk.glow), c2 = sk.cos ? sk.cos.r2 : sk.wing, n = (L * 2) >> sv.o.part;
    for (let i = 0; i < n && parts.length < PCAP(); i++) { const a = Math.random() * 6.283, v = rnd(1.5, 3.5); parts.push({ x: e.x, y: e.y, vx: Math.cos(a) * v, vy: Math.sin(a) * v, life: 26, max: 26, r: rnd(1.5, 3), c: i % 2 ? c1 : c2 }); }
    if (L >= 3 && rings.length < 24) rings.push({ x: e.x, y: e.y, r: 4, max: 18 + L * 9, c: c1 });
    if (L >= 5 && rings.length < 24) rings.push({ x: e.x, y: e.y, r: 2, max: 30 + L * 8, c: c2 });
    if (L >= 7) for (let i = 0; i < 6 && parts.length < PCAP(); i++) { const a = i * 1.047; parts.push({ x: e.x, y: e.y, vx: Math.cos(a) * 5, vy: Math.sin(a) * 5, life: 18, max: 18, r: 2.5, c: '#fff' }); }
  }
  const themeNow = () => {
    if (!(state === 'play' || state === 'pick' || state === 'pause')) return 'menu';
    const b = enemies.find(e => e.k === 3); if (b) return (b.php || 1) >= 4 ? 'final' : 'boss';
    return enemies.some(e => e.mini) ? 'mini' : wave >= 11 ? 'w3' : wave >= 6 ? 'w2' : 'w1';
  };
  function panel(x, y, w, h) { ctx.fillStyle = 'rgba(10,30,50,.5)'; ctx.beginPath(); ctx.roundRect(x, y, w, h, 12); ctx.fill(); }
  function hud() {
    ctx.textBaseline = 'alphabetic';
    panel(8, 8, 112, 50); panel(W / 2 - 52, 8, 104, 36); panel(W - 88, 8, 80, 28);
    ctx.textAlign = 'left'; ctx.fillStyle = '#fff'; ctx.font = `bold 20px ${FONT}`; ctx.fillText(score, 18, 31);
    ctx.font = `bold 13px ${FONT}`; ctx.fillStyle = '#ff6b81'; ctx.fillText('♥ ' + p.lives, 18, 48);
    if (p.shield > 0) { ctx.fillStyle = '#7fe3ff'; ctx.fillText('◈ ' + Math.ceil(p.shield / 60) + 's', 64, 48); }
    ctx.fillStyle = 'rgba(255,255,255,.25)'; ctx.fillRect(18, 53, 92, 3); ctx.fillStyle = '#b57cff'; ctx.fillRect(18, 53, 92 * Math.min(1, p.xp / p.xpN), 3);
    wTot = Math.max(wTot, wq.length + enemies.length);
    const pr = Math.max(0, Math.min(1, 1 - (wq.length + enemies.length) / wTot));
    ctx.textAlign = 'center'; ctx.fillStyle = '#fff'; ctx.font = `bold 12px ${FONT}`; ctx.fillText('VAGUE ' + Math.max(1, wave), W / 2, 24); ctx.fillStyle = DC[DNG(wave)]; ctx.beginPath(); ctx.arc(W / 2 + 46, 20, 4, 0, 7); ctx.fill();
    ctx.fillStyle = 'rgba(255,255,255,.25)'; ctx.fillRect(W / 2 - 40, 31, 80, 5); ctx.fillStyle = '#ffd966'; ctx.fillRect(W / 2 - 40, 31, 80 * pr, 5);
    ctx.textAlign = 'right'; ctx.fillStyle = '#ffd966'; ctx.font = `bold 15px ${FONT}`; ctx.fillText('◎ ' + sv.coins, W - 16, 28);
    { const sk = skin(); if (inRun) { const L = p.sl, a = SKLW[L - 1], b2 = SKLW[L]; ctx.textAlign = 'left'; ctx.font = `bold 11px ${FONT}`; ctx.fillStyle = '#fff'; ctx.fillText(sk.n.toUpperCase() + ' NIV ' + L + '/7' + (b2 ? ' · ' + Math.min(99, Math.round(lvProg(L) * 100)) + '%' : ' · MAX'), 12, 72); ctx.textAlign = 'right'; } }
    if (combo > 1) { ctx.font = `bold 12px ${FONT}`; ctx.fillStyle = '#fff'; ctx.fillText('COMBO x' + cm(), W - 16, 52); } ctx.textAlign = 'left';
  }
  let pickList = [];
  const RC = ['#9aa7b5', '#3fa0e8', '#a055e0', '#ffb020', '#ff3d7f', '#19e3ff'], RN = ['COMMUN', 'RARE', 'ÉPIQUE', 'LÉGENDAIRE', 'MYTHIQUE', 'EXOTIQUE'];
  const UPS = [
    { r: 0, n: 'Dégâts +20%', d: 'Tes tirs frappent plus fort.', f: () => { p.dmg *= 1.2; } },
    { r: 0, n: 'Cadence +15%', d: 'Tu tires plus vite.', f: () => { p.rate *= 1.15; } },
    { r: 0, n: 'Vitesse +10%', d: 'Ton avion réagit plus vite.', f: () => { p.spd *= 1.1; } },
    { r: 0, n: 'Gros tirs', d: 'Projectiles 20% plus larges.', f: () => { p.bsz *= 1.2; } },
    { r: 0, n: 'Aimant', d: 'Les bonus viennent vers toi.', ok: () => !p.mag, f: () => { p.mag = 1; } },
    { r: 1, n: '+1 projectile', d: 'Un tir de plus sur le côté.', ok: () => p.multi < 4, f: () => { p.multi++; } },
    { r: 1, n: 'Perforation', d: 'Les tirs traversent 1 ennemi de plus.', f: () => { p.pierce++; } },
    { r: 1, n: 'Critique +10%', d: '10% de chances de dégâts x2.', f: () => { p.crit += .1; } },
    { r: 1, n: '+1 vie', d: 'Une vie en plus.', f: () => { p.lives = Math.min(6, p.lives + 1); } },
    { r: 1, n: 'Bouclier', d: 'Bouclier de 10 secondes.', f: () => { p.shield = 600; } },
    { r: 2, n: 'Drone', d: 'Deux escorteurs tirent avec toi.', ok: () => !p.drone, f: () => { p.drone = 1; } },
    { r: 2, n: 'Régénération', d: '+1 vie toutes les 25 secondes.', ok: () => !p.regen, f: () => { p.regen = 1; } },
    { r: 2, n: 'Surcharge', d: 'Dégâts +40% et cadence +20%.', f: () => { p.dmg *= 1.4; p.rate *= 1.2; } },
    { r: 3, n: 'Tempête', d: 'Des éclairs frappent les ennemis.', ok: () => !p.storm, f: () => { p.storm = 1; } },
    { r: 3, n: 'Phénix', d: 'Reviens une fois si tu meurs.', ok: () => !p.phx, f: () => { p.phx = 1; } },
    { r: 3, n: 'Armement total', d: '+2 projectiles et tir triple.', ok: () => p.multi < 3, f: () => { p.multi = Math.min(4, p.multi + 2); p.power = Math.max(p.power, 3); } }
  ];
  UPS.push(
    { r: 4, n: "Tempête d'étoiles", d: 'Toutes les 8 destructions : 5 étoiles.', max: 1, f: () => { p.stars = 1; } },
    { r: 4, n: 'Armure du Vide', d: 'Absorbe 1 coup toutes les 12 s.', max: 1, f: () => { p.varm = 1; } },
    { r: 4, n: 'Nova solaire', d: 'Onde de choc toutes les 4 s.', max: 1, f: () => { p.nova = 1; } },
    { r: 5, n: 'Fracture', d: "Les tirs se divisent à l'impact.", max: 1, f: () => { p.frac = 1; } },
    { r: 5, n: 'Double destin', d: 'Une copie fantôme tire avec toi.', max: 1, f: () => { p.clone = 1; } }
  );
  UPS.push({ r: 1, n: 'Trèfle', d: 'Chance +25 % (loot, élites, bonus).', max: 3, f: () => { p.luck += .25; sv.st.luck = Math.max(sv.st.luck || 1, p.luck); } });
  UPS.forEach(u => { u.max = u.max || (u.ok ? 1 : u.r === 0 ? 3 : 2); });

  // ===================== ULTIMATE UPDATE : bonus de partie supplémentaires =====================
  RC.push('#fff1b8', '#ff2bd6'); RN.push('DIVIN', 'EXTRÊME ULTIME');
  UPS.push(
    { r: 0, n: 'Poche pleine', d: 'Pièces +10 % (cette partie).', max: 3, f: () => { p.cg *= 1.1; } },
    { r: 0, n: 'Apprenti', d: 'XP +10 % (cette partie).', max: 3, f: () => { p.xg *= 1.1; } },
    { r: 1, n: 'Marchand', d: 'Pièces +25 %.', max: 2, f: () => { p.cg *= 1.25; } },
    { r: 1, n: 'Érudit', d: 'XP +25 %.', max: 2, f: () => { p.xg *= 1.25; } },
    { r: 1, n: 'Réacteur de pouvoir', d: 'Recharge des pouvoirs -15 %.', max: 3, f: () => { p.cdr *= .85; } },
    { r: 1, n: 'Combo généreux', d: 'Récompense de combo +20 %.', max: 3, f: () => { p.comboB *= 1.2; } },
    { r: 2, n: 'Chasseur de coffres', d: '+3 % de chance de coffre.', max: 3, f: () => { p.chestC += .03; } },
    { r: 2, n: 'Esquive affûtée', d: 'Esquives parfaites +50 % de gains.', max: 3, f: () => { p.dodgeB += .5; } },
    { r: 2, n: 'Pouvoirs amplifiés', d: 'Dégâts des pouvoirs +25 %.', max: 3, f: () => { p.pwUp *= 1.25; } },
    { r: 2, n: "Prime d'élite", d: 'Pièces des élites +40 %.', max: 2, f: () => { p.eliteB *= 1.4; } },
    { r: 3, n: 'Fortune', d: 'Pièces +50 %.', max: 2, f: () => { p.cg *= 1.5; } },
    { r: 3, n: 'Prodige', d: 'XP +60 %.', max: 2, f: () => { p.xg *= 1.6; } },
    { r: 3, n: 'Pouvoir jumeau', d: 'Un second pouvoir automatique.', max: 1, ok: () => !p.pw2, f: () => { p.pw2 = 1; } },
    { r: 3, n: 'Second souffle', d: 'Survis une fois avec 1 PV.', max: 1, ok: () => !p.sec, f: () => { p.sec = 1; } },
    { r: 3, n: 'Phase fantôme', d: 'Traverse les projectiles 5 s.', max: 2, f: () => { p.ghost = 300; } },
    { r: 6, n: '✨ Aura divine', d: '+2 vies, régénération, bouclier 20 s.', max: 1, f: () => { p.lives = Math.min(6, p.lives + 2); p.regen = 1; p.shield = 1200; } },
    { r: 6, n: '✨ Pluie d\'or divine', d: 'Pièces ×2 pour toute la partie.', max: 1, f: () => { p.cg *= 2; } },
    { r: 7, n: '🌌 ASCENSION ULTIME', d: 'Pièces ×2, XP ×2, 6 vies, second souffle, ultime prêt.', max: 1, f: () => { p.cg *= 2; p.xg *= 2; p.lives = 6; p.sec = 1; p.ghost = 300; ult = 100; rings.push({ x: p.x, y: p.y, r: 8, max: 260, c: '#ff2bd6' }); shake = 14; } }
  );
  UPS.forEach(u => { u.max = u.max || (u.ok ? 1 : u.r === 0 ? 3 : 2); });
  function openPick() {
    const left = u => (p.got[u.n] || 0) < (u.max || 1) && (!u.ok || u.ok());
    const w = Math.min(.03, wave * .001), picks = [];
    for (let t = 0; picks.length < 3 && t < 60; t++) {
      const r = Math.random(); let rar = r < .6 ? 0 : r < .85 ? 1 : r < .95 ? 2 : r < .99 - w ? 3 : r < .999 - w / 3 ? 4 : 5;
      if (rar >= 3 && rar < 6 && Math.random() < .07 * p.luck) rar = 6; if (rar === 6 && Math.random() < .15) rar = 7;
      const pool = UPS.filter(u => u.r === rar && left(u) && !picks.includes(u));
      if (pool.length) picks.push(pool[(Math.random() * pool.length) | 0]);
    }
    const rest = UPS.filter(u => left(u) && !picks.includes(u));
    while (picks.length < 3 && rest.length) picks.push(rest.splice((Math.random() * rest.length) | 0, 1)[0]);
    while (picks.length < 3) picks.push({ r: 1, n: 'Prime de ◎ ' + picks.length, d: '+150 ◎ immédiats.', f: () => earn(150) });
    pickList = picks; target = null; state = 'pick'; AM.sfx('up');
  }
  function choose(u) {
    u.f(); p.got[u.n] = (p.got[u.n] || 0) + 1; p.xl++; state = 'play'; target = null; p.inv = Math.max(p.inv, 90);
    if (u.r === 4) p.myth++; if (u.r === 5) p.exo++;
    rings.push({ x: p.x, y: p.y, r: 8, max: u.r >= 4 ? 170 : 130, c: u.r === 5 ? '#19e3ff' : u.r === 4 ? '#ff3d7f' : '#9be7ff' });
    boom(p.x, p.y, u.r >= 4 ? 36 : 22, [RC[u.r], '#fff']); AM.sfx(u.r >= 4 ? 'rare' : 'level');
    msg = u.r >= 4 ? { a: u.r === 7 ? '🌌 BONUS EXTRÊME ULTIME !' : u.r === 6 ? '✨ BONUS DIVIN !' : u.r === 5 ? 'BONUS EXOTIQUE !' : 'BONUS MYTHIQUE !', b: u.n } : { a: 'ÉVOLUTION ' + p.xl, b: EVN[p.xl - 1] || 'Forme ' + p.xl }; msgT = 150;
  }
  const pickCards = () => { const h = Math.min(100, (H - 230) / 3 - 14); return pickList.map((u, i) => ({ u, x: 24, y: 140 + i * (h + 14), w: W - 48, h })); };
  function drawPick() {
    hud(); ctx.fillStyle = 'rgba(18,41,63,.65)'; ctx.fillRect(0, 0, W, H);
    ctx.textAlign = 'center'; ctx.textBaseline = 'alphabetic'; ctx.fillStyle = '#ffd966'; ctx.font = `bold 28px ${FONT}`; ctx.fillText('AMÉLIORATION !', W / 2, 100);
    ctx.fillStyle = '#fff'; ctx.font = `14px ${FONT}`; ctx.fillText('Choisis une amélioration', W / 2, 124);
    for (const c of pickCards()) {
      const u = c.u, col = RC[u.r];
      ctx.fillStyle = 'rgba(255,255,255,.95)'; ctx.beginPath(); ctx.roundRect(c.x, c.y, c.w, c.h, 14); ctx.fill();
      ctx.strokeStyle = col; ctx.lineWidth = 3; ctx.stroke();
      ctx.textAlign = 'left'; ctx.fillStyle = '#12293f'; ctx.font = `bold 19px ${FONT}`; ctx.fillText(u.n + (u.max > 1 ? '  ' + ((p.got[u.n] || 0) + 1) + '/' + u.max : ''), c.x + 16, c.y + c.h * .42);
      ctx.font = `13px ${FONT}`; ctx.fillText(u.d, c.x + 16, c.y + c.h * .7);
      ctx.fillStyle = col; ctx.font = `bold 11px ${FONT}`; ctx.textAlign = 'right'; ctx.fillText(RN[u.r], c.x + c.w - 14, c.y + 20);
    }
  }
  const SHB = { prev: { x: 14, y: 54, w: 52, h: 34 }, next: { x: W - 66, y: 54, w: 52, h: 34 }, act: { x: W / 2 - 100, y: 262, w: 200, h: 40 } };
  const gridCards = () => SKINS.filter(k => k.r === shopR).map((k, i) => ({ k, x: 20 + (i % 3) * 107, y: 316 + shopY + ((i / 3) | 0) * 100, w: 101, h: 92 }));
  const skiInfo = k => {
    const SN = { '': 'Classique', inter: 'Interceptor fin', falcon: 'Faucon pointu', stealth: 'Furtif triangulaire', ranger: 'Ranger larges ailes', mecha: 'Mécanique', dragon: 'Dragon', titan: 'Titan imposant', arrow: 'Flèche', demon: 'Cornes démoniaques', angel: "Ailes d'ange" };
    const PN2 = { fire: 'flammes', star: 'étoiles', spark: 'étincelles', void: 'vide', ghost: 'fantôme', gold: 'poussière dorée', demon: 'braises', angel: 'plumes dorées', rb: 'arc-en-ciel' };
    const BN = { flame: 'flammes', star: 'étoiles', void: 'ovales sombres', gold: 'orbes dorés', geo: 'cristaux', ghost: 'traits fantômes', rb: 'arc-en-ciel', cosmic: 'rayons cosmiques', demon: 'énergie démoniaque', arrow: 'flèches célestes' };
    const L = ['Forme : ' + (SN[k.shape || ''] || 'Classique'), 'Particules : ' + (PN2[k.fx] || 'aucune'), 'Tirs : ' + (BN[k.bul] || 'classiques'), 'Élimination : ' + (k.r >= 1 && (k.dk || k.fx) ? 'effet spécial' : 'standard')];
    if (k.cos) L[3] += ' · 7 niveaux d\'évolution'; L.push('Ultime : ' + ultOf(k).n); return L;
  };
  function shopTap(q) {
    if (inRect(q, SHB.prev)) { shopR = (shopR + SKN.length - 1) % SKN.length; shopY = 0; return; }
    if (inRect(q, SHB.next)) { shopR = (shopR + 1) % SKN.length; shopY = 0; return; }
    const k = SKINS.find(s2 => s2.id === shopSel) || skin();
    if (inRect(q, SHB.act)) {
      if (sv.owned.includes(k.id)) { if (sv.eq === k.id) { sv.eq = 'classic'; shopMsg = 'Skin retiré'; AM.sfx('equip'); } else { sv.eq = k.id; shopMsg = 'Skin équipé !'; AM.sfx('equip'); if (k.dk) { equipFx = 60; equipSk = k; } } }
      else if (k.fp ? (sv.frag || 0) >= k.fp : sv.coins >= k.price) { if (k.fp) sv.frag -= k.fp; else sv.coins -= k.price; sv.owned.push(k.id); sv.eq = k.id; shopMsg = 'Skin obtenu !'; AM.sfx('buy'); if (k.dk) { equipFx = 60; equipSk = k; } checkAll(); }
      else shopMsg = k.fp ? 'Pas assez de ◆' : 'Pas assez de ◎';
      shopT = 90; save(); return;
    }
    if (q.y > 310) for (const c of gridCards()) if (inRect(q, c)) { shopSel = c.k.id; AM.sfx('pop'); }
  }
  function drawShop() {
    const k = SKINS.find(s2 => s2.id === shopSel) || skin(), own = sv.owned.includes(k.id), eq = sv.eq === k.id, can = k.fp ? (sv.frag || 0) >= k.fp : sv.coins >= k.price, L = 1 + ((time / 100) | 0) % 7;
    ctx.textBaseline = 'alphabetic'; ctx.textAlign = 'center'; ctx.fillStyle = '#12293f'; ctx.font = `bold 24px ${FONT}`; ctx.fillText('Boutique', W / 2, 36);
    pill(backBtn(), 'Retour'); pill(SHB.prev, '◀'); pill(SHB.next, '▶');
    ctx.fillStyle = SKR[shopR]; ctx.font = `bold 15px ${FONT}`; ctx.fillText(SKN[shopR] + '  ·  page ' + (shopR + 1) + ' / 9', W / 2, 76);
    ctx.fillStyle = '#12293f'; ctx.font = `bold 12px ${FONT}`; ctx.fillText('◎ ' + sv.coins + '     ◆ ' + (sv.frag || 0), W / 2, 98);
    ctx.fillStyle = 'rgba(255,255,255,.93)'; ctx.beginPath(); ctx.roundRect(14, 106, W - 28, 148, 16); ctx.fill(); ctx.strokeStyle = SKR[k.r]; ctx.lineWidth = 2; ctx.stroke();
    ship(78, 184, 2, k, 4, Math.sin(time * .03) * .3, 4, L);
    ctx.textAlign = 'left'; ctx.fillStyle = '#12293f'; ctx.font = `bold 19px ${FONT}`; ctx.fillText(k.n.toUpperCase(), 150, 132);
    ctx.fillStyle = SKR[k.r]; ctx.font = `bold 12px ${FONT}`; ctx.fillText(SKN[k.r], 150, 148);
    ctx.fillStyle = '#3a4a5c'; ctx.font = `12px ${FONT}`; skiInfo(k).forEach((l, i) => ctx.fillText(l, 150, 168 + i * 16));
    ctx.fillStyle = eq ? '#2f6fc4' : own ? '#1d8a4a' : '#6a7a8c'; ctx.font = `bold 12px ${FONT}`; ctx.fillText(eq ? 'ÉQUIPÉ' : own ? 'POSSÉDÉ' : 'NON POSSÉDÉ', 150, 244);
    pill(SHB.act, eq ? (k.id === 'classic' ? 'ÉQUIPÉ ✓' : 'DÉSÉQUIPER') : own ? 'ÉQUIPER' : (k.fp ? 'ACHETER  ◆ ' + k.fp : 'ACHETER  ◎ ' + k.price), eq ? 'rgba(18,41,63,.5)' : own ? '#2f6fc4' : can ? '#1d8a4a' : '#9a3b3b');
    shopMax = Math.max(0, Math.ceil(SKINS.filter(s2 => s2.r === shopR).length / 3) * 100 - (H - 316) + 10); shopY = Math.max(-shopMax, Math.min(0, shopY));
    ctx.save(); ctx.beginPath(); ctx.rect(0, 310, W, H - 310); ctx.clip();
    gridCards().forEach(c => {
      const o2 = sv.owned.includes(c.k.id); ctx.fillStyle = o2 ? 'rgba(255,255,255,.92)' : 'rgba(255,255,255,.6)'; ctx.beginPath(); ctx.roundRect(c.x, c.y, c.w, c.h, 12); ctx.fill();
      ctx.strokeStyle = SKR[c.k.r]; ctx.lineWidth = c.k.id === k.id ? 3.5 : 1.2; ctx.stroke(); ship(c.x + c.w / 2, c.y + 38, .85, c.k, 3, 0, 1, L);
      ctx.textAlign = 'center'; ctx.fillStyle = '#12293f'; ctx.font = `bold 11px ${FONT}`; ctx.fillText(c.k.n, c.x + c.w / 2, c.y + 78);
      ctx.font = `10px ${FONT}`; ctx.fillStyle = o2 ? '#1d8a4a' : '#6a7a8c'; ctx.fillText(o2 ? (sv.eq === c.k.id ? 'équipé' : 'possédé') : c.k.fp ? '◆ ' + c.k.fp : '◎ ' + c.k.price, c.x + c.w / 2, c.y + 89);
    });
    ctx.restore();
    if (shopMax > 0) { const th = Math.max(24, (H - 316) * (H - 316) / (H - 316 + shopMax)); ctx.fillStyle = 'rgba(18,41,63,.4)'; ctx.fillRect(W - 5, 316 + (-shopY / shopMax) * (H - 316 - th), 3, th); }
    if (equipFx > 0) { const t = 1 - equipFx / 60, dm = equipSk.dk === 'demon'; ctx.fillStyle = `rgba(0,0,0,${.55 * Math.sin(t * 3.14)})`; ctx.fillRect(0, 0, W, H); ctx.globalAlpha = Math.min(1, t * 2.2); ship(W / 2, H / 2, 2.2 + t * .6, equipSk, 3, 0, 5, 3); ctx.globalAlpha = 1; ctx.strokeStyle = dm ? '#ff2a3a' : '#ffd966'; ctx.lineWidth = 4; ctx.beginPath(); ctx.arc(W / 2, H / 2, t * 200, 0, 7); ctx.stroke(); }
    if (shopT > 0) { ctx.globalAlpha = Math.min(1, shopT / 20); ctx.textAlign = 'center'; ctx.fillStyle = '#12293f'; ctx.font = `bold 18px ${FONT}`; ctx.fillText(shopMsg, W / 2, H - 14); ctx.globalAlpha = 1; }
  }
  function draw() {
    ctx.setTransform(S, 0, 0, S, 0, 0);
    const g = ctx.createLinearGradient(0, 0, 0, H);
    const sc2 = skyCols(Math.max(1, wave)); g.addColorStop(0, sc2[0]); g.addColorStop(1, sc2[1]);
    ctx.fillStyle = g; ctx.fillRect(0, 0, W, H);
    if (wave >= 18) { const sa = Math.min(1, (wave - 18) / 6); ctx.fillStyle = '#fff'; for (const st of STARS) { ctx.globalAlpha = sa * (.5 + .5 * Math.sin(time * .05 + st.x)); ctx.fillRect(st.x, st.y % H, st.s, st.s); } ctx.globalAlpha = 1; }
    if (shake > 0 && sv.o.cam) ctx.translate(rnd(-3, 3), rnd(-3, 3));
    ctx.fillStyle = 'rgba(255,255,255,.75)';
    for (const c of clouds) {
      ctx.globalAlpha = c.a; ctx.beginPath(); ctx.ellipse(c.x, c.y, 40 * c.s, 14 * c.s, 0, 0, 7);
      ctx.ellipse(c.x - 22 * c.s, c.y + 5 * c.s, 26 * c.s, 11 * c.s, 0, 0, 7);
      ctx.ellipse(c.x + 24 * c.s, c.y + 4 * c.s, 28 * c.s, 11 * c.s, 0, 0, 7); ctx.fill();
    }
    ctx.globalAlpha = 1;
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    for (const u of pups) {
      const d = PUP[u.t];
      ctx.fillStyle = d[0]; ctx.beginPath(); ctx.arc(u.x, u.y, 13 + Math.sin(u.ph) * 1.5, 0, 7); ctx.fill();
      ctx.strokeStyle = '#fff'; ctx.lineWidth = 2; ctx.stroke();
      ctx.fillStyle = '#fff'; ctx.font = `bold ${d[1].length > 1 ? 12 : 15}px ${FONT}`; ctx.fillText(d[1], u.x, u.y + 1);
    }
    for (const e of enemies) { enemy(e); decor(e); }
    drawWorldMeta();
    for (const w of warns) { ctx.fillStyle = `rgba(255,210,60,${.12 + .1 * Math.sin(time * .5)})`; ctx.fillRect(w.x - 8, 90, 16, H); }
    for (const s2 of strikes) { const on = s2.t <= 0; ctx.fillStyle = on ? 'rgba(255,60,60,.55)' : `rgba(255,220,60,${.15 + .15 * Math.sin(time * .6)})`; ctx.fillRect(s2.x - s2.w / 2, 0, s2.w, H); if (on) { ctx.fillStyle = '#fff'; ctx.fillRect(s2.x - 2, 0, 4, H); } }
    if (mg && mg.kind === 'z' && mg.safeX != null) { ctx.fillStyle = 'rgba(255,255,255,.2)'; ctx.fillRect(mg.safeX - 56, 0, 112, H); }
    for (const b of bullets) drawBullet(b);
    for (const b of ebullets) {
      ctx.fillStyle = 'rgba(255,61,90,.3)'; ctx.beginPath(); ctx.arc(b.x, b.y, 8, 0, 7); ctx.fill();
      ctx.fillStyle = '#ff3d5a'; ctx.beginPath(); ctx.arc(b.x, b.y, 4.5, 0, 7); ctx.fill();
      ctx.fillStyle = '#ffd0d8'; ctx.beginPath(); ctx.arc(b.x, b.y, 2, 0, 7); ctx.fill();
    }

    drawUan(1);
    if ((state === 'play' || state === 'pick') && !(p.inv > 0 && (p.inv >> 2) % 2)) {
      if (p.wing > 0 || p.drone) { plane(p.x - 38, p.y + 14, .45, '#d8f5e6', '#22a06b', '#2a4a7a'); plane(p.x + 38, p.y + 14, .45, '#d8f5e6', '#22a06b', '#2a4a7a'); }
      const sk = skin(); if (p.ghost > 0) ctx.globalAlpha = .45; ship(p.x, p.y, 1, sk, p.lvl, p.tilt, p.xl, p.sl); ctx.globalAlpha = 1; if (p.clone) { ctx.globalAlpha = .35; ship(W - p.x, p.y, 1, sk, p.lvl, -p.tilt, p.xl, p.sl); ctx.globalAlpha = 1; }
      ctx.fillStyle = sk.rb ? `hsl(${(time * 8) % 360},90%,60%)` : sk.flame; ctx.beginPath(); ctx.moveTo(p.x - 3, p.y + 20); ctx.lineTo(p.x, p.y + 28 + Math.random() * (p.lvl >= 4 ? 14 : 8)); ctx.lineTo(p.x + 3, p.y + 20); ctx.fill();
      if (cm() >= 5) { ctx.strokeStyle = 'rgba(255,230,120,.35)'; ctx.lineWidth = 2; ctx.beginPath(); ctx.arc(p.x, p.y, 26 + Math.min(10, cm() / 5), 0, 7); ctx.stroke(); }
      if (p.shield > 0) {
        ctx.strokeStyle = `rgba(80,200,255,${p.shield < 90 && (p.shield >> 2) % 2 ? .2 : .8})`; ctx.lineWidth = 3;
        ctx.beginPath(); ctx.arc(p.x, p.y, 32, 0, 7); ctx.stroke();
      }
    }
    for (const a of parts) {
      if (a.s) { drawPart(a); continue; }
      ctx.globalAlpha = Math.max(0, a.life / a.max); ctx.fillStyle = a.c;
      ctx.beginPath(); ctx.arc(a.x, a.y, a.r * (a.life / a.max + .3), 0, 7); ctx.fill();
    }
    for (const r of rings) { ctx.globalAlpha = Math.max(0, 1 - r.r / r.max); ctx.strokeStyle = r.c; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(r.x, r.y, r.r, 0, 7); ctx.stroke(); }
    drawUan();
    for (const q of floats) {
      ctx.globalAlpha = Math.min(1, q.l / 20); ctx.font = `bold 13px ${FONT}`; ctx.textAlign = 'center'; ctx.lineWidth = 3;
      ctx.strokeStyle = 'rgba(18,41,63,.7)'; ctx.fillStyle = '#ffd966'; ctx.strokeText(q.t, q.x, q.y); ctx.fillText(q.t, q.x, q.y);
    }
    ctx.globalAlpha = 1;
    ctx.setTransform(S, 0, 0, S, 0, 0);
    if (bombT > 0) { ctx.fillStyle = `rgba(255,255,255,${bombT / 22 * .8})`; ctx.fillRect(0, 0, W, H); }

    if (PAGES.includes(state)) { ctx.globalAlpha = 1; ctx.fillStyle = g; ctx.fillRect(0, 0, W, H); }
    if (state === 'menu') { drawMenu(); return; }
    if (state === 'shop') { drawShop(); return; }
    if (state === 'col') { drawCol(); return; }
    if (state === 'hub') { drawHub(); return; }
    if (state === 'prof') { drawProf(); return; }
    if (state === 'opt') { drawOpt(); return; }
    if (state === 'ach') { drawAch(); return; }
    if (state === 'best') { drawBest(); return; }
    if (state === 'stat') { drawStat(); return; }
    if (state === 'chest') { drawChest(); return; }
    if (state === 'bonus') { drawBonus(); return; }
    if (state === 'pause') { hud(); modal('PAUSE', [], ['REPRENDRE', 'OPTIONS', 'COLLECTION', 'MENU DU JEU', 'QUITTER LA PARTIE', 'RETOUR AU HUB'], .26); return; }
    if (state === 'resume') { modal('PARTIE EN COURS', ['Vague ' + wave + '  ·  Score ' + score], ['REPRENDRE', 'NOUVELLE PARTIE', 'MENU'], .3); return; }
    if (state === 'conf') { modal('RÉINITIALISER ?', ['Cette action efface tout.', 'Elle est définitive.'], ['NON, ANNULER', 'OUI, TOUT EFFACER'], .3); return; }
    if (state === 'over') {
      const sk = skin(), ln = ['Vague ' + wave + '  ·  Score ' + score, 'Ennemis ' + p.kills + '  ·  Combo max x' + (1 + (p.bc / 5 | 0)), '!◎ gagnés : +' + (p.earned || 0)];
      if (sk.cos) ln.push(sk.n + ' niveau ' + p.sl + '/7  ·  ' + (p.sl - 1) + ' transformation(s)');
      ln.push('◆ fragments : +' + (p.frag || 0)); ln.push('Esquives parfaites : ' + pdRun); if (score > 0 && score >= hi) ln.push('!🏆 NOUVEAU RECORD !');
      modal('PARTIE TERMINÉE', ln, ['REJOUER', 'MENU DU JEU', 'RETOUR AU HUB'], .58); return;
    }
    if (state === 'pick') { drawPick(); return; }
    // interface
    hud();
    if (state === 'play') {
      pill(PILL, '≡'); drawUlt(); drawBtns(); hudExtra(); if (isTest()) drawTestPanel();
      let i = 0;
      for (const t of TIMERS) if (p[t[0]] > 0) {
        const x = 24 + i++ * 30, col = PUP[t[2]][0];
        ctx.strokeStyle = col; ctx.lineWidth = 4; ctx.beginPath(); ctx.arc(x, 92, 11, -1.57, -1.57 + 6.283 * p[t[0]] / t[1]); ctx.stroke();
        ctx.fillStyle = '#12293f'; ctx.font = `bold 10px ${FONT}`; ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
        ctx.fillText(PUP[t[2]][1], x, 93); ctx.textAlign = 'left'; ctx.textBaseline = 'alphabetic';
      }
      const bs = enemies.find(e => e.k === 3);
      if (bs) {
        ctx.fillStyle = 'rgba(18,41,63,.55)'; ctx.fillRect(20, 60, W - 40, 12);
        ctx.fillStyle = BOSSES[bs.bi].c[1]; ctx.fillRect(20, 60, (W - 40) * Math.max(0, bs.hp) / bs.mhp, 12);
        ctx.fillStyle = '#fff'; ctx.font = `bold 12px ${FONT}`; ctx.textAlign = 'center'; ctx.fillText(BOSSES[bs.bi].n.toUpperCase(), W / 2, 56); ctx.textAlign = 'left';
      }
      if (mg && mg.cps) mg.cps.forEach((c, i) => { ctx.globalAlpha = i < mg.i ? .25 : 1; ctx.strokeStyle = i === mg.i ? '#ffd966' : '#fff'; ctx.lineWidth = 3; ctx.beginPath(); ctx.arc(c.x, c.y, 24 + (i === mg.i ? Math.sin(time * .2) * 3 : 0), 0, 7); ctx.stroke(); ctx.fillStyle = '#fff'; ctx.font = `bold 20px ${FONT}`; ctx.textAlign = 'center'; ctx.fillText('ABCD'[i], c.x, c.y + 7); ctx.globalAlpha = 1; });
      if (mg && (mg.need || mg.cps)) { const n = mg.cps ? 4 : mg.need, v = mg.cps ? mg.i : mg.got; ctx.fillStyle = 'rgba(18,41,63,.5)'; ctx.fillRect(60, 122, 240, 8); ctx.fillStyle = '#7dff9a'; ctx.fillRect(60, 122, 240 * Math.min(1, v / n), 8); ctx.fillStyle = '#fff'; ctx.font = `bold 11px ${FONT}`; ctx.textAlign = 'center'; ctx.fillText('OBJECTIF ' + Math.min(v, n) + ' / ' + n, W / 2, 144); }
      if (mg) { ctx.fillStyle = 'rgba(18,41,63,.5)'; ctx.fillRect(60, 108, 240, 8); ctx.fillStyle = '#ffe45e'; ctx.fillRect(60, 108, 240 * mg.t / mg.m, 8); ctx.fillStyle = '#fff'; ctx.font = `bold 12px ${FONT}`; ctx.textAlign = 'center'; ctx.fillText(mg.kind === 'c' ? 'CIBLES ' + mg.got + ' / ' + mg.need : 'BOUCLIER ACTIVÉ', W / 2, 132); ctx.textAlign = 'left'; }
      if (msgT > 0 && msg) {
        ctx.globalAlpha = Math.min(1, msgT / 30); ctx.textAlign = 'center'; ctx.lineWidth = 4; ctx.strokeStyle = 'rgba(18,41,63,.8)';
        ctx.fillStyle = '#7dff9a'; ctx.font = `bold 24px ${FONT}`; ctx.strokeText(msg.a, W / 2, H * .36); ctx.fillText(msg.a, W / 2, H * .36);
        ctx.fillStyle = '#ffd966'; ctx.font = `bold 20px ${FONT}`; ctx.strokeText(msg.b, W / 2, H * .36 + 28); ctx.fillText(msg.b, W / 2, H * .36 + 28);
        ctx.globalAlpha = 1; ctx.textAlign = 'left';
      }
      if (upT > 0) {
        ctx.globalAlpha = Math.min(1, upT / 30); ctx.textAlign = 'center'; ctx.lineWidth = 4; ctx.strokeStyle = 'rgba(18,41,63,.8)';
        ctx.fillStyle = '#ffd966'; ctx.font = `bold 22px ${FONT}`;
        ctx.strokeText('Amélioration : ' + upName, W / 2, H * .45); ctx.fillText('Amélioration : ' + upName, W / 2, H * .45);
        ctx.fillStyle = '#fff'; ctx.font = `15px ${FONT}`;
        ctx.strokeText('+1 vie · bouclier', W / 2, H * .45 + 24); ctx.fillText('+1 vie · bouclier', W / 2, H * .45 + 24);
        ctx.globalAlpha = 1; ctx.textAlign = 'left';
      }
      if (banner > 0) {
        ctx.globalAlpha = Math.min(1, banner / 30); ctx.textAlign = 'center'; ctx.fillStyle = '#fff';
        ctx.shadowColor = 'rgba(18,41,63,.6)'; ctx.shadowBlur = 8;
        ctx.font = `bold 26px ${FONT}`; ctx.fillText((wave % 5 === 0 ? 'BOSS · ' : isMini(wave) ? 'MINI-BOSS · ' : '') + 'VAGUE ' + wave, W / 2, H * .3); ctx.fillRect(W / 2 - 70, H * .3 + 10, 140, 2);
        if (bossName) { ctx.font = `bold 20px ${FONT}`; ctx.fillText(bossName, W / 2, H * .3 + 36); }
        ctx.fillStyle = DC[DNG(wave)]; ctx.font = `bold 14px ${FONT}`; ctx.fillText('DANGER : ' + DN[DNG(wave)], W / 2, H * .3 + 58);
        ctx.shadowBlur = 0; ctx.globalAlpha = 1;
      }
    } else {
      ctx.fillStyle = 'rgba(18,41,63,.6)'; ctx.fillRect(0, 0, W, H);
      ctx.textAlign = 'center'; ctx.fillStyle = '#fff';
      ctx.font = `bold 44px ${FONT}`; ctx.fillText(state === 'title' ? 'Escadrille' : 'Abattu', W / 2, H / 2 - 60);
      ctx.font = `20px ${FONT}`;
      if (state === 'over') ctx.fillText('Score ' + score + ' · Vague ' + wave + (score >= hi && score > 0 ? ' · record !' : ''), W / 2, H / 2 - 20);
      ctx.font = `15px ${FONT}`;
      const lines = state === 'title'
        ? ['Glisse le doigt : l\'avion te suit et tire seul.', 'P tir · S bouclier · + vie · B bombe', 'R cadence · ×2 points · W escorte · T ralenti', 'Mini-boss toutes les 5 vagues.', 'Ton avion évolue aux vagues 3, 6, 9 et 13.']
        : ['Ennemis : ' + p.kills + '  ·  Combo max : ' + p.bc + '  ·  Évo ' + p.xl, '◎ +' + (p.earned || 0) + ' gagnés', 'Touche l\'écran pour rejouer.'];
      lines.forEach((l, i) => ctx.fillText(l, W / 2, H / 2 + 20 + i * 24));
      if (state === 'title') { ctx.font = `bold 20px ${FONT}`; ctx.fillText('Touche pour décoller', W / 2, H / 2 + 135); }
      pill(menuBtn(), 'Menu');
    }
  }

  let last = performance.now(), acc = 0;
  function loop(now) {
    acc += Math.min(100, now - last); last = now;
    while (acc >= 16.667) { update(); acc -= 16.667; }
    AM.tick(themeNow());
    draw(); requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
  if (typeof window !== 'undefined' && window.__ARC_TEST) window.__ARC_EVAL = x => eval(x);
})();
</script>
</body>
</html>
