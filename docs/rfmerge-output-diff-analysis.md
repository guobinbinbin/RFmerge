# 为什么两段 RFmerge 脚本结果会有差异

下面按“会直接改变数值结果”的优先级说明。

## 1) 输入数据并不一致（最核心）
- 第一段按“整年多层栅格 + `chinaPPts_<year>.RData`”处理：
  - 栅格：`ERA5_<year>_multilayer.tif`、`gsmap_<year>_multilayer.tif`、`imerg_<year>_multilayer.tif`
  - 站点：`chinaPPts_<year>.RData` 中的 `year_zoo`
- 第二段按“逐日单层栅格 + 每日 CSV”处理：
  - 栅格：`anusplin_*_china/<year>/<YYYY.MM.DD>.tif`
  - 站点：`<YYYYMMDD>.csv`
- 即使来源名看起来类似，只要重采样、偏差订正、时间聚合方式不完全相同，RFmerge 输出就会变。

## 2) 时间组织方式不同（全年序列 vs 单日循环）
- 第一段把一年作为一个时序对象整体送入 `RFmerge(...)`。
- 第二段每天单独跑一次 `RFmerge(...)`，并在并行 worker 中批处理。
- 对随机森林类流程来说，样本组织方式、每次训练的数据切片不同，会导致模型和最终融合结果出现可见差异。

## 3) 并行策略和随机性来源不同
- 第一段：`parallel = "parallelWin"`，`par.nnodes = min(detectCores()-1, 12)`（RFmerge 内部并行）。
- 第二段：外层 `future::multisession` 并行（每天/每批并行），RFmerge 调用里未显式启用 `parallelWin`。
- 两段都设置了 `seed = 123`，但并行调度顺序、worker 任务切分不同，仍可能让随机过程（抽样、树构建顺序）出现细小差别。

## 4) DEM 的处理链路不同
- 第一段：每年循环里对 DEM 做 `CRS` 对齐 + `resample(..., ERA5, method="bilinear")`，然后再投影到 Albers。
- 第二段：DEM 在主进程只投影一次（到 Albers），worker 里直接复用，不再按当天 ERA5 重新对齐/重采样。
- 这会改变协变量栅格值（尤其边缘与复杂地形区），进而影响 RFmerge 结果。

## 5) 掩膜对象与矢量处理时机不同
- 第一段在主进程一次性构建 `mask_vec` 并复用。
- 第二段每个 worker 重新加载并构建 `mask_vec_local`。
- 理论上应接近一致，但不同进程下对象精度、几何修复路径不同，可能带来极小差异。

## 6) 栅格对象转换方式不同（raster -> terra）
- 第一段 `safe_rast_convert()` 先写临时 tif，再 `terra::rast()` 读回。
- 第二段 `safe_rast_convert()` 直接 `terra::rast(raster_obj)` 内存转换。
- I/O 量化、数据类型、NA 写读行为可能略有不同，通常差异很小，但会叠加到最终结果。

## 7) 线程限制与运行环境不同
- 第二段显式限制了 `OMP/GDAL/BLAS/MKL` 线程为 1；第一段没有。
- 多线程数值运算在浮点累积顺序上可能产生微小差异（最后会反映到随机森林分裂评估和预测上）。

## 8) 预处理细节有小差别
- 第一段 `clean_negative_pp()` 对 zoo 用 `apply()` 列处理。
- 第二段先 `coredata()` 转矩阵再替换负值。
- 在常规数值场景下两者应接近，但对象类型、列属性、NA 传播细节可能不完全一致。

---

## 你可以怎么验证“差异来自哪里”
建议做“逐项对齐”的 A/B 实验，每次只改一个因素：
1. 先统一输入数据（都用同一套栅格与同一套站点时间序列）。
2. 再统一时间粒度（都按天或都按年）。
3. 再统一并行方式（都串行，或都同一并行框架）。
4. 再统一 DEM 处理流程（都按 ERA5 对齐后再投影，或都一次性预处理）。
5. 每步比较输出栅格的 MAE / RMSE / bias。

通常第 1、2、4 点会解释绝大部分差异。
