# Drift Away — Three.js + TypeScript + Vite 升級與遷移計畫

這是一份針對《遠方 Drift Away》公路旅行放鬆遊戲從 2D Canvas 原型升級至 **Three.js + TypeScript + Vite** 的完整評估與分階段遷移計畫。

---

## 1. 現有程式碼架構分析與功能盤點

### 現有架構 (`drift_away_prototype.html` / `index.html`)
- **繪圖核心**：HTML5 2D Canvas，利用經典偽 3D（Pseudo-3D / 梯形透視與 $z^2$ 二次曲線投影）繪製彎曲公路、滾動地平線、雲朵、道路虛線與兩側樹木/仙人掌。
- **駕駛狀態機**：
  - 模式：`drive`（手動）、`cruise`（巡航）、`auto`（自動駕駛）。
  - 運動學：速度 `speed`（km/h）、里程 `distance`（km）、轉向角 `steer`（-1 ~ 1，放開鍵盤時以 $0.45$ 阻尼回正）、前進偏移量 `roadOffset`。
- **音訊引擎**：純前端 Web Audio API 即時頻率合成器，利用 `OscillatorNode`（正弦波/三角波）與 `GainNode` 指數衰減包絡線，即時循環合成 3 首 ambient 輕音樂（Blue Hour、Soft Horizon、After the Rain）。
- **介面層**：純 CSS 毛玻璃風格（`backdrop-filter: blur(12px)`）浮動儀表與控制卡片，包含頂部狀態、左下行車控制、右下電台音響、中下數位儀表板與啟動導引視窗。

### 功能現狀對比

| 功能模組 | 已實作狀態 | 尚未實作 / 原型限制 |
|---|---|---|
| **視覺透視** | 2D 梯形近似透視、多邊形遠山與平坦路面 | 無真正的 3D 深度、無起伏地形、無立體光影 |
| **車輛與視角** | 靜態 2D 車頭邊框與擋風玻璃遮罩 | 無 3D 車身與車內內裝、方向盤無法旋轉、無第一人稱/第三人稱動態視角切換 |
| **駕駛系統** | 鍵盤操控（WASD / 方向鍵）、巡航、自動行駛、里程計 | 缺乏懸吊上下晃動、轉向側傾、真實加速度曲線與輪胎物理感 |
| **環境系統** | 4 種色票切換（海岸、山林、曠野、夜色）與雨絲效果 | 缺少 3D 起伏地形（山脈、懸崖、沙丘）、立體植被與動態日夜光照 |
| **照明系統** | 偽色票天色與簡易月光圓形 | 無車頭大燈照射、儀表背光、環境反射或陰影投射 |
| **音樂音效** | 3 首 Web Audio 合成環境音、音量與切換 | 缺少車輛音效（引擎低鳴、行車胎噪、風切聲、轉向訊號） |

---

## 2. 升級可行性評估 (Three.js + TypeScript + Vite)

### 評估結論：**完全可行且為最佳演進方向**

1. **技術相容性與效能優勢**：
   - **Vite** 提供秒級熱更新與極快啟動速度，且產出的靜態資產可無縫在 AI Studio 的 Node.js / Express 環境中運作（維持 `PORT=3000`）。
   - **TypeScript** 能為 3D 空間數學（`Vector3`, `Euler`, `Matrix4`）、攝影機矩陣與實體狀態管理提供嚴格類型防護，防止執行期 undefined 錯誤。
   - **Three.js (r160+)**：支援 WebGL2 與 PBR 材質，配合 `InstancedMesh` 技術，可以在單一 Draw Call 內繪製數千棵道路兩側的樹木與護欄，並在 60 FPS 下流暢運作。
2. **零外部依賴與美術風格傳承**：
   - 繼續維持原創的「低飽和、溫暖、靜謐（Minimalist / Low-poly）」公路美學。
   - 車體內裝、路面、植被與天空全部採用 **程式化幾何（Procedural Geometry）** 與 **純程式化材質（CanvasTexture / PBR）**，**不依賴外部大型 GLB/PNG 檔案**，杜絕載入失敗或 404 風險。
3. **保留既有資產**：
   - 原有的 Web Audio 模組與 CSS 毛玻璃 UI 可以 100% 沿用並無縫覆蓋在 Three.js 渲染畫布上方。

---

## 3. 分階段遷移與架構計畫

```
┌────────────────────────────────────────────────────────┐
│                   Drift Away UI (DOM)                  │
│   [Brand Header]   [Radio Player]   [Bottom Controls]  │
└──────────────────────────┬─────────────────────────────┘
                           │ UI Events & Audio Triggers
┌──────────────────────────▼─────────────────────────────┐
│                 Engine Orchestrator                    │
│   (Game Loop: Clock.getDelta, State Sync, Modes)       │
├──────────────────────────┬─────────────────────────────┤
│   Vehicle System         │   World & Rendering         │
│   - First-Person Camera  │   - Three.js Scene/Renderer │
│   - Cockpit & Steering   │   - Endless Spline Road     │
│   - Driving Kinematics   │   - Procedural Sky & Light  │
│   - Speed / Odometry     │   - Instanced Roadside Props│
├──────────────────────────┴─────────────────────────────┤
│   Audio Engine (Web Audio Synth - Zero External Files) │
└────────────────────────────────────────────────────────┘
```

### 分階段實施路線：

- **第一階段（本階段目標）：核心 3D 駕駛與第一人稱車內體驗**
  - 建置 Vite + TypeScript + Three.js 開發與編譯環境。
  - 建立 3D 場景、天球底色、無限循環延伸的 3D 道路。
  - 建立第一人稱車內視角（駕駛座高度與 FOV 視角）。
  - 建構低多邊形但細緻的車內駕駛座模型（中控台、轉動方向盤、後視鏡外框、前擋風玻璃）。
  - 將鍵盤控制（WASD/方向鍵）、手動/巡航/自動駕駛邏輯與 Three.js 攝影機運動學結合。
  - 保留現有 Web Audio 電台與所有 HTML 毛玻璃控制面板。
- **第二階段：地形豐富度與 4 大環境主題遷移**
  - 以 Three.js 實作起伏地表（海岸線海浪、高低山巒、沙漠沙丘）。
  - 使用 `InstancedMesh` 生成兩側植被（杉樹、棕櫚、仙人掌）。
  - 實作 4 種色調主題與平滑過渡（海岸黃昏、晨曦山林、午後曠野、靜謐夜色）。
- **第三階段：夜間照明、動態儀表與天氣粒子**
  - 車輛頭燈投射（SpotLight 光錐）與車內儀表自發光。
  - 3D 儀表板實體指針或數位螢幕聯動當前車速。
  - 夜間雨水粒子系統與擋風玻璃雨滴效果。
- **第四階段：多視角與行車環境音效**
  - 新增視角切換：車內第一人稱 / 車頂引擎蓋視角 / 車尾第三人稱追焦視角。
  - 擴充 Web Audio 合成器加入引擎運轉低頻音與風噪。

---

## 4. 第一階段具體預計修改與新增檔案清單

1. **`package.json`**：新增 `vite`, `typescript`, `three`, `@types/three`，更新 `dev` 與 `build` 腳本。
2. **`tsconfig.json`**：TypeScript 編譯組態（ESNext, bundler module resolution）。
3. **`vite.config.ts`**：設定 Vite 監聽 `0.0.0.0:3000` 與相關構建選項。
4. **`index.html`**：保留所有精美的 UI 標籤、啟動介面與 CSS 樣式，將主要指令碼入口切換為 `/src/main.ts`。
5. **`src/main.ts`**：主應用程式入口，整合 3D 渲染器、車輛控制器、音訊播放器與 DOM 事件。
6. **`src/audio/Synthesizer.ts`**：模組化遷移現有的 Web Audio 原創環境音樂合成系統。
7. **`src/scene/World.ts`**：負責 Three.js 場景、照明、漸層天空、以及程序化無盡道路生成與捲動。
8. **`src/vehicle/Cockpit.ts`**：程序化 3D 車輛內部幾何（儀表板、可隨轉向旋轉的方向盤、車柱、前車蓋）。
9. **`src/vehicle/VehicleController.ts`**：車輛物理運動學（加速、煞車、轉向角阻尼、巡航與自動駕駛狀態機）。
10. **`src/ui/HUD.ts`**：橋接既有的 DOM 控制按鈕（駕駛模式、路段主題、電台）與 3D 遊戲狀態。

---

## 5. 驗證與測試策略

- 確保 `npm run build` 與 `compile_applet` 通過。
- 確保畫面在 `0.0.0.0:3000` 正確載入，沒有任何外部 404 資源錯誤。
- 驗證第一人稱視野：方向盤隨著 A/D 或左右箭頭平滑轉動；前進時路面標線與路面持續流動；儀表板里程與車速正確同步。
- 驗證 Web Audio 音響點擊後能正常發聲，音量調節流暢。
