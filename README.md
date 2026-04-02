# Week 6 — Prediction Shootout

雙事件降雨內插比較分析，涵蓋 Variogram fitting、四種空間內插方法，以及不確定性分析。

## 資料

| 事件 | 日期 | 颱風 | 縣市 |
|------|------|------|------|
| 事件 1 | 2024-07-25 | 凱米颱風 | 宜蘭縣、花蓮縣 |
| 事件 2 | 2024-11-11 | 鳳凰颱風 | 宜蘭縣、花蓮縣 |

資料來源：`data/rain_20240725.csv`、`data/rain_20241111.csv`
欄位：`StationLatitude`、`StationLongitude`、`Past1hr`（mm/hr）、`CountyName`
過濾條件：`Past1hr > 0`，排除 `-998`（缺值），投影轉換 EPSG:4326 → EPSG:3826

## 分析流程

### A1. Variogram Fitting
對兩個事件分別以 log 轉換後的雨量資料（`log1p(z)`）fit 兩種模型：

- **Spherical**
- **Exponential**

比較 SSE 選出各事件最佳模型，輸出參數表（Sill、Range、Nugget）。

### A2. 四種內插方法（1000m 解析度）

| 方法 | 說明 |
|------|------|
| Nearest Neighbor | scipy NearestNDInterpolator |
| IDW | power=2，分批計算 |
| Ordinary Kriging | 使用 A1 最佳 Variogram 模型，log 轉換 |
| Random Forest | n_estimators=200, min_samples_leaf=3 |

每個事件輸出 2×2 四圖並列比較圖與 Kriging vs RF 差異圖（RdBu_r colormap）。

### A3. 不確定性分析
輸出 Kriging Variance 視覺化（Sigma Map，Blues colormap）。

### A5. 跨事件綜合比較
彙整兩事件的最佳 Variogram 參數比較表。

## 輸出檔案

```
output/
├── variogram_fitting_comparison.png       # 2×2 variogram fitting 圖
├── variogram_parameters.csv              # 4 組 variogram 參數（2事件 × 2模型）
├── variogram_comparison.csv              # A5 跨事件最佳模型比較表
├── interpolation_comparison_事件_1_20240725.png
├── interpolation_comparison_事件_2_20241111.png
├── kriging_rf_difference_事件_1_20240725.png
├── kriging_rf_difference_事件_2_20241111.png
├── sigma_map_事件_1_20240725.png
├── sigma_map_事件_2_20241111.png
├── kriging_rainfall_20240725.tif          # Kriging 結果 GeoTIFF (EPSG:3826)
├── kriging_rainfall_20241111.tif
├── kriging_variance_20240725.tif          # Kriging Variance GeoTIFF
├── kriging_variance_20241111.tif
├── rf_rainfall_20240725.tif               # Random Forest 結果 GeoTIFF
└── rf_rainfall_20241111.tif
```

## reflection

## 環境需求

```
pandas
geopandas
numpy
matplotlib
shapely
pykrige
scikit-learn
scipy
rasterio
```

