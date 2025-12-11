# 浏览器指纹检测项、原理与代码片段

本文档汇总 CreepJS 在 `src/creep.ts` 中汇集的各类指纹信号，逐项说明检测原理并标出实际使用的 JavaScript 代码位置（节选自仓库源码）。

## contentWindow / Window 属性集合
- **原理**：枚举 `window` 上的自有属性与前缀特征（moz / webkit / apple），用于区分内核和可能的环境伪装。
- **代码片段**（`src/window/index.ts`）：
```ts
const win = PHANTOM_DARKNESS || window
let keys = Object.getOwnPropertyNames(win)
  .filter((key) => !/_|\d{3,}/.test(key))
const moz = keys.filter((key) => (/moz/i).test(key)).length
const webkit = keys.filter((key) => (/webkit/i).test(key)).length
```

## Navigator / UA、平台与硬件参数
- **原理**：比对 `navigator` 与 worker Scope 中的 UA、platform、deviceMemory 等信息，识别谎报与指纹特征。
- **代码片段**（`src/navigator/index.ts`）：
```ts
let lied = (
  lieProps['Navigator.appVersion'] ||
  lieProps['Navigator.deviceMemory'] ||
  lieProps['Navigator.userAgent'] ||
  lieProps['Navigator.plugins'] ||
  lieProps['Navigator.mimeTypes']
) || false
const { platform, userAgent, hardwareConcurrency } = navigator
```

## CSS System Styles & Computed Styles
- **原理**：枚举 `getComputedStyle` / `HTMLElement.style` / `CSSRuleList.style` 的属性键，并读取系统色值以揭示渲染引擎差异。
- **代码片段**（`src/css/index.ts`）：
```ts
const cssStyleDeclaration = (
  type == 'getComputedStyle' ? getComputedStyle(document.body) :
    type == 'HTMLElement.style' ? document.body.style :
      type == 'CSSRuleList.style' ? document.styleSheets[0].cssRules[0].style :
        undefined
)
const colors = [
  'ActiveBorder','ActiveCaption','ActiveText','AppWorkspace','Background',
  'ButtonBorder','ButtonFace','ButtonHighlight','ButtonShadow','ButtonText',
  'Canvas','CanvasText','CaptionText','Field','FieldText','GrayText',
  'Highlight','HighlightText','InactiveBorder','InactiveCaption',
  'InactiveCaptionText','InfoBackground','InfoText','LinkText','Mark',
  'MarkText','Menu','MenuText','Scrollbar','ThreeDDarkShadow','ThreeDFace',
  'ThreeDHighlight','ThreeDLightShadow','ThreeDShadow','VisitedText',
  'Window','WindowFrame','WindowText',
  // 其余系统色值见源码完整列表
]
```

## CSS 媒体特征（Media Queries）
- **原理**：使用 `matchMedia` 和动态注入样式，测试设备宽高、色域、指针、首选配色方案等媒体特征。
- **代码片段**（`src/cssmedia/index.ts`）：
```ts
const query = ({ body, type, rangeStart, rangeLen }) => {
  const html = [...Array(rangeLen)]
    .map((_, i) => `@media(device-${type}:${i+rangeStart}px){body{--device-${type}:${i+rangeStart};}}`)
    .join('')
  body.innerHTML = `<style>${html}</style>`
  return getComputedStyle(body).getPropertyValue(`--device-${type}`).trim()
}
const matchMediaCSS = {
  'prefers-color-scheme': (
    matchMedia('(prefers-color-scheme: light)').matches ? 'light' :
      matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : undefined
  ),
  orientation: (
    matchMedia('(orientation: landscape)').matches ? 'landscape' :
      matchMedia('(orientation: portrait)').matches ? 'portrait' : undefined
  ),
}
```

## HTMLElement 版本
- **原理**：遍历 `document.documentElement` 的可枚举键，观察 DOM 接口差异。
- **代码片段**（`src/document/index.ts`）：
```ts
const keys = []
for (const key in document.documentElement) {
  keys.push(key)
}
```

## Screen & matchMedia 校验
- **原理**：读取 `screen` 宽高、可用区域、色深与 `devicePixelRatio`，并用 `matchMedia` 交叉验证是否存在欺骗。
- **代码片段**（`src/screen/index.ts`）：
```ts
const { width, height, availWidth, availHeight, colorDepth, pixelDepth } = window.screen
const matchMediaLie = !matchMedia(
  `(device-width: ${width}px) and (device-height: ${height}px)`,
).matches
const hasLiedDPR = !matchMedia(`(resolution: ${window.devicePixelRatio}dppx)`).matches
```

## DOMRect / ClientRects & Emoji 布局
- **原理**：绘制旋转、透视的元素与 emoji，分别调用 `getClientRects`/`getBoundingClientRect`，比较矩形数值与位移差异以检测伪装。
- **代码片段**（`src/domrect/index.ts`）：
```ts
const getBestRect = (el: Element) => {
  let range
  if (!lieProps['Element.getClientRects']) {
    return el.getClientRects()[0]
  } else if (!lieProps['Element.getBoundingClientRect']) {
    return el.getBoundingClientRect()
  } else if (!lieProps['Range.getClientRects']) {
    range = DOC.createRange()
    range.selectNode(el)
    return range.getClientRects()[0]
  }
  range = DOC.createRange()
  range.selectNode(el)
  return range.getBoundingClientRect()
}
```

## JS Runtime：Math 精度
- **原理**：对 `Math.acos`/`Math.cos`/`Math.hypot` 等函数多次取值比较，若同一输入返回不一致则记为伪装。
- **代码片段**（`src/math/index.ts`）：
```ts
const check = [
  'acos','acosh','asin','asinh','atan','atanh','atan2','cbrt','cos','cosh',
  'expm1','exp','hypot','log','log1p','log10','sin','sinh','sqrt','tan','tanh','pow',
]
check.forEach((prop) => {
  const res1 = Math[prop](...test)
  const res2 = Math[prop](...test)
  const matching = isNaN(res1) && isNaN(res2) ? true : res1 == res2
  if (!matching) {
    lied = true
    const mathLie = `expected x and got y`
    documentLie(`Math.${prop}`, mathLie)
  }
})
```

## JS Engine：Console 错误模式
- **原理**：构造一组语法/运行时错误，收集 `Error.message` 文本差异以刻画引擎与本地化。
- **代码片段**（`src/engine/index.ts`）：
```ts
const errorTests = [
  () => new Function('alert(")')(),
  () => new Function('const foo;foo.bar')(),
  () => new Function('(1).toString(1000)')(),
]
const errors = getErrors(errorTests)
```

## Canvas 2D（随机像素 / 文本 / Emoji）
- **原理**：在多张 Canvas 间复制随机像素，比较噪声差异；同时绘制文本、emoji 与阴影，输出 DataURL 作为指纹。
- **代码片段**（`src/canvas/index.ts`）：
```ts
const canvas1 = document.createElement('canvas')
const canvas2 = document.createElement('canvas')
context1.fillRect(x, y, 1, 1)        // 写入随机像素
const { data: [r,g,b,a] } = context1.getImageData(x, y, 1, 1) || {}
context2.fillRect(x, y, 1, 1)        // 复制并读取差分
const patternDiffs = [...pattern1].map((_, i) => pattern1[i] != pattern2[i])
```

## WebGL 参数与渲染输出
- **原理**：采集大量 `gl.getParameter` 返回值、渲染像素与着色器输出，计算 GPU 相关哈希。
- **代码片段**（`src/webgl/index.ts`）：
```ts
const getParamNames = () => [
  'MAX_TEXTURE_SIZE','MAX_VIEWPORT_DIMS','SHADING_LANGUAGE_VERSION',
  'VENDOR','RENDERER','VERSION','MAX_VERTEX_ATTRIBS','MAX_TEXTURE_IMAGE_UNITS',
  'MAX_FRAGMENT_UNIFORM_VECTORS','MAX_RENDERBUFFER_SIZE','MAX_SAMPLES',
  // 其余 WebGL getParameter 常量见源码完整列表
].sort()
const draw = (gl) => { gl.clear(gl.COLOR_BUFFER_BIT) /* 触发渲染并读取像素 */ }
```

## 媒体能力与 MIME
- **原理**：枚举常见音视频 MIME，分别调用 `Audio/Video.canPlayType`、`MediaSource.isTypeSupported` 与 `MediaRecorder.isTypeSupported` 记录支持矩阵。
- **代码片段**（`src/media/index.ts`）：
```ts
const mimeTypes = getMimeTypeShortList()
const types = mimeTypes.reduce((acc, type) => {
  acc.push({
    mimeType: type,
    audioPlayType: audioEl.canPlayType(type),
    videoPlayType: videoEl.canPlayType(type),
    mediaSource: MediaSource.isTypeSupported(type),
    mediaRecorder: MediaRecorder.isTypeSupported(type),
  })
  return acc
}, [])
```

## WebRTC 设备与 SDP 能力
- **原理**：调用 `navigator.mediaDevices.enumerateDevices()` 获取设备类型列表，并通过 `RTCPeerConnection` 解析 SDP 中的编解码与扩展。
- **代码片段**（`src/webrtc/index.ts`）：
```ts
export async function getWebRTCDevices() {
  if (!navigator?.mediaDevices?.enumerateDevices) return null
  return navigator.mediaDevices.enumerateDevices().then((devices) =>
    devices.map((device) => device.kind).sort()
  )
}
const getMediaCapabilities = async () => navigator.mediaCapabilities.decodingInfo(config)
```

## 字体与 Emoji 渲染
- **原理**：通过 `FontFace`/`document.fonts.check` 探测已安装字体，再测量大号 emoji 的布局尺寸，推断系统与版本。
- **代码片段**（`src/fonts/index.ts`）：
```ts
const fontFaceList = fontList.map((font) => new FontFace(font, `local("${font}")`))
const responseCollection = await Promise.allSettled(fontFaceList.map((font) => font.load()))
const emojiSet = [...doc.getElementsByClassName('pixel-emoji')]
  .reduce((set, el, i) => set.add(getComputedStyle(el).inlineSize), new Set())
```

## Speech 合成声源
- **原理**：等待 `speechSynthesis` 服务加载，读取本地/远程声源、语言与默认声源，检测返回为空或语言不一致的情况。
- **代码片段**（`src/speech/index.ts`）：
```ts
await new Promise((resolve) => setTimeout(resolve, 50))
const data = speechSynthesis.getVoices()
const local = data.filter((x) => x.localService).map((x) => x.name)
const languages = [...new Set(data.map((x) => x.lang))]
```

## Offline AudioContext（音频指纹）
- **原理**：使用 `OfflineAudioContext` 生成并渲染波形，读取频域/时域数据与压缩器增益，捕捉浮点差异和硬件噪声。
- **代码片段**（`src/audio/index.ts`）：
```ts
const context = new OfflineAudioContext(1, bufferLen, 44100)
const analyser = context.createAnalyser()
const oscillator = context.createOscillator()
oscillator.frequency.value = 10000
oscillator.connect(dynamicsCompressor)
dynamicsCompressor.connect(analyser)
context.startRendering()
```

## SVG 文本与 BBox
- **原理**：在隐藏容器中插入 emoji SVG 文本，读取 `getBBox()` 与 `getComputedTextLength()`，检测字体渲染与几何异常。
- **代码片段**（`src/svg/index.ts`）：
```ts
patch(divElement, html`
  <svg><g id="svgBox">
    ${EMOJIS.map((emoji) => `<text x="32" y="32" class="svgrect-emoji">${emoji}</text>`).join('')}
  </g></svg>
`)
const bBox = svgBox.getBBox()
const dimensions = ''+el.getComputedTextLength()
```

## 时区与城市枚举
- **原理**：遍历大量 IANA 时区字符串，利用 `Intl.DateTimeFormat` 格式化结果确认系统时区并检测时区降级。
- **代码片段**（`src/timezone/index.ts`）：
```ts
const cities = ['UTC','GMT','Etc/GMT+0','Africa/Abidjan', /* ...其余时区见源码 */]
const getOffset = (tz) => new Intl.DateTimeFormat('en', { timeZone: tz }).format(0)
// 完整 IANA 名称列表见源码，逐个格式化用于比对偏移
```

## Intl 本地化指纹
- **原理**：调用 `Intl.*.resolvedOptions()` 与格式化结果，收集 locale、日期、数字、列表与相对时间的本地化输出。
- **代码片段**（`src/intl/index.ts`）：
```ts
const locale = constructors.reduce((acc, name) => {
  const obj = new intl[name]; const { locale } = obj.resolvedOptions() || {}
  return [...acc, locale]
}, [])
const dateTimeFormat = new Intl.DateTimeFormat(undefined,{ month:'long', timeZoneName:'long' }).format(963644400000)
const numberFormat = new Intl.NumberFormat(undefined,{ notation:'compact', compactDisplay:'long' }).format(21000000)
```

---
以上检测结果会在 `src/creep.ts` 中并行收集，并通过哈希汇总形成整体浏览器指纹。
