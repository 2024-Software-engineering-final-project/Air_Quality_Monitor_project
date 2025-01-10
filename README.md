### 系統概述
本系統旨在監測中正大學附近空氣品質數據。雖然目前環境部有在實施全台灣的空品監測，然而嘉義地區的監測站僅有嘉義市區一個站點，遠遠不足以兼顧整個嘉義，中正大學附近更是完全沒有相關資訊可以取得，校內化工系恰好有相關設備與技術願意提供，是故製作本系統以實現校區周圍的空氣品質數據的即時收集、存儲、警示和可視化展示，並與 ICCC 智慧校園管理平台連結，讓校內空品資訊的取得與檢視變得更加便利。
	主要功能包括：
+ 從化工系空品偵測站與環境部資料站數據收集
+ 可視化展示即時空品資訊
+ 可互動操作(顯示點位、指標)
+ 若超出安全閾值，以面板顯示警訊，並以電子郵件警示 
+ 歷史資料儲存

### 架構圖
![](https://raw.githubusercontent.com/HutakiHare/IIS_intern/main/%E8%9E%A2%E5%B9%95%E6%93%B7%E5%8F%96%E7%95%AB%E9%9D%A2%202025-01-10%20142402.png)  
+ data source: 通過API提供數據  
    + 化生實驗室空氣品質Server  
    + 環境部即時影像  
+ AI Prediction: 利用模型抓取影像並判斷影像之 PM2.5  
+ Prometheus Server: 負責拉取並存儲數據，並提供給Grafana與 Alertmanager  
+ Alertmanager: 在資料數值超出設定閾值時，發出警告  
+ Data visualization: Grafana通過Prometheus的數據接口，生成儀表板  

### Component Design
3.1 數據收集模組  
+ 技術類型: Python  
+ 輸入：空氣品質偵測數據 (PM2.5, PM10, CO, HC, O3 等)  
+ 功能：使用 API 作為數據接口，由自訂義的 exporter 進行定時收集數據、重新填寫成 Prometheus 合適的接收格式。
  
3.2 影像數據預測模組  
+ 技術類型: PyTorch  
+ 訓練資料來源:  
    + 環境部空品影像  
    + 環境部空品數據  
+ 功能:  
    + 使用 ResNet18 模型進行訓練，利用訓練後的模型進行預測  
    + 定期從環境部網站獲取測站周圍影像  
    + 根據獲取到的影像進行 PM2.5 的濃度預測  
    + 提供 Prometheus 預測的數值。 

3.3 數據存儲與警示模組  
+ 技術類型：Prometheus、 Alertmanager  
+ 功能：  
    + 數據提取，依各數據更新頻率作為設定，向不同目標抓取數據  
    + 數據存儲，支持長期時間序列的數據查詢  
    + 警示判斷，將數據與閾值做比較，若超出標準，則寄出 Email 作為示警  
    + 配置數據保留政策，保留 15 天的數據資料，以後的刪除  
    + 實現指標提取與數據聚合  
+ 查詢語言：PromQL  

3.4 可視化模組  
+ 技術類型：Grafana  
+ 功能：  
    + 通過 PromQL 提取需求時段數據  
    + 設計儀表板以可視化關鍵指標  
    + 即時數值與數據趨勢  
    + 長時間序列數據(單一與多站點)  
    + 以地圖顯示站點位置  
+ 設置警告  
    + 圖表上，以不同顏色表示不同警訊狀態（包含normal, firing等狀態）  
    + AQI 警示狀態改變時，會記錄於圖表上，供使用者回看  
+ 動態展示設計：使用者可選擇不同地區、指標，和調整時間範圍，以詳細觀察不同條件下的數據  


### 數據模型

Prometheus 中的數據存儲以時間序列為單位：
+ Metric (float variables)名稱：

| 資料         | 指標名稱                                      |
|--------------|-----------------------------------------------|
| 溫度         | air_quality_temperature_celsius              |
| 濕度         | air_quality_humidity_percentage              |
| CO2          | air_quality_co2_concentration_ppm            |
| O3           | air_quality_o3_concentration_ppb             |
| HC           | air_quality_hc_concentration_ppb             |
| CO           | air_quality_co_concentration_ppm             |
| PM2.5        | air_quality_pm25_concentration_micrograms     |
| PM10         | air_quality_pm10_concentration_micrograms     |
| VOC          | air_quality_voc_concentration_ppb            |
| AI PM2.5     | air_quality_prediction_pm25_microgram_cubic_meter |

+ station (string variables)名稱：

| 站點ID | 站點中文名稱            |
|--------|-------------------------|
| 106    | 共同教育大樓106教室     |
| B1F    | 活動中心B1              |
| ARF    | 地質環境館屋頂          |
| A1F    | unknown location        |
| M      | unknow location         |
| PL     | 化工實驗室左側          |
| PR     | 化工實驗室右側          |
| 42     | 嘉義影像拍攝地          |

### panel usage

| Panel Type   | Functions                                                                                                                                          |
|--------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| Text         | 1. Show text                                                                                                                                    |
|              | 2. Run HTML codes: grab pictures from certain URL                                                                                               |
| Clock        | Display time information, including date, current time, and time zone                                                                          |
| State        | 1. Display immediate AQI information                                                                                                           |
|              | 2. Display sparkline to briefly show trends of data                                                                                            |
|              | 3. Background color changes when alert state changes                                                                                           |
| Time series  | 1. Displays an x-y graph with time progression on the x-axis and the magnitude of the values on the y-axis.                                    |
|              | 2. Background color shows threshold of metrics                                                                                                |
|              | 3. Alert rules can be linked to Time series panel in the form of annotations to observe when alerts fire and are resolved                      |
| Geomap       | Display the locations of AQI stations                                                                                                          |
| Alert list   | 1. Display the information of AQI alerts, including metrics and stations.                                                                      |
|              | 2. The information of AQI alerts are grouped by metrics.                                                                                       |
| Row          | Folders holding detailed information (usually in folded state to maintain UI simplicity)                                                      |

### File Descriptions

| 檔案名稱                 | 功能                                           |
|--------------------------|----------------------------------------------|
| air_quality_exporter.py | 抓取化生系空品資料的 Prometheus exporter       |
| config.json             | air_quality_exporter.py 的設定檔             |
| api_air_prediction.py   | 抓取空品測站的照片並預測                     |
| test_module.py          | 使用預測模型進行預測                        |
| pm25_model.pth          | AI 預測模型                                  |
| prometheus.yml          | Prometheus Server 設定檔                     |
| air_quality_monitor.json| Grafana Dashboard 設定檔                     |