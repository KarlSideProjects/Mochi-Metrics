# Mochi-Metrics｜麻糬 Metrics

把電腦的 CPU、記憶體、GPU 與傳輸速率，送到桌上的 ESP8266 小螢幕。對需要觀察主機狀態、又不想一直切換監控視窗的人，Mochi-Metrics 提供獨立顯示器，以及用瀏覽器設定 Wi-Fi、MQTT 與顯示內容的入口。

目前包含可編譯的 ESP-12F 韌體、Go／Python 兩種發送器、設定精靈與狀態面板。下方為儲存庫既有的實機照片（含文字標註）；沒有硬體也能閱讀 [協定範例](docs/protocol/metrics-v2.md) 與執行發送器測試。

<img src="docs/images/runtime-screen.jpeg" width="640" alt="既有實機照片與文字標註：ESP8266 螢幕顯示 CPU、RAM、GPU 指標">

## 已經做出的功能

- **主機資訊發送：** Go 與 Python 讀取 CPU、RAM、GPU、網路及磁碟指標，預設每秒發布一份 metrics v2 JSON。GPU 與溫度能否取得取決於作業系統、驅動與工具；不可用欄位可能是 `0`，不代表實測零值。
- **多主機顯示：** 韌體經 MQTT 接收資料，支援裝置選擇、輪播、鎖定、欄位設定與離線狀態，目前裝置槽位上限為 8。
- **瀏覽器設定：** Wi-Fi／MQTT 設定精靈、監控設定頁，以及顯示連線狀態和 heap 資訊的狀態面板；設定存入 LittleFS。
- **小螢幕操作：** ST7789 240×240 TFT、亮度調整與可停用的觸控操作。
- **斷線恢復與局部重繪：** 非同步 MQTT、重連排程、訊息分片緩衝與 dirty mask，減少連線工作和全畫面重繪對介面的影響。

| 設定精靈 | 即時狀態面板 | 監控設定 |
| --- | --- | --- |
| <img src="docs/images/webui-wizard-mqtt.png" width="240" alt="MQTT 設定步驟"> | <img src="docs/images/webui-dashboard.png" width="240" alt="Web 狀態面板"> | <img src="docs/images/webui-monitor-config.png" width="240" alt="監控設定頁"> |

其餘既有畫面：[Wi-Fi](docs/images/webui-wizard-wifi.png)、[Sender 安裝](docs/images/webui-wizard-sender.png)、[完成頁](docs/images/webui-wizard-done.png)。圖片是展示紀錄，不是本次文件更新的實機驗收。

## 資料如何走到螢幕

```text
電腦 psutil／gopsutil → Python 或 Go Sender → MQTT broker
                                                 ↓
ESP8266：MQTTTransport → metrics v2 parser → DeviceStore → TFT
                ↕                              ↕
             Web 設定／狀態頁 ←→ MonitorConfig／LittleFS
```

發送器與顯示器只透過 `sys/agents/<hostname>/metrics/v2` 和 [metrics v2 協定](docs/protocol/metrics-v2.md) 耦合。broker 需另外準備，未包含在韌體內；電腦與 ESP 都必須能連到它。

- [韌體入口](apps/firmware/src/main.cpp)：Arduino `setup()` 初始化硬體及設定，`loop()` 協調 Wi-Fi、MQTT、顯示、觸控與 Web 工作。
- [Go Sender](apps/sender/go/main.go)／[Python Sender](apps/sender/python/sender_v2.py)：收集主機指標並發布 JSON，選一種部署即可。
- [協定解析](apps/firmware/src/include/metrics_parser_v2.h)：驗證 topic 和版本，從 topic 取得 hostname，再將陣列位置對應到指標。
- [教學文件入口](docs/site/zh/index.md)：架構、模組與學習路線；部分歷史說明可能早於目前程式，現況以原始碼與設定檔為準。

## 最短操作路徑

### 1. 準備硬體與 broker

需要 ESP-12F（ESP8266）顯示板、ST7789 SPI 240×240 螢幕（既有型號 `BL-A54038P-02`）、可燒錄的 USB 連線、PlatformIO，以及可供電腦與 ESP 存取的 MQTT broker。Go Sender 需要 Go 1.22+；Python 替代方案需要 Python 3.10+ 與 uv。

[platformio.ini](apps/firmware/platformio.ini) 指定 `esp12e` 板型：CS=GPIO16、DC=GPIO0、RST=GPIO4、BL=GPIO5、MOSI=GPIO13、SCLK=GPIO14。這是既有顯示板配置，其他硬體需先核對電路與腳位。

### 2. 編譯與燒錄

以下命令從新終端機執行；`pio` 需在 PATH 中。

```bash
git clone https://github.com/KarlSideProjects/Mochi-Metrics.git
cd Mochi-Metrics
pio run -d apps/firmware -e esp12e_wifi
pio run -d apps/firmware -e esp12e_wifi -t upload --upload-port /dev/ttyUSB0
```

將 `/dev/ttyUSB0` 換成實際序列埠（Windows 為 `COMx`）；設定檔原預設為 `/dev/ttyUSB1`。若使用 PlatformIO 自建環境，命令也可用 `~/.platformio/penv/bin/pio`。韌體建置會將 [Web 頁面](apps/firmware/web) 壓縮成嵌入式 gzip 標頭。

### 3. 設定 ESP

無有效 Wi-Fi 設定時，韌體會嘗試已儲存連線並依啟動策略進入 AP 模式。連接螢幕顯示的熱點，開啟 `http://192.168.4.1/setup`，依序設定 Wi-Fi 與 MQTT broker。電腦 Sender 要使用同一 broker 的可達位址；除非 broker 就在 Sender 主機上，否則不要使用 `127.0.0.1`。

### 4. 啟動一種 Sender

下例使用 POSIX shell，從專案根目錄執行；先把 `broker.example` 換成自己的 broker 主機名稱或 IP。若 broker 需要驗證，另設定 `MQTT_USER`、`MQTT_PASS` 環境變數。

```bash
# Go：先前景執行，確認可連線
(cd apps/sender/go && go build -o sender_v2 . && MQTT_HOST=broker.example ./sender_v2)
```

或：

```bash
# Python
(cd apps/sender/python && uv sync && MQTT_HOST=broker.example uv run python sender_v2.py)
```

兩者預設 `MQTT_PORT=1883`、`SEND_INTERVAL_SEC=1.0`、`MQTT_QOS=0`，也可指定 `SENDER_HOSTNAME`。連線成功後預期出現 Sender 啟動訊息，ESP 顯示 `MQTT OK` 與持續更新的指標。再開啟裝置的 `/status` 或 `/api/v2/status` 檢查連線與狀態；`/monitor` 可調整裝置及顯示設定。

### 常駐部署與完整指南

確認前景運作後，可從專案根目錄使用既有安裝腳本：

| 平台 | 命令（替換 broker.example） | 常駐方式 |
| --- | --- | --- |
| Linux | `bash apps/sender/go/install.sh --mqtt-host=broker.example` | systemd user service |
| macOS | `bash apps/sender/go/install-macos.sh --mqtt-host=broker.example` | launchd agent |
| Windows | `powershell -File apps/sender/go/install.ps1 -MqttHost broker.example`（系統管理員） | Scheduled Task |

腳本支援 `--uninstall`，Windows 為 `-Uninstall`。詳細操作與排錯見 [圖文安裝指南](docs/guide/安裝與使用指南.md)、[Go Sender 說明](apps/sender/go/README.md)、[Python Sender 說明](apps/sender/python/README.md)。Python 文件也保留 `senderctl.sh quickstart-compose`、狀態／日誌／重啟／停止及 Docker 開機設定命令。

## 三個值得追問的設計

**數字如何跨語言保持一致？** 協定用固定鍵與陣列位置縮短訊息，代價是讀者必須查欄位順序。韌體以 [MetricsFrameV2](apps/firmware/src/include/metrics_v2.h) 的整數欄位儲存，例如 42.3% 存為 423；解析時仍有浮點換算。16 位元容量也意味著數值可能被截限，例如 RAM MiB 欄位最多 65535。缺值以零表示的設計，尚不能清楚區分「未支援」與「真零值」。

**網路不穩時，畫面怎麼繼續工作？** [MQTTTransport](apps/firmware/src/include/mqtt_transport.h) 使用非同步 client，將重連交由主迴圈排程，並重組分片訊息。目前 [connection_policy.h](apps/firmware/include/connection_policy.h) 的重連退避上限是 5 秒、訊息上限 1024 bytes、連線後無訊息 15 秒會觸發恢復。這些門檻配合每秒資料流；若改成低頻發送，也要重新檢視沉默判斷。

**如何減少螢幕重畫成本？** [DeviceStore](apps/firmware/src/include/device_store.h) 用 dirty mask 記錄變動欄位，[TFTDriver](apps/firmware/src/include/tft_driver.h) 批次寫入 SPI。刷新策略區分強制重畫 90 ms、有更新 500 ms、無更新 1000 ms；這些是排程間隔，不是保證的畫面 FPS。既有 [效能升級報告](docs/2026-03-24-firmware-performance-upgrade.md) 說明修改與除錯經過，其中「10–20 倍」是預期加速，不能當作本次重測結果。

## 驗證與目前限制

2026-09-16 文件整理時核對程式、設定、提交紀錄與既有圖片；Python Sender 的既有測試在 Python 3.12 環境執行，**8 項通過**。本次未連接 ESP／broker、未燒錄，也未重測跨平台感測器與長時間斷線恢復；Go 與 PlatformIO 工具未安裝，未執行其建置／測試。儲存庫目前沒有 GitHub Actions 工作流程。

可在具備工具的環境重跑下列既有檢查（各行從專案根目錄執行）：

```bash
pio test -d apps/firmware -e native
(cd apps/sender/python && uv sync --extra dev && uv run python -m pytest -q)
(cd apps/sender/go && go test ./... && go build -o sender_v2 .)
```

[韌體 policy 測試](apps/firmware/test/test_connection_policy/test_main.cpp) 覆蓋純邏輯，不等於硬體端到端驗收；[Python 測試](apps/sender/python/tests) 與 [Go payload 測試](apps/sender/go/payload_test.go) 也不能保證所有硬體感測器準確。另有 [2026-02-22 穩定性紀錄](docs/2026-02-22-mqtt-stability-debug-report.md)，涉及舊 PubSubClient 路徑；目前已改用 espMqttClientAsync，舊報告的參數與成功紀錄需按日期判讀。

這是桌面與學習用監控實作，沒有歷史資料庫或告警工作流程。Web 設定與 MQTT 路徑未提供完整的公開網路安全部署方案，應在受控網路中操作。現有截圖、測試與報告提供不同層次的證據，並未構成全部平台的可靠性或效能保證。

## 學習與探究延伸

可以從一次完整資料流練習協定設計與證據判讀：為 Python／Go 輸入相同快照，比對 JSON；改變發送頻率，觀察顯示延遲與沉默判斷；或量測局部重畫和全畫面重畫的耗時。這些活動能連接取樣、單位轉換、資源限制與故障假設驗證。

[學習路線](docs/site/zh/learning-path.md) 與模組文件是既有教材資源；上述實驗仍需設計控制條件與記錄方式，不能據此宣稱已有學生學習成效或教學研究結果。文件站可在本機預覽：

```bash
(cd docs && uv sync && uv run mkdocs serve)
# 本機瀏覽 http://127.0.0.1:8000
```

## 授權與貢獻來源

本專案附 [AGPL-3.0-only](LICENSE) 與 [Commons Clause 條件](LICENSE-COMMONS-CLAUSE.md)，後者明列限制 Sell 的授權條件及 Licensor `JHIH WEI JHAN`；不能簡化成「可自由商用」。本次不變更授權文字。

提交歷史可追溯韌體、Sender 與文件的演進，本頁描述專案成果，不把整合的第三方元件算作獨立原創。依賴來源列於 [PlatformIO 設定](apps/firmware/platformio.ini)、[go.mod](apps/sender/go/go.mod) 與 [Python 設定](apps/sender/python/pyproject.toml)，包含 ArduinoJson、QRCode、ESPAsyncTCP／ESPAsyncWebServer、espMqttClient、Paho MQTT、psutil 與 gopsutil；各自授權另依其上游。既有圖片與內嵌點陣字型未附獨立來源說明，重用前需確認其權利範圍。
