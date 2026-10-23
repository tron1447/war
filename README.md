[Uploading index (21).html…]()
<!doctype html>
<html lang="et">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
<title>Eesliin VR</title>
<script>
window.GAME_VERSION = 'el2';
(function(){ try { fetch(location.pathname + '?nocache=' + Date.now(), { cache: 'no-store' }).then(r => r.text()).then(t => { var m = t.match(/window\.GAME_VERSION = '([^']+)'/); if (m && m[1] !== window.GAME_VERSION) { var k = 'el_reload_' + m[1]; if (!sessionStorage.getItem(k)) { sessionStorage.setItem(k, '1'); location.replace(location.pathname + '?v=' + m[1] + '_' + Date.now()); } } }).catch(() => {}); } catch (e) {} })();
</script>
<style>
  html,body{margin:0;height:100%;background:#1c2024;overflow:hidden;font-family:system-ui,"Segoe UI",sans-serif}
  #info{position:fixed;top:10px;left:10px;color:#e8eadf;background:rgba(20,26,20,.82);border:2px solid #8a9a5b;padding:10px 14px;border-radius:10px;font-size:14px;line-height:1.5;max-width:460px;z-index:5}
  #info b{color:#d4c86a}
  #cross{position:fixed;left:50%;top:50%;width:6px;height:6px;margin:-3px;border-radius:50%;background:#fff;box-shadow:0 0 4px #000;z-index:4;pointer-events:none}
</style>
<script type="importmap">{"imports":{"three":"https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js","three/addons/":"https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"}}</script>
</head>
<body>
<div id="cross"></div>
<div id="info">
  <b>🎖️ EESLIIN VR</b> <span style="opacity:.6">(el2)</span><br>
  Questis: <b>ENTER VR</b>. Parem päästik = tulista, A = salv välja, B = vaheta relva, vasak külgnupp vöö juures = uus salv, rinnal = granaat, Y = paus.<br>
  Arvutis: klõpsa ekraanile, <b>WASD</b> = liigu, hiir = vaata, <b>klõps</b> = tulista, <b>parem klõps</b> = sihi, <b>R</b> = laadi, <b>1–4</b> = relv, <b>G</b> = granaat, <b>H</b> = suits, <b>C</b> = kükita, <b>Esc</b> = menüü.
</div>
<script type="module">
import * as THREE from 'three';
import { VRButton } from 'three/addons/webxr/VRButton.js';
import { XRHandModelFactory } from 'three/addons/webxr/XRHandModelFactory.js';
import { Sky } from 'three/addons/objects/Sky.js';

/* ===================== ABI ===================== */
const V3 = (x = 0, y = 0, z = 0) => new THREE.Vector3(x, y, z);
const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
const rand = (a = 0, b = 1) => a + Math.random() * (b - a);
const pick = a => a[Math.floor(Math.random() * a.length)];
const store = (k, v) => { try { if (v === undefined) return JSON.parse(localStorage.getItem(k)); localStorage.setItem(k, JSON.stringify(v)); } catch (e) { return null; } };
const FONT = '"Segoe UI", system-ui, sans-serif';
const tmpV = V3(), tmpD = V3(), tmpQ = new THREE.Quaternion(), head = V3();
function roundRect(c, x, y, w, h, r){ c.beginPath(); c.moveTo(x + r, y); c.arcTo(x + w, y, x + w, y + h, r); c.arcTo(x + w, y + h, x, y + h, r); c.arcTo(x, y + h, x, y, r); c.arcTo(x, y, x + w, y, r); c.closePath(); }
function canvasTex(w, h, fn, rep){ const cv = document.createElement('canvas'); cv.width = w; cv.height = h; fn(cv.getContext('2d'), w, h); const t = new THREE.CanvasTexture(cv); t.colorSpace = THREE.SRGBColorSpace; t.anisotropy = 4; if (rep) { t.wrapS = t.wrapT = THREE.RepeatWrapping; t.repeat.set(rep[0], rep[1]); } return t; }
function noiseTex(base, var_, n, rep, extra){ return canvasTex(256, 256, (c, W, H) => { c.fillStyle = base; c.fillRect(0, 0, W, H); for (let i = 0; i < n; i++) { const v = rand(-var_, var_); c.fillStyle = `rgba(${v > 0 ? 255 : 0},${v > 0 ? 255 : 0},${v > 0 ? 255 : 0},${Math.abs(v)})`; c.fillRect(rand(0, W), rand(0, H), rand(1, 6), rand(1, 6)); } if (extra) extra(c, W, H); }, rep); }

/* ===================== RENDERER ===================== */
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2)); renderer.setSize(innerWidth, innerHeight);
renderer.xr.enabled = true; renderer.xr.setReferenceSpaceType('local-floor');
renderer.shadowMap.enabled = true; renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
try { renderer.xr.setFoveation(0.4); } catch (e) {}
document.body.appendChild(renderer.domElement);
const vrBtn = VRButton.createButton(renderer); document.body.appendChild(vrBtn); vrBtn.addEventListener('click', () => audioInit());
const scene = new THREE.Scene();
const DAY = { sky: 0x9fb7c9, fog: 0xc8c4b4, hemi: 0.7, sun: 3.4 }, NIGHT = { sky: 0x070b14, fog: 0x0a0f18, hemi: 0.35, sun: 0.0 };
scene.background = new THREE.Color(DAY.sky); scene.fog = new THREE.Fog(0xc8c4b4, 60, 260);
renderer.toneMappingExposure = 0.75;
const SUN_DIR = new THREE.Vector3().setFromSphericalCoords(1, THREE.MathUtils.degToRad(60), THREE.MathUtils.degToRad(215));
function mkSky(scale){ const s = new Sky(); s.scale.setScalar(scale); const u = s.material.uniforms; u.turbidity.value = 7; u.rayleigh.value = 1.4; u.mieCoefficient.value = 0.006; u.mieDirectionalG.value = 0.82; u.sunPosition.value.copy(SUN_DIR); return s; }
const skyM = mkSky(480); scene.add(skyM);
let envTex = null; { const pm = new THREE.PMREMGenerator(renderer), es = new THREE.Scene(); es.add(mkSky(100)); envTex = pm.fromScene(es, 0, 0.1, 200).texture; }
scene.environment = envTex; scene.background = null;
const stars = (() => { const n = 1500, p = new Float32Array(n * 3); for (let i = 0; i < n; i++) { const th = Math.random() * Math.PI * 2, ph = Math.acos(Math.random() * 0.95), r = 400; p[i * 3] = Math.sin(ph) * Math.cos(th) * r; p[i * 3 + 1] = Math.cos(ph) * r; p[i * 3 + 2] = Math.sin(ph) * Math.sin(th) * r; } const g = new THREE.BufferGeometry(); g.setAttribute('position', new THREE.BufferAttribute(p, 3)); const o = new THREE.Points(g, new THREE.PointsMaterial({ color: 0xffffff, size: 1.5, sizeAttenuation: false, fog: false })); o.visible = false; scene.add(o); return o; })();
const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.03, 600);
const rig = new THREE.Group(); rig.add(camera); scene.add(rig); camera.position.set(0, 1.65, 0);
addEventListener('resize', () => { camera.aspect = innerWidth / innerHeight; camera.updateProjectionMatrix(); renderer.setSize(innerWidth, innerHeight); });
const hemi = new THREE.HemisphereLight(0xdfe8f0, 0x5b5040, DAY.hemi); scene.add(hemi);
const sun = new THREE.DirectionalLight(0xfff1dc, DAY.sun); sun.castShadow = true; sun.shadow.mapSize.set(2048, 2048);
Object.assign(sun.shadow.camera, { left: -30, right: 30, top: 30, bottom: -30, near: 1, far: 120 }); sun.shadow.bias = -0.0006; scene.add(sun); scene.add(sun.target);
const flash = new THREE.PointLight(0xffc070, 0, 8, 2); scene.add(flash);
const torch = new THREE.SpotLight(0xffffff, 0, 30, 0.35, 0.5, 1.3); scene.add(torch); scene.add(torch.target);

/* ===================== MATERJALID ===================== */
const std = (c, o = {}) => new THREE.MeshStandardMaterial(Object.assign({ color: c, roughness: 0.85 }, o));
const M = {
  dirt: std(0xffffff, { map: noiseTex('#8a7a5c', 0.12, 4000, [60, 60]) }),
  concrete: std(0xffffff, { map: noiseTex('#a6a49c', 0.08, 2500, [1, 1], (c, W, H) => { c.fillStyle = 'rgba(60,50,40,0.15)'; for (let i = 0; i < 5; i++) c.fillRect(rand(0, W), 0, rand(4, 14), H); }) }),
  wallIn: std(0xffffff, { map: noiseTex('#c9c2b1', 0.05, 1500, [1, 1]) }),
  wood: std(0xffffff, { map: canvasTex(128, 128, (c, W, H) => { c.fillStyle = '#8b6a3e'; c.fillRect(0, 0, W, H); c.strokeStyle = '#5e4526'; c.lineWidth = 4; for (let y = 0; y < H; y += 32) { c.beginPath(); c.moveTo(0, y); c.lineTo(W, y); c.stroke(); } c.strokeRect(2, 2, W - 4, H - 4); c.beginPath(); c.moveTo(0, 0); c.lineTo(W, H); c.stroke(); }) }),
  sand: std(0xffffff, { map: noiseTex('#b29e72', 0.12, 1500, [2, 1], (c, W, H) => { c.strokeStyle = 'rgba(70,55,30,0.5)'; c.lineWidth = 3; for (let y = 0; y < H; y += 32) for (let x = (y / 32 % 2) * 40; x < W; x += 80) { roundRect(c, x, y, 78, 30, 12); c.stroke(); } }) }),
  metal: std(0x5a5f63, { roughness: 0.45, metalness: 0.7 }), gun: std(0x222426, { roughness: 0.5, metalness: 0.6 }), gunTan: std(0x8b7d5a, { roughness: 0.7 }),
  olive: std(0x4f5a34), tan: std(0xa08c62), dark: std(0x2b2e2a), skin: std(0xd2a27a), red: std(0xb32020), blue: std(0x2a5ab3),
  road: std(0xffffff, { map: noiseTex('#4a4a48', 0.1, 4000, [6, 30]) })
};
function brickTex(c1, c2){ return canvasTex(256, 256, (c, W, H) => { c.fillStyle = '#6e665c'; c.fillRect(0, 0, W, H); for (let r = 0; r < 16; r++) for (let k = -1; k < 8; k++) { const x = k * 36 + (r % 2) * 18, y = r * 16; const t = rand(); c.fillStyle = t < 0.5 ? c1 : c2; c.globalAlpha = 0.75 + rand(0, 0.25); c.fillRect(x + 1, y + 1, 34, 14); } c.globalAlpha = 1; for (let i = 0; i < 1500; i++) { c.fillStyle = `rgba(0,0,0,${rand(0, 0.15)})`; c.fillRect(rand(0, W), rand(0, H), 2, 2); } c.fillStyle = 'rgba(30,25,20,0.25)'; for (let i = 0; i < 4; i++) c.fillRect(rand(0, W), rand(0, H * 0.3), rand(6, 20), H); }, [1, 1]); }
M.brick = std(0xffffff, { map: brickTex('#8e5a43', '#7a4a36') }); M.brick2 = std(0xffffff, { map: brickTex('#a89d86', '#958a74') });
M.plaster = std(0xffffff, { map: noiseTex('#c7bba0', 0.08, 3000, [1, 1], (c, W, H) => { c.fillStyle = 'rgba(90,70,50,0.18)'; for (let i = 0; i < 6; i++) { c.beginPath(); c.arc(rand(0, W), rand(0, H), rand(10, 40), 0, 7); c.fill(); } c.fillStyle = 'rgba(40,30,20,0.25)'; c.fillRect(0, H * 0.85, W, H * 0.15); }) });
for (const [k, t, b] of [['concrete', 2.5, 0.6], ['wallIn', 2, 0.3], ['wood', 1.2, 0.8], ['sand', 1.6, 1.2], ['brick', 2, 1], ['brick2', 2, 1], ['plaster', 2.5, 0.5]]) { const m = M[k]; m.userData.tile = t; m.map.wrapS = m.map.wrapT = THREE.RepeatWrapping; m.map.repeat.set(1, 1); m.bumpMap = m.map; m.bumpScale = b; m.needsUpdate = true; }
M.dirt.map.dispose(); M.dirt.map = canvasTex(512, 512, (c, W, H) => { c.fillStyle = '#7d6f52'; c.fillRect(0, 0, W, H); for (let i = 0; i < 260; i++) { c.fillStyle = pick(['rgba(92,110,58,0.35)', 'rgba(70,90,45,0.3)', 'rgba(120,100,70,0.3)', 'rgba(60,50,40,0.25)']); c.beginPath(); c.ellipse(rand(0, W), rand(0, H), rand(8, 50), rand(6, 30), rand(0, 3), 0, 7); c.fill(); } for (let i = 0; i < 9000; i++) { const v = rand(); c.fillStyle = `rgba(${v > 0.5 ? 200 : 30},${v > 0.5 ? 190 : 25},${v > 0.5 ? 160 : 20},${rand(0.05, 0.2)})`; c.fillRect(rand(0, W), rand(0, H), 2, 2); } }, [80, 80]); M.dirt.bumpMap = M.dirt.map; M.dirt.bumpScale = 1.5;
M.camo = std(0xffffff, { map: canvasTex(128, 128, (c, W, H) => { c.fillStyle = '#6b6a4a'; c.fillRect(0, 0, W, H); for (const col of ['#4a4f32', '#8a7d5a', '#3b3a2a', '#5e6b45']) for (let i = 0; i < 14; i++) { c.fillStyle = col; c.beginPath(); c.ellipse(rand(0, W), rand(0, H), rand(6, 18), rand(4, 12), rand(0, 3), 0, 7); c.fill(); } }, [2, 2]) });
M.gunMetal = std(0x1c1d1f, { roughness: 0.35, metalness: 0.85 }); M.polymer = std(0x262626, { roughness: 0.75, metalness: 0.1 }); M.fde = std(0x8a7a58, { roughness: 0.7, metalness: 0.1 });
const cont = c => std(0xffffff, { map: canvasTex(256, 128, (x, W, H) => { x.fillStyle = c; x.fillRect(0, 0, W, H); for (let i = 0; i < W; i += 12) { x.fillStyle = 'rgba(0,0,0,0.18)'; x.fillRect(i, 0, 4, H); } x.fillStyle = 'rgba(255,255,255,0.6)'; x.font = 'bold 22px sans-serif'; x.fillText('EST ' + Math.floor(rand(1000, 9999)), 20, 30); }), roughness: 0.6, metalness: 0.3 });

/* ===================== MAAILM + KOKKUPÕRKED ===================== */
const COL = [];      // {x0,x1,y0,y1,z0,z1}
const world = new THREE.Group(); scene.add(world);
function tileUV(g, w, h, d, t){ const uv = g.attributes.uv, dims = [[d, h], [d, h], [w, d], [w, d], [w, h], [w, h]]; for (let f = 0; f < 6; f++) for (let k = 0; k < 4; k++) { const i = f * 4 + k; uv.setXY(i, uv.getX(i) * dims[f][0] / t, uv.getY(i) * dims[f][1] / t); } }
function B(x, y, z, w, h, d, mat, collide = true, parent = world){
  const geo = new THREE.BoxGeometry(w, h, d); if (mat.userData.tile) tileUV(geo, w, h, d, mat.userData.tile);
  const m = new THREE.Mesh(geo, mat); m.position.set(x, y + h / 2, z); m.castShadow = true; m.receiveShadow = true; parent.add(m);
  if (mat.map && mat.map.repeat && !mat.userData.keep) { /* tekstuurid korduvad */ }
  if (collide) COL.push({ x0: x - w / 2, x1: x + w / 2, y0: y, y1: y + h, z0: z - d / 2, z1: z + d / 2, m });
  return m;
}
const ground = new THREE.Mesh(new THREE.PlaneGeometry(600, 600), M.dirt); ground.rotation.x = -Math.PI / 2; ground.receiveShadow = true; scene.add(ground);
// hoone seintega ja ukseavadega
function building(cx, cz, w, d, h, doors = [], roof = true, mat = pick([M.brick, M.brick2, M.plaster])){
  const t = 0.25, sides = { n: [cz - d / 2, 'x', w], s: [cz + d / 2, 'x', w], w: [cx - w / 2, 'z', d], e: [cx + w / 2, 'z', d] };
  for (const k in sides) {
    const [pos, ax, len] = sides[k], ds = doors.filter(o => o.s === k).sort((a, b) => a.o - b.o); let start = -len / 2;
    const seg = (a, b) => { if (b - a < 0.05) return; const mid = (a + b) / 2; if (ax === 'x') B(cx + mid, 0, pos, b - a, h, t, mat); else B(pos, 0, cz + mid, t, h, b - a, mat); };
    for (const o of ds) { seg(start, o.o - 0.55); if (ax === 'x') B(cx + o.o, 2.2, pos, 1.1, h - 2.2, t, mat); else B(pos, 2.2, cz + o.o, t, h - 2.2, 1.1, mat); start = o.o + 0.55; }
    seg(start, len / 2);
  }
  if (roof) B(cx, h, cz, w + 0.3, 0.2, d + 0.3, mat, true);
}
const crate = (x, z, s = 1.1, h = 1.1, rot = 0) => { const m = B(x, 0, z, s, h, s, M.wood); m.rotation.y = rot; return m; };
const sandbags = (x, z, w, d, h = 1.05) => B(x, 0, z, w, h, d, M.sand);
const container = (x, z, rotX = true, c = pick(['#7a3b2a', '#2c5d7a', '#3f6b3a', '#8a6d2a'])) => rotX ? B(x, 0, z, 6, 2.6, 2.4, cont(c)) : B(x, 0, z, 2.4, 2.6, 6, cont(c));

/* --- 1) BAAS + LASKETIIR (0,0) --- */
const RANGE = { x: 0, z: 0 };
for (let i = -3; i <= 3; i++) { B(i * 1.8 - 0.9, 0, -0.6, 0.12, 1.1, 0.8, M.wood); }
B(0, 0.95, -0.5, 13, 0.08, 0.9, M.wood);              // laud
B(0, 0, -150, 40, 6, 3, M.sand); B(-20, 0, -75, 1, 4, 150, M.sand, true); B(20, 0, -75, 1, 4, 150, M.sand, true); // vall
for (let z = -12; z > -140; z -= 20) for (const x of [-20, 20]) B(x, 4, z, 1.2, 0.3, 1.2, M.wood, false);
// lipp
{ const p = B(-8, 0, 3, 0.08, 7, 0.08, M.metal, false); const fl = new THREE.Mesh(new THREE.PlaneGeometry(1.5, 0.95), new THREE.MeshStandardMaterial({ map: canvasTex(150, 95, (c, W, H) => { c.fillStyle = '#0072ce'; c.fillRect(0, 0, W, H / 3); c.fillStyle = '#000'; c.fillRect(0, H / 3, W, H / 3); c.fillStyle = '#fff'; c.fillRect(0, H * 2 / 3, W, H / 3); }), side: THREE.DoubleSide })); fl.position.set(-7.2, 6.4, 3); scene.add(fl); window._flag = fl; }
// telgid ja sõidukid baasis
for (const [x, z] of [[-14, 10], [-14, 18], [14, 12]]) { const t = new THREE.Mesh(new THREE.CylinderGeometry(0.01, 3.2, 3, 4, 1), M.olive); t.rotation.y = Math.PI / 4; t.scale.z = 1.6; t.position.set(x, 1.5, z); t.castShadow = true; scene.add(t); COL.push({ x0: x - 2.2, x1: x + 2.2, y0: 0, y1: 3, z0: z - 3.4, z1: z + 3.4 }); }
function truck(x, z, rot = 0, parent = world){ const g = new THREE.Group(); g.position.set(x, 0, z); g.rotation.y = rot; parent.add(g); const add = (w, h, d, px, py, pz, m) => { const b = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), m); b.position.set(px, py, pz); b.castShadow = b.receiveShadow = true; g.add(b); return b; }; add(2.4, 1.3, 2, 0, 1.4, -2.6, M.olive); add(2.5, 1.6, 4.4, 0, 1.6, 0.8, M.olive); add(2.3, 0.3, 6.8, 0, 0.6, 0, M.dark); add(2.2, 0.5, 0.05, 0, 1.75, -3.61, std(0x8fb3c9, { roughness: 0.1, metalness: 0.5 })); for (const [wx, wz] of [[-1.15, -2.4], [1.15, -2.4], [-1.15, 1.8], [1.15, 1.8]]) { const w = new THREE.Mesh(new THREE.CylinderGeometry(0.5, 0.5, 0.35, 16).rotateZ(Math.PI / 2), M.dark); w.position.set(wx, 0.5, wz); w.castShadow = true; g.add(w); } const c = Math.cos(rot), s = Math.sin(rot), hw = Math.abs(c) * 1.3 + Math.abs(s) * 3.4, hd = Math.abs(s) * 1.3 + Math.abs(c) * 3.4; COL.push({ x0: x - hw, x1: x + hw, y0: 0, y1: 2.4, z0: z - hd, z1: z + hd }); return g; }
truck(10, 22, 0.3); truck(-4, 26, -0.2);

/* --- 2) TAPAMAJA (killhouse) (40, -20) --- */
const KH = { x: 40, z: -20 };
building(40, -20, 16, 20, 3, [{ s: 's', o: -5 }, { s: 'n', o: 5 }], false, M.wood);
for (const m of [M.concrete]) {}
B(36, 0, -20, 0.2, 3, 8, M.wood); B(44, 0, -16, 8, 3, 0.2, M.wood); B(37.5, 0, -24, 5, 3, 0.2, M.wood); B(44, 0, -25, 0.2, 3, 6, M.wood);
B(35, 0, -14, 6, 3, 0.2, M.wood); crate(46, -12); crate(34, -27); crate(41, -21, 0.9, 0.9);
// lae talad
for (let z = -29; z <= -11; z += 3) B(40, 3, z, 16, 0.15, 0.15, M.wood, false);

/* --- 3) KOMPLEKS (lained, missioonid) (-70, -40) --- */
const CX = -70, CZ = -40;
building(CX - 12, CZ - 10, 12, 9, 3.4, [{ s: 's', o: -2 }, { s: 'e', o: 1 }]);
building(CX + 12, CZ - 12, 10, 8, 3.4, [{ s: 's', o: 2 }, { s: 'w', o: -1 }]);
building(CX - 14, CZ + 12, 9, 10, 3.4, [{ s: 'n', o: 1 }, { s: 'e', o: -2 }]);
building(CX + 14, CZ + 12, 12, 10, 6, [{ s: 'n', o: -2 }, { s: 'w', o: 2 }]);   // torn/ladu
for (const [x, z, r] of [[CX - 3, CZ - 20, true], [CX + 3, CZ + 22, true], [CX - 25, CZ, false], [CX + 26, CZ - 2, false], [CX + 2, CZ - 1, true]]) container(x, z, r);
for (const [x, z, w, d] of [[CX, CZ + 5, 4, 0.8], [CX - 5, CZ + 2, 0.8, 3], [CX + 5, CZ + 2, 0.8, 3], [CX - 8, CZ - 4, 3, 0.8], [CX + 8, CZ - 5, 3, 0.8], [CX - 20, CZ + 22, 4, 0.8], [CX + 22, CZ - 22, 4, 0.8], [CX - 22, CZ - 22, 0.8, 4], [CX, CZ + 30, 5, 0.8], [CX, CZ - 30, 5, 0.8]]) sandbags(x, z, w, d);
for (const [x, z] of [[CX - 6, CZ + 15], [CX + 7, CZ + 17], [CX - 18, CZ - 22], [CX + 20, CZ + 25], [CX - 27, CZ + 18], [CX + 25, CZ - 18], [CX - 2, CZ + 11], [CX + 4, CZ - 11], [CX - 30, CZ - 10], [CX + 30, CZ + 8]]) { crate(x, z, 1.1, 1.1, rand(0, 1)); if (Math.random() < 0.5) crate(x + 1.2, z + rand(-0.3, 0.3), 1, 1, rand(0, 1)); }
// müür ümber
for (const [x, z, w, d] of [[CX, CZ - 38, 78, 0.6], [CX, CZ + 38, 78, 0.6], [CX - 38, CZ, 0.6, 78], [CX + 38, CZ, 0.6, 78]]) { const gap = 8; if (w > d) { B(x - (w / 4 + gap / 4), 0, z, w / 2 - gap / 2, 2.6, d, M.concrete); B(x + (w / 4 + gap / 4), 0, z, w / 2 - gap / 2, 2.6, d, M.concrete); } else { B(x, 0, z - (d / 4 + gap / 4), w, 2.6, d / 2 - gap / 2, M.concrete); B(x, 0, z + (d / 4 + gap / 4), w, 2.6, d / 2 - gap / 2, M.concrete); } }
const road = new THREE.Mesh(new THREE.PlaneGeometry(8, 160), M.road); road.rotation.x = -Math.PI / 2; road.position.set(CX, 0.01, CZ); road.receiveShadow = true; scene.add(road);
// taust: puud
for (let i = 0; i < 120; i++) { const a = rand(0, Math.PI * 2), r = rand(110, 220), x = Math.cos(a) * r, z = Math.sin(a) * r; const t = new THREE.Mesh(new THREE.ConeGeometry(rand(2, 3.5), rand(7, 12), 7), std(pick([0x2f4a2a, 0x37522c, 0x2a3f26]))); t.position.set(x, 4, z); scene.add(t); }
/* --- detailid: tünnid, vrakid, postid, aiad, rohi --- */
const barrelM = [std(0x7a2e1e, { roughness: 0.55, metalness: 0.4 }), std(0x2f4a6b, { roughness: 0.55, metalness: 0.4 }), std(0x4a5530, { roughness: 0.6, metalness: 0.3 })];
function barrel(x, z){ const m = new THREE.Mesh(new THREE.CylinderGeometry(0.3, 0.3, 0.9, 14), pick(barrelM)); m.position.set(x, 0.45, z); m.castShadow = m.receiveShadow = true; world.add(m); for (const y of [0.2, 0.7]) { const r = new THREE.Mesh(new THREE.TorusGeometry(0.305, 0.012, 6, 20).rotateX(Math.PI / 2), M.metal); r.position.set(x, y, z); world.add(r); } COL.push({ x0: x - 0.3, x1: x + 0.3, y0: 0, y1: 0.9, z0: z - 0.3, z1: z + 0.3 }); }
for (const [x, z] of [[CX - 7, CZ - 14], [CX - 6.3, CZ - 14.5], [CX + 18, CZ - 5], [CX + 18.6, CZ - 5.4], [CX - 21, CZ + 6], [CX + 9, CZ + 24], [CX - 4, CZ - 26], [CX + 31, CZ + 30], [CX - 31, CZ - 30], [12, -5], [-12, -4]]) barrel(x, z);
const fires = [];
function wreck(x, z, rot, burning){ const g = new THREE.Group(); g.position.set(x, 0, z); g.rotation.y = rot; world.add(g); const burnt = std(0x2a2522, { roughness: 0.9, metalness: 0.3 }), rust = std(0x5a3220, { roughness: 0.95 }); const add = (w, h, d, px, py, pz, m, rx = 0, rz = 0) => { const b = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), m); b.position.set(px, py, pz); b.rotation.set(rx, 0, rz); b.castShadow = b.receiveShadow = true; g.add(b); }; add(1.9, 0.6, 4.2, 0, 0.55, 0, rust, 0, 0.04); add(1.7, 0.6, 2.1, 0, 1.15, 0.2, burnt, 0.05); for (const [wx, wz] of [[-0.9, -1.3], [0.9, 1.3], [-0.9, 1.4]]) add(0.25, 0.5, 0.5, wx, 0.25, wz, burnt); const c = Math.abs(Math.cos(rot)), sn = Math.abs(Math.sin(rot)); COL.push({ x0: x - (c + sn * 2.1), x1: x + (c + sn * 2.1), y0: 0, y1: 1.4, z0: z - (sn + c * 2.1), z1: z + (sn + c * 2.1) }); if (burning) { const l = new THREE.PointLight(0xff7a2a, 3, 10, 2); l.position.set(x, 1.6, z); scene.add(l); fires.push({ p: V3(x, 1.2, z), l }); } }
wreck(CX - 26, CZ + 26, 0.6, true); wreck(CX + 24, CZ + 33, 1.6, false); wreck(CX + 6, CZ + 33, -0.3, false); wreck(CX - 10, CZ - 33, 1.2, true);
// elektripostid juhtmetega
{ const pm = std(0x5a4632); let prev = null; for (let z = 40; z > -120; z -= 18) { const x = CX - 6, p = new THREE.Mesh(new THREE.CylinderGeometry(0.12, 0.15, 8, 8), pm); p.position.set(x, 4, CZ + z); p.castShadow = true; world.add(p); const bar = new THREE.Mesh(new THREE.BoxGeometry(1.6, 0.1, 0.1), pm); bar.position.set(x, 7.6, CZ + z); world.add(bar); if (prev) for (const dx of [-0.7, 0.7]) { const pts = []; for (let i = 0; i <= 10; i++) { const k = i / 10; pts.push(V3(x + dx, 7.65 - Math.sin(k * Math.PI) * 0.6, prev + (CZ + z - prev) * k)); } world.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints(pts), new THREE.LineBasicMaterial({ color: 0x111111 }))); } prev = CZ + z; COL.push({ x0: x - 0.15, x1: x + 0.15, y0: 0, y1: 8, z0: CZ + z - 0.15, z1: CZ + z + 0.15 }); } }
// võrkaed tiiru ümber
{ const fence = std(0xffffff, { map: canvasTex(64, 64, (c, W, H) => { c.clearRect(0, 0, W, H); c.strokeStyle = '#9aa0a3'; c.lineWidth = 3; for (let i = -64; i < 128; i += 16) { c.beginPath(); c.moveTo(i, 0); c.lineTo(i + 64, 64); c.stroke(); c.beginPath(); c.moveTo(i + 64, 0); c.lineTo(i, 64); c.stroke(); } }, [1, 1]), transparent: true, alphaTest: 0.4, side: THREE.DoubleSide, metalness: 0.6, roughness: 0.4 }); fence.map.wrapS = fence.map.wrapT = THREE.RepeatWrapping; for (const [x, z, w, r] of [[-24, 12, 40, 0], [24, 12, 40, 0]]) { const f = new THREE.Mesh(new THREE.PlaneGeometry(w, 2.2), fence); f.material.map.repeat.set(w / 1.5, 2.2 / 1.5); f.position.set(x, 1.1, z); world.add(f); for (let k = 0; k <= w; k += 4) { const p = new THREE.Mesh(new THREE.CylinderGeometry(0.04, 0.04, 2.3, 6), M.metal); p.position.set(x - w / 2 + k, 1.15, z); world.add(p); } } }
// rohututid (instanced)
{ const gt = canvasTex(64, 64, (c, W, H) => { c.clearRect(0, 0, W, H); for (let i = 0; i < 22; i++) { c.strokeStyle = pick(['#5d7a34', '#6f8a3c', '#4f6a2c', '#8a8a4a']); c.lineWidth = rand(2, 3.5); c.beginPath(); const x = rand(8, 56); c.moveTo(x, H); c.quadraticCurveTo(x + rand(-10, 10), H * 0.5, x + rand(-14, 14), rand(4, 20)); c.stroke(); } });
  const geo = new THREE.PlaneGeometry(0.7, 0.5).translate(0, 0.25, 0), g2 = geo.clone().rotateY(Math.PI / 2), merged = new THREE.BufferGeometry(); const pa = [...geo.attributes.position.array, ...g2.attributes.position.array], ua = [...geo.attributes.uv.array, ...g2.attributes.uv.array], na = [...geo.attributes.normal.array, ...g2.attributes.normal.array]; merged.setAttribute('position', new THREE.Float32BufferAttribute(pa, 3)); merged.setAttribute('uv', new THREE.Float32BufferAttribute(ua, 2)); merged.setAttribute('normal', new THREE.Float32BufferAttribute(na.map((v, i) => i % 3 === 1 ? 1 : 0), 3)); merged.setIndex([0, 2, 1, 2, 3, 1, 4, 6, 5, 6, 7, 5]);
  const N = 2600, im = new THREE.InstancedMesh(merged, std(0xffffff, { map: gt, alphaTest: 0.5, side: THREE.DoubleSide }), N); const mt = new THREE.Matrix4(); let n = 0;
  for (let i = 0; i < N * 3 && n < N; i++) { const x = rand(-120, 60), z = rand(-90, 40); if (blockedAt(x, z, 0.4, 0, 3)) continue; if (Math.abs(x - CX) < 4.5 && Math.abs(z - CZ) < 80) continue; if (Math.abs(x) < 20 && z < 2 && z > -150) continue; const sc = rand(0.6, 1.4); mt.compose(V3(x, 0, z), new THREE.Quaternion().setFromEuler(new THREE.Euler(0, rand(0, 6), 0)), V3(sc, sc * rand(0.7, 1.3), sc)); im.setMatrixAt(n++, mt); }
  im.count = n; im.receiveShadow = true; scene.add(im); }
// kaugel suitsusambad ja tuli
const SMOKE_COL = [V3(-180, 0, -160), V3(140, 0, -220), V3(-60, 0, -240)];
// varjupunktid automaatselt madalate takistuste juures
const COVER = [];
for (const c of COL) { const h = c.y1 - c.y0; if (h < 0.8 || h > 2.8 || c.y0 > 0.1) continue; const cx = (c.x0 + c.x1) / 2, cz = (c.z0 + c.z1) / 2; for (const [nx, nz] of [[1, 0], [-1, 0], [0, 1], [0, -1]]) { const px = nx ? (nx > 0 ? c.x1 + 0.6 : c.x0 - 0.6) : cx, pz = nz ? (nz > 0 ? c.z1 + 0.6 : c.z0 - 0.6) : cz; if (!blockedAt(px, pz, 0.35)) COVER.push({ x: px, z: pz, nx: -nx, nz: -nz, low: h < 1.5 }); } }
function blockedAt(x, z, r = 0.3, y0 = 0.3, y1 = 1.6){ for (const c of COL) if (x > c.x0 - r && x < c.x1 + r && z > c.z0 - r && z < c.z1 + r && c.y1 > y0 && c.y0 < y1) return true; return false; }
function rayBox(o, d, c, maxT){ let tmin = 0, tmax = maxT; for (const [oa, da, a0, a1] of [[o.x, d.x, c.x0, c.x1], [o.y, d.y, c.y0, c.y1], [o.z, d.z, c.z0, c.z1]]) { if (Math.abs(da) < 1e-9) { if (oa < a0 || oa > a1) return Infinity; continue; } let t1 = (a0 - oa) / da, t2 = (a1 - oa) / da; if (t1 > t2) [t1, t2] = [t2, t1]; tmin = Math.max(tmin, t1); tmax = Math.min(tmax, t2); if (tmin > tmax) return Infinity; } return tmin; }
function castWorld(o, d, maxT){ let best = maxT, hit = null; for (const c of COL) { const t = rayBox(o, d, c, best); if (t < best) { best = t; hit = c; } } if (d.y < 0) { const tg = -o.y / d.y; if (tg < best) { best = tg; hit = 'ground'; } } return { t: best, hit }; }
function los(a, b){ const d = tmpD.copy(b).sub(a), L = d.length(); d.divideScalar(L); return castWorld(a, d.clone(), L - 0.1).t >= L - 0.1 && !smokeBlocks(a, b); }

/* ===================== AUDIO ===================== */
let actx = null, master, sfx, rev, noiseBuf;
function audioInit(){
  if (actx) { if (actx.state === 'suspended' && mode !== 'pause') actx.resume(); return; }
  actx = new (window.AudioContext || window.webkitAudioContext)(); master = actx.createGain(); master.gain.value = 0.9; const comp = actx.createDynamicsCompressor(); master.connect(comp); comp.connect(actx.destination);
  sfx = actx.createGain(); sfx.connect(master);
  noiseBuf = actx.createBuffer(1, actx.sampleRate * 2, actx.sampleRate); const d = noiseBuf.getChannelData(0); for (let i = 0; i < d.length; i++) d[i] = Math.random() * 2 - 1;
  rev = actx.createConvolver(); const ir = actx.createBuffer(2, actx.sampleRate * 1.8, actx.sampleRate); for (let ch = 0; ch < 2; ch++) { const b = ir.getChannelData(ch); for (let i = 0; i < b.length; i++) b[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / b.length, 3); } rev.buffer = ir; const rg = actx.createGain(); rg.gain.value = 0.35; rev.connect(rg); rg.connect(master);
  // tuul
  const w = actx.createBufferSource(); w.buffer = noiseBuf; w.loop = true; const wf = actx.createBiquadFilter(); wf.type = 'lowpass'; wf.frequency.value = 300; const wg = actx.createGain(); wg.gain.value = 0.04; w.connect(wf); wf.connect(wg); wg.connect(master); w.start();
}
function out(pos){ if (!pos) return sfx; const p = actx.createPanner(); p.panningModel = 'HRTF'; p.distanceModel = 'inverse'; p.refDistance = 2; p.rolloffFactor = 1; p.positionX.value = pos.x; p.positionY.value = pos.y; p.positionZ.value = pos.z; p.connect(sfx); p.connect(rev); return p; }
function nz(t, dur, freq, pk, type = 'lowpass', dest){ const s = actx.createBufferSource(); s.buffer = noiseBuf; const f = actx.createBiquadFilter(); f.type = type; f.frequency.value = freq; const g = actx.createGain(); g.gain.setValueAtTime(pk, t); g.gain.exponentialRampToValueAtTime(0.0001, t + dur); s.connect(f); f.connect(g); g.connect(dest); s.start(t, Math.random()); s.stop(t + dur + 0.05); }
function tn(t, f0, f1, dur, pk, type, dest){ const o = actx.createOscillator(), g = actx.createGain(); o.type = type; o.frequency.setValueAtTime(f0, t); o.frequency.exponentialRampToValueAtTime(f1, t + dur); g.gain.setValueAtTime(pk, t); g.gain.exponentialRampToValueAtTime(0.0001, t + dur); o.connect(g); g.connect(dest); o.start(t); o.stop(t + dur + 0.05); }
const S = {
  shot(kind, pos, supp){ if (!actx) return; const t = actx.currentTime, d = out(pos); if (supp) { nz(t, 0.12, 1800, 0.35, 'bandpass', d); return; } const big = kind === 'sniper' ? 1.6 : kind === 'shotgun' ? 1.4 : kind === 'pistol' ? 0.8 : 1; nz(t, 0.25 * big, 3500, 0.9, 'lowpass', d); nz(t, 0.06, 6000, 0.5, 'highpass', d); tn(t, 160, 45, 0.18 * big, 0.9, 'sine', d); if (!pos) nz(t, 0.6, 800, 0.15, 'lowpass', rev); },
  enemyShot(pos){ if (!actx) return; const t = actx.currentTime, d = out(pos); nz(t, 0.2, 2500, 0.6, 'lowpass', d); tn(t, 140, 50, 0.15, 0.5, 'sine', d); },
  laser(pos){ if (!actx) return; const t = actx.currentTime; tn(t, 1400, 300, 0.18, 0.25, 'sawtooth', out(pos)); },
  paint(pos){ if (!actx) return; const t = actx.currentTime; nz(t, 0.08, 1200, 0.5, 'bandpass', out(pos)); tn(t, 300, 120, 0.08, 0.3, 'square', out(pos)); },
  whiz(pos){ if (!actx) return; const t = actx.currentTime; nz(t, 0.12, 4000, 0.25, 'bandpass', out(pos)); },
  click(){ if (!actx) return; const t = actx.currentTime; tn(t, 2000, 1500, 0.03, 0.2, 'square', sfx); },
  mag(){ if (!actx) return; const t = actx.currentTime; tn(t, 900, 600, 0.05, 0.25, 'square', sfx); nz(t + 0.05, 0.05, 3000, 0.3, 'bandpass', sfx); },
  rack(){ if (!actx) return; const t = actx.currentTime; nz(t, 0.06, 2500, 0.5, 'bandpass', sfx); nz(t + 0.12, 0.06, 2000, 0.5, 'bandpass', sfx); },
  empty(){ if (!actx) return; const t = actx.currentTime; tn(t, 1200, 1100, 0.03, 0.15, 'square', sfx); },
  boom(pos){ if (!actx) return; const t = actx.currentTime, d = out(pos); nz(t, 1.6, 600, 1.4, 'lowpass', d); tn(t, 90, 25, 1.0, 1.2, 'sine', d); nz(t, 2.5, 300, 0.4, 'lowpass', rev); },
  hit(){ if (!actx) return; const t = actx.currentTime; tn(t, 1800, 1700, 0.05, 0.15, 'triangle', sfx); },
  hurt(){ if (!actx) return; const t = actx.currentTime; tn(t, 200, 80, 0.2, 0.4, 'sawtooth', sfx); },
  ok(){ if (!actx) return; const t = actx.currentTime; [523, 659, 784].forEach((f, i) => tn(t + i * 0.08, f, f, 0.2, 0.15, 'triangle', sfx)); },
  bad(){ if (!actx) return; const t = actx.currentTime; tn(t, 300, 150, 0.4, 0.2, 'square', sfx); },
  beep(f = 880){ if (!actx) return; tn(actx.currentTime, f, f, 0.12, 0.15, 'square', sfx); },
  hiss(pos){ if (!actx) return; nz(actx.currentTime, 2.5, 2000, 0.3, 'highpass', out(pos)); }
};
function say(txt){ try { if (!window.speechSynthesis) return; const u = new SpeechSynthesisUtterance(txt); u.lang = 'et-EE'; u.rate = 1.1; u.volume = 0.8; speechSynthesis.speak(u); } catch (e) {} }

/* ===================== OSAKESED ===================== */
const PN = 900, pPos = new Float32Array(PN * 3), pCol = new Float32Array(PN * 3), pSize = new Float32Array(PN), pAlpha = new Float32Array(PN), pVel = Array.from({ length: PN }, () => V3()), pLife = new Float32Array(PN), pMax = new Float32Array(PN), pGrav = new Float32Array(PN), pGrow = new Float32Array(PN); let pI = 0;
const pGeo = new THREE.BufferGeometry(); for (const [n, a, s] of [['position', pPos, 3], ['pcolor', pCol, 3], ['psize', pSize, 1], ['palpha', pAlpha, 1]]) pGeo.setAttribute(n, new THREE.BufferAttribute(a, s));
const pMat = new THREE.ShaderMaterial({ uniforms: { uScale: { value: 600 } }, transparent: true, depthWrite: false,
  vertexShader: `attribute vec3 pcolor; attribute float psize; attribute float palpha; uniform float uScale; varying vec3 vC; varying float vA; void main(){ vC = pcolor; vA = palpha * (psize > 0.5 ? 0.42 : 1.0); vec4 mv = modelViewMatrix * vec4(position, 1.0); gl_PointSize = psize * uScale / max(0.05, -mv.z); gl_Position = projectionMatrix * mv; }`,
  fragmentShader: `varying vec3 vC; varying float vA; void main(){ float d = length(gl_PointCoord - 0.5); float a = smoothstep(0.5, 0.1, d); gl_FragColor = vec4(vC, a * vA); }` });
const pts = new THREE.Points(pGeo, pMat); pts.frustumCulled = false; pts.renderOrder = 5; scene.add(pts);
const tc = new THREE.Color();
function emit(p, v, col, size, life, grav = 0, grow = 0){ const i = pI = (pI + 1) % PN; pPos[i * 3] = p.x; pPos[i * 3 + 1] = p.y; pPos[i * 3 + 2] = p.z; pVel[i].copy(v); tc.set(col); pCol[i * 3] = tc.r; pCol[i * 3 + 1] = tc.g; pCol[i * 3 + 2] = tc.b; pSize[i] = size; pLife[i] = life; pMax[i] = life; pGrav[i] = grav; pGrow[i] = grow; }
function updParticles(dt){ for (let i = 0; i < PN; i++) { if (pLife[i] <= 0) { pAlpha[i] = 0; continue; } pLife[i] -= dt; pVel[i].y += pGrav[i] * dt; pVel[i].multiplyScalar(1 - dt * 0.5); pPos[i * 3] += pVel[i].x * dt; pPos[i * 3 + 1] += pVel[i].y * dt; pPos[i * 3 + 2] += pVel[i].z * dt; pSize[i] += pGrow[i] * dt; pAlpha[i] = Math.min(1, pLife[i] / pMax[i] * 1.5); } for (const k of ['position', 'pcolor', 'psize', 'palpha']) pGeo.attributes[k].needsUpdate = true; }
// kuulijäljed (tracerid)
const TR = 40, trPos = new Float32Array(TR * 6), trLife = new Float32Array(TR); let trI = 0;
const trGeo = new THREE.BufferGeometry(); trGeo.setAttribute('position', new THREE.BufferAttribute(trPos, 3));
const trCol = new Float32Array(TR * 6); trGeo.setAttribute('color', new THREE.BufferAttribute(trCol, 3));
const tracers = new THREE.LineSegments(trGeo, new THREE.LineBasicMaterial({ vertexColors: true, transparent: true, opacity: 0.85, blending: THREE.AdditiveBlending, depthWrite: false })); tracers.frustumCulled = false; scene.add(tracers);
function tracer(a, b, col = 0xffd27a){ const i = trI = (trI + 1) % TR; const s = a.clone().lerp(b, 0.15); trPos.set([s.x, s.y, s.z, b.x, b.y, b.z], i * 6); tc.set(col); trCol.set([tc.r, tc.g, tc.b, tc.r, tc.g, tc.b], i * 6); trLife[i] = 0.06; }
function updTracers(dt){ for (let i = 0; i < TR; i++) { if (trLife[i] > 0) { trLife[i] -= dt; if (trLife[i] <= 0) trPos.fill(0, i * 6, i * 6 + 6); } } trGeo.attributes.position.needsUpdate = true; trGeo.attributes.color.needsUpdate = true; }
// kuuliaugud
const holes = []; const holeGeo = new THREE.CircleGeometry(0.03, 8), holeMat = new THREE.MeshBasicMaterial({ color: 0x111111, transparent: true, opacity: 0.8, polygonOffset: true, polygonOffsetFactor: -2 });
function hole(p, n, col){ const m = new THREE.Mesh(holeGeo, col ? new THREE.MeshBasicMaterial({ color: col, polygonOffset: true, polygonOffsetFactor: -2 }) : holeMat); if (col) m.scale.setScalar(rand(2, 3.5)); m.position.copy(p).addScaledVector(n, 0.01); m.lookAt(p.clone().add(n)); scene.add(m); holes.push(m); if (holes.length > 80) holes.shift().removeFromParent(); }
// suits
const smokes = [];
function smokeBlocks(a, b){ for (const s of smokes) { if (s.t < 1) continue; const d = tmpD.copy(b).sub(a), L = d.length(); d.divideScalar(L); const k = clamp(tmpV.copy(s.p).sub(a).dot(d), 0, L); if (a.clone().addScaledVector(d, k).distanceTo(s.p) < s.r) return true; } return false; }

/* ===================== UI PANEELID ===================== */
class Panel { constructor(w, h, ppm = 500){ this.w = w; this.h = h; this.cv = document.createElement('canvas'); this.cv.width = Math.round(w * ppm); this.cv.height = Math.round(h * ppm); this.ctx = this.cv.getContext('2d'); this.tex = new THREE.CanvasTexture(this.cv); this.tex.colorSpace = THREE.SRGBColorSpace; this.mesh = new THREE.Mesh(new THREE.PlaneGeometry(w, h), new THREE.MeshBasicMaterial({ map: this.tex, transparent: true, depthWrite: false, depthTest: false, toneMapped: false, fog: false })); this.mesh.renderOrder = 50; } draw(fn){ const c = this.ctx; c.clearRect(0, 0, this.cv.width, this.cv.height); fn(c, this.cv.width, this.cv.height); this.tex.needsUpdate = true; } }
const OD = '#8a9a5b', GOLD = '#d4c86a';
function bg(c, W, H, col = OD){ roundRect(c, 6, 6, W - 12, H - 12, 24); c.fillStyle = 'rgba(18,24,18,0.93)'; c.fill(); c.lineWidth = 6; c.strokeStyle = col; c.stroke(); }
function txt(c, s, x, y, size, col = '#e8eadf', weight = 'bold', align = 'center'){ c.font = `${weight} ${size}px ${FONT}`; c.textAlign = align; c.textBaseline = 'middle'; c.fillStyle = col; c.fillText(s, x, y); }
const buttons = [];
function btn(parent, w, h, x, y, label, onClick, col = OD){
  const p = new Panel(w, h, 600); p.mesh.position.set(x, y, 0.01); p.mesh.renderOrder = 51; parent.add(p.mesh);
  const b = { p, w, h, label, col, onClick, sel: false, hover: false, sub: '', lock: false };
  b.draw = () => p.draw((c, W, H) => { roundRect(c, 6, 6, W - 12, H - 12, H * 0.2); c.fillStyle = b.lock ? 'rgba(40,40,40,0.9)' : b.sel ? b.col : b.hover ? 'rgba(80,95,60,0.95)' : 'rgba(35,44,30,0.95)'; c.fill(); c.lineWidth = b.sel || b.hover ? 8 : 4; c.strokeStyle = b.lock ? '#555' : b.col; c.stroke(); let fs = H * (b.sub ? 0.3 : 0.4); c.font = `bold ${fs}px ${FONT}`; while (c.measureText(b.label).width > W * 0.86 && fs > 9) { fs--; c.font = `bold ${fs}px ${FONT}`; } txt(c, b.label, W / 2, b.sub ? H * 0.38 : H * 0.52, fs, b.lock ? '#888' : b.sel ? '#111' : '#e8eadf'); if (b.sub) { let f2 = H * 0.2; c.font = `${f2}px ${FONT}`; while (c.measureText(b.sub).width > W * 0.86 && f2 > 8) { f2--; c.font = `${f2}px ${FONT}`; } txt(c, b.sub, W / 2, H * 0.72, f2, b.sel ? '#222' : '#b9c0a8', 'normal'); } });
  b.draw(); p.mesh.userData.btn = b; buttons.push(b); return b;
}
const isShown = o => { while (o) { if (!o.visible) return false; o = o.parent; } return true; };
function placeInFront(g, dist, yOff = 0){ camera.getWorldPosition(head); camera.getWorldDirection(tmpD); tmpD.y = 0; if (tmpD.lengthSq() < 1e-4) tmpD.set(0, 0, -1); tmpD.normalize(); g.position.copy(head).addScaledVector(tmpD, dist); g.position.y = head.y + yOff; g.rotation.set(0, Math.atan2(-tmpD.x, -tmpD.z), 0); }
// teated
const msgP = new Panel(1.6, 0.22, 400); msgP.mesh.renderOrder = 60; camera.add(msgP.mesh); msgP.mesh.position.set(0, 0.32, -1.2); msgP.mesh.visible = false; let msgT = 0;
function msg(s, col = GOLD, dur = 2.5){ msgP.draw((c, W, H) => { roundRect(c, 4, 4, W - 8, H - 8, H * 0.3); c.fillStyle = 'rgba(10,14,10,0.8)'; c.fill(); let fs = H * 0.45; c.font = `bold ${fs}px ${FONT}`; while (c.measureText(s).width > W * 0.94 && fs > 10) { fs--; c.font = `bold ${fs}px ${FONT}`; } txt(c, s, W / 2, H / 2, fs, col); }); msgP.mesh.visible = true; msgT = dur; }
// HUD (käe peal / ekraanil)
const hudP = new Panel(0.5, 0.12, 500); hudP.mesh.renderOrder = 55; camera.add(hudP.mesh); hudP.mesh.position.set(0, -0.42, -1.0); hudP.mesh.visible = false;
const hurtM = new THREE.Mesh(new THREE.SphereGeometry(0.2, 16, 12), new THREE.MeshBasicMaterial({ color: 0x880000, transparent: true, opacity: 0, side: THREE.BackSide, depthTest: false, fog: false })); hurtM.renderOrder = 99; camera.add(hurtM);
const nvgM = new THREE.Mesh(new THREE.SphereGeometry(0.21, 16, 12), new THREE.MeshBasicMaterial({ color: 0x00ff40, transparent: true, opacity: 0.18, side: THREE.BackSide, depthTest: false, blending: THREE.AdditiveBlending, fog: false })); nvgM.renderOrder = 98; nvgM.visible = false; camera.add(nvgM);
// objektiivi marker
const objMark = new Panel(0.5, 0.25, 300); objMark.mesh.renderOrder = 58; scene.add(objMark.mesh); objMark.mesh.visible = false; let objPos = null, objLabel = '';

/* ===================== SALVESTUS, AUASTMED ===================== */
const RANKS = ['Reamees', 'Kapral', 'Nooremseersant', 'Seersant', 'Vanemseersant', 'Veebel', 'Lipnik', 'Leitnant', 'Kapten', 'Major'];
const save = Object.assign({ xp: 0, enemy: 'soldier', diff: 1, reload: 'auto', move: 'smooth', best: {}, mis: {} }, store('el_save') || {});
if (!save.v2) { save.v2 = true; save.reload = 'auto'; }
const persist = () => store('el_save', save);
const rankIdx = () => Math.min(RANKS.length - 1, Math.floor(Math.sqrt(save.xp / 150)));
const rankXp = i => i * i * 150;
function addXp(n){ const r0 = rankIdx(); save.xp += n; persist(); if (rankIdx() > r0) { msg(`🎖️ Ülendus! Uus auaste: ${RANKS[rankIdx()]}`, GOLD, 4); S.ok(); say('Ülendus'); } }
const DIFF = [{ n: '🙂 Kerge', acc: 0.18, dmg: 7, react: 1.1 }, { n: '😐 Tavaline', acc: 0.3, dmg: 11, react: 0.7 }, { n: '😈 Raske', acc: 0.45, dmg: 16, react: 0.45 }];

/* ===================== RELVAD ===================== */
const WEAPONS = {
  rifle: { n: 'Automaat', auto: true, rate: 0.09, dmg: 34, spread: 0.012, mag: 30, recoil: 0.04, pellets: 1, rank: 0 },
  pistol: { n: 'Püstol', auto: false, rate: 0.14, dmg: 30, spread: 0.014, mag: 12, recoil: 0.06, pellets: 1, rank: 0 },
  shotgun: { n: 'Haavlipüss', auto: false, rate: 0.8, dmg: 16, spread: 0.06, mag: 6, recoil: 0.14, pellets: 9, rank: 2 },
  sniper: { n: 'Snaipripüss', auto: false, rate: 1.2, dmg: 160, spread: 0.0005, mag: 5, recoil: 0.2, pellets: 1, rank: 3, scope: 6 }
};
const WORDER = ['rifle', 'pistol', 'shotgun', 'sniper'];
function buildGun(kind){
  const g = new THREE.Group(), add = (w, h, d, x, y, z, m = M.gun) => { const b = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), m); b.position.set(x, y, z); b.castShadow = true; g.add(b); return b; };
  const cyl = (r, l, x, y, z, m = M.gun) => { const c = new THREE.Mesh(new THREE.CylinderGeometry(r, r, l, 10).rotateX(Math.PI / 2), m); c.position.set(x, y, z); g.add(c); return c; };
  let muzzle, magwell, fore, sightY = 0.07;
  if (kind === 'pistol') {
    add(0.028, 0.1, 0.034, 0, -0.035, 0.012, M.polymer).rotation.x = 0.18; add(0.03, 0.03, 0.17, 0, 0.012, -0.06, M.polymer);
    add(0.032, 0.032, 0.19, 0, 0.042, -0.065, M.gunMetal); for (let i = 0; i < 6; i++) add(0.033, 0.026, 0.003, 0, 0.044, 0.01 + i * 0.006, M.polymer);
    add(0.004, 0.012, 0.022, 0, -0.012, -0.03, M.gunMetal); add(0.006, 0.01, 0.008, 0, 0.063, -0.15, M.gunMetal); add(0.016, 0.009, 0.006, 0, 0.063, 0.02, M.gunMetal);
    cyl(0.006, 0.01, 0, 0.042, -0.162, M.dark); muzzle = V3(0, 0.042, -0.17); magwell = V3(0, -0.095, 0.02); fore = null; sightY = 0.068;
  } else {
    const L = kind === 'sniper' ? 0.9 : kind === 'shotgun' ? 0.75 : 0.62, furn = kind === 'rifle' ? M.fde : kind === 'sniper' ? M.camo : M.polymer;
    // lukukast ja käepide
    add(0.052, 0.06, 0.34, 0, 0.035, -0.08, M.gunMetal); add(0.05, 0.03, 0.16, 0, -0.008, 0.0, M.gunMetal);
    add(0.034, 0.12, 0.045, 0, -0.06, 0.06, furn).rotation.x = 0.28; add(0.012, 0.04, 0.05, 0, -0.03, -0.02, M.gunMetal);
    // päästikukaitse
    add(0.008, 0.006, 0.05, 0, -0.045, -0.015, M.gunMetal);
    // pära
    add(0.04, 0.05, 0.2, 0, 0.02, 0.2, furn); add(0.045, 0.11, 0.025, 0, -0.005, 0.31, M.polymer); add(0.02, 0.02, 0.12, 0, 0.045, 0.14, M.gunMetal);
    // eesmine käekaitse siinidega
    add(0.055, 0.055, 0.26, 0, 0.03, -0.37, furn); for (let i = 0; i < 9; i++) add(0.04, 0.008, 0.012, 0, 0.061, -0.27 - i * 0.025, M.gunMetal); for (const x of [-0.029, 0.029]) for (let i = 0; i < 4; i++) add(0.004, 0.012, 0.03, x, 0.03, -0.29 - i * 0.05, M.polymer);
    add(0.05, 0.008, 0.33, 0, 0.068, -0.15, M.gunMetal); for (let i = 0; i < 11; i++) add(0.054, 0.006, 0.01, 0, 0.074, 0.0 - i * 0.03, M.gunMetal);
    // toru, leegisummuti
    cyl(kind === 'shotgun' ? 0.014 : 0.0095, L * 0.55, 0, 0.04, -0.5 - L * 0.12, M.gunMetal); cyl(0.016, 0.06, 0, 0.04, -0.5 - L * 0.4, M.dark);
    if (kind === 'shotgun') { cyl(0.012, 0.3, 0, 0.012, -0.45, M.gunMetal); add(0.05, 0.05, 0.15, 0, 0.012, -0.42, furn); }
    // salv (kumer)
    if (kind !== 'shotgun') { for (let i = 0; i < 4; i++) { const sg = add(0.026, 0.045, 0.06, 0, -0.04 - i * 0.04, -0.1 - i * 0.008 * i, M.dark); sg.rotation.x = -0.1 - i * 0.1; } }
    // sihikud
    if (kind === 'sniper') { cyl(0.025, 0.3, 0, 0.115, -0.12, M.gunMetal); cyl(0.031, 0.05, 0, 0.115, -0.28, M.gunMetal); cyl(0.031, 0.05, 0, 0.115, 0.04, M.gunMetal); add(0.016, 0.04, 0.02, 0, 0.085, -0.18, M.gunMetal); add(0.016, 0.04, 0.02, 0, 0.085, -0.04, M.gunMetal); cyl(0.008, 0.035, 0.035, 0.115, -0.12, M.gunMetal).rotation.set(0, 0, Math.PI / 2); for (const sx of [-1, 1]) { const bp = add(0.01, 0.16, 0.01, sx * 0.02, -0.04, -0.6, M.gunMetal); bp.rotation.set(0.5, 0, sx * 0.3); } }
    else if (kind === 'rifle') { add(0.036, 0.034, 0.06, 0, 0.098, -0.05, M.gunMetal); add(0.032, 0.03, 0.004, 0, 0.1, -0.08, M.glassScope || (M.glassScope = std(0x335566, { transparent: true, opacity: 0.35, metalness: 0.9, roughness: 0.05 }))); const dot = new THREE.Mesh(new THREE.CircleGeometry(0.0022, 8), new THREE.MeshBasicMaterial({ color: 0xff2020, fog: false })); dot.position.set(0, 0.1, -0.078); g.add(dot); }
    else { add(0.006, 0.02, 0.006, 0, 0.085, -0.6); add(0.02, 0.018, 0.01, 0, 0.085, 0.05); }
    muzzle = V3(0, 0.04, -0.5 - L * 0.45); magwell = V3(0, -0.12, -0.1); fore = V3(0, 0.0, -0.37); sightY = kind === 'sniper' ? 0.115 : kind === 'rifle' ? 0.1 : 0.085;
  }
  const mz = new THREE.Object3D(); mz.position.copy(muzzle); g.add(mz);
  // salv relvas
  const mag = add(kind === 'pistol' ? 0.024 : 0.028, kind === 'pistol' ? 0.08 : 0.12, 0.05, magwell.x, magwell.y + (kind === 'pistol' ? 0.03 : 0.05), magwell.z, M.dark);
  // laskemoona loendur
  const ammoP = new Panel(0.06, 0.03, 1200); ammoP.mesh.renderOrder = 40; ammoP.mesh.position.set(-0.03, 0.03, kind === 'pistol' ? -0.05 : -0.1); ammoP.mesh.rotation.y = -Math.PI / 2; g.add(ammoP.mesh);
  // snaipri sihik: renderdatakse tekstuuri
  let lens = null; if (kind === 'sniper') { lens = new THREE.Mesh(new THREE.CircleGeometry(0.026, 24), new THREE.MeshBasicMaterial({ map: scopeRT.texture, fog: false, toneMapped: false })); lens.position.set(0, 0.115, 0.052); g.add(lens); }
  const flashM = new THREE.Mesh(new THREE.ConeGeometry(0.04, 0.15, 8).rotateX(-Math.PI / 2), new THREE.MeshBasicMaterial({ color: 0xffd27a, transparent: true, opacity: 0.9, blending: THREE.AdditiveBlending, depthWrite: false })); flashM.position.copy(muzzle).add(V3(0, 0, -0.07)); flashM.visible = false; g.add(flashM);
  return { g, mz, magwell, fore, mag, ammoP, lens, flashM, sightY };
}
const scopeRT = new THREE.WebGLRenderTarget(384, 384);
const laserDot = new THREE.Mesh(new THREE.SphereGeometry(0.018, 8, 6), new THREE.MeshBasicMaterial({ color: 0xff1a1a, fog: false })); laserDot.visible = false; scene.add(laserDot);
const laserBeam = new THREE.Mesh(new THREE.CylinderGeometry(0.0012, 0.0012, 1, 4).translate(0, 0.5, 0).rotateX(Math.PI / 2), new THREE.MeshBasicMaterial({ color: 0xff3030, transparent: true, opacity: 0.35, fog: false })); laserBeam.visible = false; scene.add(laserBeam);
const scopeCam = new THREE.PerspectiveCamera(75 / 6, 1, 0.1, 600);
const guns = {}; for (const k of WORDER) { const g = buildGun(k); g.g.visible = false; scene.add(g.g); guns[k] = Object.assign(g, { kind: k, def: WEAPONS[k], ammo: WEAPONS[k].mag, hasMag: true, chambered: true }); }
// granaadid, salved (esemed)
const props = [];
function newProp(type, pos){
  const g = new THREE.Group(); scene.add(g); g.position.copy(pos);
  if (type === 'mag') { const m = new THREE.Mesh(new THREE.BoxGeometry(0.03, 0.13, 0.055), M.dark); m.castShadow = true; g.add(m); }
  if (type === 'frag') { const m = new THREE.Mesh(new THREE.SphereGeometry(0.035, 12, 10), M.olive); m.scale.y = 1.2; g.add(m); const l = new THREE.Mesh(new THREE.BoxGeometry(0.012, 0.05, 0.02), M.metal); l.position.set(0.02, 0.03, 0); g.add(l); }
  if (type === 'smoke') { const m = new THREE.Mesh(new THREE.CylinderGeometry(0.03, 0.03, 0.11, 12), std(0x9aa0a6)); g.add(m); }
  const p = { type, g, vel: V3(), held: null, fuse: type === 'frag' ? 3.2 : type === 'smoke' ? 1.5 : 0, live: false, rest: false, age: 0 }; props.push(p); return p;
}
function removeProp(p){ p.g.removeFromParent(); props.splice(props.indexOf(p), 1); }

/* ===================== MÄNGIJA ===================== */
const P = { hp: 100, maxHp: 100, regenT: 0, weapon: 'rifle', cool: 0, crouch: false, armor: 0, frags: 3, smokes: 2, mags: 99, dead: false, H: 1.65, reloadT: 0, ads: false, nvg: false };
const handFactory = new XRHandModelFactory();
const hands = [0, 1].map(i => {
  const ctrl = renderer.xr.getController(i), grip = renderer.xr.getControllerGrip(i), hand = renderer.xr.getHand(i);
  rig.add(ctrl); rig.add(grip); rig.add(hand); const hm = handFactory.createHandModel(hand, 'spheres'); hand.add(hm);
  const glove = new THREE.Group(); grip.add(glove); { const palm = new THREE.Mesh(new THREE.BoxGeometry(0.06, 0.03, 0.09), M.dark); palm.position.set(0, -0.01, 0.03); glove.add(palm); for (let f = 0; f < 4; f++) { const fg = new THREE.Mesh(new THREE.CapsuleGeometry(0.01, 0.04, 4, 8), M.dark); fg.rotation.x = Math.PI / 2; fg.position.set(-0.022 + f * 0.015, -0.02, -0.03); glove.add(fg); } }
  const holder = new THREE.Group(); grip.add(holder); holder.rotation.x = -0.25;
  const hHold = new THREE.Group(); scene.add(hHold);
  const ray = new THREE.Mesh(new THREE.CylinderGeometry(0.002, 0.002, 1, 6).translate(0, -0.5, 0).rotateX(Math.PI / 2), new THREE.MeshBasicMaterial({ color: GOLD, transparent: true, opacity: 0.6 })); ray.visible = false; ctrl.add(ray);
  const h = { i, kind: 'xr', ctrl, grip, hand, hm, glove, holder, hHold, ray, side: null, isHand: false, connected: false, held: null, vel: V3(), prev: V3(), hasPrev: false, hover: null, gp: null, poke: null, trig: false, grip_: false, btns: {}, gClosed: false, pinch: false };
  ctrl.addEventListener('connected', e => { h.connected = true; h.side = e.data.handedness; h.isHand = !!e.data.hand; h.gp = e.data.gamepad; glove.visible = !h.isHand; if (h.isHand) save.reload = 'auto'; });
  ctrl.addEventListener('disconnected', () => { h.connected = false; });
  ctrl.addEventListener('selectstart', () => { audioInit(); if (h.hover) clickBtn(h.hover, h); else h.selTrig = true; });
  ctrl.addEventListener('selectend', () => { h.selTrig = false; });
  return h;
});
const R = () => hands.find(h => h.connected && h.side === 'right') || hands.find(h => h.connected);
const L = () => hands.find(h => h.connected && h.side === 'left');
function pulse(h, s = 0.5, ms = 50){ try { const a = h && h.gp && h.gp.hapticActuators && h.gp.hapticActuators[0]; if (a) a.pulse(s, ms); } catch (e) {} }
function handPos(h, out){ if (h.isHand && h.hand.joints && h.hand.joints['middle-finger-metacarpal']) return h.hand.joints['middle-finger-metacarpal'].getWorldPosition(out); return h.grip.getWorldPosition(out); }
function clickBtn(b, h){ if (!isShown(b.p.mesh) || b.lock) { if (b.lock) S.bad(); return; } S.click(); pulse(h, 0.4, 30); b.onClick(); }
// lauaarvuti juhtimine
const keys = {}; let yaw = 0, pitch = 0, mouseDown = false, rmb = false;
addEventListener('keydown', e => { keys[e.code] = true; audioInit();
  if (e.code === 'Escape') { if (mode === 'play') pauseGame(); else if (mode === 'pause') resumeGame(); }
  if (mode === 'play' || mode === 'hub') { if (e.code === 'KeyR') startReload(); if (e.code >= 'Digit1' && e.code <= 'Digit4') setWeapon(WORDER[+e.code.slice(5) - 1]); if (e.code === 'KeyG') deskThrow('frag'); if (e.code === 'KeyH') deskThrow('smoke'); if (e.code === 'KeyN') toggleNvg(); if (e.code === 'KeyQ') useStreak(); if (e.code === 'KeyE') interact(); }
});
addEventListener('keyup', e => keys[e.code] = false);
const mouseNDC = new THREE.Vector2();
renderer.domElement.addEventListener('mousedown', e => { audioInit(); if (deskHover) { clickBtn(deskHover); return; } if (document.pointerLockElement !== renderer.domElement && (mode === 'play' || mode === 'hub') && !menuOpen()) { renderer.domElement.requestPointerLock(); return; } if (e.button === 0) mouseDown = true; if (e.button === 2) rmb = true; });
addEventListener('mouseup', e => { if (e.button === 0) mouseDown = false; if (e.button === 2) rmb = false; });
renderer.domElement.addEventListener('contextmenu', e => e.preventDefault());
addEventListener('mousemove', e => { mouseNDC.set(e.clientX / innerWidth * 2 - 1, -(e.clientY / innerHeight) * 2 + 1); if (document.pointerLockElement === renderer.domElement) { const s = P.ads ? 0.0008 : 0.0022; yaw -= e.movementX * s; pitch = clamp(pitch - e.movementY * s, -1.4, 1.4); } });
let deskHover = null;
const menuOpen = () => menuG.visible || endG.visible || pauseG.visible || armG.visible || (typeof wireG !== 'undefined' && wireG.visible) || buyMenu.visible;

function setWeapon(k){ if (!k || P.weapon === k || WEAPONS[k].rank > rankIdx() && !(mission && mission.weapons && mission.weapons.includes(k))) { if (k && WEAPONS[k].rank > rankIdx()) msg(`🔒 ${WEAPONS[k].n} avaneb auastmel ${RANKS[WEAPONS[k].rank]}`, '#ff8a80', 2); return; } P.weapon = k; P.reloadT = 0; S.rack(); for (const w of WORDER) guns[w].g.visible = false; guns[k].g.visible = true; drawAmmo(guns[k]); }
function cycleWeapon(){ const av = WORDER.filter(w => WEAPONS[w].rank <= rankIdx() || mission && mission.weapons && mission.weapons.includes(w)); setWeapon(av[(av.indexOf(P.weapon) + 1) % av.length]); }
function drawAmmo(g){ g.ammoP.draw((c, W, H) => { c.fillStyle = 'rgba(0,0,0,0.6)'; c.fillRect(0, 0, W, H); txt(c, g.hasMag ? `${g.ammo}/${g.def.mag}` : '— —', W / 2, H / 2, H * 0.7, g.ammo === 0 ? '#ff5252' : '#9cff57', 'bold'); }); }
function startReload(){ const g = guns[P.weapon]; if (P.reloadT > 0 || g.ammo === g.def.mag && g.hasMag) return; P.reloadT = g.kind === 'shotgun' ? 2.2 : 1.6; S.mag(); }
function ejectMag(){ const g = guns[P.weapon]; if (!g.hasMag) return; g.hasMag = false; g.mag.visible = false; const p = newProp('mag', g.g.localToWorld(g.magwell.clone())); p.live = true; p.vel.set(0, -1, 0); p.empty = true; g.ammo = 0; drawAmmo(g); S.mag(); }
function insertMag(){ const g = guns[P.weapon]; g.hasMag = true; g.mag.visible = true; g.ammo = g.def.mag; drawAmmo(g); S.mag(); setTimeout(() => S.rack(), 200); }
function toggleNvg(){ if (!mission || !mission.night) return; P.nvg = !P.nvg; P.nvgSeen = false; nvgM.visible = P.nvg; S.beep(P.nvg ? 1200 : 600); applyLight(); }

/* ===================== TULISTAMINE ===================== */
const shootables = [];  // { test(o,d,maxT)->{t,part}, hit(point,part,dmg,dir) }
function fire(){
  const g = guns[P.weapon], d = g.def;
  if (P.cool > 0 || P.reloadT > 0 || P.dead) return;
  if (g.ammo <= 0 || !g.hasMag) { S.empty(); P.cool = 0.25; if (save.reload === 'auto' || !renderer.xr.isPresenting) startReload(); return; }
  g.ammo--; P.cool = d.rate; drawAmmo(g);
  const o = g.mz.getWorldPosition(V3()), q = g.g.getWorldQuaternion(new THREE.Quaternion()), base = V3(0, 0, -1).applyQuaternion(q);
  if (!renderer.xr.isPresenting) { camera.getWorldDirection(base); camera.getWorldPosition(tmpV); o.copy(tmpV).addScaledVector(base, 0.3); }
  const twoHand = P.twoHand || (!renderer.xr.isPresenting && rmb);
  for (let k = 0; k < d.pellets; k++) {
    const sp = d.spread * (twoHand ? 0.4 : 1) * (P.crouch ? 0.8 : 1), dir = base.clone().add(V3(rand(-sp, sp), rand(-sp, sp), rand(-sp, sp))).normalize();
    shootRay(o, dir, d.dmg, k === 0);
  }
  S.shot(P.weapon, null, false); flash.position.copy(o); flash.intensity = 4; g.flashM.visible = true; g.flashM.rotation.z = rand(0, 6);
  setTimeout(() => { g.flashM.visible = false; flash.intensity = 0; }, 45);
  emit(g.g.localToWorld(V3(0.03, 0.05, -0.08)), V3(1.2, 1.4, 0.3).applyQuaternion(q), 0xd4a74a, 0.012, 0.6, -9);
  for (let i = 0; i < 3; i++) emit(o, base.clone().multiplyScalar(rand(1, 2)), 0xaaaaaa, 0.05, 0.5, 0.3, 0.3);
  // tagasilöök
  if (renderer.xr.isPresenting) { P.kick = d.recoil; const h = R(); pulse(h, 0.6 + d.recoil * 2, 40); if (P.twoHand) pulse(L(), 0.5, 40); } else { pitch += d.recoil * 0.6; yaw += rand(-0.01, 0.01); }
  alertEnemies(o, P.weapon === 'pistol' ? 25 : 45);
}
function shootRay(o, dir, dmg, trace){
  let best = 300, sh = null, part = null;
  const w = castWorld(o, dir, best); best = w.t;
  for (const s of shootables) { const r = s.test(o, dir, best); if (r && r.t < best) { best = r.t; sh = s; part = r.part; } }
  const p = o.clone().addScaledVector(dir, best);
  if (trace) tracer(o, p, save.enemy === 'robot' ? 0x6ff7ff : save.enemy === 'paint' ? 0x7cff4f : 0xffd27a);
  if (sh) sh.hit(p, part, dmg, dir);
  else if (w.hit) { const n = w.hit === 'ground' ? V3(0, 1, 0) : boxNormal(w.hit, p); if (save.enemy === 'paint') hole(p, n, pick([0xff2bd6, 0x7cff4f, 0x00e5ff, 0xffd23f])); else hole(p, n); for (let i = 0; i < 6; i++) emit(p, n.clone().multiplyScalar(rand(0.5, 1.5)).add(V3(rand(-0.5, 0.5), rand(0, 1), rand(-0.5, 0.5))), w.hit === 'ground' ? 0x8a7a5c : 0xbbb2a0, 0.03, 0.5, -5, 0.05); emit(p, V3(0, 0, 0), 0xffe0a0, 0.06, 0.06); }
}
function boxNormal(c, p){ const d = [[Math.abs(p.x - c.x0), V3(-1, 0, 0)], [Math.abs(p.x - c.x1), V3(1, 0, 0)], [Math.abs(p.y - c.y0), V3(0, -1, 0)], [Math.abs(p.y - c.y1), V3(0, 1, 0)], [Math.abs(p.z - c.z0), V3(0, 0, -1)], [Math.abs(p.z - c.z1), V3(0, 0, 1)]]; d.sort((a, b) => a[0] - b[0]); return d[0][1]; }
function raySphere(o, d, c, r){ const oc = tmpV.copy(o).sub(c), b = oc.dot(d), cc = oc.lengthSq() - r * r, h = b * b - cc; if (h < 0) return Infinity; const t = -b - Math.sqrt(h); return t > 0 ? t : Infinity; }

/* ===================== VASTASED ===================== */
const enemies = [];
function makeEnemyMesh(kind){
  const g = new THREE.Group(); scene.add(g);
  const add = (geo, m, x, y, z) => { const o = new THREE.Mesh(geo, m); o.position.set(x, y, z); o.castShadow = true; g.add(o); return o; };
  if (kind === 'robot') {
    const met = std(0x8fa3b3, { metalness: 0.7, roughness: 0.35 }), eye = new THREE.MeshBasicMaterial({ color: 0xff3030 });
    add(new THREE.BoxGeometry(0.42, 0.55, 0.3), met, 0, 1.15, 0); add(new THREE.BoxGeometry(0.3, 0.26, 0.28), met, 0, 1.6, 0); add(new THREE.BoxGeometry(0.22, 0.05, 0.02), eye, 0, 1.62, -0.145);
    for (const x of [-0.12, 0.12]) add(new THREE.BoxGeometry(0.14, 0.85, 0.16), M.dark, x, 0.43, 0); for (const x of [-0.29, 0.29]) add(new THREE.BoxGeometry(0.11, 0.5, 0.12), met, x, 1.12, -0.05);
  } else {
    const uni = kind === 'paint' ? std(pick([0xd84315, 0x6a1b9a, 0x1565c0])) : kind === 'ally' ? std(0x3d5a80) : M.camo, vest = std(kind === 'paint' ? 0x222222 : kind === 'ally' ? 0x2b3a50 : 0x3b4030, { roughness: 0.9 }), dark = std(0x1d1d1b), skinC = std(pick([0xd2a27a, 0xb9855e, 0x8d5c3e]));
    add(new THREE.CapsuleGeometry(0.18, 0.4, 6, 12), uni, 0, 1.12, 0).scale.set(1.1, 1, 0.8);
    add(new THREE.BoxGeometry(0.42, 0.44, 0.3), vest, 0, 1.18, 0); for (const x of [-0.12, 0, 0.12]) add(new THREE.BoxGeometry(0.09, 0.12, 0.06), vest, x, 1.08, -0.17);
    add(new THREE.BoxGeometry(0.3, 0.35, 0.12), vest, 0, 1.2, 0.2);
    add(new THREE.CylinderGeometry(0.06, 0.07, 0.08, 10), skinC, 0, 1.47, 0);
    const hd = add(new THREE.SphereGeometry(0.115, 16, 12), kind === 'ally' ? skinC : std(0x22231f), 0, 1.6, 0); hd.scale.set(1, 1.1, 1.05);
    if (kind !== 'ally' && kind !== 'paint') { add(new THREE.BoxGeometry(0.16, 0.035, 0.02), skinC, 0, 1.62, -0.105); }
    add(new THREE.SphereGeometry(0.135, 16, 10, 0, Math.PI * 2, 0, Math.PI / 2), kind === 'paint' ? std(0x111111) : kind === 'ally' ? std(0x3a4a2a) : std(0x4d5236, { roughness: 0.8 }), 0, 1.63, 0).scale.set(1, 0.9, 1.08);
    if (kind === 'paint') add(new THREE.BoxGeometry(0.2, 0.08, 0.04), std(0x222222, { metalness: 0.5, roughness: 0.2 }), 0, 1.6, -0.12);
    else add(new THREE.BoxGeometry(0.17, 0.045, 0.03), std(0x111111, { metalness: 0.8, roughness: 0.2 }), 0, 1.71, -0.11);
    for (const x of [-0.11, 0.11]) { add(new THREE.CapsuleGeometry(0.075, 0.32, 4, 8), uni, x, 0.62, 0); add(new THREE.CapsuleGeometry(0.065, 0.3, 4, 8), uni, x, 0.27, 0.02); add(new THREE.BoxGeometry(0.1, 0.1, 0.08), dark, x, 0.45, -0.07); add(new THREE.BoxGeometry(0.11, 0.09, 0.22), dark, x, 0.05, -0.04); }
    add(new THREE.CapsuleGeometry(0.055, 0.22, 4, 8), uni, -0.24, 1.22, -0.12).rotation.set(-1.1, 0, -0.3); add(new THREE.CapsuleGeometry(0.05, 0.22, 4, 8), uni, -0.12, 1.2, -0.32).rotation.set(-1.45, 0, 0.6);
    add(new THREE.CapsuleGeometry(0.055, 0.22, 4, 8), uni, 0.24, 1.18, -0.06).rotation.set(-0.7, 0, 0.3); add(new THREE.CapsuleGeometry(0.05, 0.2, 4, 8), uni, 0.12, 1.2, -0.2).rotation.set(-1.3, 0, -0.5);
  }
  const gun = new THREE.Group(); gun.position.set(0.05, 1.25, -0.35); g.add(gun); { const gm = kind === 'paint' ? std(0x33cc33) : M.gunMetal; const a = (w, h, d, x, y, z, m = gm) => { const b = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), m); b.position.set(x, y, z); b.castShadow = true; gun.add(b); }; a(0.045, 0.06, 0.4, 0, 0, 0); a(0.04, 0.05, 0.2, 0, -0.01, 0.27, kind === 'paint' ? gm : M.fde); a(0.025, 0.1, 0.05, 0, -0.07, 0.03, M.dark); a(0.012, 0.012, 0.25, 0, 0.01, -0.3); }
  const mz = new THREE.Object3D(); mz.position.set(0.05, 1.26, -0.8); g.add(mz);
  return { g, mz };
}
function spawnEnemy(x, z, opts = {}){
  const kind = save.enemy, m = makeEnemyMesh(kind), y = opts.y || 0;
  const e = { m, pos: V3(x, y, z), hp: kind === 'robot' ? 120 : 100, state: opts.alert ? 'engage' : 'idle', alert: !!opts.alert, cover: null, cool: rand(0.5, 2), burst: 0, peek: 0, t: rand(0, 5), dead: false, deadT: 0, yaw: rand(0, 6), static: !!opts.static, patrol: opts.patrol || null, pi: 0, sightT: 0, kind, goal: null, hostage: false };
  m.g.position.copy(e.pos);
  e.sh = { test: (o, d, maxT) => { if (e.dead) return null; const base = e.pos, cr = e.crouchK || 0; const hc = V3(base.x, base.y + 1.6 - cr * 0.5, base.z); const th = raySphere(o, d, hc, 0.15); let tb = Infinity; for (const yy of [1.2, 0.8, 0.4]) tb = Math.min(tb, raySphere(o, d, V3(base.x, base.y + yy - cr * (yy > 0.6 ? 0.4 : 0), base.z), 0.24)); if (th < tb && th < maxT) return { t: th, part: 'head' }; if (tb < maxT) return { t: tb, part: 'body' }; return null; },
    hit: (p, part, dmg, dir) => damageEnemy(e, part === 'head' ? dmg * 3 : dmg, p, dir, part) };
  shootables.push(e.sh); enemies.push(e); return e;
}
function damageEnemy(e, dmg, p, dir, part){
  if (e.dead) return; e.hp -= dmg; S.hit(); hitMarker = 0.15;
  const col = save.enemy === 'robot' ? 0x6ff7ff : save.enemy === 'paint' ? pick([0xff2bd6, 0x7cff4f, 0x00e5ff]) : 0x6b5a4a;
  for (let i = 0; i < (save.enemy === 'robot' ? 10 : 6); i++) emit(p, V3(rand(-1, 1), rand(0, 1.5), rand(-1, 1)).add(dir.clone().multiplyScalar(1.5)), col, 0.035, 0.4, -6);
  if (!e.alert) { e.alert = true; e.state = 'engage'; }
  if (e.hp <= 0) killEnemy(e, dir, part);
  else { e.cover = null; e.flinch = 0.3; }
}
function killEnemy(e, dir, part){
  e.dead = true; e.deadT = 0; e.fallDir = Math.atan2(dir.x, dir.z); shootables.splice(shootables.indexOf(e.sh), 1);
  const pts = part === 'head' ? 150 : 100; if (game) { game.kills++; game.points += pts; } addXp(part === 'head' ? 15 : 10);
  popTxt(e.pos.clone().add(V3(0, 2, 0)), part === 'head' ? '🎯 PEAS +150' : '+100', part === 'head' ? GOLD : '#e8eadf');
  if (save.enemy === 'robot') { for (let i = 0; i < 25; i++) emit(e.pos.clone().add(V3(0, 1.2, 0)), V3(rand(-2, 2), rand(0, 3), rand(-2, 2)), pick([0xffd27a, 0x6ff7ff]), 0.04, 0.7, -6); }
  if (game && game.onKill) game.onKill(e);
  if (game) streakKill();
}
function alertEnemies(p, r){ for (const e of enemies) if (!e.dead && !e.alert && e.pos.distanceTo(p) < r) { e.alert = true; e.state = 'engage'; e.cool = DIFF[save.diff].react + rand(0, 0.5); } }
function moveEnemy(e, gx, gz, sp, dt){
  const dx = gx - e.pos.x, dz = gz - e.pos.z, d = Math.hypot(dx, dz); if (d < 0.15) return true;
  let a = Math.atan2(dx, dz); const step = Math.min(d, sp * dt);
  for (const off of [0, 0.6, -0.6, 1.2, -1.2, 1.8, -1.8]) { const nx = e.pos.x + Math.sin(a + off) * step, nzv = e.pos.z + Math.cos(a + off) * step; if (!blockedAt(nx, nzv, 0.3, e.pos.y + 0.3, e.pos.y + 1.6)) { e.pos.x = nx; e.pos.z = nzv; e.yaw = a + off + Math.PI; return false; } }
  return false;
}
function playerHead(out){ return camera.getWorldPosition(out); }
function updateEnemies(dt){
  const ph = playerHead(V3()), D = DIFF[save.diff];
  for (let i = enemies.length - 1; i >= 0; i--) {
    const e = enemies[i], g = e.m.g; e.t += dt;
    if (e.dead) { e.deadT += dt; const k = Math.min(1, e.deadT * 2.5); g.rotation.set(0, e.yaw, 0); g.rotateOnWorldAxis(V3(Math.cos(e.fallDir), 0, -Math.sin(e.fallDir)), -k * Math.PI / 2); g.position.set(e.pos.x, e.pos.y + Math.sin(k * Math.PI) * 0.15, e.pos.z); if (e.deadT > 12) { g.removeFromParent(); enemies.splice(i, 1); } continue; }
    const eye = V3(e.pos.x, e.pos.y + 1.55 - (e.crouchK || 0) * 0.5, e.pos.z), dist = eye.distanceTo(ph);
    const facing = Math.cos(Math.atan2(ph.x - e.pos.x, ph.z - e.pos.z) - (e.yaw + Math.PI)) > 0.2;
    const vis = mission && mission.night && !P.nvgSeen && !e.alert ? 16 : 70;
    const sees = dist < vis && (e.alert || facing && dist < Math.min(35, vis)) && !P.dead && !(game && game.noTargetPlayer) && los(eye, ph);
    if (sees) { e.sightT += dt; if (!e.alert && e.sightT > D.react) { e.alert = true; e.state = 'engage'; e.cool = D.react; } e.lastSeen = ph.clone(); } else e.sightT = 0;
    e.flinch = Math.max(0, (e.flinch || 0) - dt);
    if (!e.alert) { // patrull
      if (e.patrol) { const wp = e.patrol[e.pi]; if (moveEnemy(e, wp[0], wp[1], 1.2, dt)) e.pi = (e.pi + 1) % e.patrol.length; }
      g.position.copy(e.pos).setY(e.pos.y + Math.abs(Math.sin(e.t * 6)) * 0.03 * (e.patrol ? 1 : 0)); g.rotation.set(0, e.yaw, 0); continue;
    }
    // vali vari
    if (!e.static && (!e.cover || e.coverT <= 0)) {
      const tgt = e.lastSeen || ph; let best = null, bs = 1e9;
      for (const c of COVER) { const d = Math.hypot(c.x - e.pos.x, c.z - e.pos.z); if (d > 18 || Math.abs(e.pos.y) > 0.5) continue; const toP = Math.hypot(tgt.x - c.x, tgt.z - c.z); if (toP < 5) continue; const facingP = ((tgt.x - c.x) * c.nx + (tgt.z - c.z) * c.nz) / toP; if (facingP < 0.3) continue; if (enemies.some(o => o !== e && o.cover === c && !o.dead)) continue; const score = d + toP * (game && game.push ? 0.6 : 0.15) + rand(0, 3); if (score < bs) { bs = score; best = c; } }
      e.cover = best; e.coverT = rand(6, 12);
    }
    e.coverT -= dt;
    let moving = false;
    if (e.cover && !e.static) { moving = !moveEnemy(e, e.cover.x, e.cover.z, e.flinch > 0 ? 1.5 : 3.2, dt); }
    else if (!e.static && !sees && e.lastSeen) { moving = !moveEnemy(e, e.lastSeen.x, e.lastSeen.z, 2.5, dt); }
    // liitlased / veokid sihtmärgiks
    let tgtP = ph, ally = null;
    if (game && game.allies && (!sees || Math.random() < 0.02 || e.allyLock)) { let bd = 60; for (const a of game.allies) { if (a.dead) continue; const ap = a.pos.clone().add(V3(0, a.h || 1.3, 0)), d = ap.distanceTo(eye); if (d < bd && los(eye, ap)) { bd = d; ally = a; tgtP = ap; } } if (ally && !e.alert) { e.alert = true; e.state = 'engage'; } e.allyLock = !!ally && !sees; }
    // piilu ja tulista
    const atCover = e.cover && !moving;
    if (atCover && e.cover.low) { e.peek -= dt; if (e.peek < -2.2) e.peek = rand(1.5, 2.8); e.crouchK = e.peek > 0 ? 0 : 1; } else e.crouchK = 0;
    const face = Math.atan2(tgtP.x - e.pos.x, tgtP.z - e.pos.z) + Math.PI; e.yaw += Math.atan2(Math.sin(face - e.yaw), Math.cos(face - e.yaw)) * Math.min(1, dt * 6);
    e.cool -= dt;
    if ((sees || ally) && e.cool <= 0 && e.crouchK < 0.5 && e.flinch <= 0) { if (e.burst <= 0) e.burst = Math.floor(rand(2, 5)); e.burst--; e.cool = e.burst > 0 ? 0.12 : rand(0.9, 2.0) * (1.4 - save.diff * 0.25); enemyShoot(e, ally ? tgtP : ph, ally ? eye.distanceTo(tgtP) : dist, moving, ally); }
    g.position.copy(e.pos); g.position.y += moving ? Math.abs(Math.sin(e.t * 9)) * 0.04 : 0; g.rotation.set(0, e.yaw, 0);
    g.scale.y = 1 - (e.crouchK || 0) * 0.35;
  }
}
function enemyShoot(e, ph, dist, moving, ally){
  const D = DIFF[save.diff], mz = e.m.mz.getWorldPosition(V3());
  if (ally) { const hitA = Math.random() < 0.35 * clamp(1.4 - dist / 40, 0.3, 1.2); S.enemyShot(mz); tracer(mz, ph.clone().add(hitA ? V3() : V3(rand(-1, 1), rand(-0.5, 0.5), rand(-1, 1))), 0xffe08a); if (hitA) ally.hurt(D.dmg * 0.8); return; }
  let p = D.acc * clamp(1.4 - dist / 40, 0.25, 1.2) * (P.crouch ? 0.6 : 1) * (moving ? 0.6 : 1) * (P.speed > 1 ? 0.7 : 1);
  const hit = Math.random() < p, tgt = ph.clone().add(hit ? V3(rand(-0.1, 0.1), -0.2, rand(-0.1, 0.1)) : V3(rand(-1.2, 1.2), rand(-0.8, 0.8), rand(-1.2, 1.2)));
  if (save.enemy === 'robot') S.laser(mz); else if (save.enemy === 'paint') S.paint(mz); else S.enemyShot(mz);
  tracer(mz, tgt, save.enemy === 'robot' ? 0xff3030 : save.enemy === 'paint' ? 0xff2bd6 : 0xffe08a);
  emit(mz, V3(), 0xffd27a, 0.08, 0.05);
  if (hit) hurtPlayer(D.dmg * rand(0.8, 1.2), mz); else if (tgt.distanceTo(ph) < 1.2) S.whiz(tgt);
}
function hurtPlayer(d, from){
  if (P.dead || (game && game.god)) return; if (P.armor > 0) { const a = Math.min(P.armor, d * 0.6); P.armor -= a; d -= a; }
  P.hp -= d; P.regenT = 4; hurtM.material.opacity = Math.min(0.55, hurtM.material.opacity + 0.25); S.hurt(); for (const h of hands) pulse(h, 0.8, 120);
  if (save.enemy === 'paint') { hurtM.material.color.set(pick([0xff2bd6, 0x7cff4f, 0x00e5ff])); } else hurtM.material.color.set(0x880000);
  if (P.hp <= 0) { P.hp = 0; P.dead = true; endGame(false, save.enemy === 'paint' ? '🎨 Sind värviti üle!' : '❌ Sind tabati'); }
}
// hüpikud
const pops = Array.from({ length: 8 }, () => { const p = new Panel(0.7, 0.18, 300); p.mesh.renderOrder = 57; p.mesh.visible = false; scene.add(p.mesh); return { p, t: 0 }; }); let popI = 0;
function popTxt(pos, s, col){ const o = pops[popI = (popI + 1) % pops.length]; o.p.draw((c, W, H) => { c.font = `900 ${H * 0.55}px ${FONT}`; c.textAlign = 'center'; c.textBaseline = 'middle'; c.lineWidth = 8; c.strokeStyle = '#000'; c.strokeText(s, W / 2, H / 2); c.fillStyle = col; c.fillText(s, W / 2, H / 2); }); o.p.mesh.position.copy(pos); o.t = 1.2; o.p.mesh.visible = true; }
let hitMarker = 0;

/* ===================== MÄRKLAUAD (tiir, tapamaja) ===================== */
const targets = [];
const tgtTex = { enemy: canvasTex(128, 256, (c, W, H) => { c.fillStyle = '#c8b48a'; c.fillRect(0, 0, W, H); c.fillStyle = '#222'; c.beginPath(); c.arc(W / 2, 50, 30, 0, 7); c.fill(); c.fillRect(W / 2 - 40, 85, 80, 130); c.fillRect(W / 2 + 20, 110, 60, 14); c.strokeStyle = '#b00'; c.lineWidth = 4; for (const r of [10, 25, 45]) { c.beginPath(); c.arc(W / 2, 140, r, 0, 7); c.stroke(); } }),
  civ: canvasTex(128, 256, (c, W, H) => { c.fillStyle = '#c8b48a'; c.fillRect(0, 0, W, H); c.fillStyle = '#1f5fbf'; c.beginPath(); c.arc(W / 2, 50, 30, 0, 7); c.fill(); c.fillRect(W / 2 - 40, 85, 80, 130); c.fillRect(W / 2 - 60, 30, 16, 70); c.fillRect(W / 2 + 44, 30, 16, 70); c.fillStyle = '#fff'; c.font = 'bold 26px sans-serif'; c.textAlign = 'center'; c.fillText('ÄRA', W / 2, 150); c.fillText('LASE', W / 2, 180); }),
  bull: canvasTex(256, 256, (c, W, H) => { c.fillStyle = '#f2ead8'; c.fillRect(0, 0, W, H); for (let r = 10; r >= 1; r--) { c.beginPath(); c.arc(W / 2, H / 2, r * 12, 0, 7); c.fillStyle = r <= 3 ? (r % 2 ? '#c62828' : '#111') : r % 2 ? '#fff' : '#222'; c.fill(); } }) };
function makeTarget(kind, x, y, z, opts = {}){
  const g = new THREE.Group(); g.position.set(x, y, z); g.rotation.y = opts.rot || 0; scene.add(g);
  const w = kind === 'bull' ? (opts.size || 0.6) : 0.55, h = kind === 'bull' ? w : 1.6;
  const pivot = new THREE.Group(); g.add(pivot);
  const face = new THREE.Mesh(new THREE.PlaneGeometry(w, h).translate(0, h / 2, 0), new THREE.MeshStandardMaterial({ map: tgtTex[kind], side: THREE.DoubleSide, roughness: 0.9 })); face.castShadow = true; pivot.add(face);
  const post = new THREE.Mesh(new THREE.BoxGeometry(0.05, kind === 'bull' ? y : 0.3, 0.05), M.wood); post.position.y = kind === 'bull' ? -y / 2 : -0.15; g.add(post);
  const t = { kind, g, pivot, w, h, up: opts.up !== false, upK: opts.up !== false ? 1 : 0, alive: true, hits: 0, onHit: opts.onHit, timer: 0 };
  pivot.rotation.x = t.upK ? 0 : -Math.PI / 2;
  t.sh = { test: (o, d, maxT) => { if (t.upK < 0.9) return null; const n = V3(0, 0, 1).applyQuaternion(g.quaternion), p0 = g.position.clone(); const den = n.dot(d); if (Math.abs(den) < 1e-5) return null; const tt = p0.clone().sub(o).dot(n) / den; if (tt < 0 || tt > maxT) return null; const hp = o.clone().addScaledVector(d, tt), loc = g.worldToLocal(hp.clone()); if (Math.abs(loc.x) > w / 2 || loc.y < 0 || loc.y > h) return null; return { t: tt, part: loc }; },
    hit: (p, loc, dmg, dir) => { hole(p, V3(0, 0, 1).applyQuaternion(t.g.quaternion)); t.hits++; S.hit(); hitMarker = 0.12; if (t.onHit) t.onHit(t, loc, p); } };
  shootables.push(t.sh); targets.push(t); return t;
}
function clearTargets(){ for (const t of targets) { t.g.removeFromParent(); shootables.splice(shootables.indexOf(t.sh), 1); } targets.length = 0; }
function updTargets(dt){ for (const t of targets) { const goal = t.up ? 1 : 0; t.upK += (goal - t.upK) * Math.min(1, dt * 10); t.pivot.rotation.x = -(1 - t.upK) * Math.PI / 2; } }

/* ===================== MÄNGU OLEK, REŽIIMID ===================== */
let mode = 'hub', game = null, mission = null;
function clearEnemies(){ clearMission(); for (const e of enemies) e.m.g.removeFromParent(); enemies.length = 0; for (let i = shootables.length - 1; i >= 0; i--) if (!targets.some(t => t.sh === shootables[i])) shootables.splice(i, 1); }
function resetPlayer(x, z, rotY = 0, y = 0){
  P.hp = P.maxHp = 100; P.armor = 0; P.dead = false; P.regenT = 0; P.frags = 3; P.smokes = 2; P.nvg = false; nvgM.visible = false; hurtM.material.opacity = 0;
  for (const k of WORDER) { const g = guns[k]; g.ammo = g.def.mag; g.hasMag = true; g.mag.visible = true; drawAmmo(g); }
  for (const p of [...props]) removeProp(p); for (const s of smokes) s.parts = 0; smokes.length = 0;
  camera.getWorldPosition(head); rig.position.set(x - (renderer.xr.isPresenting ? head.x - rig.position.x : 0), y, z - (renderer.xr.isPresenting ? head.z - rig.position.z : 0));
  if (renderer.xr.isPresenting) { rig.rotation.y = rotY; } else { yaw = rotY; pitch = 0; rig.rotation.y = 0; }
}
function applyLight(){ const N = mission && mission.night ? NIGHT : DAY; const night = N === NIGHT; skyM.visible = !night; scene.environment = night ? null : envTex; scene.background = night ? new THREE.Color(P.nvg ? 0x031a08 : N.sky) : null; stars.visible = night; scene.fog.color.set(P.nvg ? 0x062a10 : N.fog); hemi.intensity = P.nvg ? 1.4 : N.hemi; hemi.color.set(P.nvg ? 0x9cff9c : 0xdfe8f0); sun.intensity = night ? 0.5 : N.sun; sun.color.set(night ? 0x8fa8ff : 0xfff1dc); scene.fog.near = night ? 4 : 60; scene.fog.far = night && !P.nvg ? 55 : night ? 120 : 260; torch.intensity = mission && mission.night && !P.nvg ? 80 : 0; }
function startGame(cfg){
  audioInit(); hideMenus(); clearEnemies(); clearTargets(); mission = cfg.mission || null;
  game = Object.assign({ t: 0, kills: 0, points: 0, shots: 0 }, cfg); mode = 'play';
  resetPlayer(cfg.x, cfg.z, cfg.rot || 0, cfg.y || 0); applyLight();
  if (cfg.weapon) { P.weapon = null; setWeapon(cfg.weapon); }
  if (cfg.init) cfg.init(game);
  hudP.mesh.visible = true; objPos = null;
  msg(cfg.intro || cfg.name, GOLD, 4); if (cfg.say) say(cfg.say);
}
function endGame(win, reason){
  if (mode !== 'play') return; mode = 'end'; const g = game;
  let medal = '', xp = Math.round(g.points / 10);
  if (g.score) { const r = g.score(win); medal = r.medal || ''; xp += r.xp || 0; g.result = r; }
  if (win) xp += g.winXp || 50; addXp(xp);
  const key = g.id; if (win && g.result && g.result.val != null) { const b = save.best[key]; if (b == null || (g.lowerBetter ? g.result.val < b : g.result.val > b)) { save.best[key] = g.result.val; g.rec = true; } persist(); }
  if (mission && mission.id && win) { save.mis[mission.id] = Math.max(save.mis[mission.id] || 0, g.result && g.result.stars || 1); persist(); }
  endP.draw((c, W, H) => {
    bg(c, W, H, win ? '#7cbf5a' : '#c0392b');
    txt(c, win ? '✔ ' + (g.winTitle || 'ÜLESANNE TÄIDETUD') : (reason || '❌ ÜLESANNE EBAÕNNESTUS'), W / 2, H * 0.12, H * 0.08, win ? '#9cff57' : '#ff8a80', '900');
    txt(c, g.name, W / 2, H * 0.22, H * 0.045, '#b9c0a8');
    const lines = g.result && g.result.lines ? g.result.lines : [`Tabamusi: ${g.kills}`, `Punkte: ${g.points}`];
    lines.forEach((l, i) => txt(c, l, W / 2, H * (0.34 + i * 0.08), H * 0.05));
    if (medal) txt(c, medal, W / 2, H * 0.68, H * 0.09, GOLD, '900');
    txt(c, `+${xp} XP • ${RANKS[rankIdx()]}${g.rec ? ' • 🏆 UUS REKORD!' : ''}`, W / 2, H * 0.8, H * 0.045, GOLD);
  });
  placeInFront(endG, 1.3, -0.1); endG.visible = true; hudP.mesh.visible = false; objMark.mesh.visible = false;
  if (document.pointerLockElement) document.exitPointerLock();
  if (win) S.ok(); else S.bad();
}
function toHub(){ hideMenus(); clearEnemies(); clearTargets(); mission = null; game = null; mode = 'hub'; resetPlayer(0, 3, 0); applyLight(); buildHubTargets(); openMenu(); }
function pauseGame(){ mode = 'pause'; if (actx) actx.suspend(); placeInFront(pauseG, 1.0, -0.1); pauseG.visible = true; if (document.pointerLockElement) document.exitPointerLock(); }
function resumeGame(){ pauseG.visible = false; mode = 'play'; if (actx) actx.resume(); }
function interact(){ if (game && game.interact) game.interact(); }

/* --- Hubi vabad märklauad --- */
function buildHubTargets(){ clearTargets(); for (const [x, z, s] of [[-4.5, -10, 0.5], [-1.5, -15, 0.6], [1.5, -25, 0.8], [4.5, -50, 1.2], [0, -100, 2], [-6, -140, 2.5]]) makeTarget('bull', x, 1.2 + s * 0.2, z, { size: s, onHit: (t, loc) => { const r = Math.hypot(loc.x, loc.y - t.h / 2) / (t.w / 2); popTxt(t.g.position.clone().add(V3(0, t.h + 0.4, 0)), r < 0.25 ? '🎯 10' : r < 0.5 ? '8' : r < 0.8 ? '5' : '1', r < 0.25 ? GOLD : '#fff'); } }); }

/* --- 🎯 TIIR: täpsus / kiirus / snaiper --- */
function rangeDrill(type){
  const cfg = { id: 'range_' + type, x: 0, z: 0.8, rot: 0, name: { acc: '🎯 Täpsuslaskmine', speed: '⚡ Kiiruslaskmine', snip: '🔭 Snaiprilaskmine' }[type], weapon: type === 'snip' ? 'sniper' : 'rifle', mission: { weapons: ['sniper', 'rifle', 'pistol', 'shotgun'] },
    intro: { acc: '🎯 10 lasku 3 märklauale – sihi keskele!', speed: '⚡ Lase hüpikmärklaudu kiiresti! 30 sekundit', snip: '🔭 Taba kauged märklauad (100–200 m)' }[type] };
  cfg.init = g => {
    g.sc = 0; g.shotsLeft = type === 'acc' ? 10 : type === 'snip' ? 8 : 999; g.left = type === 'speed' ? 30 : 999;
    const score = (t, loc) => { const r = Math.hypot(loc.x, loc.y - t.h / 2) / (t.w / 2); const v = r < 0.2 ? 10 : r < 0.4 ? 8 : r < 0.6 ? 6 : r < 0.8 ? 4 : 2; g.sc += v; popTxt(t.g.position.clone().add(V3(0, t.h + 0.4, 0)), '+' + v, v >= 8 ? GOLD : '#fff'); };
    if (type === 'acc') for (const [x, z] of [[-3, -10], [0, -25], [3, -50]]) makeTarget('bull', x, 1.2, z, { size: 0.6 + (-z) / 60, onHit: score });
    if (type === 'snip') for (const [x, z] of [[-8, -100], [0, -150], [8, -200], [-4, -130]]) makeTarget('bull', x, 1.5, z, { size: 1.6, onHit: score });
    if (type === 'speed') { for (let i = 0; i < 8; i++) makeTarget('enemy', -7 + i * 2, 0, -12 - (i % 3) * 8, { up: false, onHit: (t) => { if (!t.up) return; t.up = false; g.sc += 1; popTxt(t.g.position.clone().add(V3(0, 2, 0)), '+1', GOLD); S.ok(); } }); g.popT = 0.5; }
  };
  cfg.tick = (g, dt) => {
    if (type === 'speed') { g.left -= dt; g.popT -= dt; if (g.popT <= 0) { const dn = targets.filter(t => !t.up); if (dn.length) { const t = pick(dn); t.up = true; t.timer = 1.6; S.beep(700); } g.popT = rand(0.5, 1.0); } for (const t of targets) if (t.up) { t.timer -= dt; if (t.timer <= 0) t.up = false; } if (g.left <= 0) endGame(true); return `⏱ ${Math.ceil(g.left)}s • 🎯 ${g.sc}`; }
    g.shotsLeft = (type === 'acc' ? 10 : 8) - g.used; if (g.shotsLeft <= 0 && !g.endT) g.endT = 1.5; if (g.endT) { g.endT -= dt; if (g.endT <= 0) endGame(true); }
    return `Lasud ${Math.max(0, g.shotsLeft)} • Punkte ${g.sc}`;
  };
  cfg.score = () => { const g = game, s = g.sc, max = type === 'acc' ? 100 : type === 'snip' ? 80 : 25; const k = s / max; return { val: s, medal: k >= 0.85 ? '🥇 KULD' : k >= 0.65 ? '🥈 HÕBE' : k >= 0.4 ? '🥉 PRONKS' : '', xp: Math.round(s), lines: [`Tulemus: ${s}${type === 'speed' ? ' tabamust' : ' / ' + max}`, `Rekord: ${Math.max(save.best[cfg.id] || 0, s)}`] }; };
  startGame(cfg); game.used = 0; game.onShot = () => game.used++;
}

/* --- 🏃 TAPAMAJA --- */
function killhouse(){
  startGame({ id: 'kh', lowerBetter: true, name: '🏃 Tapamaja', x: 35, z: -6, rot: 0, weapon: 'rifle', intro: '🏃 Mine majja! 🔴 vastased • 🔵 tsiviilid – ÄRA LASE neid. Lõpp põhjauksest.', say: 'Mine, mine, mine!',
    init: g => {
      g.started = false; g.pen = 0; g.civHit = 0;
      const spots = [[34, -13, 0, 'enemy'], [38, -15.5, Math.PI / 2, 'civ'], [45.5, -14, -Math.PI / 2, 'enemy'], [46.5, -18.5, -Math.PI / 2, 'enemy'], [41.5, -19, 0, 'civ'], [33, -22, Math.PI / 2, 'enemy'], [37, -25.5, 0, 'enemy'], [35, -28, 0, 'civ'], [42, -27, 0, 'enemy'], [46.5, -28.5, -Math.PI / 2, 'enemy'], [44.5, -22, Math.PI, 'enemy'], [39, -29, 0, 'enemy']];
      for (const [x, z, r, k] of spots) makeTarget(k, x, 0, z, { rot: r, up: false, onHit: t => { if (!t.alive) return; t.alive = false; t.up = false; if (k === 'civ') { g.civHit++; g.pen += 10; msg('❌ TSIVIILISIK! +10 s', '#ff5252', 1.5); S.bad(); } else { popTxt(t.g.position.clone().add(V3(0, 2, 0)), '✔', '#9cff57'); } } });
      for (const t of targets) t.kindT = t.kind;
    },
    tick: (g, dt) => {
      camera.getWorldPosition(head);
      if (!g.started && head.z < -9.6) { g.started = true; S.beep(1000); }
      if (g.started) g.t2 = (g.t2 || 0) + dt;
      for (const t of targets) if (t.alive && !t.up && head.distanceTo(t.g.position) < 7 && los(head, t.g.position.clone().add(V3(0, 1.2, 0)))) { t.up = true; S.beep(500); }
      if (g.started && head.z < -30.2) { const missed = targets.filter(t => t.kind === 'enemy' && t.alive).length; g.pen += missed * 5; g.missed = missed; endGame(true); }
      return g.started ? `⏱ ${(g.t2 + g.pen).toFixed(1)} s  (trahv +${g.pen})` : 'Mine lõunauksest sisse →';
    },
    score: () => { const g = game, tt = g.t2 + g.pen; return { val: +tt.toFixed(1), medal: tt < 25 ? '🥇 KULD' : tt < 40 ? '🥈 HÕBE' : tt < 60 ? '🥉 PRONKS' : '', xp: Math.max(10, Math.round(80 - tt)), lines: [`Aeg: ${g.t2.toFixed(1)} s + trahv ${g.pen} s = ${tt.toFixed(1)} s`, `Vahele jäänud vastaseid: ${g.missed || 0} • Tsiviile tabatud: ${g.civHit}`, `Rekord: ${save.best.kh != null ? Math.min(save.best.kh, tt).toFixed(1) : tt.toFixed(1)} s`] }; }
  });
}

/* --- 🧟 LAINED --- */
const EDGE = [[CX - 34, CZ - 34], [CX + 34, CZ - 34], [CX - 34, CZ + 34], [CX + 34, CZ + 34], [CX, CZ - 36], [CX, CZ + 36], [CX - 36, CZ], [CX + 36, CZ]];
function waves(){
  startGame({ id: 'waves', name: '🧟 Lainete kaitse', x: CX - 2, z: CZ + 8, rot: 0, weapon: 'rifle', push: true, intro: '🧟 Kaitse positsiooni! Lainete vahel: kast keskel = osta (E / puuduta)', say: 'Valmistu, vastased tulevad',
    init: g => { g.wave = 0; g.breakT = 4; g.toSpawn = 0; g.money = 0; buildBuy(g); },
    onKill: () => { game.money += 100; },
    tick: (g, dt) => {
      const alive = enemies.filter(e => !e.dead).length;
      if (g.toSpawn > 0) { g.spawnT -= dt; if (g.spawnT <= 0 && alive < 8) { const [x, z] = pick(EDGE); const e = spawnEnemy(x + rand(-3, 3), z + rand(-3, 3), { alert: true }); e.lastSeen = V3(CX, 1.6, CZ); g.toSpawn--; g.spawnT = rand(0.8, 2); } }
      else if (alive === 0 && g.breakT <= 0) { g.breakT = 20; msg(`✔ Laine ${g.wave} läbi! Osta varustust – järgmine 20 s pärast`, '#9cff57', 4); S.ok(); addXp(30); buyG.visible = true; }
      if (buyG.visible && !buyMenu.visible) { camera.getWorldPosition(head); if (Math.hypot(head.x - (CX - 3), head.z - (CZ + 3)) < 1.6) { if (!g.nearBuy) { placeInFront(buyMenu, 0.9, -0.15); buyMenu.visible = true; } g.nearBuy = true; } else g.nearBuy = false; }
      if (g.breakT > 0) { g.breakT -= dt; if (g.breakT <= 0) { g.wave++; g.toSpawn = 3 + g.wave * 2; g.spawnT = 1; buyG.visible = false; msg(`⚠️ LAINE ${g.wave}!`, '#ff8a80', 2.5); say('Kontakt!'); } return `☕ Paus ${Math.ceil(g.breakT)} s • 💰 ${g.money}`; }
      return `🌊 Laine ${g.wave} • vastaseid ${alive + g.toSpawn} • 💰 ${g.money}`;
    },
    score: () => ({ val: game.wave, xp: game.wave * 20, lines: [`Jõudsid laineni: ${game.wave}`, `Tabamusi: ${game.kills}`, `Rekord: laine ${Math.max(save.best.waves || 0, game.wave)}`] }),
    interact: () => { if (buyG.visible) { placeInFront(buyMenu, 0.9, -0.15); buyMenu.visible = true; } }
  });
}
const buyG = new THREE.Group(); scene.add(buyG); buyG.visible = false;
{ const c = B(CX - 3, 0, CZ + 3, 1.2, 0.9, 0.8, std(0x4f5a34), false, buyG); const lab = new Panel(0.8, 0.25, 300); lab.draw((c, W, H) => { bg(c, W, H, GOLD); txt(c, '🛒 VARUSTUS (E)', W / 2, H / 2, H * 0.45, GOLD); }); lab.mesh.position.set(CX - 3, 1.4, CZ + 3); buyG.add(lab.mesh); }
const buyMenu = new THREE.Group(); scene.add(buyMenu); buyMenu.visible = false;
function buildBuy(g){
  while (buyMenu.children.length) { const m = buyMenu.children.pop(); const i = buttons.findIndex(b => b.p.mesh === m); if (i >= 0) buttons.splice(i, 1); }
  const p = new Panel(1.0, 0.75); p.draw((c, W, H) => { bg(c, W, H, GOLD); txt(c, '🛒 Varustus', W / 2, H * 0.1, H * 0.08, GOLD); }); buyMenu.add(p.mesh);
  const items = [['🧱 Soomusvest +50', 400, () => P.armor = 50], ['❤️ Ravi täis', 300, () => P.hp = P.maxHp], ['💣 +2 granaati', 300, () => P.frags += 2], ['🌫️ +2 suitsu', 200, () => P.smokes += 2], ['🔫 Haavlipüss', 600, () => { mission = { weapons: ['shotgun', 'sniper', 'rifle', 'pistol'] }; setWeapon('shotgun'); }], ['🔭 Snaipripüss', 800, () => { mission = { weapons: ['shotgun', 'sniper', 'rifle', 'pistol'] }; setWeapon('sniper'); }]];
  items.forEach(([n, cost, fn], i) => btn(buyMenu, 0.45, 0.11, -0.24 + (i % 2) * 0.48, 0.17 - Math.floor(i / 2) * 0.14, `${n} • 💰${cost}`, () => { if (game.money < cost) { S.bad(); msg('💰 Raha ei jätku', '#ff8a80', 1.2); return; } game.money -= cost; fn(); S.ok(); msg(`✔ Ostetud: ${n}`, '#9cff57', 1.2); }));
  btn(buyMenu, 0.4, 0.1, 0, -0.3, '✖ Sulge', () => buyMenu.visible = false);
}


/* ===================== MISSIOONID ===================== */
const missionG = new THREE.Group(); scene.add(missionG);
let extraMeshes = [];
let COL_BASE = 0;
function clearMission(){ if (COL_BASE) COL.length = COL_BASE; while (missionG.children.length) missionG.children[0].removeFromParent(); for (const m of extraMeshes) m.removeFromParent(); extraMeshes = []; wireG.visible = false; heliSup.g.visible = false; }
function heliModel(){
  const g = new THREE.Group(), add = (geo, m, x, y, z) => { const o = new THREE.Mesh(geo, m); o.position.set(x, y, z); o.castShadow = true; g.add(o); return o; };
  const body = std(0x3e4a33, { roughness: 0.6, metalness: 0.3 });
  add(new THREE.CapsuleGeometry(1.1, 2.6, 8, 16).rotateX(Math.PI / 2), body, 0, 0, 0); add(new THREE.BoxGeometry(0.4, 0.5, 5), body, 0, 0.3, 3.6); add(new THREE.BoxGeometry(0.1, 1.2, 0.8), body, 0, 0.9, 6);
  add(new THREE.SphereGeometry(0.9, 16, 10, 0, Math.PI * 2, 0, Math.PI / 2).rotateX(-Math.PI / 2), std(0x223344, { roughness: 0.1, metalness: 0.8 }), 0, 0.15, -1.7);
  for (const x of [-0.9, 0.9]) add(new THREE.BoxGeometry(0.08, 0.08, 3), M.metal, x, -1.3, 0);
  const rotor = new THREE.Group(); rotor.position.set(0, 1.3, 0); g.add(rotor); for (let i = 0; i < 4; i++) { const b = new THREE.Mesh(new THREE.BoxGeometry(0.25, 0.04, 7), M.dark); b.rotation.y = i * Math.PI / 4 * 2; rotor.add(b); }
  const tail = new THREE.Group(); tail.position.set(0.25, 0.9, 6); g.add(tail); for (let i = 0; i < 2; i++) { const b = new THREE.Mesh(new THREE.BoxGeometry(0.03, 1.4, 0.12), M.dark); b.rotation.x = i * Math.PI / 2; tail.add(b); }
  scene.add(g); return { g, rotor, tail };
}
function jetModel(){ const g = new THREE.Group(), m = std(0x6d747a, { metalness: 0.6, roughness: 0.4 }); const f = new THREE.Mesh(new THREE.CylinderGeometry(0.5, 0.7, 12, 10).rotateX(Math.PI / 2), m); g.add(f); const n = new THREE.Mesh(new THREE.ConeGeometry(0.5, 3, 10).rotateX(-Math.PI / 2), m); n.position.z = -7.5; g.add(n); const w = new THREE.Mesh(new THREE.BoxGeometry(11, 0.15, 3.5), m); w.position.z = 0.5; g.add(w); const t = new THREE.Mesh(new THREE.BoxGeometry(4, 0.12, 1.6), m); t.position.z = 5.5; g.add(t); const v = new THREE.Mesh(new THREE.BoxGeometry(0.12, 2.2, 2), m); v.position.set(0, 1.1, 5.4); g.add(v); const fl = new THREE.Mesh(new THREE.SphereGeometry(0.5, 8, 6), new THREE.MeshBasicMaterial({ color: 0xffa040 })); fl.position.z = 6.2; g.add(fl); g.visible = false; scene.add(g); return g; }
const heliSup = heliModel(); heliSup.g.visible = false;
const jet = jetModel(); let jetT = 0, jetFrom = V3(), jetTo = V3(), jetBomb = null;
function flyJet(from, to, bombAt){ jet.visible = true; jetT = 0; jetFrom.copy(from); jetTo.copy(to); jet.position.copy(from); jet.lookAt(to); jet.rotateY(Math.PI); jetBomb = bombAt; if (actx) { const t = actx.currentTime, o = out(from.clone().lerp(to, 0.4)); nz(t, 4, 500, 0.6, 'lowpass', o); tn(t, 220, 90, 4, 0.15, 'sawtooth', o); } }
function updJet(dt){ if (!jet.visible) return; jetT += dt / 4; jet.position.lerpVectors(jetFrom, jetTo, jetT); if (jetBomb && jetT > 0.45) { const b = jetBomb; jetBomb = null; b.forEach((p, i) => setTimeout(() => explode(p, true), 500 + i * 220)); } if (jetT >= 1) jet.visible = false; }
const intel = () => { const g = new THREE.Group(); const t = new THREE.Mesh(new THREE.BoxGeometry(1.2, 0.05, 0.7), M.wood); t.position.y = 0.9; g.add(t); for (const [x, z] of [[-0.55, -0.3], [0.55, -0.3], [-0.55, 0.3], [0.55, 0.3]]) { const l = new THREE.Mesh(new THREE.BoxGeometry(0.05, 0.9, 0.05), M.wood); l.position.set(x, 0.45, z); g.add(l); } const lb = new THREE.Mesh(new THREE.BoxGeometry(0.34, 0.02, 0.24), M.dark); lb.position.y = 0.94; g.add(lb); const sc = new THREE.Mesh(new THREE.PlaneGeometry(0.32, 0.2), new THREE.MeshBasicMaterial({ color: 0x55ff99 })); sc.position.set(0, 1.04, -0.11); sc.rotation.x = -0.25; g.add(sc); const sb = new THREE.Mesh(new THREE.BoxGeometry(0.34, 0.22, 0.01), M.dark); sb.position.set(0, 1.04, -0.125); sb.rotation.x = -0.25; g.add(sb); missionG.add(g); return g; };
function makeAlly(x, z, kind = 'ally'){ const m = makeEnemyMesh(kind); extraMeshes.push(m.g); const a = { m, pos: V3(x, 0, z), hp: 100, dead: false, t: rand(0, 5), yaw: Math.PI, h: 1.3 }; a.hurt = d => { if (a.dead) return; a.hp -= d; for (let i = 0; i < 4; i++) emit(a.pos.clone().add(V3(0, 1.2, 0)), V3(rand(-1, 1), rand(0, 1), rand(-1, 1)), 0x6b5a4a, 0.03, 0.4, -5); if (a.hp <= 0) { a.dead = true; msg('❗ Liitlane langes!', '#ff8a80', 2); } }; m.g.position.copy(a.pos); return a; }
function updAlly(a, dt, goal, speed){ a.t += dt; const g = a.m.g; if (a.dead) { g.rotation.set(-Math.PI / 2 * Math.min(1, (a.deadT = (a.deadT || 0) + dt) * 2), a.yaw, 0, 'YXZ'); g.position.y = 0.2; return false; } let moving = false; if (goal) { const dx = goal.x - a.pos.x, dz = goal.z - a.pos.z, d = Math.hypot(dx, dz); if (d > 0.3) { const st = Math.min(d, speed * dt); a.pos.x += dx / d * st; a.pos.z += dz / d * st; a.yaw = Math.atan2(dx, dz) + Math.PI; moving = true; } } g.position.copy(a.pos); g.position.y += moving ? Math.abs(Math.sin(a.t * 8)) * 0.04 : 0; g.rotation.set(0, a.yaw, 0); return moving; }
// objektiivid
function objTick(g, dt){
  if (g.oi == null) { g.oi = 0; }
  const o = g.objs && g.objs[g.oi]; if (!o) return '';
  camera.getWorldPosition(head); const p = typeof o.pos === 'function' ? o.pos() : o.pos;
  if (p) { objPos = p; objLabel = o.icon || '◆'; }
  let done = false;
  if (o.done) done = o.done(g);
  else if (o.near && p) { const d = Math.hypot(head.x - p.x, head.z - p.z); if (d < o.near) { if (o.hold) { g.holdT = (g.holdT || 0) + dt; if (g.holdT >= o.hold) done = true; } else done = true; } else g.holdT = 0; }
  if (done) { g.holdT = 0; if (o.onDone) o.onDone(g); g.oi++; S.ok(); const nx = g.objs[g.oi]; if (nx) { msg('✔ ' + (typeof nx.t === 'function' ? nx.t(g) : nx.t), GOLD, 3); if (nx.onStart) nx.onStart(g); } else endGame(true); }
  const t = typeof o.t === 'function' ? o.t(g) : o.t;
  return o.hold && g.holdT ? `${t} ${'▮'.repeat(Math.round(g.holdT / o.hold * 8))}${'▯'.repeat(8 - Math.round(g.holdT / o.hold * 8))}` : t;
}
function missionScore(win){ const g = game, par = g.par || 240, st = !win ? 0 : 1 + (P.hp > 50 ? 1 : 0) + (g.t < par ? 1 : 0); return { stars: st, medal: '⭐'.repeat(st) + '☆'.repeat(3 - st), xp: win ? 100 : 20, val: win ? Math.round(g.t) : null, lines: [`Aeg: ${Math.floor(g.t / 60)}:${String(Math.floor(g.t % 60)).padStart(2, '0')}`, `Vastaseid tabatud: ${g.kills} • Elud: ${Math.round(P.hp)}%`] }; }
const alive = () => enemies.filter(e => !e.dead).length;
function squad(list){ return list.map(([x, z, p]) => spawnEnemy(x, z, { patrol: p })); }
const MISSIONS = [
  { id: 'm1', n: '🌙 Öine haarang', d: 'öövaatlus • luure' }, { id: 'm2', n: '🧍 Pantvang', d: 'päästa ja too välja' }, { id: 'm3', n: '🔭 Snaipri kate', d: 'kaitse salka katuselt' }, { id: 'm4', n: '🚚 Konvoi', d: 'kaitse veokeid' }, { id: 'm5', n: '💣 Pomm', d: 'desarmeeri ajaga' }
];
function startMission(i){
  const base = { lowerBetter: true, tick: (g, dt) => { const s = g.extraTick ? g.extraTick(g, dt) : ''; return s || objTick(g, dt); }, score: missionScore, winTitle: 'MISSIOON TÄIDETUD' };
  if (i === 0) startGame(Object.assign({}, base, { id: 'm1', mission: { id: 'm1', night: true, weapons: ['rifle', 'pistol', 'shotgun'] }, name: '🌙 Öine haarang', x: CX, z: CZ + 33, rot: 0, weapon: 'rifle', par: 300,
    intro: '🌙 Vajuta X (arvutis N) – öövaatlus! Kõrvalda laohoone valvurid.', say: 'Öine haarang. Lülita öövaatlus sisse.',
    init: g => { g.n = 10; squad([[CX + 12, CZ + 10], [CX + 16, CZ + 14], [CX + 10, CZ + 4, [[CX + 10, CZ + 4], [CX + 2, CZ + 4], [CX + 2, CZ + 20]]], [CX + 22, CZ + 4], [CX + 6, CZ + 19], [CX + 23, CZ + 20], [CX - 2, CZ + 12, [[CX - 2, CZ + 12], [CX - 2, CZ - 6]]], [CX + 5, CZ - 5], [CX - 9, CZ + 2], [CX + 27, CZ + 11]]);
      const lp = intel(); lp.position.set(CX + 14, 0, CZ + 12); const hl = heliModel(); hl.g.visible = false; extraMeshes.push(hl.g); g.heli = hl;
      g.objs = [{ t: g2 => `Kõrvalda valvurid (${g2.n - alive()}/${g2.n})`, pos: () => V3(CX + 14, 2, CZ + 12), done: () => alive() === 0, icon: '⚔' },
        { t: 'Võta luureandmed – seisa sülearvuti juures', pos: V3(CX + 14, 1.2, CZ + 12), near: 1.5, hold: 2.5, icon: '💻', onDone: () => { lp.children[lp.children.length - 2].material.color.set(0x222222); say('Luureandmed käes. Mine kopteri juurde.'); } },
        { t: 'Mine kopteri juurde! 🚁', pos: V3(CX, 0, CZ - 28), near: 4, icon: '🚁', onStart: g2 => { g2.heli.g.visible = true; g2.heli.g.position.set(CX - 60, 25, CZ - 60); g2.heliIn = 0; for (let k = 0; k < 4; k++) { const e = spawnEnemy(EDGE[k][0], EDGE[k][1], { alert: true }); e.lastSeen = V3(CX, 1.6, CZ - 20); } } }];
      g.extraTick = (g2, dt) => { if (g2.heli.g.visible) { g2.heliIn = Math.min(1, g2.heliIn + dt / 10); const p = V3(CX - 60, 25, CZ - 60).lerp(V3(CX, 1.6, CZ - 28), g2.heliIn); g2.heli.g.position.copy(p); g2.heli.g.rotation.y = -Math.PI * 0.75; g2.heli.rotor.rotation.y += dt * 25; g2.heli.tail.rotation.x += dt * 30; } return ''; };
    } }));
  if (i === 1) startGame(Object.assign({}, base, { id: 'm2', mission: { id: 'm2', weapons: ['rifle', 'pistol', 'shotgun'] }, name: '🧍 Pantvangi päästmine', x: CX + 30, z: CZ - 28, rot: Math.PI / 4, weapon: 'rifle', par: 280,
    intro: '🧍 Pantvang on loodeosa majas. ÄRA TABA PANTVANGI!', say: 'Leia pantvang ja too ta välja.',
    init: g => { squad([[CX - 12, CZ + 9], [CX - 16, CZ + 15], [CX - 8, CZ + 20], [CX - 20, CZ + 5], [CX - 4, CZ + 6, [[CX - 4, CZ + 6], [CX - 4, CZ - 10]]], [CX + 6, CZ - 8], [CX - 24, CZ + 20], [CX + 2, CZ + 14], [CX - 10, CZ - 2], [CX - 28, CZ + 10]]);
      const h = makeAlly(CX - 15.5, CZ + 15, 'ally'); h.m.g.scale.set(1, 0.7, 1); h.yaw = 0; g.hostage = h; h.free = false;
      h.sh = { test: (o, d, maxT) => { const c = h.pos.clone().add(V3(0, h.free ? 1.1 : 0.8, 0)); const t = raySphere(o, d, c, 0.3); return t < maxT ? { t, part: 'b' } : null; }, hit: () => { if (game === g) { g.hostageHit = true; endGame(false, '❌ Tabasid pantvangi!'); } } }; shootables.push(h.sh); g.cleanupSh = h.sh;
      g.objs = [{ t: 'Leia pantvang ja vabasta (seisa tema juures)', pos: () => h.pos.clone().add(V3(0, 1.4, 0)), near: 1.4, hold: 2, icon: '🧍', onDone: () => { h.free = true; h.m.g.scale.set(1, 1, 1); say('Tule minuga!'); } },
        { t: 'Vii pantvang lõunaväravasse 🚩', pos: V3(CX, 0, CZ + 36), done: () => Math.hypot(h.pos.x - CX, h.pos.z - (CZ + 36)) < 4, icon: '🚩', onStart: () => { for (let k = 0; k < 3; k++) { const e = spawnEnemy(EDGE[k + 4][0], EDGE[k + 4][1], { alert: true }); e.lastSeen = V3(CX - 15, 1.6, CZ + 15); } } }];
      g.extraTick = (g2, dt) => { if (h.free) { camera.getWorldPosition(head); const d = Math.hypot(head.x - h.pos.x, head.z - h.pos.z); updAlly(h, dt, d > 1.8 ? V3(head.x, 0, head.z) : null, 2.6); } else { h.m.g.position.copy(h.pos); } return ''; };
    } }));
  if (i === 2) startGame(Object.assign({}, base, { id: 'm3', mission: { id: 'm3', weapons: ['sniper', 'rifle', 'pistol'] }, name: '🔭 Snaipri kate', x: CX + 14, z: CZ + 14, y: 6.2, rot: 0, weapon: 'sniper', par: 200, bounds: [CX + 8.6, CX + 19.4, CZ + 7.6, CZ + 16.4],
    intro: '🔭 Kaitse salka katuselt! Salk liigub mööda teed põhja poole.', say: 'Snaipri kate. Kaitse salka.',
    init: g => { g.allies = [0, 1, 2].map(k => makeAlly(CX - 1 + k * 0.9, CZ + 36 + k * 1.5)); g.spawned = 0; g.total = 12; g.wp = CZ + 36;
      g.objs = [{ t: g2 => `Kaitse salka: ${g2.allies.filter(a => !a.dead).length}/3 elus • vastaseid ${g2.total - g2.spawned + alive()}`, pos: () => g.allies.find(a => !a.dead) ? g.allies.find(a => !a.dead).pos.clone().add(V3(0, 2.2, 0)) : null, done: g2 => g2.wp <= CZ - 32 && alive() === 0 && g2.spawned >= g2.total, icon: '🛡' }];
      g.extraTick = (g2, dt) => {
        const live = g2.allies.filter(a => !a.dead); if (!live.length) { endGame(false, '❌ Salk langes'); return ''; }
        const contact = enemies.some(e => !e.dead && e.alert && e.pos.distanceTo(live[0].pos) < 28);
        if (!contact) g2.wp = Math.max(CZ - 34, g2.wp - dt * 1.4);
        g2.allies.forEach((a, k) => { updAlly(a, dt, contact ? null : V3(CX - 1 + k * 0.9, 0, g2.wp + k * 1.5), 1.6); if (!a.dead && contact) { a.shootT = (a.shootT || rand(1, 3)) - dt; if (a.shootT <= 0) { a.shootT = rand(1.5, 3); const tgt = enemies.find(e => !e.dead && e.pos.distanceTo(a.pos) < 30); if (tgt) { const mz = a.m.mz.getWorldPosition(V3()); tracer(mz, tgt.pos.clone().add(V3(0, 1.3, 0)), 0x9fd7ff); S.enemyShot(mz); if (Math.random() < 0.25) damageEnemy(tgt, 40, tgt.pos.clone().add(V3(0, 1.2, 0)), V3(0, 0, 1), 'body'); } } } });
        if (g2.spawned < g2.total && (alive() < 3 || g2.t2 === undefined)) { g2.t2 = 0; const n = Math.min(3, g2.total - g2.spawned); for (let k = 0; k < n; k++) { const z = g2.wp - rand(14, 26), x = CX + pick([-1, 1]) * rand(6, 18); const e = spawnEnemy(x, z, { alert: true }); e.lastSeen = live[0].pos.clone(); g2.spawned++; } }
        return '';
      };
    } }));
  if (i === 3) startGame(Object.assign({}, base, { id: 'm4', mission: { id: 'm4', weapons: ['rifle', 'pistol', 'shotgun', 'sniper'] }, name: '🚚 Konvoi kaitse', x: CX, z: CZ - 22, rot: Math.PI, weapon: 'rifle', par: 260,
    intro: '🚚 Veokid on rünnaku all! Kaitse neid 3 lainet.', say: 'Kaitse konvoid!',
    init: g => { g.allies = []; g.truckHp = 900; for (const x of [CX - 14, CX - 6, CX + 6]) { const t = truck(x, CZ - 27.5, Math.PI / 2, missionG); const a = { pos: V3(x, 0, CZ - 27.5), h: 1.4, dead: false, hurt: d => { g.truckHp -= d; if (Math.random() < 0.3) emit(a.pos.clone().add(V3(rand(-2, 2), 2, rand(-1, 1))), V3(0, 1, 0), 0x333333, 0.8, 3, 0, 0.5); } }; g.allies.push(a); }
      g.wave = 0; g.toSpawn = 0; g.breakT = 3;
      g.objs = [{ t: g2 => `Laine ${g2.wave}/3 • veokid ${Math.max(0, Math.round(g2.truckHp / 9))}% • vastaseid ${alive() + g2.toSpawn}`, pos: V3(CX - 4, 3, CZ - 27.5), done: g2 => g2.wave >= 3 && g2.toSpawn === 0 && alive() === 0, icon: '🚚' }];
      g.extraTick = (g2, dt) => { if (g2.truckHp <= 0) { endGame(false, '❌ Konvoi hävitati'); return ''; }
        if (g2.toSpawn > 0) { g2.spT = (g2.spT || 0) - dt; if (g2.spT <= 0 && alive() < 8) { const [x, z] = pick(EDGE.filter(e => e[1] > CZ - 30)); const e = spawnEnemy(x + rand(-3, 3), z, { alert: true }); e.lastSeen = V3(CX, 1.4, CZ - 27); g2.toSpawn--; g2.spT = rand(0.8, 1.8); } }
        else if (alive() === 0 && g2.wave < 3) { g2.breakT -= dt; if (g2.breakT <= 0) { g2.wave++; g2.toSpawn = 4 + g2.wave * 2; g2.breakT = 8; msg(`⚠️ LAINE ${g2.wave}/3!`, '#ff8a80', 2.5); say('Kontakt põhjast!'); } }
        return ''; };
    } }));
  if (i === 4) { const COLS = [['PUNANE', 0xd32f2f], ['SININE', 0x1976d2], ['ROHELINE', 0x388e3c], ['KOLLANE', 0xfbc02d]], ok = Math.floor(rand(0, 4));
    startGame(Object.assign({}, base, { id: 'm5', mission: { id: 'm5', weapons: ['rifle', 'pistol', 'shotgun'] }, name: '💣 Pommi desarmeerimine', x: CX - 30, z: CZ + 30, rot: -Math.PI / 4, weapon: 'rifle', par: 120,
      intro: `💣 Luure: lõika ${COLS[ok][0]} juhe! Pomm on idaosa majas. Aega 2:30.`, say: `Lõika ${COLS[ok][0].toLowerCase()} juhe.`,
      init: g => { squad([[CX + 12, CZ - 10], [CX + 10, CZ - 14], [CX + 15, CZ - 5], [CX + 4, CZ - 12], [CX + 20, CZ - 18, [[CX + 20, CZ - 18], [CX + 20, CZ - 2]]], [CX + 2, CZ - 2], [CX - 6, CZ + 10], [CX + 24, CZ - 8], [CX - 14, CZ + 2]]);
        const bomb = new THREE.Group(); bomb.position.set(CX + 12, 0, CZ - 12.5); missionG.add(bomb); const cr = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.8, 0.8), M.wood); cr.position.y = 0.4; bomb.add(cr); const bx = new THREE.Mesh(new THREE.BoxGeometry(0.5, 0.25, 0.35), M.dark); bx.position.y = 0.93; bomb.add(bx); COLS.forEach(([n, c], k) => { const w = new THREE.Mesh(new THREE.CylinderGeometry(0.008, 0.008, 0.4, 6).rotateZ(Math.PI / 2), std(c)); w.position.set(0, 0.97, -0.1 + k * 0.06); bomb.add(w); }); const led = new THREE.Mesh(new THREE.SphereGeometry(0.03, 8, 6), new THREE.MeshBasicMaterial({ color: 0xff0000 })); led.position.set(0.18, 1.08, 0); bomb.add(led); g.led = led; g.left = 150;
        g.objs = [{ t: g2 => `Leia pomm (lõika ${COLS[ok][0]})`, pos: V3(CX + 12, 1.3, CZ - 12.5), near: 1.4, icon: '💣', onDone: () => { openWires(COLS, ok); } }, { t: 'Lõika õige juhe!', pos: V3(CX + 12, 1.3, CZ - 12.5), done: () => false, icon: '✂' }];
        g.extraTick = (g2, dt) => { g2.left -= dt; g2.led.visible = Math.sin(g2.t * (g2.left < 30 ? 20 : 6)) > 0; if (Math.floor(g2.left) !== g2.lastBeep) { g2.lastBeep = Math.floor(g2.left); if (g2.left < 30) S.beep(1400); } if (g2.left <= 0) { wireG.visible = false; explode(V3(CX + 12, 1, CZ - 12.5), true, 14); endGame(false, '💥 Pomm plahvatas!'); return ''; } const tt = `⏱ ${Math.floor(g2.left / 60)}:${String(Math.floor(g2.left % 60)).padStart(2, '0')} • `; return tt + objTick(g2, dt); };
      } })); }
}
// juhtmepaneel
const wireG = new THREE.Group(); scene.add(wireG); wireG.visible = false; const wireP = new Panel(0.9, 0.5); wireG.add(wireP.mesh); const wireBtns = [];
function openWires(COLS, ok){
  for (const b of wireBtns) { b.p.mesh.removeFromParent(); buttons.splice(buttons.indexOf(b), 1); } wireBtns.length = 0;
  wireP.draw((c, W, H) => { bg(c, W, H, '#ff5252'); txt(c, '✂ LÕIKA JUHE', W / 2, H * 0.15, H * 0.12, '#ff8a80', '900'); });
  COLS.forEach(([n, col], k) => { const b = btn(wireG, 0.19, 0.16, -0.31 + k * 0.205, -0.05, n, () => { wireG.visible = false; if (k === ok) { S.ok(); say('Pomm desarmeeritud'); msg('✔ Pomm desarmeeritud!', '#9cff57', 3); game.oi = 99; endGame(true); } else { explode(V3(CX + 12, 1, CZ - 12.5), true, 14); endGame(false, '💥 Vale juhe!'); } }, '#' + new THREE.Color(col).getHexString()); wireBtns.push(b); });
  placeInFront(wireG, 0.7, -0.25); wireG.visible = true; if (document.pointerLockElement) document.exitPointerLock();
}
/* ===================== TAPMISSEERIAD (üllatus) ===================== */
const STREAKS = [[3, 'uav', '📡 DROON'], [5, 'air', '✈️ ÕHULÖÖK'], [8, 'heli', '🚁 KOPTERI TOETUS']];
let uavT = 0, heliT = 0, heliShootT = 0;
const uavMarks = Array.from({ length: 16 }, () => { const sp = new THREE.Sprite(new THREE.SpriteMaterial({ map: canvasTex(64, 64, (c) => { c.fillStyle = '#ff2020'; c.beginPath(); c.moveTo(32, 60); c.lineTo(8, 16); c.lineTo(56, 16); c.closePath(); c.fill(); c.strokeStyle = '#fff'; c.lineWidth = 4; c.stroke(); }), depthTest: false, fog: false })); sp.scale.set(0.5, 0.5, 1); sp.renderOrder = 70; sp.visible = false; scene.add(sp); return sp; });
function streakKill(){ game.streak = (game.streak || 0) + 1; for (const [n, k, name] of STREAKS) if (game.streak === n) { game.ready = k; S.ok(); msg(`${name} VALMIS! Vajuta X (arvutis Q)`, GOLD, 4); say(name.replace(/[^A-Za-zÕÄÖÜõäöü ]/g, '') + ' valmis'); } }
function useStreak(){
  if (!game || !game.ready || mode !== 'play') return false; const k = game.ready; game.ready = null;
  if (k === 'uav') { uavT = 25; msg('📡 Droon üleval – vastased on näha punasega', GOLD, 3); say('Droon üleval'); }
  if (k === 'air') { const g = guns[P.weapon]; let o, d; if (renderer.xr.isPresenting) { o = g.mz.getWorldPosition(V3()); d = V3(0, 0, -1).applyQuaternion(g.g.getWorldQuaternion(new THREE.Quaternion())); } else { o = camera.getWorldPosition(V3()); d = camera.getWorldDirection(V3()); } const t = Math.min(castWorld(o, d, 160).t, 160); const p = o.clone().addScaledVector(d, t); p.y = 0; camera.getWorldPosition(head); if (Math.hypot(p.x - head.x, p.z - head.z) < 12) { msg('⚠️ Liiga lähedal! Siht kaugemale', '#ff8a80', 2); game.ready = 'air'; return true; } const dir = V3(p.x - head.x, 0, p.z - head.z).normalize(), side = V3(-dir.z, 0, dir.x); const bombs = [-8, -4, 0, 4, 8].map(k2 => p.clone().addScaledVector(dir, k2).addScaledVector(side, rand(-1.5, 1.5))); flyJet(p.clone().addScaledVector(dir, -250).add(V3(0, 60, 0)), p.clone().addScaledVector(dir, 250).add(V3(0, 60, 0)), bombs); msg('✈️ Õhulöök tulekul!', GOLD, 2.5); say('Õhulöök tulekul'); }
  if (k === 'heli') { heliT = 30; heliSup.g.visible = true; msg('🚁 Kopter toetab!', GOLD, 3); say('Kopter saabub'); }
  return true;
}
function updStreaks(dt){
  uavT -= dt; let k = 0; for (const e of enemies) { if (uavT > 0 && !e.dead && k < uavMarks.length) { uavMarks[k].position.copy(e.pos).add(V3(0, 2.2, 0)); uavMarks[k].visible = true; k++; } } for (; k < uavMarks.length; k++) uavMarks[k].visible = false;
  if (heliT > 0 && mode === 'play') { heliT -= dt; camera.getWorldPosition(head); const a = performance.now() / 4000; heliSup.g.position.set(head.x + Math.cos(a) * 20, 22, head.z + Math.sin(a) * 20); heliSup.g.rotation.y = -a; heliSup.rotor.rotation.y += dt * 25; heliSup.tail.rotation.x += dt * 30; heliShootT -= dt;
    if (heliShootT <= 0) { heliShootT = 1.3; const hp = heliSup.g.position.clone().add(V3(0, -1.5, 0)); const tg = enemies.find(e => !e.dead && los(hp, e.pos.clone().add(V3(0, 1.2, 0)))); if (tg) { for (let i = 0; i < 5; i++) setTimeout(() => { tracer(hp, tg.pos.clone().add(V3(rand(-0.5, 0.5), rand(0.5, 1.5), rand(-0.5, 0.5))), 0xffd27a); S.enemyShot(hp); }, i * 70); setTimeout(() => damageEnemy(tg, 250, tg.pos.clone().add(V3(0, 1.2, 0)), V3(0, -1, 0), 'body'), 350); } }
    if (heliT <= 0) heliSup.g.visible = false; }
}
/* ===================== ATMOSFÄÄR ===================== */
let ambT = 8, jetAmbT = 30;
function updAtmos(dt){
  const night = mission && mission.night;
  for (const c of SMOKE_COL) if (Math.random() < dt * 6) emit(c.clone().add(V3(rand(-4, 4), rand(0, 6), rand(-4, 4))), V3(rand(0.5, 1.5), rand(2, 4), rand(-0.5, 0.5)), pick([0x6a6662, 0x7a7570, 0x5a5652]), rand(6, 10), rand(8, 12), 0, 2.5);
  for (const f of fires) { f.l.intensity = 2.5 + Math.sin(performance.now() / 70 + f.p.x) * 0.8 + rand(-0.4, 0.4); if (Math.random() < dt * 30) emit(f.p.clone().add(V3(rand(-0.6, 0.6), 0, rand(-1, 1))), V3(rand(-0.2, 0.2), rand(1, 2), rand(-0.2, 0.2)), pick([0xff7a2a, 0xffb347, 0xff5020]), rand(0.3, 0.6), rand(0.4, 0.8), 0, -0.4); if (Math.random() < dt * 6) emit(f.p.clone().add(V3(0, 1.2, 0)), V3(rand(-0.3, 0.3), 1.5, rand(-0.3, 0.3)), 0x55514d, 1.2, 4, 0, 0.8); }
  ambT -= dt; if (ambT <= 0) { ambT = rand(8, 22); camera.getWorldPosition(head); const a = rand(0, 6.28), p = head.clone().add(V3(Math.cos(a) * 220, 0, Math.sin(a) * 220)); S.boom(p); if (Math.random() < 0.5) for (let i = 0; i < 6; i++) setTimeout(() => S.enemyShot(head.clone().add(V3(Math.cos(a + 0.3) * 150, 0, Math.sin(a + 0.3) * 150))), i * 110); }
  jetAmbT -= dt; if (jetAmbT <= 0 && !night && !jet.visible) { jetAmbT = rand(40, 80); camera.getWorldPosition(head); const a = rand(0, 6.28), d = V3(Math.cos(a), 0, Math.sin(a)); flyJet(head.clone().addScaledVector(d, -400).add(V3(0, 90, 0)), head.clone().addScaledVector(d, 400).add(V3(0, 90, 0)), null); }
  updJet(dt);
  // objektiivi marker
  if (objPos && mode === 'play') { objMark.mesh.visible = true; camera.getWorldPosition(head); const d = head.distanceTo(objPos); objMark.mesh.position.copy(objPos).add(V3(0, 1.0, 0)); objMark.mesh.lookAt(head); objMark.mesh.scale.setScalar(clamp(d / 8, 0.6, 6)); if (Math.floor(d) !== objMark.lastD) { objMark.lastD = Math.floor(d); objMark.draw((c, W, H) => { txt(c, objLabel, W / 2, H * 0.35, H * 0.5, GOLD); txt(c, `${Math.round(d)} m`, W / 2, H * 0.82, H * 0.28, '#fff'); }); } } else objMark.mesh.visible = false;
  if (P.nvg) P.nvgSeen = false;
}

/* ===================== MENÜÜD ===================== */
const menuG = new THREE.Group(); scene.add(menuG);
const menuP = new Panel(1.7, 1.3, 450); menuG.add(menuP.mesh);
function drawMenu(){ menuP.draw((c, W, H) => { bg(c, W, H, OD); txt(c, '🎖️ EESLIIN VR', W / 2, H * 0.07, H * 0.065, GOLD, '900'); const r = rankIdx(), nx = rankXp(r + 1), cur = rankXp(r); txt(c, `${RANKS[r]} • ${save.xp} XP${r < RANKS.length - 1 ? ` (järgmine: ${RANKS[r + 1]} @ ${nx})` : ''}`, W / 2, H * 0.135, H * 0.032, '#b9c0a8'); c.fillStyle = '#333'; c.fillRect(W * 0.25, H * 0.16, W * 0.5, H * 0.012); c.fillStyle = GOLD; c.fillRect(W * 0.25, H * 0.16, W * 0.5 * clamp((save.xp - cur) / Math.max(1, nx - cur), 0, 1), H * 0.012);
  txt(c, 'TREENING', W * 0.2, H * 0.22, H * 0.03, GOLD); txt(c, 'LAHING', W * 0.5, H * 0.22, H * 0.03, GOLD); txt(c, 'SEADED', W * 0.8, H * 0.22, H * 0.03, GOLD); txt(c, '🎬 MISSIOONID', W / 2, H * 0.665, H * 0.032, GOLD); txt(c, 'Tapmisseeria: 3 = 📡 droon • 5 = ✈️ õhulöök • 8 = 🚁 kopter (X / Q) • Y/Esc = menüü', W / 2, H * 0.955, H * 0.024, '#8a917a', 'normal'); }); }
const MB = {};
MB.acc = btn(menuG, 0.46, 0.12, -0.56, 0.27, '🎯 Täpsus', () => rangeDrill('acc')); MB.speed = btn(menuG, 0.46, 0.12, -0.56, 0.13, '⚡ Kiirus', () => rangeDrill('speed')); MB.snip = btn(menuG, 0.46, 0.12, -0.56, -0.01, '🔭 Snaiper', () => rangeDrill('snip')); MB.kh = btn(menuG, 0.46, 0.12, -0.56, -0.15, '🏃 Tapamaja', () => killhouse());
MB.waves = btn(menuG, 0.46, 0.14, 0, 0.26, '🧟 Lained', () => waves(), '#c0392b'); MB.free = btn(menuG, 0.46, 0.12, 0, 0.1, '🔫 Vaba harjutus', () => { hideMenus(); msg('🔫 Lase märklaudu! Y/Esc = menüü', GOLD, 2.5); }); MB.arm = btn(menuG, 0.46, 0.12, 0, -0.05, '🛠️ Relvad', () => openArmory());
MISSIONS.forEach((m, i) => { MB['m' + i] = btn(menuG, 0.3, 0.13, -0.64 + i * 0.32, -0.3, m.n, () => startMission(i), '#c9a227'); });
MB.enemy = btn(menuG, 0.46, 0.11, 0.56, 0.27, '', () => { save.enemy = { soldier: 'robot', robot: 'paint', paint: 'soldier' }[save.enemy]; persist(); refreshMenu(); });
MB.diff = btn(menuG, 0.46, 0.11, 0.56, 0.13, '', () => { save.diff = (save.diff + 1) % 3; persist(); refreshMenu(); });
MB.reload = btn(menuG, 0.46, 0.11, 0.56, -0.01, '', () => { save.reload = save.reload === 'manual' ? 'auto' : 'manual'; persist(); refreshMenu(); });
MB.move = btn(menuG, 0.46, 0.11, 0.56, -0.15, '', () => { save.move = save.move === 'smooth' ? 'teleport' : 'smooth'; persist(); refreshMenu(); });
function refreshMenu(){ MB.enemy.label = { soldier: '🪖 Vastased: Sõdurid', robot: '🤖 Vastased: Robotid', paint: '🎨 Vastased: Värvikuulid' }[save.enemy]; MB.diff.label = 'Raskus: ' + DIFF[save.diff].n; MB.reload.label = save.reload === 'manual' ? '🔄 Laadimine: käsitsi' : '🔄 Laadimine: automaatne'; MB.move.label = save.move === 'smooth' ? '🚶 Liikumine: sujuv' : '✨ Liikumine: teleport';
  MB.acc.sub = save.best.range_acc ? `rekord ${save.best.range_acc}/100` : 'staatilised märklauad'; MB.speed.sub = save.best.range_speed ? `rekord ${save.best.range_speed}` : 'hüpikmärklauad'; MB.snip.sub = rankIdx() >= 0 ? (save.best.range_snip ? `rekord ${save.best.range_snip}/80` : '100–200 m') : ''; MB.kh.sub = save.best.kh != null ? `rekord ${save.best.kh} s` : 'vastased + tsiviilid'; MB.waves.sub = save.best.waves ? `rekord: laine ${save.best.waves}` : 'kaitse positsiooni';
  MISSIONS.forEach((m, i) => { const st = save.mis[m.id] || 0; MB['m' + i].sub = st ? '⭐'.repeat(st) + '☆'.repeat(3 - st) : m.d; });
  Object.values(MB).forEach(b => b.draw()); drawMenu(); }
function openMenu(){ hideMenus(); refreshMenu(); placeInFront(menuG, 1.35, -0.1); menuG.visible = true; if (document.pointerLockElement) document.exitPointerLock(); }
// relvaruum
const armG = new THREE.Group(); scene.add(armG); armG.visible = false;
const armP = new Panel(1.3, 0.9); armG.add(armP.mesh);
const armBtns = WORDER.map((k, i) => btn(armG, 0.55, 0.13, -0.3 + (i % 2) * 0.6, 0.15 - Math.floor(i / 2) * 0.17, WEAPONS[k].n, () => { if (WEAPONS[k].rank > rankIdx()) { S.bad(); return; } setWeapon(k); openArmory(); }));
btn(armG, 0.4, 0.11, 0, -0.33, '✖ Tagasi', () => openMenu());
function openArmory(){ hideMenus(); armP.draw((c, W, H) => { bg(c, W, H, GOLD); txt(c, '🛠️ RELVAD', W / 2, H * 0.09, H * 0.08, GOLD, '900'); txt(c, 'Uued relvad avanevad auastmetega • kinnitused (sihikud, summuti) tulekul', W / 2, H * 0.19, H * 0.035, '#b9c0a8', 'normal'); }); armBtns.forEach((b, i) => { const k = WORDER[i], d = WEAPONS[k]; b.lock = d.rank > rankIdx(); b.sel = P.weapon === k; b.sub = b.lock ? `🔒 ${RANKS[d.rank]}` : `salv ${d.mag} • ${d.auto ? 'automaatne' : 'üksik'}`; b.draw(); }); placeInFront(armG, 1.2, -0.1); armG.visible = true; }
const pauseG = new THREE.Group(); scene.add(pauseG); pauseG.visible = false;
{ const p = new Panel(0.8, 0.6); p.draw((c, W, H) => { bg(c, W, H); txt(c, '⏸ PAUS', W / 2, H * 0.15, H * 0.12, GOLD, '900'); }); pauseG.add(p.mesh); btn(pauseG, 0.55, 0.11, 0, 0.05, '▶ Jätka', () => resumeGame()); btn(pauseG, 0.55, 0.11, 0, -0.1, '🔁 Alusta uuesti', () => { const c = game.restart; c(); }); btn(pauseG, 0.55, 0.11, 0, -0.24, '🏠 Baasi', () => { if (actx) actx.resume(); toHub(); }); }
const endG = new THREE.Group(); scene.add(endG); endG.visible = false;
const endP = new Panel(1.3, 0.9); endG.add(endP.mesh);
btn(endG, 0.45, 0.12, -0.25, -0.33, '🔁 Uuesti', () => game.restart()); btn(endG, 0.45, 0.12, 0.25, -0.33, '🏠 Baasi', () => toHub());
function hideMenus(){ menuG.visible = endG.visible = pauseG.visible = armG.visible = buyMenu.visible = false; }
// taaskäivitused
const _rd = rangeDrill, _kh = killhouse, _wv = waves;
const wrapStart = startGame; // game.restart määratakse allpool
const origStart = startGame;

/* ===================== GRANAADID ===================== */
function throwProp(type, pos, vel){ if (type === 'frag' && P.frags <= 0 || type === 'smoke' && P.smokes <= 0) { msg(type === 'frag' ? '💣 Granaadid otsas' : '🌫️ Suitsud otsas', '#ff8a80', 1.2); return null; } if (type === 'frag') P.frags--; else P.smokes--; const p = newProp(type, pos); p.vel.copy(vel); p.live = true; S.click(); return p; }
function deskThrow(type){ camera.getWorldDirection(tmpD); camera.getWorldPosition(tmpV); throwProp(type, tmpV.clone().addScaledVector(tmpD, 0.4), tmpD.clone().multiplyScalar(13).add(V3(0, 3.5, 0))); }
function explode(p, big = false, R = 7){
  S.boom(p); flash.position.copy(p).add(V3(0, 0.5, 0)); flash.intensity = 30; setTimeout(() => flash.intensity = 0, 120);
  for (let i = 0; i < 60; i++) emit(p, V3(rand(-1, 1), rand(0.3, 1.5), rand(-1, 1)).normalize().multiplyScalar(rand(3, 9)), pick([0xffb74d, 0xff7043, 0xffe082]), 0.12, rand(0.3, 0.6), -4);
  for (let i = 0; i < 30; i++) emit(p.clone().add(V3(0, 0.5, 0)), V3(rand(-1.5, 1.5), rand(0.5, 2.5), rand(-1.5, 1.5)), pick([0x555555, 0x777777, 0x3a3a3a]), 0.6, rand(1.5, 3), 0, 0.8);
  if (big) { for (let i = 0; i < 40; i++) emit(p.clone().add(V3(0, 1, 0)), V3(rand(-2, 2), rand(1, 4), rand(-2, 2)), pick([0x222222, 0x444444]), 2, rand(3, 5), 0, 1.2); }
  for (const e of enemies) { if (e.dead) continue; const d = e.pos.distanceTo(p); if (d < R && los(p.clone().add(V3(0, 0.4, 0)), e.pos.clone().add(V3(0, 1, 0)))) damageEnemy(e, 220 * (1 - d / (R + 0.5)), e.pos.clone().add(V3(0, 1, 0)), e.pos.clone().sub(p).normalize(), 'body'); }
  camera.getWorldPosition(head); const dp = head.distanceTo(p); if (dp < 6) hurtPlayer(70 * (1 - dp / 6), p); if (dp < 15) for (const h of hands) pulse(h, 1, 300);
  for (const t of targets) if (t.up && t.g.position.distanceTo(p) < 5) t.sh.hit(t.g.position.clone().add(V3(0, 1, 0)), V3(0, 1, 0), 0, V3());
}
function updProps(dt){
  for (let i = props.length - 1; i >= 0; i--) {
    const p = props[i]; p.age += dt;
    if (p.held) continue;
    if (p.live && !p.rest) { p.vel.y -= 9.8 * dt; const o = p.g.position.clone(), step = p.vel.clone().multiplyScalar(dt), L = step.length(); if (L > 0) { const d = step.clone().divideScalar(L), w = castWorld(o, d, L + 0.04); if (w.t < L + 0.04) { const n = w.hit === 'ground' ? V3(0, 1, 0) : boxNormal(w.hit, o.clone().addScaledVector(d, w.t)); p.g.position.addScaledVector(d, Math.max(0, w.t - 0.04)); p.vel.reflect(n).multiplyScalar(0.35); if (p.vel.length() < 0.6 && n.y > 0.5) p.rest = true; } else p.g.position.add(step); } p.g.rotation.x += dt * 6; }
    if ((p.type === 'frag' || p.type === 'smoke') && p.live) { p.fuse -= dt; if (p.fuse <= 0) { if (p.type === 'frag') { explode(p.g.position.clone()); removeProp(p); continue; } else if (!p.smoked) { p.smoked = true; smokes.push({ p: p.g.position.clone().add(V3(0, 1.2, 0)), r: 0.5, t: 0, prop: p }); S.hiss(p.g.position); } } }
    if (p.type === 'mag' && p.empty && p.age > 6) { removeProp(p); continue; }
  }
  for (let i = smokes.length - 1; i >= 0; i--) { const s = smokes[i]; s.t += dt; s.r = Math.min(4, 0.5 + s.t * 1.2); if (s.t < 18 && Math.random() < dt * 25) emit(s.p.clone().add(V3(rand(-s.r, s.r) * 0.6, rand(-1, 1), rand(-s.r, s.r) * 0.6)), V3(rand(-0.2, 0.2), 0.1, rand(-0.2, 0.2)), pick([0xdddddd, 0xcfcfcf, 0xe8e8e8]), 1.4, 4, 0, 0.3); if (s.t > 22) { if (s.prop) removeProp(s.prop); smokes.splice(i, 1); } }
}

/* ===================== VR SISEND ===================== */
const btnPrev = {};
function edge(k, v, fn){ if (v && !btnPrev[k]) fn(); btnPrev[k] = v; }
let snapReady = true, tpAim = null;
const tpMark = new THREE.Mesh(new THREE.RingGeometry(0.25, 0.32, 32).rotateX(-Math.PI / 2), new THREE.MeshBasicMaterial({ color: GOLD, transparent: true, opacity: 0.8 })); tpMark.visible = false; scene.add(tpMark);
function readXR(dt){
  const s = renderer.xr.getSession(); if (!s) return; let mx = 0, my = 0;
  for (const src of s.inputSources) {
    const gp = src.gamepad, h = hands.find(x => x.side === src.handedness); if (!gp || !h || h.isHand) continue;
    const ax = gp.axes, b = gp.buttons, sx = ax.length >= 4 ? ax[2] : ax[0] || 0, sy = ax.length >= 4 ? ax[3] : ax[1] || 0;
    h.trig = b[0] && b[0].value > 0.5; const gr = b[1] && b[1].value > 0.5;
    if (src.handedness === 'left') {
      mx = sx; my = sy;
      edge('lgrip', gr, () => leftGrab(h)); if (!gr && btnPrev.lgripHeld) leftRelease(h); btnPrev.lgripHeld = gr;
      edge('X', b[4] && b[4].pressed, () => { if (!useStreak()) toggleNvg(); }); edge('Y', b[5] && b[5].pressed, () => { if (mode === 'play') pauseGame(); else if (mode === 'pause') resumeGame(); else if (mode === 'hub') { if (menuG.visible) hideMenus(); else openMenu(); } });
    } else {
      if (Math.abs(sx) > 0.7 && snapReady) { snapReady = false; turn(sx > 0 ? -Math.PI / 6 : Math.PI / 6); } else if (Math.abs(sx) < 0.3) snapReady = true;
      edge('A', b[4] && b[4].pressed, () => { if (mode === 'play' || mode === 'hub') { if (save.reload === 'auto') startReload(); else ejectMag(); } }); edge('B', b[5] && b[5].pressed, () => { if (mode === 'play' || mode === 'hub') cycleWeapon(); });
      if (save.move === 'teleport') { if (sy < -0.6) tpAim = h; else if (tpAim === h && Math.abs(sy) < 0.2) { doTeleport(); tpAim = null; } }
    }
  }
  if (save.move === 'smooth' && (mode === 'play' || mode === 'hub')) movePlayer(mx, my, dt, 2.2);
}
function turn(a){ camera.getWorldPosition(head); rig.rotation.y += a; const after = camera.getWorldPosition(V3()); rig.position.x += head.x - after.x; rig.position.z += head.z - after.z; }
function movePlayer(mx, my, dt, sp){
  const len = Math.hypot(mx, my); P.speed = 0; if (len < 0.15) return;
  camera.getWorldDirection(tmpD); tmpD.y = 0; tmpD.normalize(); const rx = -tmpD.z, rz = tmpD.x;
  const vx = (tmpD.x * -my + rx * mx) / Math.max(1, len) * sp * dt, vz = (tmpD.z * -my + rz * mx) / Math.max(1, len) * sp * dt;
  camera.getWorldPosition(head); const fy = rig.position.y;
  if (!blockedAt(head.x + vx, head.z + vz, 0.3, fy + 0.3, fy + 1.6) && inBounds(head.x + vx, head.z + vz)) { rig.position.x += vx; rig.position.z += vz; }
  else if (!blockedAt(head.x + vx, head.z, 0.3, fy + 0.3, fy + 1.6) && inBounds(head.x + vx, head.z)) rig.position.x += vx;
  else if (!blockedAt(head.x, head.z + vz, 0.3, fy + 0.3, fy + 1.6) && inBounds(head.x, head.z + vz)) rig.position.z += vz;
  P.speed = sp;
}
const inBounds = (x, z) => !(game && game.bounds) || (x > game.bounds[0] && x < game.bounds[1] && z > game.bounds[2] && z < game.bounds[3]);
function doTeleport(){ if (!tpMark.visible) return; camera.getWorldPosition(head); rig.position.x += tpMark.position.x - head.x; rig.position.z += tpMark.position.z - head.z; tpMark.visible = false; }
function updTeleport(){ tpMark.visible = false; if (!tpAim) return; tpAim.ctrl.getWorldPosition(tmpV); tpAim.ctrl.getWorldQuaternion(tmpQ); const d = V3(0, 0, -1).applyQuaternion(tmpQ); let p = tmpV.clone(), v = d.multiplyScalar(8); for (let i = 0; i < 40; i++) { v.y -= 9.8 * 0.05; const n = p.clone().addScaledVector(v, 0.05); if (n.y <= rig.position.y) { if (!blockedAt(n.x, n.z, 0.3, rig.position.y + 0.3, rig.position.y + 1.6) && inBounds(n.x, n.z)) { tpMark.position.set(n.x, rig.position.y + 0.02, n.z); tpMark.visible = true; } return; } p = n; } }
// vasak käsi: salv (vöölt) või granaat (rinnalt)
function bodyPoint(dx, dy, dz){ camera.getWorldPosition(head); camera.getWorldDirection(tmpD); tmpD.y = 0; tmpD.normalize(); const r = V3(-tmpD.z, 0, tmpD.x); return head.clone().addScaledVector(r, dx).add(V3(0, dy, 0)).addScaledVector(tmpD, dz); }
function leftGrab(h){
  if (h.held || mode === 'end') return; const hp = handPos(h, V3()), g = guns[P.weapon];
  // kahe käega hoie
  if (g.fore && g.g.visible) { const f = g.g.localToWorld(g.fore.clone()); if (hp.distanceTo(f) < 0.16) { P.twoHand = true; return; } }
  const hip = bodyPoint(-0.2, -0.65, 0.05), chestL = bodyPoint(-0.1, -0.35, 0.18), chestR = bodyPoint(0.1, -0.35, 0.18);
  if (hp.distanceTo(hip) < 0.22) { const p = newProp('mag', hp); p.held = h; h.held = p; (h.isHand ? h.hHold : h.holder).attach(p.g); p.g.position.set(0, 0, 0); S.click(); pulse(h, 0.3, 30); return; }
  if (hp.distanceTo(chestL) < 0.18 || hp.distanceTo(chestR) < 0.18) { const type = hp.distanceTo(chestL) < hp.distanceTo(chestR) ? 'frag' : 'smoke'; if (type === 'frag' ? P.frags <= 0 : P.smokes <= 0) { msg(type === 'frag' ? '💣 Granaadid otsas' : '🌫️ Suitsud otsas', '#ff8a80', 1.2); return; } if (type === 'frag') P.frags--; else P.smokes--; const p = newProp(type, hp); p.held = h; h.held = p; (h.isHand ? h.hHold : h.holder).attach(p.g); p.g.position.set(0, 0, 0); S.click(); pulse(h, 0.4, 40); msg(type === 'frag' ? '💣 Viska! (lase külgnupp lahti)' : '🌫️ Viska! (lase lahti)', GOLD, 1.2); return; }
  if (hip && hp.distanceTo(hip) < 0.4) msg('💡 Salv: vasak käsi vöö vasakule poolele', '#b9c0a8', 1.5);
}
function leftRelease(h){
  P.twoHand = false; const p = h.held; if (!p) return; h.held = null; p.held = null; scene.attach(p.g);
  if (p.type === 'mag') { p.live = true; p.empty = true; p.vel.copy(h.vel); return; }
  p.live = true; p.vel.copy(h.vel).multiplyScalar(1.4); if (p.type === 'frag') p.fuse = 3.0;
}
function checkMagInsert(){ const h = L(); if (!h || !h.held || h.held.type !== 'mag') return; const g = guns[P.weapon]; if (g.hasMag && g.ammo === g.def.mag) return; const mw = g.g.localToWorld(g.magwell.clone()); if (handPos(h, V3()).distanceTo(mw) < 0.13 || h.held.g.getWorldPosition(V3()).distanceTo(mw) < 0.11) { const p = h.held; h.held = null; removeProp(p); if (g.hasMag) { g.hasMag = false; } insertMag(); pulse(h, 0.6, 50); pulse(R(), 0.6, 50); } }
// päris käte žestid
const J = (h, n) => h.hand.joints && h.hand.joints[n];
function handGestures(h){
  const w = J(h, 'wrist'), palm = J(h, 'middle-finger-metacarpal'), it = J(h, 'index-finger-tip'), th = J(h, 'thumb-tip'), mt = J(h, 'middle-finger-tip'), rt = J(h, 'ring-finger-tip'); if (!w || !palm || !it || !th || !mt || !rt) return;
  const Pp = palm.getWorldPosition(V3()), I = it.getWorldPosition(V3()), T = th.getWorldPosition(V3());
  const curl = (I.distanceTo(Pp) + mt.getWorldPosition(V3()).distanceTo(Pp) + rt.getWorldPosition(V3()).distanceTo(Pp)) / 3, pinch = I.distanceTo(T);
  h.hHold.position.copy(Pp); w.getWorldQuaternion(h.hHold.quaternion);
  if (h.side === 'right') { h.trig = pinch < 0.03 && !h.hover; h.selTrig = false; const mm = J(h, 'middle-finger-metacarpal'), im = J(h, 'index-finger-metacarpal'), pm = J(h, 'pinky-finger-metacarpal'); if (mm && im && pm) { const W0 = w.getWorldPosition(V3()), fwd = mm.getWorldPosition(V3()).sub(W0).normalize(), side = im.getWorldPosition(V3()).sub(pm.getWorldPosition(V3())).normalize(); const up = side.sub(fwd.clone().multiplyScalar(side.dot(fwd))).normalize(), zAx = fwd.clone().negate(), xAx = up.clone().cross(zAx); h.gunQ = new THREE.Quaternion().setFromRotationMatrix(new THREE.Matrix4().makeBasis(xAx, up, zAx)); } }
  else { const closed = curl < 0.07, open = curl > 0.085; if (!h.gClosed && closed) { h.gClosed = true; leftGrab(h); } else if (h.gClosed && open) { h.gClosed = false; leftRelease(h); } }
}

/* ===================== RELVA ASEND JA SIHIK ===================== */
function updateWeapon(dt){
  const g = guns[P.weapon]; if (!g) return; g.g.visible = true;
  if (renderer.xr.isPresenting) {
    const h = R(); if (!h) { g.g.visible = false; return; }
    if (h.isHand) { g.g.position.copy(h.hHold.position); if (h.gunQ) g.g.quaternion.copy(h.gunQ); else g.g.quaternion.copy(h.hHold.quaternion); g.g.translateY(0.04); }
    else { h.holder.getWorldPosition(g.g.position); h.holder.getWorldQuaternion(g.g.quaternion); }
    const lh = L(); if (P.twoHand && lh && g.fore) { const lp = handPos(lh, V3()); const dir = lp.sub(g.g.position).normalize(); const up = V3(0, 1, 0).applyQuaternion(g.g.quaternion); const m = new THREE.Matrix4().lookAt(V3(), dir.clone().negate().negate(), up); g.g.quaternion.setFromRotationMatrix(new THREE.Matrix4().lookAt(g.g.position, g.g.position.clone().add(dir), up)); }
    if (P.kick) { g.g.rotateX(P.kick * 2); g.g.translateZ(P.kick * 0.5); P.kick *= Math.exp(-dt * 20); if (P.kick < 0.002) P.kick = 0; }
  } else {
    const ads = rmb && mode !== 'end'; P.ads = ads; const targ = ads ? V3(0, -g.sightY - 0.0, -0.3) : V3(0.17, -0.2, -0.38);
    camera.updateMatrixWorld(); const lp = targ; if (P.reloadT > 0) lp.y -= 0.15; g.g.position.copy(camera.localToWorld(lp.clone())); g.g.quaternion.copy(camera.getWorldQuaternion(new THREE.Quaternion()));
    if (P.reloadT > 0) g.g.rotateX(0.5);
    const fov = ads ? (g.def.scope ? 75 / g.def.scope : 55) : 75; if (Math.abs(camera.fov - fov) > 0.1) { camera.fov += (fov - camera.fov) * Math.min(1, dt * 12); camera.updateProjectionMatrix(); }
    document.getElementById('cross').style.display = ads && !g.def.scope ? 'none' : '';
  }
  // laser (VR sihtimisabi)
  laserDot.visible = laserBeam.visible = false;
  if (renderer.xr.isPresenting && g.kind !== 'sniper' && mode !== 'end') { const o = g.mz.getWorldPosition(V3()), d = V3(0, 0, -1).applyQuaternion(g.g.getWorldQuaternion(new THREE.Quaternion())); let t = castWorld(o, d, 120).t; for (const sh of shootables) { const r = sh.test(o, d, t); if (r && r.t < t) t = r.t; } if (t < 120) { laserDot.position.copy(o).addScaledVector(d, t - 0.02); laserDot.scale.setScalar(Math.max(1, t * 0.08)); laserDot.visible = true; } laserBeam.position.copy(o); laserBeam.lookAt(o.clone().add(d)); laserBeam.scale.set(1, 1, Math.min(t, 120)); laserBeam.rotation.z = 0; laserBeam.visible = true; }
  // laadimine (taimer)
  if (P.reloadT > 0) { P.reloadT -= dt; if (P.reloadT <= 0) { insertMag(); } }
  // snaipri sihiku pilt
  if (g.lens && renderer.xr.isPresenting) { g.mz.getWorldPosition(scopeCam.position); scopeCam.position.copy(g.g.localToWorld(V3(0, 0.115, 0.05))); g.g.getWorldQuaternion(scopeCam.quaternion); const xrOn = renderer.xr.enabled; renderer.xr.enabled = false; g.g.visible = false; renderer.setRenderTarget(scopeRT); renderer.render(scene, scopeCam); renderer.setRenderTarget(null); renderer.xr.enabled = xrOn; g.g.visible = true; }
}
// sihiku ristnitkud tekstuuri peale
scopeRT.texture.colorSpace = THREE.SRGBColorSpace;

/* ===================== HUD ===================== */
let hudKey = '';
function drawHud(line){
  const g = guns[P.weapon], key = `${Math.ceil(P.hp)}|${Math.ceil(P.armor)}|${g.ammo}|${P.frags}|${P.smokes}|${line}|${P.weapon}`; if (key === hudKey) return; hudKey = key;
  hudP.draw((c, W, H) => { roundRect(c, 4, 4, W - 8, H - 8, 20); c.fillStyle = 'rgba(10,14,10,0.6)'; c.fill(); const hp = P.hp / P.maxHp; c.fillStyle = '#333'; c.fillRect(W * 0.04, H * 0.18, W * 0.3, H * 0.2); c.fillStyle = hp > 0.5 ? '#7cbf5a' : hp > 0.25 ? '#e0b030' : '#e53935'; c.fillRect(W * 0.04, H * 0.18, W * 0.3 * hp, H * 0.2); if (P.armor > 0) { c.fillStyle = '#64b5f6'; c.fillRect(W * 0.04, H * 0.42, W * 0.3 * P.armor / 50, H * 0.08); }
    txt(c, `${WEAPONS[P.weapon].n} ${g.hasMag ? g.ammo : '–'}/${g.def.mag}`, W * 0.55, H * 0.3, H * 0.22, g.ammo ? '#e8eadf' : '#ff5252'); txt(c, `💣${P.frags} 🌫️${P.smokes}`, W * 0.88, H * 0.3, H * 0.2); txt(c, line || '', W / 2, H * 0.72, H * 0.2, GOLD); });
}

/* ===================== SESSIOON JA TSÜKKEL ===================== */
const info = document.getElementById('info');
renderer.xr.addEventListener('sessionstart', () => { info.style.display = 'none'; document.getElementById('cross').style.display = 'none'; audioInit(); camera.fov = 75; setTimeout(() => { if (menuG.visible) placeInFront(menuG, 1.35, -0.1); if (endG.visible) placeInFront(endG, 1.3, -0.1); }, 400); hudP.mesh.position.set(0, -0.35, -0.9); const s = renderer.xr.getSession(); s.addEventListener('visibilitychange', () => { if (s.visibilityState !== 'visible' && mode === 'play') pauseGame(); }); });
renderer.xr.addEventListener('sessionend', () => { info.style.display = ''; document.getElementById('cross').style.display = ''; camera.position.set(0, 1.65, 0); rig.rotation.y = 0; });
// taaskäivitamise sidumine
const RESTART = { m1: () => startMission(0), m2: () => startMission(1), m3: () => startMission(2), m4: () => startMission(3), m5: () => startMission(4), range_acc: () => rangeDrill('acc'), range_speed: () => rangeDrill('speed'), range_snip: () => rangeDrill('snip'), kh: killhouse, waves };
const _startGame = startGame;
function bindRestart(){ if (game && !game.restart) game.restart = RESTART[game.id] || toHub; }
COL_BASE = COL.length; toHub(); P.weapon = null; setWeapon('rifle');
const clock = new THREE.Clock();
renderer.setAnimationLoop(() => {
  const dt = Math.min(clock.getDelta(), 0.05), T = clock.elapsedTime; bindRestart();
  if (renderer.xr.isPresenting) readXR(dt);
  else {
    let mx = (keys.KeyD ? 1 : 0) - (keys.KeyA ? 1 : 0), my = (keys.KeyS ? 1 : 0) - (keys.KeyW ? 1 : 0);
    P.crouch = !!(keys.KeyC || keys.ControlLeft); camera.position.y += ((P.crouch ? 1.0 : 1.65) - camera.position.y) * Math.min(1, dt * 10);
    camera.rotation.set(pitch, yaw, 0, 'YXZ');
    if (mode === 'play' || mode === 'hub') movePlayer(mx, my, dt, keys.ShiftLeft ? 4.5 : P.ads ? 1.3 : 2.6);
    if ((mode === 'play' || mode === 'hub') && mouseDown && !menuOpen() && document.pointerLockElement) { const w = WEAPONS[P.weapon]; if (w.auto || !P.semiLock) { fire(); P.semiLock = true; } } if (!mouseDown) P.semiLock = false;
    // menüü hiirega
    deskHover = null; if (menuOpen()) { const rc = new THREE.Raycaster(); rc.setFromCamera(mouseNDC, camera); const hit = rc.intersectObjects(buttons.filter(b => isShown(b.p.mesh)).map(b => b.p.mesh), false)[0]; deskHover = hit ? hit.object.userData.btn : null; for (const b of buttons) { const hv = b === deskHover; if (b.hover !== hv) { b.hover = hv; b.draw(); } } }
  }
  // VR: käed, tulistamine, nupud
  if (renderer.xr.isPresenting) {
    const ph = playerHead(V3()); P.crouch = ph.y - rig.position.y < (game ? 1.25 : 1.2);
    const btnMeshes = buttons.filter(b => isShown(b.p.mesh)).map(b => b.p.mesh);
    for (const h of hands) {
      h.ray.visible = false; if (!h.connected) continue;
      const hp = handPos(h, V3()); if (h.hasPrev) h.vel.lerp(tmpD.copy(hp).sub(h.prev).divideScalar(Math.max(dt, 1e-3)), 0.5); h.prev.copy(hp); h.hasPrev = true;
      if (h.isHand) handGestures(h);
      let hov = null; if (btnMeshes.length) { h.ctrl.getWorldPosition(tmpV); h.ctrl.getWorldQuaternion(tmpQ); const rc = new THREE.Raycaster(tmpV, V3(0, 0, -1).applyQuaternion(tmpQ), 0, 5); const hit = rc.intersectObjects(btnMeshes, false)[0]; if (hit) { hov = hit.object.userData.btn; h.ray.visible = true; h.ray.scale.set(1, 1, hit.distance); } }
      if (hov !== h.hover) { if (h.hover) { h.hover.hover = false; h.hover.draw(); } h.hover = hov; if (hov) { hov.hover = true; hov.draw(); } }
      if (h.isHand && J(h, 'index-finger-tip')) { const tip = J(h, 'index-finger-tip').getWorldPosition(V3()); let touched = null; for (const m of btnMeshes) { const lp = m.worldToLocal(tip.clone()), b = m.userData.btn; if (Math.abs(lp.x) < b.w / 2 && Math.abs(lp.y) < b.h / 2 && Math.abs(lp.z) < 0.03) touched = b; } if (touched && h.poke !== touched) clickBtn(touched, h); h.poke = touched; }
    }
    const rh = R();
    if (rh && (mode === 'play' || mode === 'hub')) { const w = WEAPONS[P.weapon]; if ((rh.trig || rh.selTrig) && !rh.hover) { if (w.auto || !rh.semiLock) { fire(); rh.semiLock = true; } } else rh.semiLock = false; }
    checkMagInsert(); updTeleport();
    // käe juhised
    const lh = L(); if (lh && !lh.held && (mode === 'play' || mode === 'hub')) { const hp = handPos(lh, V3()); const near = hp.distanceTo(bodyPoint(-0.2, -0.65, 0.05)) < 0.22 ? '🔋 Salv' : hp.distanceTo(bodyPoint(-0.1, -0.35, 0.18)) < 0.18 ? '💣 Granaat' : hp.distanceTo(bodyPoint(0.1, -0.35, 0.18)) < 0.18 ? '🌫️ Suits' : null; if (near && near !== lh.lastNear) pulse(lh, 0.2, 20); lh.lastNear = near; }
  }
  P.cool -= dt;
  if (game && game.onShot && P.lastAmmo != null && guns[P.weapon] && guns[P.weapon].ammo < P.lastAmmo) game.onShot(); P.lastAmmo = guns[P.weapon] ? guns[P.weapon].ammo : null;
  updateWeapon(dt);
  if (mode === 'play') {
    game.t += dt; const line = game.tick ? game.tick(game, dt) : ''; updateEnemies(dt); drawHud(line);
    if (!P.dead) { P.regenT -= dt; if (P.regenT <= 0 && P.hp < P.maxHp) P.hp = Math.min(P.maxHp, P.hp + 25 * dt); }
  } else if (mode === 'hub') { updateEnemies(dt); }
  updProps(dt); updTargets(dt); updAtmos(dt); if (mode === 'play') updStreaks(dt); updParticles(dt); updTracers(dt);
  hurtM.material.opacity = Math.max(0, hurtM.material.opacity - dt * 0.6 * (P.hp > 30 ? 1 : 0.3));
  for (const p of pops) if (p.p.mesh.visible) { p.t -= dt; p.p.mesh.position.y += dt * 0.4; camera.getWorldPosition(head); p.p.mesh.lookAt(head); p.p.mesh.material.opacity = clamp(p.t, 0, 1); if (p.t <= 0) p.p.mesh.visible = false; }
  if (msgP.mesh.visible) { msgT -= dt; msgP.mesh.material.opacity = clamp(msgT * 2, 0, 1); if (msgT <= 0) msgP.mesh.visible = false; }
  if (window._flag) window._flag.rotation.y = Math.sin(T * 2) * 0.15;
  // varjud järgnevad mängijale
  camera.getWorldPosition(head); sun.position.set(head.x + 25, 50, head.z + 15); sun.target.position.set(head.x, 0, head.z);
  if (torch.intensity > 0) { const g = guns[P.weapon]; g.mz.getWorldPosition(torch.position); torch.target.position.copy(torch.position).add(V3(0, 0, -1).applyQuaternion(g.g.getWorldQuaternion(new THREE.Quaternion()))); }
  if (actx && actx.listener.positionX) { const l = actx.listener, f = camera.getWorldDirection(V3()); l.positionX.value = head.x; l.positionY.value = head.y; l.positionZ.value = head.z; l.forwardX.value = f.x; l.forwardY.value = f.y; l.forwardZ.value = f.z; l.upX.value = 0; l.upY.value = 1; l.upZ.value = 0; }
  pMat.uniforms.uScale.value = renderer.xr.isPresenting ? 900 : renderer.domElement.height / 2 * camera.projectionMatrix.elements[5];
  renderer.render(scene, camera);
});
window.__el = { startMission, useStreak, shootRay, startGame, rangeDrill, killhouse, waves, toHub, spawnEnemy, enemies, P, guns, fire, get game(){ return game; }, save, explode, setWeapon };
</script>
</body>
</html>
