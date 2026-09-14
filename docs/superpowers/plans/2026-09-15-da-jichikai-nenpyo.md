# DA自治会 年表アプリ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 班員が各端末からブラウザだけで出来事・予定を追加・閲覧できる、Firebase Firestoreで共有される年表Webアプリを作り、GitHub Pagesで公開する。

**Architecture:** 単一の `index.html`（インラインCSS＋`<script type="module">`）。Firebase Firestore（CDN経由のESM SDK, v10）をデータストアとして使う。ビルドツール・npmは不要。

**Tech Stack:** HTML / CSS / Vanilla JS（ESモジュール）、Firebase JS SDK v10（`https://www.gstatic.com/firebasejs/10.13.2/` からCDN import）、Firestore、GitHub Pages。

## Global Constraints

- ファイルは `index.html` 1枚に完結させる（既存プロジェクトの単一ファイルパターンを踏襲）
- Firebaseプロジェクト: `dachronology`（`docs/superpowers/specs/2026-09-15-da-jichikai-nenpyo-design.md` にconfig記載済み）
- team値と色: `support`=赤`#e53935` / `survey`=青`#1e88e5` / `pr`=黄`#fdd835`（文字は濃色） / `project`=紫`#8e24aa` / `promotion`=グレー`#757575` / `all`=黒`#212121`
- eventType（固定6種）: 定例会 / 班長会議 / 全体集会 / プロジェクト会議 / 行事 / 意見交換会
- 合言葉: `"DA"`（追加・更新の書き込み時にFirestoreルールで検証）
- サイト全体のテーマカラーは赤`#e53935`
- 自動テストは設けない。各タスクの検証はブラウザでの手動確認（Browser preview toolを使用）
- 自動テストがないため、各タスクの「テスト」は実際にブラウザ(preview)で動作確認するステップに置き換える

---

### Task 1: Firestoreセキュリティルールの設定（ユーザー操作が必要）

**Files:**
- Create: `firestore.rules`（リポジトリに参考として保存するのみ。実際の適用はFirebase Consoleで手動）

**Interfaces:**
- Produces: `events`コレクションへの `read`（誰でも可）、`create`/`update`（`pass`フィールドが`"DA"`と一致する場合のみ可）というルール。以降の全タスクがこのルールに依存する

- [ ] **Step 1: ルールファイルを作成**

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /events/{eventId} {
      allow read: if true;
      allow create, update: if request.resource.data.pass == "DA";
    }
  }
}
```

このテキストを `firestore.rules` として保存する。

- [ ] **Step 2: Firebase Consoleに貼り付けて公開（ユーザー操作）**

ユーザーに以下を依頼する:
1. https://console.firebase.google.com/project/dachronology/firestore/rules を開く
2. エディタの中身を上記ルールに置き換える
3. 「公開」ボタンを押す

- [ ] **Step 3: 確認**

ユーザーから「公開した」と確認が取れたら次のタスクに進む。（このプロジェクトは削除を`updateDoc`による論理削除で行うため、Firestoreの`delete`操作自体は使わない。ルールに`delete`は不要）

- [ ] **Step 4: コミット**

```bash
git add firestore.rules
git commit -m "Firestoreセキュリティルールを追加（参考用）"
```

---

### Task 2: HTML雛形・Firebase初期化・購読の疎通確認

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces: グローバル定数 `db`（Firestoreインスタンス）、`eventsRef`（`events`コレクション参照）。以降のタスクはこれらを使う
- Consumes: Task 1で公開されたFirestoreルール

- [ ] **Step 1: `index.html` を作成し、骨格・テーマカラー・Firebase初期化・空の購読を実装**

```html
<!doctype html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>DA自治会 年表</title>
<style>
  :root{
    --da-red:#e53935;
    --bg:#fafafa;
    --card:#ffffff;
    --text:#212121;
    --muted:#757575;
    --border:#e0e0e0;
  }
  *{box-sizing:border-box;}
  body{
    margin:0;
    font-family:"Hiragino Kaku Gothic ProN","Yu Gothic",sans-serif;
    background:var(--bg);
    color:var(--text);
    line-height:1.6;
  }
  header{
    background:var(--da-red);
    color:#fff;
    padding:20px 16px;
    text-align:center;
  }
  header h1{margin:0;font-size:1.4rem;}
  main{max-width:720px;margin:0 auto;padding:16px;}
  .empty-msg{color:var(--muted);text-align:center;padding:24px 0;}
</style>
</head>
<body>
<header>
  <h1>DA自治会 年表</h1>
</header>
<main>
  <section id="filters"></section>
  <section id="add-form"></section>
  <section id="timeline">
    <p class="empty-msg" id="loading-msg">読み込み中...</p>
  </section>
</main>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.13.2/firebase-app.js";
import {
  getFirestore, collection, onSnapshot, query, orderBy
} from "https://www.gstatic.com/firebasejs/10.13.2/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "AIzaSyCM2EyiVohQzSEuKPvNJ-SzHpxLwIlVpFI",
  authDomain: "dachronology.firebaseapp.com",
  projectId: "dachronology",
  storageBucket: "dachronology.firebasestorage.app",
  messagingSenderId: "241472496249",
  appId: "1:241472496249:web:afc2f3be730205202f2aa8"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const eventsRef = collection(db, "events");

let allEvents = [];

const q = query(eventsRef, orderBy("date", "asc"));
onSnapshot(q, snap => {
  allEvents = snap.docs.map(d => ({ id: d.id, ...d.data() })).filter(e => !e.deleted);
  document.getElementById("timeline").innerHTML =
    `<p class="empty-msg">接続OK。イベント件数: ${allEvents.length}</p>`;
}, err => {
  document.getElementById("timeline").innerHTML =
    `<p class="empty-msg">データの読み込みに失敗しました: ${err.message}</p>`;
});
</script>
</body>
</html>
```

- [ ] **Step 2: ブラウザで開いて疎通確認**

Browser preview toolで `index.html` を開く（`preview_start` に `url: "file:///Users/taku/Desktop/index/da-jichikai-nenpyo/index.html"` を渡す、もしくは簡易HTTPサーバー経由）。
期待結果: 「読み込み中...」の後、「接続OK。イベント件数: 0」と表示される。コンソールにエラーが出ていないことを`read_console_messages`で確認する。

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "HTML雛形とFirebase接続を追加"
```

---

### Task 3: 班カラー・年表カード描画・フィルター機能

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: Task 2の `allEvents`（配列）、`onSnapshot`購読
- Produces: `TEAMS`定数（team key → `{label, color, textDark}`）、`renderTimeline()`関数、`renderFilters()`関数、`activeTeams`（Set）。Task 4・5がこれらを使う

- [ ] **Step 1: `<style>`にカード・バッジ・フィルターチップのスタイルを追加**

`</style>`の直前に追加:

```css
#filters{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:16px;}
.filter-chip{
  display:inline-flex;align-items:center;gap:6px;
  padding:6px 12px;border-radius:999px;
  border:2px solid var(--chip-color,#999);
  background:#fff;color:var(--chip-color,#333);
  font-size:0.85rem;cursor:pointer;user-select:none;
}
.filter-chip.off{background:#eee;color:#aaa;border-color:#ccc;}
.filter-chip .dot{width:10px;height:10px;border-radius:50%;background:var(--chip-color,#999);}
#timeline{display:flex;flex-direction:column;gap:12px;}
.event-card{
  background:var(--card);border:1px solid var(--border);
  border-left:6px solid var(--card-color,#999);
  border-radius:8px;padding:12px 14px;position:relative;
}
.event-card .row{display:flex;flex-wrap:wrap;align-items:center;gap:8px;margin-bottom:6px;}
.badge{display:inline-block;padding:2px 10px;border-radius:999px;font-size:0.75rem;color:#fff;background:var(--card-color,#999);}
.badge.pr-badge{color:#212121;}
.event-date{font-size:0.85rem;color:var(--muted);}
.event-type{font-size:0.8rem;color:var(--muted);}
.event-title{font-weight:bold;font-size:1.05rem;margin:4px 0;}
.event-memo{font-size:0.9rem;color:#555;white-space:pre-wrap;}
```

- [ ] **Step 2: `<script type="module">`内に`TEAMS`定数とレンダリング関数を追加**

`const eventsRef = collection(db, "events");` の直後に追加:

```javascript
const TEAMS = {
  support:   { label: "学生支援班",         color: "#e53935", textDark:false },
  survey:    { label: "アンケート班",       color: "#1e88e5", textDark:false },
  pr:        { label: "広報班",             color: "#fdd835", textDark:true  },
  project:   { label: "プロジェクト",       color: "#8e24aa", textDark:false },
  promotion: { label: "DA自治会推進委員会", color: "#757575", textDark:false },
  all:       { label: "全体共通",           color: "#212121", textDark:false },
};

const activeTeams = new Set(Object.keys(TEAMS));

function escapeHtml(s){
  return String(s).replace(/[&<>"']/g, c => ({
    "&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"
  }[c]));
}

function renderFilters(){
  const el = document.getElementById("filters");
  el.innerHTML = "";
  Object.entries(TEAMS).forEach(([key, t]) => {
    const chip = document.createElement("button");
    chip.type = "button";
    chip.className = "filter-chip" + (activeTeams.has(key) ? "" : " off");
    chip.style.setProperty("--chip-color", t.color);
    chip.innerHTML = `<span class="dot"></span>${t.label}`;
    chip.addEventListener("click", () => {
      if (activeTeams.has(key)) activeTeams.delete(key);
      else activeTeams.add(key);
      renderFilters();
      renderTimeline();
    });
    el.appendChild(chip);
  });
}

function renderTimeline(){
  const el = document.getElementById("timeline");
  const visible = allEvents.filter(e => activeTeams.has(e.team));
  if (visible.length === 0){
    el.innerHTML = '<p class="empty-msg">表示できる出来事がありません</p>';
    return;
  }
  el.innerHTML = "";
  visible.forEach(e => {
    const t = TEAMS[e.team] || TEAMS.all;
    const card = document.createElement("div");
    card.className = "event-card";
    card.style.setProperty("--card-color", t.color);
    card.innerHTML = `
      <div class="row">
        <span class="event-date">${escapeHtml(e.date)}</span>
        <span class="badge${t.textDark ? " pr-badge" : ""}">${t.label}</span>
        <span class="event-type">${escapeHtml(e.eventType)}</span>
      </div>
      <div class="event-title">${escapeHtml(e.title)}</div>
      ${e.memo ? `<div class="event-memo">${escapeHtml(e.memo)}</div>` : ""}
    `;
    el.appendChild(card);
  });
}
```

`t.label`は`TEAMS`定数（アプリ側で固定定義した文字列）からのみ来るためエスケープ不要。`e.date`・`e.eventType`・`e.title`・`e.memo`はFirestore経由でユーザーが書き込める値なので、UIを介さない直接書き込み（合言葉さえ分かればAPI経由で可能）によるXSSを防ぐため全て`escapeHtml()`を通す。

- [ ] **Step 3: `onSnapshot`のコールバックを新しい描画関数を呼ぶように差し替え**

Task 2で書いた以下のブロック:

```javascript
onSnapshot(q, snap => {
  allEvents = snap.docs.map(d => ({ id: d.id, ...d.data() })).filter(e => !e.deleted);
  document.getElementById("timeline").innerHTML =
    `<p class="empty-msg">接続OK。イベント件数: ${allEvents.length}</p>`;
}, err => {
```

を以下に置き換える:

```javascript
onSnapshot(q, snap => {
  allEvents = snap.docs.map(d => ({ id: d.id, ...d.data() })).filter(e => !e.deleted);
  renderTimeline();
}, err => {
```

ファイル末尾（`</script>`の直前）に以下を追加:

```javascript
renderFilters();
```

- [ ] **Step 4: Firebase Consoleで手動データ投入して表示確認**

Firebase Console → Firestore → `events`コレクションにドキュメントを1件手動追加する:
`date`(string)="2026-09-15", `team`(string)="support", `eventType`(string)="定例会", `title`(string)="テストイベント", `memo`(string)="", `deleted`(boolean)=false

Browser preview toolで `index.html` を再読み込みし、赤い左バーのカードが「学生支援班」バッジ付きで表示されることを確認する。フィルターチップをクリックして「学生支援班」を非表示にすると、カードが消えて「表示できる出来事がありません」になることを確認する。

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "班カラーの年表カード描画とフィルター機能を追加"
```

---

### Task 4: 追加フォーム（カレンダー日付選択・合言葉ゲート）

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: Task 2の`db`, `eventsRef`。Task 3の`TEAMS`
- Produces: `askPass()`関数（Task 5の削除機能も再利用する）

- [ ] **Step 1: `<style>`にフォームのスタイルを追加**

`</style>`の直前に追加:

```css
#add-form{
  background:var(--card);border:1px solid var(--border);
  border-radius:8px;padding:16px;margin-bottom:24px;
}
#add-form h2{margin-top:0;font-size:1.1rem;}
#add-form label{display:block;font-size:0.85rem;margin-top:10px;margin-bottom:4px;color:var(--muted);}
#add-form input, #add-form select, #add-form textarea{
  width:100%;padding:8px;border:1px solid var(--border);border-radius:6px;
  font-size:1rem;font-family:inherit;
}
#add-form button{
  margin-top:14px;background:var(--da-red);color:#fff;border:none;
  padding:10px 20px;border-radius:6px;font-size:1rem;cursor:pointer;
}
#add-form button:hover{opacity:0.9;}
```

- [ ] **Step 2: `<section id="add-form"></section>`の中身を実装**

```html
<section id="add-form">
  <h2>出来事・予定を追加</h2>
  <label for="f-date">日付</label>
  <input type="date" id="f-date" required>

  <label for="f-team">班・カテゴリ</label>
  <select id="f-team">
    <option value="support">学生支援班</option>
    <option value="survey">アンケート班</option>
    <option value="pr">広報班</option>
    <option value="project">プロジェクト</option>
    <option value="promotion">DA自治会推進委員会</option>
    <option value="all">全体共通</option>
  </select>

  <label for="f-type">予定種別</label>
  <select id="f-type">
    <option value="定例会">定例会</option>
    <option value="班長会議">班長会議</option>
    <option value="全体集会">全体集会</option>
    <option value="プロジェクト会議">プロジェクト会議</option>
    <option value="行事">行事</option>
    <option value="意見交換会">意見交換会</option>
  </select>

  <label for="f-title">内容</label>
  <input type="text" id="f-title" required placeholder="例: 第3回定例会">

  <label for="f-memo">メモ（任意）</label>
  <textarea id="f-memo" rows="2"></textarea>

  <button id="f-submit" type="button">追加する</button>
</section>
```

- [ ] **Step 3: importに`addDoc`, `serverTimestamp`を追加し、合言葉ゲートと送信処理を実装**

importを以下に差し替え:

```javascript
import {
  getFirestore, collection, addDoc, updateDoc, doc,
  onSnapshot, query, orderBy, serverTimestamp
} from "https://www.gstatic.com/firebasejs/10.13.2/firebase-firestore.js";
```

`renderFilters();`（ファイル末尾）の直前に追加:

```javascript
const PASS_KEY = "da-nenpyo-pass";
const CORRECT_PASS = "DA";

function getSavedPass(){ return localStorage.getItem(PASS_KEY) || ""; }
function setSavedPass(p){ localStorage.setItem(PASS_KEY, p); }

function askPass(){
  const saved = getSavedPass();
  if (saved === CORRECT_PASS) return saved;
  const input = prompt("合言葉を入力してください");
  if (input === CORRECT_PASS) {
    setSavedPass(input);
    return input;
  }
  if (input !== null) alert("合言葉が違います");
  return null;
}

document.getElementById("f-submit").addEventListener("click", async () => {
  const date = document.getElementById("f-date").value;
  const team = document.getElementById("f-team").value;
  const eventType = document.getElementById("f-type").value;
  const title = document.getElementById("f-title").value.trim();
  const memo = document.getElementById("f-memo").value.trim();

  if (!date || !title){
    alert("日付と内容は必須です");
    return;
  }
  const pass = askPass();
  if (!pass) return;

  await addDoc(eventsRef, {
    date, team, eventType, title, memo,
    deleted: false,
    pass,
    createdAt: serverTimestamp()
  });

  document.getElementById("f-title").value = "";
  document.getElementById("f-memo").value = "";
});
```

- [ ] **Step 4: ブラウザで動作確認**

Browser preview toolで再読み込みし、フォームに日付・班・予定種別・内容を入力して「追加する」を押す。合言葉プロンプトが出るので `DA` と入力すると、年表に新しいカードがリアルタイムで追加されることを確認する（`read_console_messages`でエラーが出ていないことも確認）。誤った合言葉（例: `xx`）で「合言葉が違います」と表示され、Firestoreに書き込まれないことも確認する。

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "追加フォームと合言葉ゲートを実装"
```

---

### Task 5: 削除機能（合言葉ゲート・論理削除）

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: Task 4の`askPass()`, `db`, `doc`, `updateDoc`

- [ ] **Step 1: カードに削除ボタンを追加**

Task 3の`renderTimeline()`内、`card.innerHTML = ...`を以下に差し替え:

```javascript
    card.innerHTML = `
      <button class="delete-btn no-print" title="削除">✕</button>
      <div class="row">
        <span class="event-date">${escapeHtml(e.date)}</span>
        <span class="badge${t.textDark ? " pr-badge" : ""}">${t.label}</span>
        <span class="event-type">${escapeHtml(e.eventType)}</span>
      </div>
      <div class="event-title">${escapeHtml(e.title)}</div>
      ${e.memo ? `<div class="event-memo">${escapeHtml(e.memo)}</div>` : ""}
    `;
    card.querySelector(".delete-btn").addEventListener("click", () => deleteEvent(e.id));
```

- [ ] **Step 2: `<style>`に削除ボタンのスタイルを追加**

```css
.delete-btn{
  position:absolute;top:10px;right:10px;
  background:none;border:none;cursor:pointer;
  font-size:1rem;color:#bbb;
}
.delete-btn:hover{color:var(--da-red);}
```

- [ ] **Step 3: `deleteEvent`関数を追加**

`document.getElementById("f-submit").addEventListener(...)`ブロックの直後に追加:

```javascript
async function deleteEvent(id){
  const pass = askPass();
  if (!pass) return;
  if (!confirm("この出来事を削除しますか？")) return;
  await updateDoc(doc(db, "events", id), { deleted: true, pass });
}
```

- [ ] **Step 4: ブラウザで動作確認**

Task 4で追加したテストイベントの✕ボタンを押す。合言葉プロンプトで`DA`を入力し、確認ダイアログでOKを押すと、カードが年表から消えることを確認する。

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "削除機能（合言葉ゲート付き論理削除）を追加"
```

---

### Task 6: 印刷用CSS

**Files:**
- Modify: `index.html`

- [ ] **Step 1: `<style>`の末尾に`@media print`ブロックを追加**

```css
@media print{
  #filters, #add-form, .delete-btn, .no-print{display:none !important;}
  body{background:#fff;}
  main{max-width:100%;padding:0;}
  .event-card{break-inside:avoid;border:1px solid #ccc;}
}
```

- [ ] **Step 2: ブラウザで印刷プレビュー確認**

Browser preview toolで開いた状態で印刷プレビュー相当の確認を行う（`computer`ツールで`cmd+p`相当のショートカットは使えないため、`javascript_tool`で`window.matchMedia('print')`のスタイル適用を目視確認するか、`read_page`でフォーム・フィルターが印刷用スタイルでは非表示指定になっていることをCSS上で確認する）。年表カード（`.event-card`）に`break-inside:avoid`が効いていること、`#add-form`と`#filters`に`display:none`が適用されることを確認する。

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "印刷用CSSを追加"
```

---

### Task 7: GitHubリポジトリ作成・push・GitHub Pages公開

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: Task 1〜6で完成した`index.html`

- [ ] **Step 1: READMEを作成**

```markdown
# DA自治会 年表

DA自治会（学生支援班・アンケート班・広報班・プロジェクト・DA自治会推進委員会）の出来事・予定を記録する共有年表です。

- 公開URL: https://taku0o0o0.github.io/da-jichikai-nenpyo/
- データはFirebase Firestoreで共有・リアルタイム反映されます
- 出来事の追加・削除には合言葉が必要です
```

- [ ] **Step 2: コミット**

```bash
git add README.md
git commit -m "READMEを追加"
```

- [ ] **Step 3: GitHubリポジトリを作成してpush**

```bash
gh repo create taku0o0o0/da-jichikai-nenpyo --public --source=. --remote=origin --push
```

- [ ] **Step 4: GitHub Pagesを有効化**

```bash
gh api -X POST repos/taku0o0o0/da-jichikai-nenpyo/pages -f "source[branch]=main" -f "source[path]=/"
```

- [ ] **Step 5: 公開確認**

数分待ってから `https://taku0o0o0.github.io/da-jichikai-nenpyo/` をBrowser preview toolで開き、年表アプリが正しく表示されること、フォームからイベントを追加できることを確認する。
