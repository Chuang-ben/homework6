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

## Reflection

#### 1. 不確定性分析

```
事件 1（凱米）variance 反而更高，因為凱米是強烈颱風，降雨強度最高達 102 mm/hr，空間梯度劇烈，variogram sill 遠大於事件 2（鳳凰，最高僅 14.5 mm/hr）。Kriging variance 的上限由 sill 決定，空間變異越大、不確定性就越高，即使測站數較多也無法完全抵消。事件 2 降雨較弱且空間分布平緩，sill 較小，variance 整體偏低，預測信心反而更高。

若為指揮官，面對高 variance 區域代表 Kriging 估計不確定性大，應採取保守策略：優先在該區域加密觀測（臨時雨量站或雷達補強），並以較高降雨量的情境作為防災決策依據，避免低估風險。

Random Forest 本身為確定性模型，無法直接輸出空間不確定性。雖可透過各決策樹的預測標準差估算不確定性，但其意義與 Kriging variance 不同——Kriging variance 源自空間結構與測站幾何關係，具備明確地統計意義；RF 的預測分散度僅反映模型內部的集成差異，無法量化空間插值本身的誤差。
```

#### 2. 最佳模型選擇理由

```
兩個事件皆選用 Spherical 模型，因其 SSE 均低於 Exponential（事件 1：0.1547 vs 0.1573；事件 2：0.0705 vs 0.0768）。Spherical 模型在達到 Range 後 semivariance 趨於平穩的特性，較符合颱風降雨在一定距離後空間相關性消失的物理行為。
```

#### 4. 兩事件 Variogram 參數差異

```
Sill：兩者相近（0.157 vs 0.130），反映整體降雨的空間變異量級相差不大，但凱米略高，對應其更強烈的降雨強度。
Nugget：事件 1 明顯較高（0.325 vs 0.110），代表測量誤差或極短距離內的隨機變異更大。
Range：差異最顯著（39km vs 148km）。Range 反映地理上的空間自相關尺度。
```

#### 5. 加分題回答
```
選擇事件 2（鳳凰）的參數，因為其 Nugget 較低、空間結構性較強，套用到其他事件時較不容易過度放大隨機誤差；Range 較大也讓內插在測站稀疏區域仍能產生合理的平滑結果。

這樣做的風險：

Variogram 參數本質上是針對特定降雨事件的空間結構所 fit 出來的，不同天氣系統（颱風、鋒面、對流）的 Sill、Range、Nugget 差異極大。若套用固定參數到高對流性降雨（如凱米），低估了真實的 Nugget，Kriging 會誤以為短距離內的降雨高度連續，產生過度平滑的結果，掩蓋局部極端值；若 Range 設得過大，則會將空間相關性延伸到實際上已無關聯的區域，導致內插失真。用一組參數以一擋百，本質上是用平均情況去近似所有情況，在極端事件下誤差最大、風險最高。
```

#### 6. 心得

```
這次的操作十分有趣，我學會了Kriging和Random Forest的應用，其中kriging非常強大，他可以根據空間自相關做出最佳的數據，而且也最貼近真實資料。不過由於windsurf的致命失誤使我必須全部重新操作一次，我可能再也不會想要使用這個軟體了。
```

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

