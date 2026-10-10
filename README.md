# 🎮 Sky Assistant (ctrl-identity)

> 一款專為《光遇 (Sky: Children of the Light)》國際服打造的模組化 Python 智慧自動化與輔助助手，集成了 OCR 每日任務辨識、防卡點地標導航、手動路徑錄製與輕量化 CNN 行為複製自動駕駛。

---

## 📂 專案目錄結構

本專案採用清晰的模組化架構設計，便於維護與功能擴充：

```text
ctrl-identity/
│
├── main.py                  # 程式唯一入口點（負責啟動圖形介面）
├── requirements.txt         # 專案依賴套件清單
│
├── gui/                     # 【介面層】
│   └── app.py               # Tkinter 圖形控制面板與非同步執行緒封裝
│
├── core/                    # 【核心功能層】
│   ├── task_checker.py      # OCR 每日任務辨識（支援 CPU 優化與影像前處理）
│   ├── route_recorder.py    # 手動路徑錄製器（同步收集 AI 訓練資料集）
│   └── vision_ai.py         # 輕量化 CNN 模型訓練與 AI 自動駕駛推理
│
├── utils/                   # 【工具層】
│   └── navigator.py         # 跑圖執行引擎（含防卡點、自動脫困與視覺地標校正）
│
├── routes/                  # 存放自訂 JSON 跑圖路線檔
├── landmarks/               # 存放視覺地標對齊小圖
└── game_dataset/            # AI 行為複製訓練資料集（images/ 與 labels.csv）
⚙️ 環境需求與安裝
Python 版本：建議使用 Python 3.11 - 3.13。

安裝依賴套件：
請在專案根目錄打開終端機，執行以下指令安裝所有必要套件：

Bash
pip install -r requirements.txt

(如果需要手動安裝，包含：easyocr, torch, torchvision, opencv-python, mss, pydirectinput, pynput, pandas)

控制面板功能說明：
1. 執行自訂跑圖路線：載入 routes/cloud_route.json，自動執行導航，並在角色遭遇牆壁卡住時自動進行側移跳躍脫困。

2. OCR 每日任務辨識檢查：自動擷取遊戲任務面板畫面，透過 EasyOCR 辨識當日任務完成狀態。

2-1. 預覽 OCR 截圖與除錯範圍：即時擷取並二值化處理任務區塊，自動開啟圖片以協助校對截圖座標。

3. 錄製新的跑圖路線：監聽鍵盤操作（W, A, S, D, Space, F），自動記錄持續時間並同步擷取畫面作為 AI 訓練資料集。
🛠️ 技術亮點
多執行緒架構：所有繁重的自動化任務與 OCR 掃描均在背景執行緒中運行，確保 Tkinter 圖形介面流暢不卡頓。

智慧防卡點機制：即時比對畫面像素變化，連續無變動時自動觸發脫困演習。

高效能影像前處理：結合灰階、雙邊過濾（Bilateral Filter）與局部二值化，大幅提升半透明遊戲介面下的 OCR 辨識準確率。

4. 訓練純影像 AI 模型：使用收集到的畫面與標籤訓練輕量化 SkyCNN 模型，並儲存至 models/sky_vision_model.pth。
