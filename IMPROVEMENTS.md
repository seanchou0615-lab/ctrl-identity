# IMPROVEMENTS.md — Sky Assistant（CtrlIdentity）可以更進步的點

> 基準版本：**Alpha 0.94**（`core/roles.py:2`）。本文件**只是提案**：這一輪沒有改動任何程式碼或 SPEC，
> 依 `SPEC.md:3`「改設計請先改本文件，再改程式」，標記「需先改 SPEC」的項目請先取得使用者同意。

## 0. 這一輪怎麼驗證的

| 檢查 | 方式 | 結果 |
|---|---|---|
| 測試套件 | `.\.venv\Scripts\python.exe tests\run_tests.py` | **114 passed, 0 failed**（Python 3.13.14） |
| 詞典條數 | `core.glossary.load()` | 原始 591 條 → 可用 **475** 條（排除 116），與 `SPEC.md:113` 一致 |
| 時效資料 | `core.knowledge.context_block()` | 出廠檔為空 → 正確輸出「本地時效資料：無」＋「不確定」規則 |
| 詞典召回 | `glossary.match("復刻先祖什麼時候來")` | 命中 `復刻時間`（同義詞表運作正常，此項**不構成問題**） |
| 截圖能力判定 | `ollama /api/show` 逐模型讀 `capabilities` | 本機 6 個模型全部只有 `completion,tools,thinking`，**沒有任何 vision 模型** |
| 本機既有模型 | `ollama /api/show` 的 `system` 欄位 | `xiaoguang-max:latest`／`xiaoyu-max:latest` 的 system prompt **與 `core/roles.py` 不同** |
| 舊模型輸出 | `ollama /api/generate`（`xiaoguang-max:latest`，`num_predict=120`） | 回應：「您好，請告訴小光您要去哪裡…`{"action": "navigate", "target": "start_village"}`」 |
| 官方角色 prompt 實測 | `ollama.chat`（`qwen2.5:7b` ＋ `core/roles.py` 小光 prompt ＋ 注入區塊） | 注不注入時效／詞典區塊都能正常作答，**未觀察到區塊毒化**（見附錄） |
| 打包產物 | `dist\CtrlIdentity\_internal\` 與 `payload.zip` 內容 | **缺 `ollama\`、`keyboard\`、`pygetwindow\`**（只剩 `ollama-0.6.3.dist-info`）；`data\` 在 `_internal\`，`config.json` 只在 `_internal\` |
| 熱鍵 | `keyboard.add_hotkey('alt+s', …)`／`unhook_all()` 後 `press_and_release` | 註冊成功；`unhook_all()` 後再按鍵**不拋例外**（先前假設的「呼叫已移除的 hotkey 會丟例外」不成立，已從本文件移除） |
| 安裝程式參數 | `installer.setup_main.validate_install_dir(expandvars(...))` | `main.py` 的擴充在呼叫端；`validate_install_dir("%LOCALAPPDATA%\\…")` 會回「請輸入完整路徑」，**正常流程不會踩到**，此項不列為問題 |

**這一輪的唯一副作用（已復原）**：探查時呼叫了一次 `core.config.save()`，覆寫了專案根的 `config.json`；
已立即以原始內容還原成 247 bytes、LF 換行、內容與 `dist\CtrlIdentity\_internal\config.json` 相同（即出廠預設值），
`tests/_tmp\` 也被 `run_tests.py` 自行清理。除此之外沒有修改任何檔案。

---

## P0 — 正確性／風險（建議全部處理）

### P0-1 發佈產物缺少三個第三方套件：exe 有崩潰／功能靜默失效的風險

**問題**
- `dist\CtrlIdentity\_internal\` 有 `ollama-0.6.3.dist-info\`、`pydantic_core\`、`certifi\`，但**沒有 `ollama\` 套件本體**；`keyboard\`、`pygetwindow\` 也都不存在（`payload.zip` 解出來的結果相同）。`pyexpat.pyd` 卻在 `_internal\` 根層——這正是不依賴 .pyd shim 時的動態分析特徵。
- 原因：`build_exe.ps1:62-70` 只給 `--collect-all customtkinter`，`ollama`、`keyboard`、`pygetwindow` 全靠靜態分析；`core/llm.py:1` 是 `import ollama`、`gui/app.py:15-18` 是 `try: import keyboard`、`core/window_finder.py:2-5` 是 `try: import pygetwindow`。
- 三個套件的失敗後果不同，這是關鍵：
  - **`ollama` 缺** → `gui/app.py:4` 的 import 直接失敗 → `main.py` 從 `gui.app` 匯入失敗 → 程式完全開不起來（**推論**：發佈產物的 .pyd 佈局與 venv 不同，動態分析結果無法在 venv 內重現；請實機雙擊 `dist\CtrlIdentity\CtrlIdentity.exe` 確認）。
  - **`keyboard` 缺** → `gui/app.py:333` `if not keyboard: return` 直接返回 → Alt+S／Alt+M **靜默失效**，沒有任何提示（`SPEC.md:74` 的熱鍵形同不存在）。
  - **`pygetwindow` 缺** → `core/window_finder.py:9-10` 回 `None` → 永遠顯示「⚪ STANDBY: 全螢幕擷取模式」（`gui/app.py:292`），截圖永遠抓整個螢幕。
- 測試抓不到：`tests/test_imports.py:19,44-56` 對缺套件會補空殼 `_StubModule`，測試環境永遠綠燈，但與真實產物無關。

**影響**：這是最嚴重的一類問題——本機 `python main.py` 一切正常，發佈版卻「打不開或熱鍵沒反應」，而使用者只會以為程式壞了。README（`README.md:11-14`）與 CHANGELOG（`CHANGELOG.md:171`）

**建議做法**
1. `build_exe.ps1` 的 PyInstaller 參數補上：`--hidden-import ollama --hidden-import ollama._types --hidden-import keyboard --hidden-import pygetwindow --hidden-import httpx`（或 `--collect-all ollama`）。
2. 建置結尾加**產物自檢**（腳本化，不是人工）：檢查 `dist\CtrlIdentity\_internal\` 下 `ollama\`、`keyboard\`、`pygetwindow\` 三個目錄存在，缺失就 `throw`；`installer\build_setup.ps1` 打包後同樣檢查 `_internal\payload.zip` 內的 `_internal/ollama` 等 entry。
3. 新增 `tests/test_packaging.py`：純檔案系統檢查（產物不存在就跳過），把上面兩點固化成回歸測試，並在 `installer\README.md` §6.2 直接引用這支測試。

**工量**：S（腳本＋自檢＋一支測試；不含重新建置與實機驗證）
**風險／SPEC**：低。**不需先改 SPEC**；建議把「建置後必須驗證第三方模組與 payload 完整」補進 `SPEC.md` §10 當驗收條件。

### P0-2 時效資料檔的範本與使用者資料同一個檔名，「還原範本」等於清空使用者資料；錯誤訊息也指向程式碼路徑

**問題**
1. `core/paths.py:15-17` 的 `RUNTIME_FILES` 把 `data/knowledge.json` 當「缺少時要補的範本」複製過去；但 `base_dir()` 同時正是使用者資料的位置（`core/paths.py:25-29`），預設檔內容是空的（`data/knowledge.json:1-7`）。目前沒有 API 會主動刪檔，所以實際觸發點是：使用者手動刪掉或搬走資料檔（想「重來一次」）→ 下次啟動 `paths.ensure_runtime_files()`（`gui/app.py:573`）**靜默用空檔重建**，之前填的每日任務／先祖全部消失，而且沒有任何提示。
2. `core/knowledge.py:148` 把「`data/knowledge.json` 不存在、損壞，或尚未填入任何可用內容」整句放進**給模型的 prompt**；`:131-133` 的規則句再寫死同一路徑（原始碼共 3 處、一次輸出出現 2 次）。但打包後真正的路徑是 `base_dir()\data\knowledge.json`（exe 旁邊，`SPEC.md:121`），字串只是原始碼模式的相對路徑。
3. `display_block()`（`core/knowledge.py:185-188`）只濾掉「規則：」開頭的行，所以 **Lite 模式首屏**直接把檔案路徑與「不存在、損壞」字樣顯示給一般玩家；且一次顯示全部三類資料，`do_daily()`／`do_candle()`（`gui/app.py:561-569`）在 Lite 下**顯示完全相同的內容**（兩顆磚做同一件事）。

**影響**：使用者資料可能被「修復機制」靜默清空（最痛）；其餘是訊息品質——玩家看到的是實作細節（檔名與相對路徑），而不是「資料還沒更新、要去哪裡補」。

**建議做法**
- `core/paths.py`：把「種子範本」與「使用者資料」分開——新增 `SEED_FILES`（只含 `config.json` 與 `data\knowledge_base.json` 這類**使用者不常改**的檔案）；使用者資料（`knowledge.json`）缺檔時**只記 log、建立一個 `knowledge.json.missing` 標記或直接在 UI 提示**，不要用空範本覆蓋可能還原的原檔。若仍要自動建立，先檢查是否為 0 bytes 且備份舊檔為 `knowledge.json.bak`。
- `core/knowledge.py`：新增 `DATA_LABEL = "本地時效資料"` 與 `data_file_hint()`（回傳實際絕對路徑）；prompt 內只寫「本地時效資料檔未更新」，**路徑只在 UI 顯示**。`display_block()` 拆成 `section_block(kind)`（`tasks`／`spirit`／`candles`），`do_daily()` 只顯示任務＋先祖、`do_candle()` 只顯示燭火路線。
- `data/README.md` 補上「不小心刪檔怎麼還原」。

**工量**：S（路徑／訊息／分段顯示）＋ S（種子與使用者資料分離）
**風險／SPEC**：**需先改 SPEC §10**（現在寫「首次啟動若 exe 旁缺少上述檔案，從打包資源複製一份範本」——正是要改的這條）。

### P0-3 本機既有角色模型的人格與 `core/roles.py` 不一致，會輸出舊版自動化指令 JSON

**問題**
- 實測 `ollama /api/show model=xiaoguang-max` 的 system 是「…專屬《Sky光·遇》的**自動化AI導航助手**…當用戶要求『帶路、前往、導航』時，句尾輸出：`{"action": "navigate", "target": "地點英文代碼"}`」；`xiaoguang-max:latest` 的 `temperature` 是 **0.2**、`xiaoyu-max:latest` 是 **0.3**，而 `tools/build_models.py:33` 的 `DEFAULT_TEMPERATURE` 是 0.6——兩邊參數也不同步。
- 實測 `api/generate` 對「請帶我去雲野」的回應尾端確實是 `{"action": "navigate", "target": "start_village"}`。
- `gui/app.py:454` 把串流文字原樣貼進對話框、`:461` 寫回歷史，沒有任何剝除。`core/glossary.py:54-61` 只在**載入詞典**時剝除 JSON，管不到模型輸出。

**影響**：`SPEC.md:6` 的定位是「不模擬任何遊戲輸入」，但使用者看到的回覆像在執行遊戲操作；`SPEC.md:63` 明文要求舊版指令 JSON 不得進入知識流，現在從「模型人格」這條路繞回來。同一台機器上 `python main.py` 的體驗還會因模型不同而漂移。

**建議做法**
1. `gui/app.py` 新增 `_strip_action_json(text)`（正則可沿用 `core/glossary.py:56` 的 `_ACTION_JSON`），在 `_chat_chunk()`／`_chat_done()`／`_aspect()` 顯示與寫歷史前剝除；`log()` 保留原文供除錯。
2. 角色模型→prompt 的一致性檢查：`core/llm.py` 的 `_model_payload()`（`:22-36`）已能拿到 `ollama.show()` 結果，新增 `system_prompt(model)`；`gui/app.py` 首次 `submit()` 前比對 `ROLES[key]["prompt"]` 是否是模型 system 的前綴，不一致時在狀態列提示「模型人格與程式不符，請重建（`python tools\build_models.py --create`）」。
3. `tools/build_models.py` 的 `--create` 預設溫度與程式一致（0.6），或把 `--temperature` 預設改成「從 `config` 讀」。

**工量**：S（剝除＋提示）／M（自動重建模型並換手）
**風險／SPEC**：**不需先改 SPEC**（§6 已要求不得讓舊指令 JSON 進入知識流，這只是把防線補到顯示層）；自動重建屬新增行為，需在 `SPEC.md` §4／§9 加一句。

### P0-4 打包後第一次啟動等於「空白資料」＋精靈會把 `setup_done` 寫死

**問題**
- `paths.ensure_runtime_files()`（`core/paths.py:68-70`）只補 **exe 旁**的檔案；但 `build_exe.ps1:65` 把 `data\` 打進 `_internal\`（`SPEC.md:120` 的 `resource_path()` 才是讀取端），實測 `dist\CtrlIdentity\` 根目錄只有 `CtrlIdentity.exe` 與 `_internal\`，`config.json` 只在 `_internal\`。第一次啟動會**複製一份 `data\` 到 exe 旁**——這本身符合 §10，但等於每次全新安裝都多出約 85 KB 的可編輯資料副本，且使用者可能改到 `_internal\` 的那一份而毫無效果。
- `gui/setup_wizard.py:286-289` 的 `skip_all()`：無論從哪一步按「略過設定」，都寫入 `setup_done=True`。首次啟動按了「略過」卻沒選模式，之後**不會再自動跳出**，程式停在 `config.json` 的 `mode: "ai"`（`config.json:2`）→ 直接撞上 P0-5。
- `gui/app.py:274-278` 的 `open_settings()` 把記憶體快照 `self.cfg` 傳給精靈（`gui/setup_wizard.py:23` `dict(cfg)`）；`_reload_config()`（`:280-282`）只在關閉時重讀。使用者先手改 `config.json` 再開精靈，精靈顯示／寫回的仍是舊值。
- `gui/setup_wizard.py:291-305` 的 `finish()` 在**主執行緒同步**呼叫 `setup.detect()`（內部 `llm.is_running()`＋`list_models()`＋每個模型一次 `supports_vision()`，`core/setup.py:49-66`）。Ollama 未回應時每支 HTTP 呼叫都要等 timeout，UI 會白屏——與 `CHANGELOG.md:132` 宣稱的「背景工作移出主執行緒」相反。

**影響**：全新安裝的使用者最常見的兩條路（「略過設定」或「精靈卡住」）都會導向「AI 模式但沒有可用模型」；第一次開精靈時也可能凍結數秒到數十秒。

**建議做法**：`skip_all()` 只寫 `setup_done=True` 而**不**改 `mode`，並讓 `_refresh_status_hint()` 對「`setup_done` 但 `mode=ai` 且 `role_models` 為空」顯示「尚未完成設定」；`open_settings()` 先 `self.cfg = config.load()` 再開精靈；`finish()` 改走 `_detect_async()` 同樣的背景執行緒路徑，偵測完成後才 `config.save()`。

**工量**：S
**風險／SPEC**：**需先改 SPEC §11**（補「略過設定後的行為」與「AI 模式下無可用模型時的降級」）。

### P0-5 預設角色模型不存在，且文件教使用者安裝的視覺模型名稱不一致

**問題**
- `core/roles.py:7,13` 的預設模型是 `xiaoguang-max`／`xiaoyu-max`；`core/setup.py:15-30` 的 `CATALOG` 沒有這兩個，`build_models.py` 是開發者工具（`SPEC.md:45`），安裝流程不會建立它們。
- `gui/app.py:270-272` 的 `_role_model()` 在 `config.role_models` 為空時**退回預設名**，`gui/app.py:435` 直接拿它呼叫 `llm.chat_stream()`；本機若沒建 `xiaoguang-max` 就是 404，錯誤原文經 `gui/app.py:445`（`⚠️ 核心連線異常：\n404 …`）原封不動顯示——違反 `SPEC.md:61`「不得把原始 API 錯誤丟給使用者」。
- 視覺模型名稱三處不一致：`README.md:61`／`gui/app.py:409,473` 寫 `ollama pull llama3.2-vision`，`core/setup.py:28` 的 `CATALOG` 是 `llama3.2-vision:11b`（實測 `llama3.2-vision:11b` 也不存在，會 404）。
- 補充：本機 `.venv` 缺 `eval_type_backport` 時 `pydantic` 連 import 都會失敗（實測），錯誤訊息同樣會原文外露；`requirements.txt` 沒有列出它。

**影響**：全新安裝＋略過精靈就是「第一次對話拿到 404 原文」；照 README 裝的視覺模型名稱與精靈清單對不上，截圖功能可能永遠停在「沒有可用的視覺模型」。

**建議做法**：`_role_model()` 在預設名不在 `llm.list_models()` 時回 `None`，`_start_chat()` 在背景執行緒改回白話「找不到模型 X，請到 ⚙️ 環境設定選擇／下載」；`core/llm.py` 新增 `is_missing_model_error(exc)`（認得 `model not found`／`404`），與 `is_multimodal_error()`（`:59-62`）並列；把視覺模型名稱抽成 `core/setup.py:VISION_HINT_MODEL = "llama3.2-vision:11b"`，`README.md:61`、`gui/app.py:409,473` 全部引用它；`requirements.txt` 加 `eval_type_backport`。

**工量**：S
**風險／SPEC**：**需先改 SPEC §6**（安裝指引文字）與 §3 註記（預設模型名只是預設，需補「找不到時降級」行為）。

---

## P1 — 使用者體驗

### P1-1 Toast 的「與面板同色」讓熱鍵回饋實際上幾乎看不到

**問題**：`gui/toast.py:53-56` 的 Toast 底色是 `T.PANEL`（`#111827`），而主面板的標題列也是 `T.PANEL`（`gui/app.py:99`）；Toast 定位在螢幕最頂端置中（`gui/toast.py:25-27`，`MARGIN_TOP = 12`）。面板預設放在 `+100+100`（`gui/app.py:52`），高度 600，所以面板頂端 `y=100` 正好壓在螢幕頂端置中的 Toast 上：**面板開著時 Toast 被面板蓋住**（主面板 `-topmost`，`:55`）。CHANGELOG 已自承多螢幕限制（`CHANGELOG.md:34`），但更常見的「單螢幕、面板開著」情境同樣看不到。

**影響**：0.94 的主打功能（Alt+S／Alt+M 有回饋）在最常見的情境下失效，使用者會以為熱鍵沒反應、連按多次反覆切換。

**建議做法**：`gui/toast.py` 的 `compute_position()` 增加 `anchor_rect` 參數（面板 `winfo_x/y/width/height`），Toast 定位在面板**上方或下方**（取空間較大的一側，間距 8 px）；`window` 無法取得面板（`withdraw`，`gui/app.py:322`）時維持現行螢幕頂端。`gui/app.py:336-337` 的熱鍵轉發時把目前視窗矩形一起帶過去。

**工量**：S
**風險／SPEC**：**需先改 SPEC §7.2**（現在寫死「螢幕頂端置中、距頂 12 px」）。

### P1-2 主面板會搶焦點，與 §7.2 自己畫的紅線不一致

**問題**
- `gui/app.py:558` 的 `do_nav()` 用 `ctk.CTkInputDialog(...).get_input()` 開**系統 Modal 對話框**，會取得焦點；跑圖中誤按「👀 環景導航」就把鍵盤從遊戲搶走。
- `LogWindow`（`gui/app.py:27-37`）與 `open_settings()`／`gui/setup_wizard.py:334-340` 的 `open_dialog()` 都只設 `-topmost`，沒有 `-toolwindow`，會進 ALT+TAB 與工作列；`SPEC.md:73` 只對 `show_titlebar` 給了一條例外，子視窗沒有規則。
- 另外 `do_nav()` 會把「目標：X。請給予導航指引！」丟給模型（`gui/app.py:559`）——在「不模擬遊戲輸入」的定位下，這顆磚的文案（「環景導航」）與實際能力（文字建議）落差很大。

**影響**：`SPEC.md:89` 說「搶焦點會讓玩家斷開操作」是遊戲場景的紅線，但只守住 Toast；玩家在遊戲中誤按就會斷輸入手感。

**建議做法**：`do_nav()` 改成面板內的第二列輸入框（或直接併入主輸入框），不開 Modal；`LogWindow`、settings 視窗加 `attributes("-toolwindow", True)`；功能磚文案改成「🗺 地圖問答」之類，避免暗示會代操遊戲。

**工量**：M
**風險／SPEC**：**需先改 SPEC §7**（新增子視窗焦點／ALT+TAB 規則）與 §7 功能磚命名（若要改文案）。

### P1-3 失敗後狀態列的階段文字會殘留：`_set_stage` 設了就不會自動回復

**問題**：`gui/app.py:284-292` 的 `_refresh_status_hint()` 只被呼叫四次（`:282`、`:355`、`:466`、`:521`）；`_chat_fail`（`:468-477`）、`_debate_fail`（`:523-530`）、`_no_model`（`:506-509`）都只呼叫 `_set_stage("⚠️ …", T.WARN)` 而**沒有**安排恢復。唯一會讓它回到一般狀態的是下一次視窗掃描（`gui/app.py:341-355` 每 3 秒）——但如果遊戲視窗狀態沒變化，`_apply_window()` 還是會呼叫 `_refresh_status_hint()`，所以會**在 3 秒內被蓋掉**；也就是「⚠️ 連線異常」只顯示不到 3 秒就消失，玩家往往看不到錯誤階段（反過來說，若掃描執行緒卡住，紅字就永遠留著）。

**影響**：使用者很難在遊戲中捕捉到「剛剛到底失敗了」這件事；錯誤訊息只留在對話框，但對話框可能同時被下一句 `say()` 蓋掉（`gui/app.py:180-182` 的 `say()` 是**整段 replace** 而非 append）。

**建議做法**：`_chat_fail`／`_debate_fail`／`_no_model` 改成 `_set_failed_stage(msg)`：寫階段 + `after(5000, self._refresh_status_hint)` 明確排程；`say()` 的失敗訊息應 append 或至少在對話框保留 `[ERROR]` 前綴一行。

**工量**：S
**風險／SPEC**：不需改 SPEC（§7.1 只規定忙碌時的行為；建議在 §7.1 補「失敗後階段至少顯示 5 秒」）。

### P1-4 錯誤訊息品質：多數失敗仍把原始例外丟給使用者

**問題**：`gui/app.py:445`（`f"⚠️ 核心連線異常：\n{e}"`）、`:503`（辯論 `str(e)`）、`:528`（同上）都是**原始例外字串**。`SPEC.md:61` 只規範了 multimodal 一種；`Ollama 未啟動`、`404 model not found`、`pydantic import 失敗`、`httpx timeout` 全部會原樣顯示。`core/llm.py:59-62` 的 `is_multimodal_error()` 是目前唯一的翻譯層。

**影響**：最常見的三種失敗（Ollama 沒開、模型不存在、套件缺失）都以英文堆疊式訊息出現在對話框，玩家無法自救。

**建議做法**：`core/llm.py` 新增 `explain_error(exc) -> str`，集中三類判別（連線失敗→「Ollama 沒有在 127.0.0.1:11434 執行」、404 →「找不到模型」、multimodal → 既有白話），`gui/app.py` 三個 fail 分支都改呼叫它；測試可用 `tests/test_llm_vision.py` 的假 ollama 模式補案例。

**工量**：S
**風險／SPEC**：**需先改 SPEC §6**（把「不得丟原始錯誤」從 multimodal 擴大到所有失敗）。

### P1-5 截圖瞬間面板消失／抓錯區域

**問題**：`gui/app.py:545` 先 `self.attributes("-alpha", 0.0)`、`after(150, self._shot)`，截圖完成才在 `:550`／`:554` 恢復 1.0——面板會「憑空消失 150 ms」；若這段期間按 Alt+S（`gui/app.py:322` `withdraw`），恢復 alpha 與重新顯示的順序會錯亂。`core/window_finder.py:19-22` 的 `bbox_of()` 用 `max(0, …)` 裁切，負座標（副螢幕在主螢幕左／上）會被壓成 0 → `ImageGrab.grab(bbox=…)` 抓到錯誤區域。本機只有 `DISPLAY1 {0,0,1920×1080}`，**無法實測多螢幕**，以上是程式碼層面的推論。

**影響**：遊戲中可見閃爍（違反 §7.1「不得使用閃爍」的精神）；多螢幕使用者可能把不相關畫面送給模型，得到「與畫面不符」的回答。

**建議做法**：`do_vision()` 改 `self.withdraw()` → `after` 截圖 → `deiconify()`（或只隱藏 `self.panel` 內的對話區）；`bbox_of()` 不做 `max(0,…)`，改由呼叫端 `ImageGrab.grab(bbox=..., all_screens=True)` 並處理負座標；截圖前在對話框寫明「抓取區域：<視窗標題或整個螢幕>」。

**工量**：S
**風險／SPEC**：不需改 SPEC（§6 只規定送出條件）。

### P1-6 AI 模式下「任務／先祖」「燭火路線」在沒有本地資料時仍會呼叫模型，結果必然是「不確定」

**問題**：`gui/app.py:561-569` 在非 Lite 模式直接 `self.submit(...)`；`data\knowledge.json` 出廠為空（`CHANGELOG.md:165`），模型拿到的時效區塊只有「本地時效資料：無」，依 `SPEC.md:62` 只能回答「不確定」。使用者付了一次冷啟動（本機 7B 模型數十秒）換到一句「不確定」。

**影響**：在資料檔填好前（也就是所有新安裝的預設狀態），這兩顆磚的最佳結果就是「不確定」，浪費推論與等待。

**建議做法**：`do_daily()`／`do_candle()` 先用 `knowledge.daily_tasks()/traveling_spirit()/candle_routes()` 判斷有無資料；沒有就直接顯示本地「不確定＋更新指引」（與 Lite 分支共用同一個 `_local_data_view()`），**不呼叫模型**。

**工量**：S
**風險／SPEC**：**需先改 SPEC §11**（新增「AI 模式下無資料時不呼叫模型」一條）。

### P1-7 熱鍵與視窗掃描的失敗完全靜默

**問題**：`gui/app.py:332-339` 的 `_hotkeys()` 用 `except Exception: pass`；`:345-351` 的 `_scan_work()` 也吞掉所有例外。缺 `keyboard` 或註冊失敗（企業環境的端點防護常攔鍵盤鉤子）時，使用者按 Alt+S 沒反應也查不到原因；`pygetwindow` 缺失時狀態列永遠停在「⚪ STANDBY」，分不出「沒開遊戲」與「套件壞了」。

**影響**：這是最常見的「功能壞了但沒有任何線索」求助；也讓 P0-1 的打包問題更難被發現。

**建議做法**：`_hotkeys()` 失敗時在狀態列一次性顯示「⚠️ 熱鍵註冊失敗，請用面板按鈕」並在 `log()` 記錄原因；`_scan_work()` 區分「未安裝 pygetwindow」與「比對不到視窗」兩種訊息。

**工量**：S
**風險／SPEC**：低。

---

## P2 — 可維護性／測試

### P2-1 測試全綠但與真實環境脫鉤（P0-1 的成因之一）

**問題**：114 項全過，但 `tests/test_imports.py:19,44-56` 會在缺套件時補空殼模組；`_bare_app()`（`:81-107`）用手工補 10 個屬性模擬 `SkyAssistantApp`。沒有任何測試看過 `dist\` 或 `installer\payload.zip` 的真實內容，`installer\README.md` §6.2 的 payload 檢查也只是人工指令。

**建議做法**：新增 `tests/test_packaging.py`：`dist\CtrlIdentity\_internal` 存在時檢查 `ollama\`、`keyboard\`、`pygetwindow\` 目錄；`dist\setup\...\_internal\payload.zip` 存在時用 `zipfile` 檢查 `_internal/ollama/` 等 entry 與頂層 `CtrlIdentity.exe`。產物不存在就 `return`（clone 後不紅燈）。

**工量**：S
**風險**：低。

### P2-2 `gui/app.py`（577 行）同時做視窗、狀態機、對話、辯論、功能磚，測試只能靠手寫替身

**問題**：視窗建置 `gui/app.py:94-177`、狀態機 `:189-250`、對話與辯論 `:363-530`、功能磚 `:533-569` 全在同一類別；`tests/test_imports.py:_bare_app()` 因此要手工補 `role`、`busy`、`cfg`、`histories`、`_vision_model`、`bridge`、`set_busy`、`say`、`log`、`_set_stage`、`_schedule_slow_hint`、`_cancel_slow_hint`，任何新增欄位都會讓替身過期。

**建議做法**：抽出 `gui/controller.py`：`ChatController(cfg, roles, llm, knowledge, glossary)` 純邏輯（組 messages、挑模型、Lite 分支、歷史裁剪——即 `:369-530` 的非 Tk 部分），`SkyAssistantApp` 只保留 `say/status/busy` 回呼。測試直接對 controller 做，不必 mock Tk。這是讓 P1-3／P1-4 的改動可回歸的唯一實際做法。

**工量**：M
**風險／SPEC**：**需先改 SPEC §4**（新增檔案與職責）。

### P2-3 詞典／時效資料的 `_CACHE` 是模組級單鍵快取，測試靠檔名隔離

**問題**：`core/knowledge.py:23,53-64` 與 `core/glossary.py:52,80-97` 都用單一 `{"key","data"}` 快取；兩支測試（`tests/test_knowledge.py:_path`、`tests/test_glossary.py:_path`）只好用不同檔名避免互相踩。使用者同秒內存檔且長度不變時 mtime/size 可能一樣，快取不會失效（`data/README.md:55` 因此寫「重啟最保險」）。

**建議做法**：改成 `functools.lru_cache(maxsize=4)` 包 `_load_cached(path, mtime_ns, size)`，並提供 `reset_cache()` 給測試；`data/README.md` 改成「存檔後應自動重新載入」。

**工量**：S
**風險**：低。

### P2-4 `core.debate.parse_vote()` 對「沒有表決行」一律算「保留」，會系統性拉高共識分數

**問題**：`core/debate.py:9-12` 找不到「表決：X」時回「保留」（0.5 分）；`consensus()`（`:14-18`）只在 `votes` 為空時才「無效」。小型模型（1.5B）常忽略格式指示，三票都是「保留」→ 0.5 → **需修正**，永遠不會被判「無效」。`SPEC.md:59` 只規定「失敗的切面不計票」、`:64` 規定「空回覆視同失敗」，沒有規定「有內容但沒表決行」。

**建議做法**：`run_debate()`（`core/debate.py:41`）在 `ok` 為真但正則沒命中時標記 `vote_parsed=False`；`consensus()` 回傳 `unparsed` 數量，`gui/app.py:517-519` 顯示「2/3 切面完成表決」。

**工量**：S
**風險／SPEC**：**需先改 SPEC §6**（明訂「無表決行」的計分規則）。

### P2-5 `core/setup.py:plan()` 是死碼，精靈下載前不排除已安裝模型

**問題**：`core/setup.py:69-84` 的 `plan()` 只有測試在用（`tests/test_setup.py:56-67`），GUI 沒有任何呼叫；`gui/setup_wizard.py:192-216` 的 `_start_download()` 直接對 `chosen()` 全部呼叫 `llm.pull()`，已安裝的模型也會重跑一次檢查，而 `_update_total()`（`:195-199`）把已安裝模型也算進容量。

**建議做法**：`_start_download()` 先 `setup.plan([i["id"] for i in items], self.detected["models"])`，只下載 `download` 清單；`_update_total()` 改用 `plan()` 的 `total_gb`。

**工量**：S
**風險**：低。

### P2-6 版本字串與安裝程式文件不同步

**問題**：`installer/setup_main.py:37` 的 `APP_VERSION = "0.94"`，但 `installer/README.md` 的登錄值表格與測試步驟仍寫 `0.93`（installer/README.md 第 5 節與 §6.3）。`core/roles.py:2`、`installer/setup_main.py:37` 兩處各自硬編版號。

**建議做法**：新增 `tests/test_version_sync.py`：`setup_main.APP_VERSION == VERSION.split()[-1]`；同步更新 `installer/README.md`。

**工量**：S
**風險**：低；能直接防住「版號忘了改」這類最常見的發佈瑕疵（`SPEC.md:112` 已要求版本字串統一）。

### P2-7 兩支 PowerShell 建置腳本約 90% 重複

**問題**：`build_exe.ps1:40-74` 與 `installer\build_setup.ps1` 的 PyInstaller 段落各自處理 `.buildtmp`、找 Python、stderr 導檔、`$LASTEXITCODE` 檢查。

**建議做法**：抽出 `scripts\build_common.ps1` dot-source，兩支腳本只留各自參數與驗證，順便放進 P0-1 的產物自檢。

**工量**：S
**風險**：低。

---

## P3 — 產品／功能想法（不含模擬遊戲輸入）

> `core/route_recorder.py`、`utils/navigator.py` 依 `SPEC.md:101` 維持不實作；以下都不碰鍵盤／滑鼠模擬。

### P3-1 每日任務 OCR（`core/task_checker.py`）：只讀畫面，不代操作

**問題**：`core/task_checker.py:1-4` 仍是 `NotImplementedError`。它正好能解 P0-2／`CHANGELOG.md:165` 的痛點：不用手動改 `knowledge.json`，也能讓時效資料有來源。

**建議做法**：`check_daily_tasks(bbox=None) -> dict` 回傳 `{"date","items","confidence"}`；UI 在「📅 任務／先祖」磚加「掃描今日任務」，結果**先顯示給使用者確認**再寫回 `daily_tasks`；`easyocr` 作可選依賴，未安裝時沿用 `core/vision_ai.py:80-89` 的安裝指引模式。明訂「不自動上傳、不模擬輸入」。

**工量**：L
**風險／SPEC**：**需先改 SPEC §8**（把 `task_checker` 從「另行評估」改成排定），並規定 OCR 結果須經使用者確認才寫入時效資料。

### P3-2 資料過期提醒與「開啟資料資料夾」

**問題**：出廠 `data\knowledge.json` 是空的（`CHANGELOG.md:165`），之後每天都回答「不確定」，但使用者沒有任何補資料的入口；Lite 狀態列只顯示「本機資料，無需模型」（`gui/app.py:288`）。

**建議做法**：`core/knowledge.py` 新增 `staleness()`（最後更新日、落後天數）；`_refresh_status_hint()` 在久未更新時顯示「🗓 資料已 N 天未更新」，頁尾加一顆「開啟資料資料夾」按鈕（`os.startfile(paths.base_dir() / "data")`，純本機操作）。

**工量**：S
**風險／SPEC**：**需先改 SPEC §7**（狀態列多一種狀態）。

### P3-3 辯論表決視覺化（SPEC 已列為後續）

**問題**：`SPEC.md:115` 明列「辯論表決上色（含 🟩🟩🟨 票型摘要）與邊緣吸附列為後續」，目前 `_debate_done()`（`gui/app.py:517-521`）只印一行「共識：通過（67%）」。

**建議做法**：`core/debate.py` 新增 `vote_badges(votes)` 純函式（🟩／🟨／🟥），由 `_debate_done()` 渲染，並與 P2-4 的 `unparsed` 一起顯示。純顯示層。

**工量**：S
**風險／SPEC**：不需改 SPEC。

### P3-4 截圖在沒有 vision 模型時，改用已訓練的離線分類器給保守結果

**問題**：`📷 視覺辨識` 必須有 vision 模型（`SPEC.md:61`），但本機實測**沒有任何 vision 模型**，這顆磚實際上永遠是「請安裝視覺模型」的提示。`core/vision_ai.py` 已有 `scan_dataset`／`train` 與輸出 `models\sky_vision.pt`，執行期卻沒有載入它的地方。

**建議做法**：新增 `core/vision_infer.py`（只讀截圖 → 分類標籤，不碰輸入），模型檔存在時優先使用，讓這顆磚在沒有 vision LLM 時仍給「目前畫面看起來像：雲野／蠟燭」的保守結果，並明確標示「離線分類，非模型描述」。

**工量**：M
**風險／SPEC**：**需先改 SPEC §6／§8**（§8 現在只把 `vision_ai` 定位為離線訓練工具）。

---

## 建議下一步先做的 3 件

1. **P0-1 打包缺模組＋產物自檢**：這是「發佈版可能是壞的」等級，而且只需改 `build_exe.ps1` 參數與加一段自檢；不改的話，後面所有 UX／功能提案都可能在 exe 上不存在。先雙擊 `dist\CtrlIdentity\CtrlIdentity.exe` 實機確認一次，再修。
2. **P0-5／P0-4 首次啟動不掉坑**：模型不存在時的 404 原文（`gui/app.py:445`）與「略過設定後卡在 AI 模式沒模型」是同一個首次體驗斷點，一起修才不會出現「打得開但一句話都不能問」。涉及 §6 文字與 §11 行為，請先取得使用者核准。
3. **P0-3 舊模型人格吐 `{"action": …}`**：只要在 `gui/app.py` 加一個 `_strip_action_json()` 就能擋住顯示層外洩，同時加「模型人格與程式不符」的狀態提示，確保使用者不會長期在錯誤人格下使用（涉及 `SPEC.md:6` 的硬性定位）。

次批：**P1-1 Toast 被面板蓋住**（0.94 主打功能實際看不到）→ **P1-3／P1-4 失敗訊息與階段殘留**→ **P2-1 打包測試**（把 P0-1 變成長期防護）。

---

## 附錄：檢查過但判定「不需要動」的項目（避免湊數）

- `core/glossary.py` 的同義詞召回：實測 `復刻先祖什麼時候來`→`復刻時間`、`白蠟上限多少`→`蠟燭上限` 皆正確；475 條可用詞典與 SPEC 一致。
- 時效資料／詞典區塊**不會毒化**官方角色 prompt：實測 `qwen2.5:7b` ＋ `core/roles.py` 小光 prompt，注不注入區塊都能正常回答（唯一觀察到的是模型回覆偶有簡體字，屬模型行為，非程式缺陷；若要處理可考慮在 prompt 規則補一行，但那會動到 §6）。
- `core/debate.py` 的 `PASS_THRESHOLD = 2/3`：與 `tests/test_debate.py:36` 及 `SPEC.md:59` 一致。
- `core/paths.py` 的 `base_dir()`／`resource_path()`：唯讀行為符合 §10。
- 視窗掃描不在主執行緒（`gui/app.py:341-355`）；呼吸點在 `busy=False` 時會 `after_cancel`（`gui/app.py:219-229`），符合 §7.1「idle 時停掉計時器」。
- `keyboard` 套件註冊與 `unhook_all()`：本機（未提權、Python 3.13）實測正常。
