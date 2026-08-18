# 语音打卡助手 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 给「猫咪生活报」工作台的「猫咪日常」(checkin) 模块加语音交互：说话即可新增 / 完成 / 删除打卡。

**Architecture:** Chrome 内置 Web Speech API 做免费语音识别 → 文本 + 当前条目清单发给 DeepSeek → 返回 JSON 指令 → 前端执行器操作 `data.checkin` 并 `persist()`。全部代码新增在 `workbench-desktop.html` 内，现有代码零改动；删除确认复用模板已有的 `confirmDelete()`。

**Tech Stack:** 单文件 HTML/JS（无构建）、Web Speech API（`webkitSpeechRecognition`）、DeepSeek `/chat/completions`（`response_format: json_object`）、localStorage。

**Spec:** `docs/superpowers/specs/2026-08-17-voice-checkin-design.md`

**测试说明（重要）：** 本项目是无构建、无测试框架的单文件静态页（YAGNI：不为此引入测试基建）。验证方式为「语法静态检查 + 本地服务手动验收」。每个任务都给出可执行的验证命令与预期结果；Task 6 为完整手动验收清单。

**已知锚点（行号基于当前文件）：**
- `ICONS` 字典末尾（`paw:` 之后）：第 647-648 行
- `CONFIG` 定义结束 `};`：第 615 行
- 主 `<style>` 块结束 `</style>`：第 520 行
- 主 `<script>` 块末尾（`buildNav(); render();`）：第 1324-1326 行
- 已有可复用：`confirmDelete(key,id)`（1279 行）、`.overlay`/`.modal`/`.btn`/`.btn.ghost` 样式、`esc()`/`icon()`/`today()`/`persist()`/`data` 全局

---

### Task 1: ICONS 新增 mic 图标 + VOICE_CONFIG 配置块

**Files:**
- Modify: `/workspace/cat-press-workbench/workbench-desktop.html`（第 647 行附近、第 615 行之后）

- [ ] **Step 1: 在 ICONS 字典加 Lucide `mic` 图标**

找到（第 647 行）：

```js
  paw:'<circle cx="11" cy="4" r="2"/><circle cx="18" cy="8" r="2"/><circle cx="20" cy="16" r="2"/><path d="M9 10a5 5 0 0 1 5 5v3.5a3.5 3.5 0 0 1-6.84 1.045Q6.52 17.48 4.46 16.84A3.5 3.5 0 0 1 5.5 10Z"/>',
};
```

改为：

```js
  paw:'<circle cx="11" cy="4" r="2"/><circle cx="18" cy="8" r="2"/><circle cx="20" cy="16" r="2"/><path d="M9 10a5 5 0 0 1 5 5v3.5a3.5 3.5 0 0 1-6.84 1.045Q6.52 17.48 4.46 16.84A3.5 3.5 0 0 1 5.5 10Z"/>',
  mic:'<path d="M12 2a3 3 0 0 0-3 3v7a3 3 0 0 0 6 0V5a3 3 0 0 0-3-3Z"/><path d="M19 10v2a7 7 0 0 1-14 0v-2"/><line x1="12" x2="12" y1="19" y2="22"/>',
};
```

- [ ] **Step 2: 在 CONFIG 定义之后插入 VOICE_CONFIG**

找到（第 615 行）：

```js
  ],
};

/* ============================================================
   ICONS — 单色线性图标库（stroke 跟随 color）
   ============================================================ */
```

在 `};` 与 ICONS 注释之间插入：

```js
  ],
};

/* ============================================================
   VOICE_CONFIG — 语音打卡助手（猫咪日常 × 大模型指令解析）
   换 provider 只需改 baseURL / model（如 Gemini 免费档）
   ============================================================ */
const VOICE_CONFIG = {
  baseURL: "https://api.deepseek.com/chat/completions",
  model: "deepseek-chat",
  keyStoreKey: "cat-press-llm-key",   // API Key 在 localStorage 里的键名
  moduleKey: "checkin",               // 本期只接猫咪日常；未来扩展改这里+动作表
};

/* ============================================================
   ICONS — 单色线性图标库（stroke 跟随 color）
   ============================================================ */
```

- [ ] **Step 3: 验证页面无语法错误**

Run:
```bash
python3 -c "
import re
html=open('/workspace/cat-press-workbench/workbench-desktop.html').read()
js=re.findall(r'<script>(.*?)</script>', html, re.S)[0]
open('/tmp/wb.js','w').write(js)
" && node --check /tmp/wb.js && echo "JS syntax OK"
```
Expected: 输出 `JS syntax OK`（无报错）

- [ ] **Step 4: Commit**

```bash
cd /workspace/cat-press-workbench && git add workbench-desktop.html && git -c user.name="Trae Agent" -c user.email="trae-agent@localhost" commit -m "feat: 语音助手配置块 + mic 图标"
```

---

### Task 2: 语音助手 CSS（麦克风按钮、录音脉冲、toast）

**Files:**
- Modify: `/workspace/cat-press-workbench/workbench-desktop.html`（第 520 行 `</style>` 之前）

- [ ] **Step 1: 在主 `</style>` 之前追加样式**

找到（第 520 行）：

```css
</style>
```

在 `</style>` 前一行插入：

```css
  /* ---------- 语音打卡助手 ---------- */
  .vmic { position:fixed; right:28px; bottom:28px; z-index:50; width:56px; height:56px; border-radius:50%;
    background:var(--module-3); color:var(--on-accent); border:none; cursor:pointer; display:grid; place-items:center;
    box-shadow:var(--shadow-overlay); transition:transform .15s; }
  .vmic:hover { transform:scale(1.06); }
  .vmic.on { background:var(--accent); animation:vmic-pulse 1.2s ease-in-out infinite; }
  @keyframes vmic-pulse { 0%,100% { box-shadow:0 0 0 0 color-mix(in srgb, var(--accent) 45%, transparent); }
    50% { box-shadow:0 0 0 14px transparent; } }
  .vtoast { position:fixed; left:50%; bottom:100px; transform:translateX(-50%) translateY(8px); z-index:70;
    background:var(--drawer-bg); color:var(--drawer-text); font-size:13.5px; padding:10px 18px; border-radius:999px;
    box-shadow:var(--shadow-overlay); opacity:0; transition:opacity .25s, transform .25s; pointer-events:none; max-width:70vw; }
  .vtoast.show { opacity:1; transform:translateX(-50%) translateY(0); }
```

- [ ] **Step 2: 验证 CSS 插入位置正确（仍在 `<style>` 内）**

Run:
```bash
python3 -c "
html=open('/workspace/cat-press-workbench/workbench-desktop.html').read()
css=html.split('<style>')[1].split('</style>')[0]
assert '.vmic' in css and '.vtoast' in css, 'CSS not inside style block'
assert html.count('<style>')==1 and html.count('</style>')==1, 'style tag broken'
print('CSS insertion OK')
"
```
Expected: 输出 `CSS insertion OK`

- [ ] **Step 3: Commit**

```bash
cd /workspace/cat-press-workbench && git add workbench-desktop.html && git -c user.name="Trae Agent" -c user.email="trae-agent@localhost" commit -m "feat: 语音助手样式（麦克风按钮/录音脉冲/toast）"
```

---

### Task 3: 语音脚本（上）—— toast、Key 管理、麦克风按钮、语音识别

**Files:**
- Modify: `/workspace/cat-press-workbench/workbench-desktop.html`（第 1324-1326 行，`buildNav(); render();` 之后、`</script>` 之前）

> **质量审查修正（已并入）：** `vlisten` 增加 `vstarting` 启动态防多实例竞态；`openKeyDialog` 增加重入守卫与 350ms 点外关闭宽限防双击闪关。

- [ ] **Step 1: 插入脚本上半部分**

找到（文件末尾）：

```js
buildNav();
render();
</script>
```

在 `render();` 与 `</script>` 之间插入：

```js
buildNav();
render();

/* ============================================================
   VOICE — 语音打卡助手（新增代码块，独立闭环）
   链路：麦克风按钮 → 语音识别 → parseCommand(在下方定义) → execCommand(在下方定义)
   ============================================================ */
function vtoast(msg, ms=2600){
  const el=document.createElement("div"); el.className="vtoast"; el.textContent=msg;
  document.body.appendChild(el);
  requestAnimationFrame(()=>el.classList.add("show"));
  setTimeout(()=>{ el.classList.remove("show"); setTimeout(()=>el.remove(),300); }, ms);
}

/* ---------- API Key 管理（localStorage，仅发往 provider 官方接口） ---------- */
function vkey(){ return localStorage.getItem(VOICE_CONFIG.keyStoreKey)||""; }
function ensureKey(){ if(vkey()) return true; openKeyDialog(); return false; }
function openKeyDialog(){
  if(document.querySelector(".overlay")) return;   // 防重入：已有弹窗时不再叠加
  const overlay=document.createElement("div"); overlay.className="overlay";
  overlay.innerHTML=`<div class="modal" style="width:420px"><h3>设置大模型 Key</h3>
    <div class="sub">语音指令由 DeepSeek 解析。Key 只存在本机浏览器，仅发往官方接口。<br>没有 Key？到 platform.deepseek.com 注册，充值 1 元可用数月。</div>
    <input id="vk-input" type="password" placeholder="sk-..." autocomplete="off"
      style="width:100%;padding:10px 12px;border:1px solid var(--border-input);border-radius:var(--radius-tile);background:var(--surface-nested);color:var(--text);font-size:14px"/>
    <div class="modal-actions"><div class="spacer"></div>
      <button class="btn ghost" id="vk-cancel">取消</button><button class="btn" id="vk-save">保存</button></div></div>`;
  document.body.appendChild(overlay);
  const close=()=>overlay.remove();
  const bornAt=Date.now();
  overlay.onclick=e=>{ if(e.target===overlay && Date.now()-bornAt>350) close(); };
  overlay.querySelector("#vk-cancel").onclick=close;
  overlay.querySelector("#vk-save").onclick=()=>{
    const k=overlay.querySelector("#vk-input").value.trim();
    if(!k){ vtoast("Key 不能为空"); return; }
    localStorage.setItem(VOICE_CONFIG.keyStoreKey,k); close(); vtoast("Key 已保存，点麦克风开始说话");
  };
}

/* ---------- 语音识别（Chrome / Edge 免费内置） ---------- */
let vrecog=null, vlistening=false, vstarting=false;
function vlisten(){
  const SR=window.SpeechRecognition||window.webkitSpeechRecognition;
  if(!SR){ vtoast("当前浏览器不支持语音识别，请用 Chrome / Edge"); return; }
  if(vstarting) return;                        // 启动中：忽略重复点击，防多实例并行
  if(vlistening){ vrecog&&vrecog.stop(); return; }
  vrecog=new SR(); vrecog.lang="zh-CN"; vrecog.interimResults=false; vrecog.maxAlternatives=1;
  const btn=document.querySelector("#vmic");
  vrecog.onstart=()=>{ vstarting=false; vlistening=true; btn.classList.add("on"); };
  vrecog.onend=()=>{ vstarting=false; vlistening=false; btn.classList.remove("on"); };
  vrecog.onerror=e=>{ vstarting=false; vlistening=false; btn.classList.remove("on");
    if(e.error==="not-allowed") vtoast("麦克风权限被拒绝，请在地址栏允许后重试",3600);
    else if(e.error!=="aborted") vtoast("语音识别出错："+e.error); };
  vrecog.onresult=async ev=>{
    const text=ev.results[0][0].transcript.trim(); if(!text) return;
    vtoast(`听到：「${text}」`,1800);
    try{ execCommand(await parseCommand(text)); }
    catch(err){ vtoast("网络或 Key 有问题："+err.message,3600); }
  };
  vstarting=true;
  vrecog.start();
}

/* ---------- 悬浮麦克风按钮 ---------- */
(function(){
  const b=document.createElement("button");
  b.id="vmic"; b.className="vmic"; b.title="语音打卡（猫咪日常）";
  b.innerHTML=icon("mic",24);
  b.onclick=()=>{ if(ensureKey()) vlisten(); };
  document.body.appendChild(b);
})();
</script>
```

- [ ] **Step 2: 验证语法 + 按钮出现**

Run:
```bash
python3 -c "
import re
html=open('/workspace/cat-press-workbench/workbench-desktop.html').read()
js=re.findall(r'<script>(.*?)</script>', html, re.S)[0]
open('/tmp/wb.js','w').write(js)
" && node --check /tmp/wb.js && echo "JS syntax OK"
```
Expected: `JS syntax OK`

然后手动验证：
```bash
python3 -m http.server 8321 --directory /workspace/cat-press-workbench &
# 浏览器打开 http://localhost:8321/workbench-desktop.html
```
预期：右下角出现橘黄色圆形麦克风按钮；点击后弹出「设置大模型 Key」弹窗（因尚无 Key）；输入任意字符串保存后按钮可用。验证后 `kill %1`。

- [ ] **Step 3: Commit**

```bash
cd /workspace/cat-press-workbench && git add workbench-desktop.html && git -c user.name="Trae Agent" -c user.email="trae-agent@localhost" commit -m "feat: 语音识别 + Key 管理 + 悬浮麦克风按钮"
```

---

### Task 4: 语音脚本（下）—— LLM 指令解析、模糊匹配、命令执行器

**Files:**
- Modify: `/workspace/cat-press-workbench/workbench-desktop.html`（Task 3 插入的 `/* ---------- 悬浮麦克风按钮 ---------- */` 注释之前）

> **质量审查修正（已并入）：** `fuzzyFind` 收紧编辑距离阈值（随标题/输入长度取 min），修复 2 字符标题被任意 2 字符输入误命中的问题；includes 与阈值统一 trim 口径。

- [ ] **Step 1: 在悬浮麦克风按钮 IIFE 之前插入**

找到（Task 3 插入的代码内）：

```js
/* ---------- 悬浮麦克风按钮 ---------- */
(function(){
```

在其前面插入：

```js
/* ---------- LLM 指令解析（DeepSeek，JSON 输出） ---------- */
async function parseCommand(text){
  const titles=(data.checkin||[]).map(x=>x.title).join("、")||"（空）";
  const sys=`你是「猫咪生活报」打卡助手的指令解析器。当前打卡条目：${titles}。
把用户的话解析为 JSON：{"action":"add|delete|complete|unknown","title":"..."}。
规则：add=新增打卡，title 用用户说的新条目名；complete=完成打卡/打了卡，title 用清单里最匹配的条目标题原文；delete=删除，title 用清单里最匹配的条目标题原文；无法理解用 unknown。只输出 JSON，不要解释。`;
  const res=await fetch(VOICE_CONFIG.baseURL,{ method:"POST",
    headers:{ "Content-Type":"application/json", "Authorization":"Bearer "+vkey() },
    body:JSON.stringify({ model:VOICE_CONFIG.model, temperature:0,
      response_format:{ type:"json_object" },
      messages:[ {role:"system",content:sys}, {role:"user",content:text} ] }) });
  if(res.status===401){ localStorage.removeItem(VOICE_CONFIG.keyStoreKey); throw new Error("Key 无效，已清除，请点麦克风重新设置"); }
  if(!res.ok) throw new Error("API 错误 "+res.status);
  const j=await res.json();
  const content=((j.choices||[])[0]||{}).message?.content||"";
  try{ return JSON.parse(content); }catch(e){ return {action:"unknown"}; }
}

/* ---------- 模糊匹配条目（包含匹配 → 编辑距离兜底） ---------- */
function lev(a,b){
  const m=a.length,n=b.length; const dp=Array.from({length:m+1},(_,i)=>[i,...Array(n).fill(0)]);
  for(let j=0;j<=n;j++) dp[0][j]=j;
  for(let i=1;i<=m;i++) for(let j=1;j<=n;j++)
    dp[i][j]=Math.min(dp[i-1][j]+1, dp[i][j-1]+1, dp[i-1][j-1]+(a[i-1]===b[j-1]?0:1));
  return dp[m][n];
}
function fuzzyFind(title){
  const items=data.checkin||[]; if(!items.length) return null;
  const t=String(title||"").trim().toLowerCase(); if(!t) return null;
  const hit=items.find(x=>x.title.trim().toLowerCase()===t)
    || items.find(x=>x.title.trim().toLowerCase().includes(t))
    || items.find(x=>t.includes(x.title.trim().toLowerCase()));
  if(hit) return hit;
  let best=null,bestD=Infinity;
  items.forEach(x=>{ const d=lev(t,x.title.trim().toLowerCase()); if(d<bestD){bestD=d;best=x;} });
  const L=best.title.trim().length;   // 阈值随标题/输入长度收紧，防短标题无确认误命中
  return bestD<=Math.min(Math.max(1,Math.floor(L/3)),Math.max(1,Math.min(t.length,L)-1)) ? best : null;
}

/* ---------- 命令执行器（add/complete 即时执行；delete 复用 confirmDelete 二次确认） ---------- */
function execCommand(cmd){
  if(!cmd||!cmd.action||cmd.action==="unknown"){ vtoast("没听清，请再说一次"); return; }
  data.checkin=data.checkin||[];
  if(cmd.action==="add"){
    const title=String(cmd.title||"").trim();
    if(!title){ vtoast("没听清要新增什么"); return; }
    data.checkin.push({id:Date.now(),title,log:{}}); persist();
    vtoast(`已新增打卡「${title}」`);
  }else if(cmd.action==="complete"){
    const x=fuzzyFind(cmd.title); if(!x){ vlistItems(); return; }
    x.log=x.log||{}; const t=today();
    if(x.log[t]){ vtoast(`「${x.title}」今天已打过卡`); return; }
    x.log[t]=true; persist();
    vtoast(`已为「${x.title}」盖上今日猫爪章`);
  }else if(cmd.action==="delete"){
    const x=fuzzyFind(cmd.title); if(!x){ vlistItems(); return; }
    confirmDelete("checkin",x.id);   // 复用模板内置删除确认弹窗
  }else vtoast("没听清，请再说一次");
}
function vlistItems(){
  const names=(data.checkin||[]).map(x=>"「"+x.title+"」").join(" ");
  vtoast("没找到对应条目，当前有："+(names||"（空）"),3600);
}

/* ---------- 悬浮麦克风按钮 ---------- */
(function(){
```

- [ ] **Step 2: 验证语法**

Run:
```bash
python3 -c "
import re
html=open('/workspace/cat-press-workbench/workbench-desktop.html').read()
js=re.findall(r'<script>(.*?)</script>', html, re.S)[0]
open('/tmp/wb.js','w').write(js)
" && node --check /tmp/wb.js && echo "JS syntax OK"
```
Expected: `JS syntax OK`

- [ ] **Step 3: 验证函数齐全**

Run:
```bash
grep -c "function \(vtoast\|vkey\|ensureKey\|openKeyDialog\|vlisten\|parseCommand\|lev\|fuzzyFind\|execCommand\|vlistItems\)" /workspace/cat-press-workbench/workbench-desktop.html
```
Expected: 输出 `10`

- [ ] **Step 4: Commit**

```bash
cd /workspace/cat-press-workbench && git add workbench-desktop.html && git -c user.name="Trae Agent" -c user.email="trae-agent@localhost" commit -m "feat: LLM 指令解析 + 模糊匹配 + 语音命令执行器"
```

---

### Task 5: 联调验收（需真实 DeepSeek Key，人工执行）

**Files:** 无改动，纯验收。

- [ ] **Step 1: 起本地服务**

```bash
python3 -m http.server 8321 --directory /workspace/cat-press-workbench
```
浏览器打开 `http://localhost:8321/workbench-desktop.html`，允许麦克风权限，首次点击麦克风输入真实 Key。

- [ ] **Step 2: 按 spec §6 验收清单逐项过**

| # | 操作（对麦克风说） | 预期 |
|---|---|---|
| 1 | 「新增一个梳毛打卡」 | 列表出现「梳毛打卡」 |
| 2 | 「喝水打卡完成了」 | 「喝够 8 杯水」今日已勾选 |
| 3 | 「把晒太阳删掉」 | 弹「删除记录」确认弹窗；点取消不删，点删除才消失 |
| 4 | 「水喝完了」 | 命中「喝够 8 杯水」（模糊匹配） |
| 5 | 「把遛弯删掉」 | toast 列出当前条目 |
| 6 | 控制台执行 `localStorage.removeItem("cat-press-llm-key")` 后点麦克风 | 弹出 Key 设置窗 |
| 7 | 控制台 | 全程无红色报错 |
| 8 | 清单含 2 字符条目时说无关 2 字符词（如「吃饭完成了」且清单无此条目） | toast 列出当前条目，不误完成 |

- [ ] **Step 3: 若全部通过，收尾提交**

```bash
cd /workspace/cat-press-workbench && git status && git log --oneline -5
```
Expected: 工作区干净（除 assets/docs 已有提交），共 4 个 feat 提交。
