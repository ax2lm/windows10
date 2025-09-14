<!doctype html>
<html lang="ar">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>محاكاة نظام تشغيل - Mini OS</title>
<style>
  :root{
    --taskbar-h:56px;
    --accent:#2a9df4;
    --glass: rgba(255,255,255,0.06);
    --text: #fff;
    --win-bg: rgba(255,255,255,0.05);
    --win-border: rgba(255,255,255,0.12);
    font-family: "Segoe UI", Tahoma, Arial, sans-serif;
  }
  html,body{height:100%;margin:0;padding:0;background:#123;color:var(--text);direction:rtl}
  body{overflow:hidden}
  /* Desktop */
  .desktop{position:relative;height:100vh;width:100vw;background-size:cover;background-position:center;}
  .icons{position:absolute;left:18px;top:18px;display:grid;grid-template-columns:repeat(4,88px);gap:18px;z-index:2}
  .icon{width:88px;text-align:center;cursor:pointer;user-select:none}
  .icon img{width:64px;height:64px;display:block;margin:0 auto 6px;filter: drop-shadow(0 6px 8px rgba(0,0,0,0.45))}
  .icon span{font-size:13px;display:block;color:var(--text);overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
  /* Taskbar */
  .taskbar{position:fixed;right:0;left:0;bottom:0;height:var(--taskbar-h);display:flex;align-items:center;padding:6px 10px;background:linear-gradient(to top, rgba(0,0,0,0.5), rgba(255,255,255,0.02));backdrop-filter: blur(6px);z-index:10}
  .start-btn{background:var(--glass);border-radius:8px;padding:8px 14px;cursor:pointer;margin-left:8px}
  .task-middle{flex:1;display:flex;gap:8px;align-items:center;overflow:auto;padding-right:8px}
  .taskitem{background:rgba(255,255,255,0.04);padding:6px 10px;border-radius:6px;cursor:pointer;white-space:nowrap}
  .task-clock{min-width:120px;text-align:left;font-weight:600}
  /* Start menu */
  .start-menu{position:fixed;left:12px;bottom:calc(var(--taskbar-h) + 12px);width:320px;background:rgba(11,22,40,0.94);border-radius:10px;padding:12px;box-shadow:0 8px 30px rgba(0,0,0,0.6);display:none;z-index:11}
  .start-grid{display:grid;grid-template-columns:1fr 1fr;gap:8px}
  .start-app{background:rgba(255,255,255,0.03);padding:10px;border-radius:8px;cursor:pointer;text-align:center}
  /* Windows */
  .window{position:absolute;width:640px;height:420px;background:var(--win-bg);border:1px solid var(--win-border);border-radius:8px;box-shadow:0 12px 40px rgba(0,0,0,0.6);overflow:hidden;display:flex;flex-direction:column;z-index:5}
  .win-title{height:42px;display:flex;align-items:center;justify-content:space-between;padding:0 10px;background:linear-gradient(to bottom, rgba(255,255,255,0.02), rgba(0,0,0,0.06));cursor:grab}
  .win-title .title {font-weight:700}
  .win-controls{display:flex;gap:6px}
  .btn{background:transparent;border:none;color:var(--text);cursor:pointer;padding:6px 8px;border-radius:6px}
  .win-content{flex:1;padding:10px;overflow:auto}
  .hidden{display:none}
  /* Small app styles */
  textarea{width:100%;height:100%;background:rgba(0,0,0,0.6);color:var(--text);border:none;padding:8px;border-radius:6px;resize:none}
  .calc-keys{display:grid;grid-template-columns:repeat(4,1fr);gap:8px}
  .key{padding:12px;border-radius:8px;background:rgba(255,255,255,0.04);text-align:center;cursor:pointer}
  /* file manager */
  .files{display:flex;flex-wrap:wrap;gap:12px}
  .file{width:120px;padding:8px;background:rgba(255,255,255,0.02);border-radius:6px;text-align:center;cursor:pointer}
  .toolbar{display:flex;gap:8px;margin-bottom:8px}
  /* responsive */
  @media (max-width:900px){ .window {width:90%; height:70%} .icons {grid-template-columns:repeat(3,88px)} }
</style>
</head>
<body>
  <div id="desktop" class="desktop"></div>

  <!-- Taskbar -->
  <div class="taskbar" id="taskbar">
    <div style="display:flex;align-items:center">
      <div class="start-btn" id="startBtn">ابدأ</div>
    </div>
    <div class="task-middle" id="runningApps"></div>
    <div class="task-clock" id="clock"></div>
  </div>

  <!-- Start menu -->
  <div class="start-menu" id="startMenu" role="menu" aria-hidden="true">
    <h3 style="margin:0 0 8px;color:var(--text)">قائمة ابدأ</h3>
    <div class="start-grid">
      <div class="start-app" data-app="notepad">المفكرة</div>
      <div class="start-app" data-app="browser">المتصفح</div>
      <div class="start-app" data-app="calc">الحاسبة</div>
      <div class="start-app" id="openFileManager">مستكشف الملفات</div>
      <div class="start-app" id="changeWallpaper">تغيير الخلفية</div>
      <div class="start-app" id="clearStorage">مسح ملفات المحاكي</div>
    </div>
  </div>

  <!-- Templates (hidden) -->
  <template id="tmpl-icon">
    <div class="icon" draggable="true">
      <img src="">
      <span>اسم</span>
    </div>
  </template>

  <template id="tmpl-window">
    <div class="window" tabindex="0">
      <div class="win-title">
        <div class="title">عنوان</div>
        <div class="win-controls">
          <button class="btn btn-min" title="تصغير">_</button>
          <button class="btn btn-max" title="تكبير">⬜</button>
          <button class="btn btn-close" title="إغلاق">✕</button>
        </div>
      </div>
      <div class="win-content"></div>
    </div>
  </template>

<script>
/* ======= إعداد بيئة الديسكتوب و أيقونات افتراضية ======= */
const desktop = document.getElementById('desktop');
const iconsContainer = document.createElement('div');
iconsContainer.className = 'icons';
desktop.appendChild(iconsContainer);

// بيانات أيقونات افتراضية
const ICONS = [
  {id:'thispc', title:'هذا الجهاز', icon: 'data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64"><rect width="64" height="48" y="8" rx="6" fill="%23ffffff" opacity="0.15"/></svg>'},
  {id:'notepad', title:'المفكرة', icon: 'data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64"><rect rx="8" width="64" height="64" fill="%23ffffff" opacity="0.12"/><text x="18" y="38" font-size="18" fill="%23000">✎</text></svg>'},
  {id:'browser', title:'المتصفح', icon: 'data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64"><circle cx="32" cy="32" r="28" fill="%23ffffff" opacity="0.12"/></svg>'},
  {id:'calc', title:'الحاسبة', icon: 'data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64"><rect rx="8" width="64" height="64" fill="%23ffffff" opacity="0.12"/><text x="18" y="38" font-size="18" fill="%23000'>∑</text></svg>'},
  {id:'files', title:'الملفات', icon: 'data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64"><rect rx="8' width='64' height='64' fill='%23ffffff' opacity='0.12'/></svg>'}
];

// خلق الأيقونات
const iconTmpl = document.getElementById('tmpl-icon');
ICONS.forEach(i=>{
  const node = iconTmpl.content.firstElementChild.cloneNode(true);
  node.dataset.app = i.id;
  node.querySelector('img').src = i.icon;
  node.querySelector('span').textContent = i.title;
  // فتح التطبيق عند دبل كليك
  node.addEventListener('dblclick', ()=>openApp(i.id));
  // سحب الأيقونات وإسقاطها (تخزين موضع)
  node.addEventListener('dragstart', dragStart);
  iconsContainer.appendChild(node);
});

// وضع خلفية افتراضية (يمكن تغييرها)
const DEFAULT_WALLPAPER = 'https://images.unsplash.com/photo-1503264116251-35a269479413?q=80&w=1920&auto=format&fit=crop&ixlib=rb-4.0.3&s=7d8b11d1b9a8e7d0a3f0f7b5f9f1f1b6';
const wallpaperKey = 'mini_os_wallpaper';
function setWallpaper(url){
  desktop.style.backgroundImage = url?`url("${url}")`:`linear-gradient(135deg,#1e3c72,#2a5298)`;
  if(url) localStorage.setItem(wallpaperKey, url); else localStorage.removeItem(wallpaperKey);
}
setWallpaper(localStorage.getItem(wallpaperKey) || DEFAULT_WALLPAPER);

/* ======= شريط المهام و start menu ======= */
const startBtn = document.getElementById('startBtn');
const startMenu = document.getElementById('startMenu');
startBtn.addEventListener('click', ()=> {
  startMenu.style.display = startMenu.style.display === 'block' ? 'none' : 'block';
  startMenu.setAttribute('aria-hidden', startMenu.style.display!=='block');
});
// اغلاق عند الضغط خارج
document.addEventListener('click', e=>{
  if(!startMenu.contains(e.target) && e.target !== startBtn) startMenu.style.display = 'none';
});
// تحديث الساعة
const clock = document.getElementById('clock');
function updateClock(){ const d=new Date(); clock.textContent = d.toLocaleTimeString(); }
setInterval(updateClock,1000); updateClock();

/* ======= مدير النوافذ (فتح / تصغير / تكبير / اغلاق) ======= */
const wndTmpl = document.getElementById('tmpl-window');
let zCount = 20;
const runningApps = document.getElementById('runningApps');

function createWindow(title, contentHTML){
  const w = wndTmpl.content.firstElementChild.cloneNode(true);
  w.querySelector('.title').textContent = title;
  const content = w.querySelector('.win-content');
  content.innerHTML = contentHTML;
  desktop.appendChild(w);
  w.style.left = (50 + Math.random()*200) + 'px';
  w.style.top = (60 + Math.random()*120) + 'px';
  w.style.zIndex = ++zCount;
  makeDraggable(w);
  // controls
  const btnClose = w.querySelector('.btn-close');
  const btnMin = w.querySelector('.btn-min');
  const btnMax = w.querySelector('.btn-max');
  let isMax=false, prevRect=null;
  btnClose.addEventListener('click', ()=>{ w.remove(); updateRunningBar(); });
  btnMin.addEventListener('click', ()=>{ w.style.display='none'; });
  btnMax.addEventListener('click', ()=>{
    if(!isMax){
      prevRect = {left:w.style.left, top:w.style.top, width:w.style.width, height:w.style.height};
      w.style.left = '0px'; w.style.top='0px'; w.style.width = '100%'; w.style.height = `calc(100vh - var(--taskbar-h) - 6px)`;
      isMax=true;
    } else {
      w.style.left = prevRect.left; w.style.top = prevRect.top; w.style.width = prevRect.width; w.style.height = prevRect.height;
      isMax=false;
    }
  });
  // عند النقر ركز النافذة
  w.addEventListener('mousedown', ()=>{ w.style.zIndex = ++zCount; });
  // زر في الشريط
  const tbtn = document.createElement('div'); tbtn.className='taskitem'; tbtn.textContent = title;
  tbtn.onclick = ()=>{ if(w.style.display==='none'){ w.style.display='flex'; w.style.zIndex = ++zCount; } else { w.style.display='none'; } };
  runningApps.appendChild(tbtn);
  w._taskBtn = tbtn;
  return w;
}

function updateRunningBar(){
  // حذف الأزرار المرتبطة بنوافذ مفرغة
  Array.from(runningApps.children).forEach(btn=>{
    const exists = Array.from(document.querySelectorAll('.window')).some(win => win._taskBtn === btn);
    if(!exists) btn.remove();
  });
}

/* ======= سحب النوافذ ======= */
function makeDraggable(el){
  const title = el.querySelector('.win-title');
  let dragging=false, startX=0, startY=0, origX=0, origY=0;
  title.addEventListener('pointerdown', (e)=>{
    dragging=true; startX=e.clientX; startY=e.clientY;
    const rect = el.getBoundingClientRect(); origX = rect.left; origY = rect.top;
    title.setPointerCapture(e.pointerId); title.style.cursor='grabbing';
    el.style.zIndex = ++zCount;
  });
  title.addEventListener('pointermove', (e)=>{ if(!dragging) return;
    const dx = e.clientX - startX; const dy = e.clientY - startY;
    el.style.left = (origX + dx) + 'px'; el.style.top = (origY + dy) + 'px';
  });
  title.addEventListener('pointerup', (e)=>{ dragging=false; title.releasePointerCapture(e.pointerId); title.style.cursor='grab'; });
}

/* ======= فتح التطبيقات الأساسية ======= */
function openApp(id){
  if(id === 'notepad') openNotepad();
  if(id === 'browser') openBrowser();
  if(id === 'calc') openCalc();
  if(id === 'thispc' || id === 'files') openFileManager();
}

// -- مفكرة مع حفظ/تحميل
function openNotepad(){
  const content = `
    <div style="display:flex;flex-direction:column;height:100%">
      <textarea id="notepadArea" placeholder="اكتب ملاحظاتك هنا..." style="flex:1;"></textarea>
      <div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-start">
        <button id="saveNote" class="btn">حفظ كملف</button>
        <button id="downloadNote" class="btn">تحميل</button>
        <button id="clearNote" class="btn">مسح</button>
        <button id="lsave" class="btn">حفظ بالمحاكي</button>
      </div>
    </div>`;
  const w = createWindow('المفكرة', content);
  // أحداث بعد الإدراج
  w.querySelector('#saveNote').addEventListener('click', ()=>{
    const txt = w.querySelector('#notepadArea').value;
    const blob = new Blob([txt], {type:'text/plain;charset=utf-8'});
    const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'note.txt'; a.click();
  });
  w.querySelector('#downloadNote').addEventListener('click', ()=>{
    const txt = w.querySelector('#notepadArea').value;
    alert('تنزيل نص كمحاكاة - نفس زر حفظ (سيبدأ تحميل الملف).');
    const blob = new Blob([txt], {type:'text/plain;charset=utf-8'});
    const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'note.txt'; a.click();
  });
  w.querySelector('#clearNote').addEventListener('click', ()=> w.querySelector('#notepadArea').value = '');
  w.querySelector('#lsave').addEventListener('click', ()=>{
    const txt = w.querySelector('#notepadArea').value;
    const notes = JSON.parse(localStorage.getItem('mini_os_notes')||'[]');
    notes.push({id:Date.now(), text:txt});
    localStorage.setItem('mini_os_notes', JSON.stringify(notes));
    alert('تم حفظ الملاحظة في ذاكرة المتصفح (localStorage).');
  });
}

// -- متصفح وهمي (iframe) مع تحذير CORS
function openBrowser(){
  const content = `
    <div style="display:flex;flex-direction:column;height:100%">
      <div style="display:flex;gap:8px;margin-bottom:8px">
        <input id="url" placeholder="https://example.com" style="flex:1;padding:8px;border-radius:6px;border:1px solid rgba(255,255,255,0.06);background:transparent;color:var(--text)" value="https://example.com">
        <button id="go" class="btn">اذهب</button>
      </div>
      <div style="flex:1;background:rgba(0,0,0,0.6);border-radius:6px;overflow:hidden">
        <iframe id="

