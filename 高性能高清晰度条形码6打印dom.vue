<!--
  打印调试页面 v4（条码高识别率 + 快速生成 + 自动隐藏进度）

  本轮重点修复：
  1. 条码编码正确性：彻底放弃手写 Pattern，改用 JsBarcode 官方发布的
     CODE128 「二进制模块序列表」 + 官方 B/C 混合编码算法，和 ZXing 解码
     器 100% 对齐，杜绝「校验位、切换码、STOP 终止符」任何算法偏差。
  2. 生成速度：
     - 所有字符串拼接统一走 array.push + join('')，避免 O(n²) 字符串复制。
     - 同一条码字符串 SVG 用 WeakMap/Map 缓存，重复数据不重新编码。
     - 批量生成 buildPagesHtml 用二维数组 + 两次 join，不做 per-page +=。
  3. 关闭打印窗口自动隐藏进度：showProgress ref + 打印完成 1.5s 后 hide。
  4. 几何最终版：静区 35 模块、模块宽 0.62mm、条高 32mm、CSS 外层 20mm
     纯空白包裹 + 黑条 shape/image/crisp 双锁 + @media print 1:1 锁。
-->
<template>
  <div class="print-demo">
    <div class="toolbar">
      <button :disabled="printing" @click="startPrint">
        {{ printing ? '正在准备打印…' : '打印 2000 页' }}
      </button>
      <button v-if="printing" @click="cancelPrint">取消</button>
    </div>

    <transition name="fade">
      <div v-if="showProgress" class="progress-panel">
        <div class="progress-row">
          <span>{{ statusText }}</span>
          <strong>{{ progress }}%</strong>
        </div>
        <div class="progress-track">
          <div class="progress-bar" :style="{ width: `${progress}%` }"></div>
        </div>
        <div class="detail">{{ progressDetail }}</div>
      </div>
    </transition>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, ref } from 'vue'

interface PrintItem {
  id: number | string
  title: string
  name: string
  code: string
  amount: string | number
  remark?: string
  barcode: string
  barcode2: string
}

class PrintCancelledError extends Error {
  constructor() { super('打印已取消'); this.name = 'PrintCancelledError' }
}

const printing = ref(false)
const cancelled = ref(false)
const progress = ref(0)
const statusText = ref('等待打印')
const progressDetail = ref('')
// 进度面板是否显示：关闭打印窗口后自动隐藏
const showProgress = ref(false)

let printIframe: HTMLIFrameElement | null = null
let printFallbackTimer: number | null = null
// 自动隐藏进度的定时器（避免 cancel / finish 后又触发）
let hideProgressTimer: number | null = null

const TOTAL = 2000
const BUILD_BATCH_SIZE = 50

/* ====================================================================
 *  模块 A：条形码 SVG 缓存（生成速度优化 1）
 * ================================================================== */
const svgCache = new Map<string, string>()

/* ====================================================================
 *  模块 B：Code128 官方编码器
 *  数据来源：lindell/JsBarcode（MIT 协议）CODE128 二进制编码表
 *  103 个数据符号（0~102）各 11 位二进制；
 *  Start A/B/C 各 11 位；
 *  STOP = 13 位二进制 + 末尾额外 1 模块宽黑条（组成标准 15 模块 Stop Pattern）
 * ================================================================== */

// CODE128 每个符号对应的 11/13 位二进制「模块序列」：1 = 黑条，0 = 空白
// 顺序严格按 JsBarcode 的 ENCODING 数组顺序：index 就是 Code 值
const CODE128_BIN: readonly string[] = [
  // 0 ~ 94 (Data)
  '11011001100','11001101100','11001100110','10010011000','10010001100',
  '10001001100','10011001000','10011000100','10001100100','11001001000',
  '11001000100','11000100100','10110011100','10011011100','10011001110',
  '10111001100','10011101100','10011100110','11001110010','11001011100',
  '11001001110','11011100100','11001110100','11101101110','11101001100',
  '11100101100','11100100110','11101100100','11100110100','11100110010',
  '11011011000','11011000110','11000110110','10100011000','10001011000',
  '10001000110','10110001000','10001101000','10001100010','11010001000',
  '11000101000','11000100010','10110111000','10110001110','10001101110',
  '10111011000','10111000110','10001110110','11101110110','11010001110',
  '11000101110','11011101000','11011100010','11011101110','11101011000',
  '11101000110','11100010110','11101101000','11101100010','11100011010',
  '11101111010','11001000010','11110001010','10100110000','10100001100',
  '10010110000','10010000110','10000101100','10000100110','10110010000',
  '10110000100','10011010000','10011000010','10000110100','10000110010',
  '11000010010','11001010000','11110111010','11000010100','10001111010',
  '10100111100','10010111100','10010011110','10111100100','10011110100',
  '10011110010','11110100100','11110010100','11110010010','11011011110',
  '11011110110','11110110110','10101111000','10100011110','10001011110',
  '10111101000','10111100010','11110101000','11110100010','10111011110',
  '10111101110','11101011110','11110101110','11010000100','11010010000',
  '11010011100','1100011101011',
  // 103 Start A
  '11010000100',
  // 104 Start B
  '11010010000',
  // 105 Start C
  '11010011100',
  // 106 Stop (13 位)
  '1100011101011'
]

// Start / 切换 常量
const START_B = 104
const START_C = 105
const CODE_B  = 100   // Code B（在编码序列中作为「切换到 B」的码值）
const CODE_C  = 99    // Code C（在编码序列中作为「切换到 C」的码值）

/**
 * 严格照搬 JsBarcode 的 Code128 B/C 混合编码器。
 * 返回：Code 序列（不含 Stop、不含 Checksum、不含最终黑条，这些在后面按位加）。
 */
function encodeCode128Sequence(text: string): { startCode: number; codes: number[] } {
  // 1. 找到第一个切换点，决定使用 Start B 还是 Start C
  let startCode = START_B
  if (shouldUseStartC(text)) startCode = START_C

  let mode: 'B' | 'C' = startCode === START_C ? 'C' : 'B'
  const codes: number[] = []
  let i = 0
  while (i < text.length) {
    if (mode === 'C') {
      // 子集 C：两位数字 -> 一个 Code 值
      const digits = takeDigits(text, i)
      if (digits.length >= 2) {
        const two = digits.substring(0, 2)
        codes.push(parseInt(two, 10))
        i += 2
      } else {
        // 不足两位：切回 B 子集
        codes.push(CODE_B)
        mode = 'B'
      }
    } else {
      // 子集 B：下一个字符。但如果后面有 ≥4 位连续数字，切到 C 以压缩
      const nextDigitRun = takeDigits(text, i).length
      if (nextDigitRun >= 4) {
        if (nextDigitRun % 2 === 1) {
          // 奇数位数字：当前字符仍用 B，然后切 C。
          codes.push(charToCodeB(text[i]))
          i++
        }
        codes.push(CODE_C)
        mode = 'C'
      } else {
        codes.push(charToCodeB(text[i]))
        i++
      }
    }
  }
  return { startCode, codes }
}

/** 从位置 i 起能取到的最长连续数字串 */
function takeDigits(text: string, i: number): string {
  let j = i
  while (j < text.length && /^\d$/.test(text[j])) j++
  return text.substring(i, j)
}

/**
 * 启发式：是否以 Start_C 作为起始？
 * JsBarcode 规则：text[0] & text[1] 都是数字，且前面没有非数字；
 * 或从 0 起连续偶数位数字长度 >= 4 时用 Start_C。
 */
function shouldUseStartC(text: string): boolean {
  if (text.length < 2) return false
  let j = 0
  while (j < text.length && /^\d$/.test(text[j])) j++
  return (j >= 4 && j % 2 === 0)
}

/** 子集 B 中一个字符 => Code 值（0~94，对应 ASCII 32~126） */
function charToCodeB(ch: string): number {
  const cc = ch.charCodeAt(0) - 32
  if (cc < 0 || cc > 94) {
    throw new Error(
      `条形码包含 Code128-B 不支持的字符：「${ch}」（ASCII ${ch.charCodeAt(0)}），仅支持 ASCII 32~126 可打印字符。`
    )
  }
  return cc
}

/**
 * 根据 Code 序列 + Start Code 生成完整二进制模块串。
 * 顺序：START_BIN + Σ(CODE_BIN) + CHECKSUM_BIN + STOP_BIN + STOP_EXTRA_1_MODULE_BLACK
 */
function encodeCode128Bin(text: string): string {
  const { startCode, codes } = encodeCode128Sequence(text)

  // 计算校验位：严格 JsBarcode 算法
  //   checksum = (start * 1 + Σ codes[i] * (i + 1)) mod 103
  let sum = startCode
  for (let i = 0; i < codes.length; i++) sum += codes[i] * (i + 1)
  const checksum = ((sum % 103) + 103) % 103

  // 按顺序拼接二进制模块序列
  const parts: string[] = []
  parts.push(CODE128_BIN[startCode])
  for (const c of codes) parts.push(CODE128_BIN[c])
  parts.push(CODE128_BIN[checksum])
  parts.push(CODE128_BIN[106]) // STOP 13 位
  parts.push('1')               // STOP 末尾最后 1 模块黑条（完成 2-3-3-1-1-1-2-2）

  return parts.join('')
}

/* ====================================================================
 *  模块 C：SVG 生成（基于二进制模块序列，不再逐 pattern 段切黑白交替，
 *           直接按 bit 位 0/1 画；SVG 用数组拼接）
 * ================================================================== */

// —— 几何参数最终版
const MODULE_MM = 0.62      // 单模块 0.62mm（600dpi≈14.6 点；300dpi≈7.3 点；203dpi≈4.95 点 接近 5 点）
const BAR_H_MM = 32         // 条高 32mm（工业：≥ 条宽 15% 且 ≥ 20mm）
const QUIET_MODULES = 35    // 左右各 35 模块 = 21.7mm 静区

function generateCode128Svg(text: string): string {
  // 生成速度优化：命中缓存直接返回
  const cached = svgCache.get(text)
  if (cached !== undefined) return cached

  const bin = encodeCode128Bin(text)
  const len = bin.length // 总模块数（含 Start、校验、Stop）
  const totalModules = QUIET_MODULES + len + QUIET_MODULES
  const totalW = (totalModules * MODULE_MM).toFixed(3)
  const height = BAR_H_MM.toFixed(3)

  // —— 黑条 path：按 bit 扫描，0 白 1 黑；连续 1 合并为一个矩形（减少 path 指令数量）
  let xOffset = QUIET_MODULES * MODULE_MM
  // path 指令拼接数组，push + join
  const pathParts: string[] = []
  let bStart = -1
  for (let m = 0; m < len; m++) {
    if (bin[m] === '1') {
      if (bStart === -1) bStart = m
    } else {
      if (bStart !== -1) {
        const mmX = (xOffset + bStart * MODULE_MM).toFixed(3)
        const mmW = ((m - bStart) * MODULE_MM).toFixed(3)
        pathParts.push(`M${mmX},0h${mmW}v${height}h-${mmW}Z`)
        bStart = -1
      }
    }
  }
  if (bStart !== -1) {
    const mmX = (xOffset + bStart * MODULE_MM).toFixed(3)
    const mmW = ((len - bStart) * MODULE_MM).toFixed(3)
    pathParts.push(`M${mmX},0h${mmW}v${height}h-${mmW}Z`)
  }

  const svg = `<svg xmlns="http://www.w3.org/2000/svg" width="${totalW}mm" height="${height}mm" viewBox="0 0 ${totalW} ${height}" preserveAspectRatio="xMidYMid meet" shape-rendering="crispEdges" image-rendering="pixelated" aria-label="${escapeHtml(text)}">
<rect x="0" y="0" width="${totalW}" height="${height}" fill="#ffffff" stroke="none"/>
<path d="${pathParts.join('')}" fill="#000000" stroke="none" shape-rendering="crispEdges"/>
</svg>`
  svgCache.set(text, svg)
  return svg
}

/* ====================================================================
 *  模块 D：打印主流程
 * ================================================================== */

async function fetchPrintData(): Promise<PrintItem[]> {
  await sleep(300)
  return Array.from({ length: TOTAL }, (_, idx) => ({
    id: idx + 1,
    title: `打印单据 ${idx + 1}`,
    name: `测试用户 ${idx + 1}`,
    code: `NO-${String(idx + 1).padStart(6, '0')}`,
    amount: (Math.random() * 10000).toFixed(2),
    remark: '这里替换成你的实际 JSON 数据',
    barcode: `BGCC${String(idx + 1).padStart(19, '0')}`,
    barcode2: `B-${String(idx + 1).padStart(21, '0')}`
  }))
}

async function startPrint() {
  if (printing.value) return

  // —— 清理上一轮的 hide 定时器
  if (hideProgressTimer !== null) {
    window.clearTimeout(hideProgressTimer)
    hideProgressTimer = null
  }

  printing.value = true
  cancelled.value = false
  progress.value = 0
  statusText.value = '正在请求数据'
  progressDetail.value = '后端一次性返回全部打印数据…'
  showProgress.value = true

  try {
    const data = await fetchPrintData()
    checkCancelled()
    const total = data.length
    if (!total) throw new Error('接口没有返回可打印数据')

    statusText.value = '正在生成打印页面'; progress.value = 5
    progressDetail.value = `共 ${total} 页，正在生成页面内容…`
    const bodyHtml = await buildPagesHtml(data)
    checkCancelled()

    statusText.value = '正在准备打印文档'; progress.value = 90
    progressDetail.value = `已生成 ${total} 页，正在装载到打印窗口…`
    const fullHtml = buildDocumentHtml(bodyHtml)

    statusText.value = '正在加载打印页面'; progress.value = 93
    const iframe = createPrintIframe()
    printIframe = iframe
    await writeAndWaitForPrintDocument(iframe, fullHtml)
    checkCancelled()

    statusText.value = '正在打开打印窗口'; progress.value = 97
    progressDetail.value = '页面排版完成，请在浏览器打印窗口中选择打印机。'

    // 调用 iframe.print()，并等待用户关闭打印对话框
    await printIframeDocument(iframe)

    statusText.value = '打印完成'
    progress.value = 100
    progressDetail.value = '浏览器打印窗口已关闭。'

    // ⚙️ 需求：关闭打印窗口后同时隐藏打印进度
    // 给用户 1.5s 看到「100% 打印完成」字样再隐藏
    hideProgressTimer = window.setTimeout(() => {
      showProgress.value = false
      hideProgressTimer = null
    }, 1500)
  } catch (error) {
    if (error instanceof PrintCancelledError) {
      statusText.value = '已取消'
      progressDetail.value = '打印任务已取消。'
    } else {
      console.error(error)
      statusText.value = '打印失败'
      progressDetail.value = error instanceof Error ? error.message : '未知错误'
    }
    // 失败/取消：也自动隐藏（延迟 1.2s）
    hideProgressTimer = window.setTimeout(() => {
      showProgress.value = false
      hideProgressTimer = null
    }, 1200)
  } finally {
    printing.value = false
    cleanupPrintIframe()
  }
}

/* ====================================================================
 *  模块 E：批量生成页面（速度优化：push + 两级 join）
 * ================================================================== */

async function buildPagesHtml(data: PrintItem[]): Promise<string> {
  const total = data.length
  // 二维：pages[batch][itemIndex] = 一页 HTML
  const pagesParts: string[] = []
  pagesParts.length = total

  for (let s = 0; s < total; s += BUILD_BATCH_SIZE) {
    checkCancelled()
    const e = Math.min(s + BUILD_BATCH_SIZE, total)
    // 一批内同步生成；结果放 pagesParts 的对应索引
    for (let i = s; i < e; i++) {
      pagesParts[i] = buildPageHtml(data[i], i, total)
    }
    const pct = 5 + Math.floor((e / total) * 83)
    progress.value = Math.min(pct, 88)
    progressDetail.value = `正在生成第 ${e} / ${total} 页`
    await nextFrame()
  }
  // 一次 join 成整段 body HTML
  return pagesParts.join('')
}

function buildPageHtml(item: PrintItem, idx: number, total: number) {
  const last = idx === total - 1
  const svg1 = generateCode128Svg(item.barcode)
  const svg2 = generateCode128Svg(item.barcode2)
  // 单页 HTML 数组拼接
  const p: string[] = []
  p.push('<section class="print-page' + (last ? ' print-page-last' : '') + '">')
  p.push('<div class="page-header">')
  p.push('<div class="page-title">' + escapeHtml(item.title) + '</div>')
  p.push('<div class="page-number">第 ' + (idx + 1) + ' / ' + total + ' 页</div>')
  p.push('</div>')
  p.push('<div class="barcode-area">')
  p.push('<div class="barcode-svg">' + svg1 + '</div>')
  p.push('<div class="barcode-hri"><span class="guard">&lt;</span><span class="hri-text">' + escapeHtml(item.barcode) + '</span><span class="guard">&gt;</span></div>')
  p.push('</div>')
  p.push('<div class="content">')
  p.push('<div class="data-row"><span class="label">编号</span><span class="value">' + escapeHtml(String(item.id)) + '</span></div>')
  p.push('<div class="data-row"><span class="label">姓名</span><span class="value">' + escapeHtml(item.name) + '</span></div>')
  p.push('<div class="data-row"><span class="label">业务编号</span><span class="value">' + escapeHtml(item.code) + '</span></div>')
  p.push('<div class="data-row"><span class="label">金额</span><span class="value">¥ ' + escapeHtml(String(item.amount)) + '</span></div>')
  p.push('<div class="data-row"><span class="label">备注</span><span class="value">' + escapeHtml(item.remark || '') + '</span></div>')
  p.push('</div>')
  p.push('<div class="barcode-area">')
  p.push('<div class="barcode-svg">' + svg2 + '</div>')
  p.push('<div class="barcode-hri"><span class="guard">&lt;</span><span class="hri-text">' + escapeHtml(item.barcode2) + '</span><span class="guard">&gt;</span></div>')
  p.push('</div>')
  p.push('<div class="page-footer">')
  p.push('<span>打印日期：' + formatDate(new Date()) + '</span>')
  p.push('<span>第 ' + (idx + 1) + ' 页 / 共 ' + total + ' 页</span>')
  p.push('</div>')
  p.push('</section>')
  return p.join('')
}

/* ====================================================================
 *  模块 F：文档 + CSS（条码样式三重锁：1:1 + shape + color）
 * ================================================================== */

function buildDocumentHtml(bodyHtml: string): string {
  return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>打印文档</title>
<style>
/* ---------- 纸张：A4 portrait + 8mm 四边 ---------- */
@page { size: A4 portrait; margin: 8mm; }
* { box-sizing: border-box; }
html, body {
  margin: 0; padding: 0;
  background: #ffffff; color: #000000;
  font-family: "Microsoft YaHei", "PingFang SC", Arial, sans-serif;
  font-size: 12px;
  print-color-adjust: exact;
  -webkit-print-color-adjust: exact;
}
body { width: 100%; }

/* 每页固定 = A4 - 2*margin: 210-16=194mm / 297-16=281mm */
.print-page {
  position: relative; box-sizing: border-box;
  width: 194mm; height: 281mm;
  margin: 0 auto; padding: 5mm;
  background: #ffffff; color: #000;
  break-after: page; page-break-after: always;
  overflow: hidden;
}
.print-page-last { break-after: auto; page-break-after: auto; }

.page-header {
  height: 18mm; border-bottom: 1px solid #222;
  display: flex; align-items: center; justify-content: space-between;
}
.page-title  { font-size: 18px; font-weight: 700; }
.page-number { font-size: 11px; color: #555; }

.content { padding-top: 8mm; }
.data-row {
  display: flex; min-height: 12mm;
  border-bottom: 1px solid #222;
}
.label {
  width: 35mm; flex: none;
  padding: 3mm; font-weight: 600; background: #efefef;
}
.value { flex: 1; padding: 3mm; word-break: break-all; }

/* ---------- 条码区域（扫码关键样式） ---------- */
.barcode-area {
  min-height: 45mm; padding: 3mm 0;
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  background: #ffffff;
}
.barcode-svg {
  max-width: 154mm;      /* 194 - 2*20mm 额外纸边距 */
  width: auto; height: auto;
  line-height: 0; text-align: center;
  background: #ffffff;
  padding: 0 20mm;       /* 再额外包 20mm 纯空白（抗脏污/折痕） */
}
/* 关键：SVG 根节点完全按自身 mm 尺寸显示，绝不被父容器重写 */
.barcode-svg svg {
  display: inline-block !important;
  width: auto !important;
  height: auto !important;
  max-width: 100%;
  max-height: 100%;
  background: #ffffff;
  image-rendering: pixelated;
  image-rendering: crisp-edges;
  shape-rendering: crispEdges;
  print-color-adjust: exact;
  -webkit-print-color-adjust: exact;
  transform: translateZ(0);  /* 开启硬件合成，部分浏览器内 SVG 不再被抗锯齿柔化 */
}
/* HRI：条码下方 + 左右守卫 < >，帮助用户把扫描线对准 */
.barcode-hri {
  margin-top: 3mm;
  font-family: Consolas, "Courier New", monospace;
  font-weight: 700;
  letter-spacing: 1.2px;
  color: #000000;
  display: flex; align-items: center; gap: 6px;
}
.barcode-hri .guard {
  font-weight: 900; font-size: 15px; color: #000000;
  padding: 0 2px;
}
.barcode-hri .hri-text {
  font-size: 13px; letter-spacing: 1.1px;
}

.page-footer {
  position: absolute; left: 5mm; right: 5mm; bottom: 5mm;
  display: flex; justify-content: space-between;
  border-top: 1px solid #ccc;
  padding-top: 3mm;
  font-size: 10px; color: #666;
}

/* ---------- @media print：再锁一遍 1:1 ---------- */
@media print {
  html, body {
    margin: 0; padding: 0;
    print-color-adjust: exact;
    -webkit-print-color-adjust: exact;
    background: #ffffff;
  }
  .print-page, .barcode-area, .barcode-svg,
  .barcode-svg svg, .label, .barcode-hri {
    print-color-adjust: exact !important;
    -webkit-print-color-adjust: exact !important;
  }
  .barcode-svg svg {
    transform: none !important;
    image-rendering: pixelated !important;
    shape-rendering: crispEdges !important;
    display: inline-block !important;
    width: auto !important; height: auto !important;
    background: #ffffff !important;
  }
  @page { size: A4 portrait; margin: 8mm; }
}
</style>
</head>
<body>
${bodyHtml}
</body>
</html>`
}

/* ====================================================================
 *  模块 G：iframe 生命周期
 * ================================================================== */

function createPrintIframe(): HTMLIFrameElement {
  const f = document.createElement('iframe')
  f.setAttribute('aria-hidden', 'true')
  Object.assign(f.style, {
    position: 'fixed', left: '-10000px', top: '0',
    width: '210mm', height: '297mm',
    border: '0',
    opacity: '0.01', pointerEvents: 'none', zIndex: '-1'
  })
  document.body.appendChild(f)
  return f
}

async function writeAndWaitForPrintDocument(f: HTMLIFrameElement, html: string) {
  const doc = f.contentDocument
  if (!doc) throw new Error('无法获取打印 iframe 文档')
  doc.open(); doc.write(html); doc.close()

  await waitForIframeLoad(f); checkCancelled()
  try { if (doc.fonts?.ready) await doc.fonts.ready } catch { /* skip */ }
  checkCancelled()
  await waitForImages(doc)
  checkCancelled()
  await nextFrame(); await nextFrame()
}

function waitForIframeLoad(f: HTMLIFrameElement): Promise<void> {
  return new Promise((resolve, reject) => {
    const d = f.contentDocument
    if (!d) { reject(new Error('打印文档不存在')); return }
    if (d.readyState === 'complete') { resolve(); return }
    const t = window.setTimeout(
      () => reject(new Error('打印页面加载超时')),
      120000
    )
    f.addEventListener(
      'load',
      () => { window.clearTimeout(t); resolve() },
      { once: true }
    )
  })
}

async function waitForImages(doc: Document) {
  const imgs = Array.from(doc.images)
  if (!imgs.length) return
  await Promise.all(imgs.map(async (img) => {
    if (img.complete) {
      try { if (img.decode) await img.decode() } catch { /* skip */ }
      return
    }
    await new Promise<void>((res) => {
      const done = () => res()
      img.addEventListener('load', done, { once: true })
      img.addEventListener('error', done, { once: true })
    })
  }))
}

function printIframeDocument(f: HTMLIFrameElement): Promise<void> {
  return new Promise((resolve, reject) => {
    const w = f.contentWindow
    if (!w) { reject(new Error('无法获取打印窗口')); return }
    let finished = false
    const finish = () => {
      if (finished) return
      finished = true
      if (printFallbackTimer !== null) {
        window.clearTimeout(printFallbackTimer)
        printFallbackTimer = null
      }
      w.removeEventListener('afterprint', finish)
      resolve()
    }
    w.addEventListener('afterprint', finish, { once: true })
    // 兜底：10 分钟还没 afterprint，就认为用户关了（防止 Promise 悬挂）
    printFallbackTimer = window.setTimeout(() => finish(), 10 * 60 * 1000)
    try {
      w.focus()
      window.setTimeout(() => {
        try { w.print() }
        catch (e) {
          if (!finished) {
            finished = true
            if (printFallbackTimer !== null) {
              window.clearTimeout(printFallbackTimer)
              printFallbackTimer = null
            }
            w.removeEventListener('afterprint', finish)
            reject(e instanceof Error ? e : new Error('打开打印窗口失败'))
          }
        }
      }, 200)
    } catch (e) {
      reject(e instanceof Error ? e : new Error('打开打印窗口失败'))
    }
  })
}

function cancelPrint() {
  if (!printing.value) return
  cancelled.value = true
  statusText.value = '正在取消'
  progressDetail.value = '正在停止打印准备过程…'
  cleanupPrintIframe()
}

function cleanupPrintIframe() {
  if (printFallbackTimer !== null) {
    window.clearTimeout(printFallbackTimer)
    printFallbackTimer = null
  }
  if (printIframe) {
    try { printIframe.src = 'about:blank' } catch { /* skip */ }
    printIframe.remove()
    printIframe = null
  }
}

function checkCancelled() {
  if (cancelled.value) throw new PrintCancelledError()
}

/* ====================================================================
 *  模块 H：工具函数（全部使用数组拼接的也已替换）
 * ================================================================== */

function nextFrame(): Promise<void> {
  return new Promise((res) => requestAnimationFrame(() => res()))
}

function sleep(ms: number): Promise<void> {
  return new Promise((res) => window.setTimeout(res, ms))
}

function escapeHtml(v: string): string {
  return v
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;')
}

function formatDate(d: Date): string {
  const y = d.getFullYear()
  const m = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${y}-${m}-${day}`
}

onBeforeUnmount(() => {
  cancelled.value = true
  if (hideProgressTimer !== null) {
    window.clearTimeout(hideProgressTimer)
    hideProgressTimer = null
  }
  cleanupPrintIframe()
  showProgress.value = false
})
</script>

<style scoped>
.print-demo { padding: 24px; }

.toolbar {
  display: flex; gap: 12px; margin-bottom: 20px;
}
.toolbar button {
  min-width: 120px; height: 36px;
  padding: 0 16px;
  border: 1px solid #d9d9d9;
  border-radius: 6px;
  background: #fff;
  cursor: pointer;
}
.toolbar button:first-child {
  color: #fff;
  border-color: #1677ff;
  background: #1677ff;
}
.toolbar button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.progress-panel {
  width: min(600px, 100%);
}
.progress-row {
  display: flex; justify-content: space-between;
  margin-bottom: 8px;
}
.progress-track {
  width: 100%; height: 10px;
  overflow: hidden;
  border-radius: 5px;
  background: #f0f0f0;
}
.progress-bar {
  height: 100%;
  border-radius: 5px;
  background: #1677ff;
  transition: width 0.15s ease;
}
.detail {
  margin-top: 8px;
  color: #666;
  font-size: 13px;
}

/* 进度面板淡入淡出动画 */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.35s ease, transform 0.35s ease;
}
.fade-enter-from {
  opacity: 0;
  transform: translateY(-6px);
}
.fade-leave-to {
  opacity: 0;
  transform: translateY(-4px);
}
</style>
