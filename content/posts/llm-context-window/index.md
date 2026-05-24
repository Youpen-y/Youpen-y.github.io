---
title: "LLM context window and tokens"
date: 2026-05-23T10:58:58+08:00
draft: false
tags: ["ai"]
comments: true
---
大模型的上下文窗口（context window）指模型在生成响应时可以参考的所有文本，包括响应本身。它以词元（tokens）为单位，词元不是单词，也不是字符，而是模型分词器生成的单元。通过下方示例查看输入的分词与对应的ID：

<style>
.tokenizer-app {
  --tk-bg: #fff;
  --tk-border: #d0d7de;
  --tk-input-bg: #f6f8fa;
  --tk-text: #1f2328;
  --tk-muted: #656d76;
  --tk-tab-bg: #f3f4f6;
  --tk-tab-active: #fff;
  --tk-tab-active-text: #111;
  --tk-tab-text: #555;
  border: 1px solid var(--tk-border);
  border-radius: 14px;
  padding: 16px;
  font-family: system-ui, -apple-system, sans-serif;
  background: var(--tk-bg);
  color: var(--tk-text);
  margin: 1.5em 0;
}
@media (prefers-color-scheme: dark) {
  .tokenizer-app {
    --tk-bg: #0d1117;
    --tk-border: #30363d;
    --tk-input-bg: #161b22;
    --tk-text: #e6edf3;
    --tk-muted: #8b949e;
    --tk-tab-bg: #21262d;
    --tk-tab-active: #30363d;
    --tk-tab-active-text: #fff;
    --tk-tab-text: #c9d1d9;
  }
}
.tk-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  font-size: 13px;
  gap: 12px;
}
.tk-select {
  appearance: none;
  padding: 5px 28px 5px 10px;
  border: 1px solid var(--tk-border);
  border-radius: 8px;
  background: var(--tk-input-bg);
  color: var(--tk-text);
  font-size: 13px;
  font-family: inherit;
  cursor: pointer;
  outline: none;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%23656d76' d='M3 4.5l3 3 3-3'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 8px center;
}
.tk-select:focus {
  border-color: #58a6ff;
  box-shadow: 0 0 0 3px rgba(31,111,235,0.12);
}
.tk-count { font-weight: 600; font-variant-numeric: tabular-nums; white-space: nowrap; }

.tk-textarea {
  width: 100%;
  min-height: 72px;
  padding: 10px 12px;
  border: 1px solid var(--tk-border);
  border-radius: 8px;
  background: var(--tk-input-bg);
  color: var(--tk-text);
  font-family: ui-monospace, SFMono-Regular, "Cascadia Code", monospace;
  font-size: 14px;
  line-height: 1.5;
  resize: vertical;
  box-sizing: border-box;
  outline: none;
  transition: border-color 0.15s;
}
.tk-textarea:focus {
  border-color: #58a6ff;
  box-shadow: 0 0 0 3px rgba(31,111,235,0.12);
}

/* segmented tabs */
.tk-tabs {
  position: relative;
  display: inline-flex;
  background: var(--tk-tab-bg);
  border-radius: 10px;
  padding: 3px;
  gap: 2px;
  margin-top: 12px;
}
.tk-tabs input[type="radio"] { display: none; }
.tk-slider {
  position: absolute;
  top: 3px; left: 3px;
  width: calc(50% - 3px);
  height: calc(100% - 6px);
  background: var(--tk-tab-active);
  border-radius: 8px;
  transition: transform 0.25s ease;
  box-shadow: 0 1px 2px rgba(0,0,0,0.08), 0 1px 1px rgba(0,0,0,0.04);
}
#tk-tab-ids:checked ~ .tk-tabs .tk-slider { transform: translateX(100%); }
.tk-tabs label {
  position: relative; z-index: 1;
  padding: 5px 14px;
  cursor: pointer;
  font-size: 13px;
  color: var(--tk-tab-text);
  user-select: none;
  transition: color 0.2s;
}
#tk-tab-colored:checked ~ .tk-tabs label[for="tk-tab-colored"],
#tk-tab-ids:checked    ~ .tk-tabs label[for="tk-tab-ids"] {
  color: var(--tk-tab-active-text);
  font-weight: 600;
}

/* output */
.tk-output {
  margin-top: 10px;
  padding: 10px 12px;
  border: 1px solid var(--tk-border);
  border-radius: 8px;
  background: var(--tk-input-bg);
  font-family: ui-monospace, SFMono-Regular, "Cascadia Code", monospace;
  font-size: 14px;
  line-height: 1.8;
  min-height: 40px;
  word-break: break-all;
}
.tk-tokens-view,
.tk-ids-view { display: none; }
#tk-tab-colored:checked ~ .tk-output .tk-tokens-view { display: block; }
#tk-tab-ids:checked    ~ .tk-output .tk-ids-view     { display: block; }

.tk-token {
  display: inline-block;
  padding: 2px 4px;
  margin: 1px;
  border-radius: 3px;
  white-space: pre-wrap;
  line-height: 1.7;
}

.tk-loading {
  color: var(--tk-muted);
  animation: tk-pulse 1.5s ease-in-out infinite;
}
@keyframes tk-pulse {
  0%, 100% { opacity: 0.5; }
  50%      { opacity: 1;   }
}
.tk-error { color: #e74c3c; }
</style>

<div class="tokenizer-app">
  <input type="radio" id="tk-tab-colored" name="tk-view" checked>
  <input type="radio" id="tk-tab-ids" name="tk-view">
  <div class="tk-header">
    <select id="tk-model" class="tk-select">
      <option value="gpt-4o">GPT-4o · o200k_base</option>
      <option value="glm-5.1">GLM-5.1</option>
      <option value="deepseek-v4">DeepSeek-V4-Pro</option>
      <option value="kimi-k2.6">Kimi-K2.6</option>
    </select>
    <span class="tk-count" id="tk-count">Loading…</span>
  </div>
  <textarea id="tk-input" class="tk-textarea" rows="3" spellcheck="false"
    placeholder="Type or paste text here…">Hello world</textarea>
  <div class="tk-tabs">
    <div class="tk-slider"></div>
    <label for="tk-tab-colored">Tokens</label>
    <label for="tk-tab-ids">Token IDs</label>
  </div>
  <div class="tk-output">
    <div class="tk-tokens-view" id="tk-tokens-view">
      <span class="tk-loading">Loading tokenizer…</span>
    </div>
    <div class="tk-ids-view" id="tk-ids-view"></div>
  </div>
</div>

<script type="module">
const $ = id => document.getElementById(id);
const select     = $('tk-model');
const input      = $('tk-input');
const countEl    = $('tk-count');
const tokensView = $('tk-tokens-view');
const idsView    = $('tk-ids-view');

const cache = new Map();
let currentModel = select.value;

/* ── Color helpers ────────────────────────────── */
function tokenColor(i) {
  const hue = (i * 47 + 15) % 360;
  const dark = window.matchMedia('(prefers-color-scheme: dark)').matches;
  return dark
    ? { bg: `hsl(${hue},45%,22%)`, fg: `hsl(${hue},55%,72%)` }
    : { bg: `hsl(${hue},65%,88%)`, fg: `hsl(${hue},75%,25%)` };
}
function esc(s) {
  return s.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
}

/* ── Unified tokenizer interface ──────────────── */
function makeTiktokenWrapper(enc) {
  return {
    encode(text) {
      const ids = Array.from(enc.encode(text));
      const texts = ids.map(id => enc.decode([id]));
      return { ids, texts };
    }
  };
}

async function loadGPT4o() {
  const { encodingForModel } = await import('https://esm.sh/js-tiktoken@1');
  return makeTiktokenWrapper(encodingForModel('gpt-4o'));
}

async function loadHF(repo) {
  const { AutoTokenizer } = await import('https://esm.sh/@huggingface/transformers@3');
  const hf = await AutoTokenizer.from_pretrained(repo);
  return {
    encode(text) {
      const result = hf(text);
      const ids = Array.from(result.input_ids.data).map(Number);
      const texts = ids.map(id => hf.decode([id], { skip_special_tokens: false }).trimStart());
      return { ids, texts };
    }
  };
}

async function loadKimi() {
  const [{ Tiktoken }, resp] = await Promise.all([
    import('https://esm.sh/js-tiktoken@1'),
    fetch('https://huggingface.co/moonshotai/Kimi-K2.6/resolve/main/tiktoken.model'),
  ]);
  const raw = await resp.text();
  // Convert per-line "base64 rank" → js-tiktoken single-line "char 0 b64_0 b64_1 …"
  const b64s = raw.trim().split('\n').map(l => l.split(' ')[0]);
  const bpeRanks = '! 0 ' + b64s.join(' ');
  const p = String.fromCharCode(92);
  const patStr =
    `[${p}p{Script=Han}]+|` +
    `[^${p}r${p}n${p}p{L}${p}p{N}]?[${p}p{Lu}${p}p{Lt}${p}p{Lm}${p}p{Lo}${p}p{M}]*[${p}p{Ll}${p}p{Lm}${p}p{Lo}${p}p{M}]+(?:'s|'S|'t|'T|'re|'rE|'Re|'RE|'ve|'vE|'Ve|'VE|'m|'M|'ll|'lL|'Ll|'LL|'d|'D)?|` +
    `[^${p}r${p}n${p}p{L}${p}p{N}]?[${p}p{Lu}${p}p{Lt}${p}p{Lm}${p}p{Lo}${p}p{M}]+[${p}p{Ll}${p}p{Lm}${p}p{Lo}${p}p{M}]*(?:'s|'S|'t|'T|'re|'rE|'Re|'RE|'ve|'vE|'Ve|'VE|'m|'M|'ll|'lL|'Ll|'LL|'d|'D)?|` +
    `${p}p{N}{1,3}|` +
    ` ?[^${p}s${p}p{L}${p}p{N}]+[${p}r${p}n]*|` +
    `${p}s*[${p}r${p}n]+|` +
    `${p}s+(?!${p}S)|` +
    `${p}s+`;
  const enc = new Tiktoken(
    { pat_str: patStr, bpe_ranks: bpeRanks, special_tokens: {} },
    {}
  );
  return makeTiktokenWrapper(enc);
}

async function getTokenizer(id) {
  if (cache.has(id)) return cache.get(id);
  switch (id) {
    case 'gpt-4o':     var tok = await loadGPT4o(); break;
    case 'glm-5.1':    var tok = await loadHF('zai-org/GLM-5.1'); break;
    case 'deepseek-v4': var tok = await loadHF('deepseek-ai/DeepSeek-V4-Pro'); break;
    case 'kimi-k2.6':  var tok = await loadKimi(); break;
    default: throw new Error('Unknown model: ' + id);
  }
  cache.set(id, tok);
  return tok;
}

/* ── Render ───────────────────────────────────── */
function render(result) {
  if (!result) {
    countEl.textContent = '—';
    tokensView.innerHTML = '';
    idsView.textContent = '';
    return;
  }
  const { ids, texts } = result;
  countEl.textContent =
    ids.length + ' token' + (ids.length !== 1 ? 's' : '') +
    ' · ' + input.value.length + ' char' + (input.value.length !== 1 ? 's' : '');
  tokensView.innerHTML = texts.map((t, i) => {
    const c = tokenColor(i);
    const vis = esc(t).replace(/\n/g, '⏎\n').replace(/\t/g, '⇥');
    return `<span class="tk-token" style="background:${c.bg};color:${c.fg}">${vis || '&nbsp;'}</span>`;
  }).join('');
  idsView.textContent = '[' + ids.join(', ') + ']';
}

/* ── Events ───────────────────────────────────── */
let tokenizeTask = Promise.resolve();

async function tokenize() {
  const model = select.value;
  if (model !== currentModel) {
    tokensView.innerHTML = '<span class="tk-loading">Loading…</span>';
    idsView.textContent = '';
    currentModel = model;
  }
  const text = input.value;
  if (!text) { render(null); return; }
  try {
    const tok = await getTokenizer(model);
    // Guard against stale results if model changed while loading
    if (select.value !== model) return;
    render(tok.encode(text));
  } catch (e) {
    console.error('Tokenize error:', e);
    tokensView.innerHTML = '<span class="tk-error">⚠ ' + esc(e.message) + '</span>';
    idsView.textContent = '';
  }
}

select.addEventListener('change', tokenize);
input.addEventListener('input', tokenize);
window.matchMedia('(prefers-color-scheme: dark)')
  .addEventListener('change', () => { if (input.value) tokenize(); });

tokenize();
</script>
下面是不同模型采用分词库的一些指标：
| 指标 | GPT-4o | GLM-5.1 | DeepSeek-V4 | Kimi-K2.6 |
|---|---|---|---|---|
| 词表大小 | 200k | 155k | 128k | 164k |
| 英文占比 | 67.2% | 64.8% | 55.8% | 49.0% |
| 中文占比 | 3.8% | 18.5% | 27.6% | 42.5% |
| 覆盖汉字数 | 2539 | 4223 | 5258 | 4933 |
| 3字词 | 826 | 5893 | 7168 | 18252 |
| 4字词 | 377 | 1925 | 2357 | 9895 |
| 5字+词 | 399 | 414 | 343 | 2034 |
| 平均汉字数/token | 1.92 | 2.20 | 2.22 | 2.58 |
| 平均英文字母数/token | 5.47 | 5.71 | 5.66 | 5.48 |

值得注意的是，GPT-4o 的中文多字词质量很差，充斥大量网络垃圾文本（如赌博、色情网站等词汇）。

经验法则：

> 对于英文文本，一个 token 通常对应大约 4-5 个字符 ≈ 3/4 个单词，因此 100 个 tokens 相当于 75 个单词。

> 对于中文文本，一个 token 通常对应大约两个汉字，100 个 tokens 大约200多汉字。

自2023年来，模型的上下文窗口经历了从`数千 → 数十万 → 百万` tokens 的演变，其中 Gemini 3 Pro，Llama 4 Scout 甚至达到了千万层级。目前先进大模型的上下文窗口数在 1M tokens左右，如果按照经验法则的话，大概相当于：
- 75 万单词或 200 万汉字
- 150 篇论文（5000单词/篇）
- 10本+ 书（20万字/本）

更大的上下文窗口允许模型处理更复杂、更长的提示，但上下文并非越多越好。一方面，研究发现模型的“宣称的窗口大小”和其“有效利用能力”之间存在差距 —— 当相关信息出现在输入的开头或结尾时，模型表现最佳，而位于中间的信息则往往利用不足（**Lost in the Middle**）。另一方面，随着上下文中词元数量增加，模型的准确率和召回率会逐步下降，即所谓的 **Context Rot** 现象。