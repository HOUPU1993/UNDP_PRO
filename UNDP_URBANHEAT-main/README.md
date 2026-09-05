# UNDP_URBANHEAT

深圳 / 上海 / 北京三城市地表城市热岛（SUHI）网格化分析项目。

## 项目目标

用 Landsat 8/9 的地表温度（LST）数据，把三座城市统一切成 500m×500m 的网格，逐年（2018–2024，夏季6-8月）
算出每个格子的 LST（以及 NDVI、NDBI），最终产出一张跨城市、跨年份可比的网格化热岛数据表，作为后续空间分析
（相关性分析、热点分析等）的基础数据。

## 目录结构

```
UNDP_URBANHEAT-main/
├── README.md                      # 本文档
└── urban_heat_SZ_SH_BJ.ipynb      # 主分析notebook（Step0-6）
```

数据产出不放在这个代码文件夹里，而是存在旁边的 `UNDP_PRO/data/` 目录（notebook里的`LOCAL_DATA_DIR`），
包括：

- `{City}_boundary.geojson`：三座城市的市域边界
- `{City}_grid_500m.geojson`：三座城市的500m网格
- `urban_heat_grid_master.csv`：最终合并后的网格化结果表（城市 × 年份 × 格子）

GEE端的中间结果（每个 城市×年份 的 `reduceRegions` 输出）存在你GEE项目`houpuprj`下的Asset路径
`projects/houpuprj/assets/urban_heat_grid/`里，不占用本地或Google Drive空间。

## 关键方法决定

这些是前期讨论后拍板的几个关键选择，记录下来方便以后回顾/交接：

- **网格**：目标是500m×500m的方格，但geohash8实际只有约19m×38m（不是500m），所以放弃了"用geohash精度
  直接当网格"的想法。改成：每座城市在各自本地UTM投影下（深圳~UTM49/50N，上海UTM51N，北京UTM50N）手切
  规则500m网格，裁到市域边界。每个格子的ID是`城市_行_列`，同时给格子中心点顺带算一个geohash6字符串
  （约1.22km×0.61km大小的编码），纯粹当一个人类可读的位置标签用，跟格子实际大小无关。
- **城市边界**：用阿里云DataV.GeoAtlas（`https://geo.datav.aliyun.com/areas_v3/bound/{adcode}.json`）
  拿市域行政边界，深圳440300/上海310000/北京110000，dissolve成单一城市轮廓。
- **LST数据源**：只用Landsat 8/9 Collection 2 Level-2的`ST_B10`波段（USGS官方已做大气校正的LST产品，
  单位开尔文，notebook里转成摄氏度），不额外做split-window/emissivity反演。8号和9号卫星数据merge到一起
  增加影像密度、减少云污染。
- **时间范围**：2018–2024逐年夏季（6-8月）composite，共7期，可以看逐年变化趋势。
- **跨城市可比性**：三座城市气候背景差异很大（深圳亚热带 vs 北京温带），直接比较绝对LST意义不大，所以
  额外算了`LST_anomaly_C` = 格子LST − 该城市当年市域均值，作为初版的热岛强度指标。这是一个简化版本，
  更严谨的做法是用郊区/农村参照点而不是全市均值当基准（见notebook末尾"Ideas for extending this"）。

## 怎么运行

在你已经配置好的`houpu_py`环境（VS Code + Earth Engine Python API）里，按notebook里的Step0到Step6顺序
跑：

1. **Step0-1**：配置 + 拉三城市边界，跑完看一下地图，确认三个城市边界画对了。
2. **Step2**：本地切500m网格（纯Python计算，不用GEE），跑完看一下每个城市的格子数量和平均面积
   （应该接近0.25 km²）。
3. **Step3**：定义Landsat预处理函数，抽查Shenzhen 2023的LST地图，确认数值合理（大概20-45°C范围）。
4. **Step4**：批量提交21个导出任务（3城市×7年）到EE Asset。**跑之前**要先在GEE网页版Assets面板里，
   在`houpuprj`项目下手动建一个叫`urban_heat_grid`的folder。提交后去
   [Tasks面板](https://code.earthengine.google.com/tasks) 看进度，北京最大的年份格子数有6万+，
   可能要等一阵子。
5. **Step5**：等Step4全部任务变成COMPLETED后，读回Asset数据，合并成一张表，存到本地`data/`目录。
6. **Step6**：画一版单城市网格热力图 + 三城市逐年趋势对比图。

## 已知问题 / Troubleshooting

- **`geemap.geopandas_to_ee`报`Read-only file system`错误**：这个函数内部会把GeoDataFrame先写成一个临时
  geojson文件到当前工作目录，再读回来转成`ee.FeatureCollection`。如果notebook kernel的工作目录恰好不可写
  （比如cwd是`/`），这一步就会失败。已经在notebook里加了一个`gdf_to_ee_fc`辅助函数替代所有
  `geemap.geopandas_to_ee`调用——它直接在内存里把GeoDataFrame转成GeoJSON dict再喂给
  `geemap.geojson_to_ee`，不落盘，不受cwd是否可写影响。如果你在其他notebook里也遇到同样的报错，可以照搬
  这个函数。

## 维护约定

这个项目往后由Claude帮忙持续维护。每次改动notebook代码时，Claude会：

1. 把这份README同步更新（新增/调整的步骤、参数、方法决定都要反映进来）；
2. 重新检查一遍notebook——语法、逻辑一致性、注释是否还准确——而不只是改动的那一小块。

notebook里的代码注释和markdown说明统一用英文（方便以后可能的合作者/审阅），这份README保持中文。

## 后续可以扩展的方向

- 用`LST_anomaly_C`对`NDVI`/`NDBI`做散点图+回归，量化植被/建成度对局地热岛强度的解释力。
- 换成郊区/农村参照点定义SUHI基准，而不是全市均值。
- 用`esda`/`pysal`做Getis-Ord Gi*热点分析，找每个城市统计显著的热岛核心区。
- 按区/县拆开看（尤其北京郊区山地和城六区差异会很大，市域均值可能被拉平）。
