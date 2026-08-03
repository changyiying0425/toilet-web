# CLAUDE.md — 廁所互動小網頁

> 專案給 Claude 的常駐說明檔。每次開啟專案先讀本檔掌握進度，需求細節見 [需求規格.md](需求規格.md)。

## 專案是什麼

桌機用互動小網頁：一個**跟隨滑鼠、前後帶括號的英文字**（如 `(URGENCY)`、`(OPEN)`），隨游標所在區域切換，透過「移動 → 提示 → 點擊 / 長按」引導體驗者走完一段廁所流程（尿急 → 開門 → 尿尿 → 沖水 → 洗手 → 解脫）。

- **視覺風格**：Dot Art 黑白點點藝術風（場景圖為使用者提供，互動層由程式繪製）。
- **情緒主軸**：常駐字 `(URGENCY)`，完成沖水後全站永久轉為 `(RELIEF)`。
- **文字格式**：所有顯示文字前後一律加括號 `()`。

## 場景地圖（5 圖 / 4 場景，B 為中樞 HUB）

```
A 門 door.jpg → (OPEN) 點門 → 漸暗 →
B 全景 hub.jpg（含馬桶+洗手台）
   ├ 馬桶區 (TOILET) → C
   └ 洗手台區 (SINK) → D
C 馬桶 toilet-closed.jpg / toilet-open.jpg：(LIFT) 掀蓋 → (PEE) 長按堆積 → (FLUSH) 旋轉沖走 → 完成轉 (RELIEF)
D 洗手台 sink.jpg：(OPEN/CLOSE) 水龍頭出水，撥水物理，噴濺出界 0.5 秒消失
子場景左上角固定 (← BACK) 返回全景。
```

## 檔案結構

| 檔案 | 說明 |
|------|------|
| `index.html` | 主程式（單一檔，含 HTML/CSS/JS） |
| `door.jpg` `hub.jpg` `toilet-closed.jpg` `toilet-open.jpg` `sink.jpg` | 5 張場景圖（16:9，Dot Art 風） |
| `cursor.png` | BACK 區塊用的像素箭頭鼠標（24×34，白底黑邊） |
| `door sound.mp3` `pee sound.mp3` `flush sound.mp3` `sink sound.mp3` | 使用者提供音效（開門 / 尿尿循環 / 沖水 / 洗手出水） |
| `需求規格.md` | 完整需求規格（唯一需求依據） |
| `CLAUDE.md` | 本檔 |

## 如何預覽

本地起靜態伺服器（互動需在瀏覽器操作）：

```bash
cd "C:/AI/claude/網頁嘗試" && python -m http.server 8123
```

開 `http://localhost:8123/index.html`。按 `D` 可切換熱區外框（debug）。

## 目前進度（2026-07-30）

- [x] 需求確認 + 規格書
- [x] 5 張場景圖到位（16:9；尺寸不足 1920×1080 但點點風可接受）
- [x] **場景 A 門**：游標跟隨、`(URGENCY)`↔`(OPEN)`、點擊漸暗轉場、debug 熱區
- [x] 字體 **DotGothic16**；文字約 18px、描邊 0.75px
- [x] **字色依背景深淺即時切換**：背景圖縮小為 140px 離屏 canvas 取樣，忽略單點、只看區塊明暗；暗→白字、亮→黑字（`inkFor()` / `applyInk()`，門檻 THRESHOLD=128）
- [x] 門把熱區調到使用者指定位置（40.5% / 46% / 7% / 13.5%）
- [x] **場景 B 全景 HUB**：`(TOILET)`（右）/`(SINK)`（左）兩區，點擊進子場景
- [x] 場景切換系統 `goScene()`：漸暗淡入；C 馬桶 / D 洗手台先作預覽（互動待建）
- [x] **三個 `(← BACK)`**：左上角常駐文字，hub→門、toilet/sink→hub（`data-target`）
- [x] **游標視覺系統**：平常無外框；進可點區 → 文字加 2px 反色外框 + 同色柔光（`#cursor.glow` + `::before`）；文字中心對齊滑鼠 (0,0)（`translate(-50%,-50%)`）
- [x] **BACK 區塊**：滑入時隱藏跟隨文字、顯示像素箭頭鼠標 `cursor.png`（`body.over-back`）
- [x] **場景 C 馬桶**：掀蓋 `(LIFT)` 漸暗換 `toilet-open.jpg`、常駐字→`(PEE)`
- [x] 馬桶內長按 → `(PEE)` **從游標直線射向藍區最低點 `DRAIN` 並堆積**（`fillR` 向外填、夾限在綠色 `BOWL` 橢圓內）；字級 13–23px、黃色
- [x] 沖水改為 **`(PEE)` 達 `FLUSH_MIN` 後按空白鍵** → 旋轉沖走 → 永久 `(RELIEF)`（無沖水把手）
- [x] `(PEE)` 拋物線落向藍區（`SPEED`/`ARC`）；沖水改為**螺旋捲入排水口**（`flush()` rAF 動畫）
- [x] **音效**：開門/尿尿/沖水/洗手皆用使用者音檔；沖完解脫大三和弦（合成）
- [x] **音效工程**：各音效經 Web Audio 增益統一至 ≈ −25 dBFS；sink 用緩衝無縫循環 + `loopStart/loopEnd` 去頭尾靜音接縫；沖水音效於完成時停止
- [ ] 馬桶物理/沖水效果持續微調中（座標：BOWL 綠橢圓、DRAIN 藍圈、hot-bowl 觸發框）
- [x] **場景 D 洗手台**：水龍頭 `(OPEN/CLOSE)` 出水；`WATER` 朝最低點 `SDRAIN` 流動、夾限在 `SINK` 橢圓內、抵達排水口淡出消失；滑鼠撥水斥力（`MOUSE_R/F`）
- [ ] 場景 D 水音效待使用者提供音檔後接上
- [ ] 部署 Netlify 公開連結
- [ ] 場景 C 馬桶（掀蓋、(PEE) 堆積、沖水旋轉）
- [ ] 場景 D 洗手台（水龍頭出水、撥水物理）
- [ ] 部署 Netlify 公開連結

## 開發約定

- 製作 / 更動前先對照 `需求規格.md`；有需求變更先更新該文件再動程式碼。
- 熱區座標用**百分比**定位（隨舞台縮放），不用像素硬值。
- 點點風下避免過小熱區，優先「較大好點區域 + 文字引導」。
- 進度推 GitHub 做版本控制；最終部署 Netlify。
