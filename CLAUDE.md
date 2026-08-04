# CLAUDE.md — 廁所互動小網頁（交接文件）

> 每次開啟專案先讀本檔掌握全貌。需求細節見 [需求規格.md](需求規格.md)。
> **本檔是「重置後也能無縫接手」的依據**：含現況、程式架構速查、關鍵座標/參數、測試方式、部署流程。
>
> 🔗 線上版：**https://yiying-toilet.netlify.app** ｜ GitHub：`changyiying0425/toilet-web`（public）

---

## 1. 專案是什麼

桌機 + 手機用互動小網頁：一個**跟隨游標 / 手指、前後帶括號的英文字**（`(URGENCY)` 等），隨所在區域切換，透過「移動 → 提示 → 點擊 / 長按」走完廁所流程（尿急→開門→尿尿→沖水→洗手→解脫）。

- 視覺：**Dot Art 黑白點點風**（場景圖使用者提供，互動層程式繪製）。
- 情緒主軸：常駐字 `(URGENCY)`，**沖水完成後全站永久 `(RELIEF)`**（`state.relieved`）。
- 文字格式：**所有顯示字前後都加括號** `()`。

## 2. 場景地圖（5 圖 / 4 場景，B 為中樞）

```
A 門 door.jpg      → 點門把(OPEN) → 漸暗 → B
B 全景 hub.jpg     → (TOILET)右→C馬桶 ／ (SINK)左→D洗手台
C 馬桶 toilet-closed/open.jpg：(LIFT)掀蓋 → 長按(PEE)堆積 → 空白鍵/點鈕沖水(螺旋捲入) → (RELIEF)
D 洗手台 sink.jpg：水龍頭(OPEN/CLOSE)出水 → WATER流向排水口消失 → 撥水物理
子場景左上角 (← BACK) 返回；離開 C/D 會自動停音效/重置。
```

## 3. 目前狀態（2026-08-05）— **基本完成，進行手機微調**

**全部完成：** A/B/C/D 四場景與互動、游標視覺系統、字色依背景切換、(URGENCY) 緊急效果、全套 7 音效（已平衡至 ≈ −25dBFS）、GitHub 版控、**Netlify 上線（push 自動部署）**、規格書已同步。

**手機觸控支援（剛合併進 main，⚠️ 待使用者手機實測）：**
- 方案 B「直接點」：偵測觸控裝置 → `body.touch`；touch 事件對應（跟隨文字跟手指、輕點觸發熱區、按住馬桶尿尿、拖曳撥水）。
- 沖水手機改「**點此沖水**」按鈕（桌機仍空白鍵）；直向顯示「請橫置」。
- **下一步：使用者在手機測，回報後在 main 上微調（點擊判定、尿尿/撥水手感、沖水鈕）。桌機邏輯不受影響（touch 只在觸控時啟動）。**

**選配未做：** 手機體驗更進一步優化（目前為堪玩版）。

## 4. 程式架構速查（全部在單一 `index.html` 的一個 IIFE 內）

**場景/游標核心**
- `.scene` 四個 div（`#scene-door/hub/toilet/sink`），一個 `.active`；`data-src`=背景圖（供取樣）。
- `setCursor(word)`：設文字 + 依 word==='URGENCY' 切 `.urgent`。`idleWord()`：relieved→RELIEF／toilet.opened→PEE／else URGENCY。
- 字色：`getSampler/inkFor/applyInk`（背景縮 `SAMPLE_W=140` 取樣，`THRESHOLD=128`，暗→白字 `body.inverted`）。
- 熱區：`.hot` 用**百分比** inline style 定位；`wireHotspot(el,onClick)`=hover 換 `data-label`+glow、click 動作。
- 轉場：`goScene(fromId,toId,afterReveal)` 漸暗 520ms；離開 sink→`resetSink()`、離開 toilet→`resetToilet()`。
- 視覺：`#cursor::before`=柔光；`.glow`=2px 反色外框；`.urgent::before`=紅光脈動 `urgentPulse`；`#urgency-burst`/`.ubit`=小字噴發（`spawnUrgencyBit`，setInterval 110ms，用 `void offsetWidth` 強制 reflow 讓靜止時也噴）。

**場景 C 馬桶（關鍵常數在該 JS 區塊，改前先在檔案確認實際值）**
- `BOWL {cx,cy,rx,ry}`＝綠色橢圓（PEE 夾限範圍）；`DRAIN {x,y}`＝藍色最低點（堆積中心）。
- 熱區：`hot-lift`（掀蓋）、`hot-bowl`（長按尿尿，範圍大）。**無沖水把手** → 沖水＝`keydown Space`（`particles.length>=FLUSH_MIN`）或手機點 `#flush-hint`。
- 物理：`spawnPee`（拋物線飛向 DRAIN，`SPEED`/`ARC`）、`step`、`inPile/settleAt`（`fillR` 由內往外長到 `FILL_MAX`，`PILE_STRETCH` y 拉長）。生成 `setInterval(spawnPee,85)`。
- `flush()`：螺旋捲入 DRAIN（`flushRaf`，`DUR=2500`），結束 `stopFlushSound()`+`playRelief()`。
- 可調：`SPEED ARC FILL_MAX PILE_STRETCH MAX_PEE FLUSH_MIN` + 生成間隔。

**場景 D 洗手台**
- `SPOUT`（出水口）、`SDRAIN`（綠色最低點）、`SINK {cx,cy,rx,ry}`（水槽橢圓）。
- 水流：`spawnWater/stepWater` 朝 SDRAIN 流、`clampSink` 夾限、到排水口淡出。撥水斥力 `MOUSE_R/MOUSE_F`；引力/阻尼 `W_ACC/W_DAMP`。
- 水龍頭 `hot-faucet` 切 OPEN/CLOSE；`resetSink()` 停水/清空/關龍頭。

**音效（JS 音效區塊）**
- `audio()`=Web Audio ctx；`route(el,gain)`=HTML5 音效導入增益節點（可放大）；`playRelief()`=合成大三和弦。
- 元素與增益：door 0.36 / pee(loop) 1.2 / flush 1.62(+`stopFlushSound`) / click 1.2 / lid 1.0 / sink＝Web Audio 緩衝迴圈 gain 1.74（`loopStart/loopEnd` 去頭尾靜音）。
- **全部 ≈ −25 dBFS**。要重新平衡：用 `decodeAudioData` 量 RMS，`gain = 10^((-25 - rms)/20)`（點擊類短音別拉滿，避免破音）。
- 檔名有空格者 HTML 用 `%20`（如 `sink%20sound.mp3`）。

**手機觸控（JS 末段「觸控支援」區塊）**
- `TOUCH = matchMedia('(hover:none) and (pointer:coarse)')` + 首次 touchstart 動態切換 → `body.touch`。
- `touchApply`（更新游標/字/取樣、回傳手指下熱區）；tap→`hot.dispatchEvent(click)`；按住 `hot-bowl`→`startPour`；拖曳→`lastFrac` 更新→撥水。點 `#flush-hint`（.on 時 `pointer-events:auto`）→`flush()`。
- `#rotate-hint`：`@media (orientation:portrait)` 顯示。

## 5. 如何測試 / 驗證（重要，避免踩雷）

**本地：**
```bash
cd "C:/AI/claude/網頁嘗試" && python -m http.server 8123
```
開 `http://localhost:8123/index.html`。**改完務必 `Ctrl+Shift+R` 強制重載**（會吃快取）。按 `D`＝顯示熱區框 + 右下即時 `X% Y%` 座標。

**⚠️ 瀏覽器面板（mcp__Claude_Browser__）的限制：**
面板常「未顯示」→ `getBoundingClientRect()` 回 **0**、`requestAnimationFrame`/CSS transition **凍結**、截圖失敗。**所以無法在面板裡實測互動物理/動畫/座標**。可行的驗證：
- 讀 **console error**（應為 0）、**network** 狀態（資產 200）。
- 跑 **JS 檢查**：dispatch 合成事件、讀 DOM/class/dataset/狀態。
- 需要「看得到的實測」→ **請使用者在自己瀏覽器/手機測並回報**（截圖）。

**熱區/座標校正流程：** 使用者在截圖畫框 → 我用「已知%的 debug 框當比例尺」反推，或請他把滑鼠移到角落唸右下的 `X% Y%`。座標一律用**百分比**（隨舞台縮放）。

## 6. 部署 / 版控

- GitHub `changyiying0425/toilet-web`（public）；Netlify 站 `yiying-toilet` 已連 main。
- **push 到 main → Netlify 自動部署**（約 30–60s）。`netlify.toml`：publish 根目錄、無 build。
- commit：**繁中訊息** + 結尾加 `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。
- git 身分：`git -c user.name="changyiying0425" -c user.email="yiying0425@gmail.com" commit ...`。

## 7. 與原規格的主要差異（實作已演進，規格書已同步）

- **沖水**：原「點沖水把手」→ 改**空白鍵 / 手機點鈕**（點點圖看不出把手）。
- **洗手台**：原「噴濺出界 0.5 秒消失」→ 改**水限制在槽內、流向排水口後消失**。
- **游標視覺**：平常無外框、進可點區才加 2px 反色外框 + 同色柔光；(URGENCY) 另有紅光脈動 + 小字噴發。

## 8. 開發約定

- 動程式前先對照 `需求規格.md`；有需求變更**先更新該文件再改碼**。
- 熱區用百分比、避免過小熱區（點點風下優先大區 + 文字引導）。
- 每完成一階段就 commit + push（= 自動上線）。改動盡量「加法」，勿破壞既有桌機/場景邏輯。
