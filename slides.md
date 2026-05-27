---
theme: default
title: プロンプト確認地獄からの脱出
info: |
  ガードレール整備と sandbox 有効化で実現したこと
class: text-center title-slide
highlighter: shiki
background: '#1e1e1e'
colorSchema: dark
transition: slide-left
fonts:
  sans: 'Noto Sans JP, Hiragino Sans, sans-serif'
  mono: 'Menlo, Consolas, monospace'
defaults:
  layout: default
---

<style>
.slidev-layout {
  background-color: #1e1e1e;
  color: #e0e0e0;
  font-family: 'Noto Sans JP', sans-serif;
}
.slidev-layout h1 {
  color: #fbbf24;
  border-bottom: 2px solid #fbbf24;
  padding-bottom: 0.2em;
}
.slidev-layout h2 {
  color: #60a5fa;
}
.slidev-layout.title-slide {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
.slidev-layout.title-slide h1 {
  width: 92%;
  margin: 0 0 1.8rem;
}
.slidev-layout.title-slide h3 {
  margin: 0;
}
.slidev-layout.title-slide .talk-meta {
  margin-top: 4rem;
}
.learning-body {
  width: fit-content;
  max-width: 88%;
  margin: 4.5rem auto 0;
  text-align: left;
}
.slidev-layout strong {
  color: #fbbf24;
}
.slidev-layout table {
  margin: 0 auto;
}
.slidev-layout th {
  color: #fbbf24;
  background: #1f2937;
}
.slidev-layout blockquote {
  background: #1f2937;
  border-left: 4px solid #60a5fa;
  color: #cbd5e1;
}
.term {
  background: #0a0a0a;
  border: 2px solid #f59e0b;
  border-radius: 8px;
  padding: 1em 1.5em;
  font-family: 'Menlo', monospace;
  font-size: 0.85em;
  white-space: pre;
  line-height: 1.5;
}
.term-ok { border-color: #10b981; }
.term-err { border-color: #ef4444; }
/* スライドピッカー(右上のドロップダウン)を非表示 */
.autocomplete-list {
  display: none !important;
}
</style>

# プロンプト確認地獄からの脱出

### ガードレール整備と sandbox 有効化で実現したこと

<div class="talk-meta text-gray-400">

Lightning Talk - 2026-05-26

</div>

---

# Claude Code を devcontainer で使ってる方へ

<pre class="term max-w-2xl mx-auto mt-8">
Bash command (unsandboxed)

git status
Show working tree status

Do you want to proceed?
❯ 1. Yes
  2. No
</pre>

<v-click>

<div class="mt-8 text-center text-yellow-400 text-xl">

↑ これ、毎回 Yes 押してませんか?<br>
　 サブエージェント、止まっていませんか?

</div>

</v-click>

---

# 原因調査: allowlist 登録済でも確認が発生

<div class="term max-w-4xl mx-auto mt-6 text-sm">
<div>プロンプトでコマンド指示</div>
<div v-click>└─ allowlist には登録済</div>
<div v-click>&nbsp;&nbsp;&nbsp;└─ sandbox 内なら自動許可されるはず</div>
<div v-click>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└─ そのために bwrap が隔離環境を作ろうとした</div>
<div v-after class="text-xs text-gray-400">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;※ bwrap = bubblewrap: Linux 名前空間で軽量 sandbox を作るツール</div>
<div v-click>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└─ devcontainer の権限不足で失敗</div>
<div v-click>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└─ sandbox 初期化に失敗</div>
<div v-click>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└─ unsandboxed 実行へフォールバック</div>
<div v-click>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└─ sandbox 自動許可の対象外になる</div>
<div v-click>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└─ コマンドの確認が発生</div>
</div>

<div class="max-w-4xl mx-auto mt-7 text-sm text-gray-300 leading-relaxed">
<div v-click class="text-yellow-400 font-bold mb-1">まとめ</div>

<div v-click>
<span class="text-yellow-400 font-bold">①</span>
&nbsp;Claude Code は、指示された Bash コマンドを sandbox 内で実行するため、<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Docker Desktop VM 内の Linux で動く OS レベル隔離ツール bwrap に隔離環境を作らせようとした。<br>
</div>

<div v-click class="mt-1">
<span class="text-yellow-400 font-bold">②</span>
しかし devcontainer の権限不足で bwrap が隔離環境を初期化できず、<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;sandbox 実行が成立しないためコマンドの確認が発生した。<br>
</div>
</div>

---

#  状況整理: どの層が壊れていたのか

<div class="grid grid-cols-2 gap-8 mt-8">

<div class="p-5 border-l-4 border-blue-400 bg-slate-800">

## 実行前に止める

<div v-click class="mt-4 text-lg">
<strong>① permissions.deny</strong><br>
Claude Code の権限ルールで止める
</div>

<div v-click class="mt-5 text-lg">
<strong>② PreToolUse hook</strong><br>
独自チェックで危険パターンを止める
</div>

</div>

<div class="p-5 border-l-4 border-emerald-400 bg-slate-800">

## 実行時に閉じ込める

<div v-click class="mt-4 text-lg">
<strong>③ sandbox 制御層</strong><br>
sandbox 方針・許可フローを適用する
</div>

<div v-click class="mt-5 text-lg">
<strong>④ bwrap OS レベル隔離</strong><br>
Linux の機能でプロセスを実際に隔離する
</div>

</div>

</div>

<div v-click class="mt-8 text-center text-xl text-yellow-400 font-bold">
④ が失敗すると、sandbox 内で実行できず<br>
自動許可の対象外になって確認が出る
</div>

<div v-click class="term term-err max-w-2xl mx-auto mt-6">
bwrap: Can't mount proc on /newroot/proc
       Operation not permitted
</div>

---

# 原因: sandbox 実隔離が成立していない

## Claude Code を中心とした 4 層防御

<div class="max-w-3xl mx-auto mt-4">

| 層 | タイミング | 名称 | 状態 |
|:-:|:-:|---|:-:|
| ① | 実行前 | permissions.deny | <span v-click>✅ 有効</span> |
| ② | 実行前 | PreToolUse hook での検査 | <span v-click>✅ 有効</span> |
| ③ | 実行時 | sandbox 制御層 | <span v-click>⚠️ 設定上は有効</span> |
| ④ | 実行時 | **bwrap OS レベル隔離** | <span v-click>**❌ 失敗**</span> |

</div>

<div v-click class="mt-4 text-center">

③ 設定は ✅有効 / ④ 実隔離は ❌失敗

</div>

<div v-click class="mt-1 text-center">

→ unsandboxed 実行へフォールバック

</div>

<div v-click class="mt-1 text-center">

→ **allowlist による自動許可が効かず、コマンドの確認が発生**

</div>

---

# AI で詰めた結論

## まず考えた解決策: コンテナに権限を足す

<div class="max-w-4xl mx-auto mt-6">

| 観点 | 内容                                                                                                 |
|---|----------------------------------------------------------------------------------------------------|
| 目的 | <span v-click>allowlist 済コマンドを sandbox 内で確認なく実行させたい</span>                                         |
| 観察 | <span v-click>sandbox 実体の <strong>bwrap</strong> が <code>/proc</code> mount で失敗</span>             |
| 推論 | <span v-click><code>mount</code> には <strong>CAP_SYS_ADMIN</strong> が必要では?</span>                   |
| 行動 | <span v-click><code>docker-compose.yml（自分用）</code>に<code>cap_add: SYS_ADMIN</code> を足す計画を作成</span> |
| 疑問 | <span v-click><strong>CAP_SYS_ADMIN は コンテナで root 相当の特権を付与する。コンテナ境界を弱めるのでは?</strong></span>         |

</div>

<div v-click class="mt-4 max-w-3xl mx-auto p-3 text-center text-base text-yellow-300 border-l-4 border-yellow-400 bg-slate-800/60 leading-relaxed">
これはセキュリティリスクの観点で、容認できない対応では？
</div>

---

# 転機: 公式リファレンスを読み直す

## AI の結論を疑う

<div v-click class="max-w-3xl mx-auto p-4 bg-slate-800 border-l-4 border-blue-400 italic">

"enableWeakerNestedSandbox mode... enables it to work inside Docker environments without privileged namespaces..."<br>
<span class="text-sm text-gray-400">— Claude Code docs: Sandboxing</span><br>
<span class="text-xs text-gray-500">https://code.claude.com/docs/en/sandboxing</span>

</div>

<div v-click class="mt-5 text-center text-sm text-gray-300 leading-relaxed">
非特権 Docker では、強い bwrap 隔離に必要な <code>/proc</code> mount ができない
</div>

<div v-click class="mt-2 text-center text-sm text-gray-300 leading-relaxed">
だから新規 namespace を作りつつ <code>/proc</code> remount を skip する「弱い nested sandbox」に切り替える
</div>

<div v-click class="mt-6 text-center text-xl text-yellow-400 font-bold">
インフラ変更 + 権限拡張（SYS_ADMIN）せずに&nbsp;&nbsp; settings.json 1 行追加&nbsp;&nbsp;で済む？
</div>

<div v-click class="mt-4 max-w-3xl mx-auto p-3 text-center text-base text-green-300 border-l-4 border-green-400 bg-slate-800/60 leading-relaxed">
→ Docker socket 経由など、bwrap が直接防げない経路はガードレールで塞ぐ
</div>

---

# ガードレール整備 + 設定 1 行で解決

<div class="grid grid-cols-2 gap-6 mt-2">

<div>

<div class="text-center text-blue-400 font-bold mb-2">Before</div>

<pre class="term text-xs h-60">
❯ git status

● Bash(git status)
⎿ Error: bwrap: Can't mount
  proc on /newroot/proc

Bash command (unsandboxed)
Do you want to proceed?
❯ 1. Yes  2. No
</pre>

<div class="text-center mt-2 text-sm">😩 毎回確認 / サブエージェント停止</div>

</div>

<div v-click>

<div class="text-center text-blue-400 font-bold mb-2">After</div>

<pre class="term term-ok text-xs h-60">
❯ git status

● Bash(git status)
⎿ On branch main
  Your branch is up to date
  nothing to commit

✓ 即実行
</pre>

<div class="text-center mt-2 text-sm">⚡ 確認なし / 流れるように動く</div>

</div>

</div>

<div class="mt-6 text-sm">

<div v-click>
<strong>やったこと:</strong>
</div>

<div v-click>
<strong>①</strong> ガードレール: <code>permissions.deny</code> + <code>PreToolUse hook での検査</code> に必要なコマンドを追加<br>
（docker socket 経由など bwrap が直接防げない経路を補完）
</div>

<div v-click>
<strong>②</strong> 設定 1 行: <code>settings.json</code> に <code>enableWeakerNestedSandbox: true</code>
</div>

</div>

---

# 学び

<div class="learning-body">

<div class="text-lg my-5">
<v-click><span class="text-yellow-400 font-bold text-2xl mr-2">①</span>
インフラ・権限の変更を考える前に、<strong>設定変更で対応できないか疑う</strong><br></v-click>
<v-click><span class="ml-10">→ 副作用が限定的でロールバックも容易</span></v-click>
</div>

<div class="text-lg my-5">
<v-click><span class="text-yellow-400 font-bold text-2xl mr-2">②</span>
単独では穴が残る。<strong>ガードレールと sandbox を組み合わせる</strong><br></v-click>
<v-click><span class="ml-10">→ ガードレールは Claude Code、アプリケーションレベルの防御<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;sandbox は Docker Desktop VM 内、OSレベルの防御</span></v-click>
</div>

<div class="text-lg my-5">
<v-click><span class="text-yellow-400 font-bold text-2xl mr-2">③</span>
結論を確定する前に、<strong>公式ドキュメントを読み直す</strong><br></v-click>
<v-click><span class="ml-10">→ AI で論理の深掘りは進むが、探索範囲は固定されがち</span></v-click>
</div>

</div>
