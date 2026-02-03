# sml-webscraping
1. introduction
Record a web scraping task in 2025 end. With assistance of AI.

AI model: Copilot Auto

Objective url: https://rfidlab.org/alec-submissionform/

Objective: collect full retailer list


/***** ① 基本配置（如页面不同，请修改选择器） *****/
const INPUT_SELECTOR = '#retailer1';                 // 自动补全输入框
const CONTAINER_SELECTOR = '#suggestionsContainer1'; // 建议列表容器（可滚动）
const OPTION_SELECTOR_CANDIDATES = [
  '#suggestionsContainer1 [role="option"]',
  '#suggestionsContainer1 .suggestion, #suggestionsContainer1 .item',
  '#suggestionsContainer1 li',
  '#suggestionsContainer1 div' // 兜底，会再过滤空文本
];

/***** ② 只做单字母 a–z *****/
const LETTERS = 'abcdefghijklmnopqrstuvwxyz'.split('');

/***** ③ 时序参数（可按站点情况微调） *****/
const TYPE_DELAY_MS = 60;      // 模拟逐字输入延时
const WAIT_DEBOUNCE_MS = 500;  // 输入后等待站点防抖/网络返回
const BETWEEN_QUERIES_MS = 150;// 每次查询之间的间隔
const SCROLL_ROUNDS = 8;       // 滚动次数（应对懒加载）
const SCROLL_DELAY_MS = 150;   // 每次滚动后等待

/***** ④ 工具函数 *****/
const sleep = (ms) => new Promise(r => setTimeout(r, ms));
function getEl(sel){ const el = document.querySelector(sel); if(!el) throw new Error(`找不到元素：${sel}`); return el; }
function isVisible(el){
  const cs = getComputedStyle(el);
  if (cs.display === 'none' || cs.visibility === 'hidden' || cs.opacity === '0') return false;
  const rect = el.getBoundingClientRect();
  return rect.width > 0 && rect.height > 0;
}
function extractOptionTexts(container) {
  let nodes = [];
  for (const sel of OPTION_SELECTOR_CANDIDATES) {
    nodes = Array.from(container.querySelectorAll(sel)).filter(isVisible);
    if (nodes.length) break;
  }
  if (!nodes.length) {
    // 兜底：取容器里所有可见“叶子节点”
    nodes = Array.from(container.querySelectorAll('*')).filter(el => isVisible(el) && el.children.length === 0);
  }
  const texts = nodes
    .map(el => (el.innerText ?? el.textContent ?? '').trim())
    .filter(Boolean);
  return texts;
}
async function scrollContainerToBottom(container) {
  let last = -1;
  for (let i = 0; i < SCROLL_ROUNDS; i++) {
    const h = container.scrollHeight;
    if (h === last) break;
    last = h;
    container.scrollTop = h;
    await sleep(SCROLL_DELAY_MS);
  }
}
async function ensureSuggestionsOpen() {
  const cont = getEl(CONTAINER_SELECTOR);
  for (let i = 0; i < 20; i++) {
    if (isVisible(cont) && cont.children.length > 0) return cont;
    await sleep(80);
  }
  return cont;
}
async function clearAndType(input, text) {
  input.focus();
  // 清空并触发监听
  input.value = '';
  input.dispatchEvent(new Event('input', { bubbles: true }));
  await sleep(50);
  // 模拟逐字输入
  for (const ch of text.split('')) {
    input.value += ch;
    input.dispatchEvent(new Event('input', { bubbles: true }));
    input.dispatchEvent(new KeyboardEvent('keyup', { bubbles: true, key: ch }));
    await sleep(TYPE_DELAY_MS);
  }
}
async function typeAndCollect(query) {
  const input = getEl(INPUT_SELECTOR);
  await clearAndType(input, query);
  await sleep(WAIT_DEBOUNCE_MS);
  const container = await ensureSuggestionsOpen();
  await scrollContainerToBottom(container);
  // 滚动后再取一次，避免漏项
  return extractOptionTexts(container);
}

/***** ⑤ 主流程：仅跑 a–z *****/
(async () => {
  const all = new Set();
  let done = 0;

  console.log('▶ 开始：单字母查询 a–z …');
  for (const q of LETTERS) {
    try {
      const items = await typeAndCollect(q);
      items.forEach(t => all.add(t));
      done++;
      if (done % 5 === 0) console.log(`…进度 ${done}/26，当前收集 ${all.size} 条`);
      await sleep(BETWEEN_QUERIES_MS);
    } catch (e) {
      console.warn(`查询 "${q}" 出错：`, e);
      await sleep(500);
    }
  }

  /***** ⑥ UTF‑8 + BOM 导出（避免 Excel 乱码） *****/
  // HTML 实体解码
  const decodeHtml = (s) => { const el = document.createElement('textarea'); el.innerHTML = s; return el.value; };
  // 规范化：去首尾空白、合并空白、Unicode NFC
  const result = Array.from(all)
    .map(s => decodeHtml(String(s).trim()).replace(/\s+/g,' ').normalize('NFC'))
    .filter(Boolean)
    .sort((a,b)=>a.localeCompare(b));

  // 生成 CSV 文本
  const csv = 'Retailer\n' + result.map(v => `"${v.replace(/"/g,'""')}"`).join('\n');

  // **关键**：加上 BOM，明确 UTF‑8，Excel 直接双击打开不会乱码
  const BOM = '\uFEFF';
  const blob = new Blob([BOM + csv], { type: 'text/csv;charset=utf-8;' });

  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = 'retailers.csv'; a.click();
  URL.revokeObjectURL(url);

  console.log(`✅ 完成！仅单字母收集，共 ${result.length} 条，已下载 retailers.csv（UTF‑8+BOM）`);
})();










<img width="1916" height="959" alt="image" src="https://github.com/user-attachments/assets/6e32352c-c184-47d5-84fc-40933449304d" />


Notes:
.js 前端本地過濾
.php 後端服務器拉取數據
本次task的retailer list屬於不敏感，少改動的數據，更適合作爲常量放在前端直接讀取處理
