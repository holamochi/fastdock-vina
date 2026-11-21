# FastDock Vina

**更快、更省内存的 AutoDock Vina：结果与原版逐字节一致，纯 CPU 运行，不需要 GPU。**

[性能实验记录](docs/BENCHMARKS.md) · [逐项数据](benchmarks/benchmark.csv) · [源码来源](docs/PROVENANCE.md) · [上游 AutoDock Vina](https://github.com/ccsb-scripps/AutoDock-Vina)

FastDock Vina 是 [AutoDock Vina](https://github.com/ccsb-scripps/AutoDock-Vina/tree/3c65c0b3e6c2c1d183f6a175ecb65e3c5ba91645) 的 CPU 优化版本，保留双精度计算和原有搜索逻辑，命令行用法与原版相同，可以直接替换。

![25 项测试的加速比及汇总指标](docs/figures/speedup-summary.png)

## 为什么用 FastDock Vina

做虚拟筛选时，Vina 常常要在 CPU 集群上跑成千上万次对接，耗时和内存直接决定一批任务要排多久。很多加速方案要么依赖 GPU，要么改动搜索策略，导致结果和原版对不上，难以放心用在正式研究里。

FastDock Vina 只做不改变计算结果的工程优化：**在已测配置下，同样的输入和随机种子，输出的 PDBQT 与原版逐字节一致**。

## 效果

测试平台为 Intel Xeon Platinum 8362，每个案例重复 3 次并取中位数：

| 指标 | 结果 |
| --- | --- |
| 25 项测试的等权几何平均加速比（19 项完整对接 + 6 项评分 / 局部优化） | **1.228×** |
| 19 项完整对接的合计耗时（原版 3242.525 秒 → 2989.231 秒） | 降低 **7.81%**（约 1.085×） |
| 峰值内存（RSS） | 降低 **28.71%–73.27%** |
| 对接输出 | 已测配置下与原版**逐字节一致** |

两个速度指标口径不同：1.228× 给每项测试相同权重；7.81% 是 19 项完整对接的总耗时变化。19 个完整对接案例都比原版快。所有结果只适用于已测的环境和输入，详细数据与验证范围见 [实验记录](docs/BENCHMARKS.md)。

![1iep 配体在受体中的位置，以及原版与 FastDock 构象的叠加](docs/figures/docking-overlay.png)

上图右侧放大了 MODEL 1：原版为蓝色，FastDock 为橙色，两者坐标完全重合（40 个原子的 RMSD 为 **0.000 Å**，未做结构对齐）。[对接输出](examples/basic/1iep_docked.pdbqt) · [图表数据与复现方法](docs/FIGURES.md)

## 做了哪些优化

- 相同原子类型对共享同一张评分查找表，减少重复计算和内存占用。
- 缓存热循环里的查表指针，用移动语义代替深拷贝。
- 调整 BFGS 矩阵布局，用 CPU SIMD 同时处理相互独立的行，并保留原有的累加顺序。

## 编译

需要 Python 3.8+、支持 C++14 的 GCC 或 Clang，以及 Boost（已验证 1.82）。

```bash
git clone https://github.com/holamochi/fastdock-vina.git
cd fastdock-vina
```

Ubuntu / Debian：

```bash
sudo apt-get install build-essential python3 libboost-all-dev
python3 tools/build.py --jobs 4
```

macOS（需要 Xcode 命令行工具和 Homebrew）：

```bash
brew install boost
python3 tools/build.py --boost-prefix "$(brew --prefix boost)" --jobs 4
```

## 运行

用仓库自带的 1iep 示例跑一次对接：

```bash
mkdir -p demo_out
./build/vina \
  --receptor examples/basic/1iep_receptor.pdbqt \
  --ligand examples/basic/1iep_ligand.pdbqt \
  --config examples/basic/1iep_receptor.box.txt \
  --scoring vina --cpu 1 --seed 20261022 \
  --exhaustiveness 8 --max_evals 0 --num_modes 9 \
  --out demo_out/1iep_out.pdbqt
```

结果保存在 `demo_out/1iep_out.pdbqt`，更多参数见 `./build/vina --help`。示例使用 `exhaustiveness=8`，上面的性能测试使用的是 `32`。

## 引用

研究中使用时，请引用 Vina 的原始论文：

- Trott, O. & Olson, A. J. AutoDock Vina: improving the speed and accuracy of docking with a new scoring function, efficient optimization, and multithreading. *J. Comput. Chem.* 31, 455–461 (2010). https://doi.org/10.1002/jcc.21334
- Eberhardt, J., Santos-Martins, D., Tillack, A. F. & Forli, S. AutoDock Vina 1.2.0: New Docking Methods, Expanded Force Field, and Python Bindings. *J. Chem. Inf. Model.* 61, 3891–3898 (2021). https://doi.org/10.1021/acs.jcim.1c00203

## Topic / Keywords

- **解决的问题**：AutoDock Vina 在 CPU 上做大规模分子对接和虚拟筛选时耗时长、内存占用高，而常见的加速方案依赖 GPU，或会改变计算结果。
- **实现方式**：在不改变双精度、搜索算法和累加顺序的前提下做 C++ 工程优化，包括共享评分查找表、缓存热循环指针、移动语义替代深拷贝、BFGS 矩阵 SIMD 化；用 25 项基准测试和逐字节输出比对来验证。
- **项目类别**：计算化学 · 分子对接 · 虚拟筛选 · 药物发现 · 高性能计算（HPC）· C++ 性能优化 · AutoDock Vina 分支
- **容易混淆**：它不是 GPU 版 Vina（如 Vina-GPU、AutoDock-GPU），没有新的打分函数，也不改变对接结果；它也不是 AutoDock Vina 官方发行版。
- **中文关键词**：分子对接、AutoDock Vina 加速、虚拟筛选、药物发现、蛋白-配体对接、CPU 优化、SIMD、内存优化、可复现、结果一致
- **English keywords**: molecular docking, AutoDock Vina, faster Vina, Vina CPU optimization, virtual screening, drug discovery, protein-ligand docking, PDBQT, SIMD, BFGS, memory reduction, bit-identical results, reproducible benchmark, computational chemistry, HPC, C++

## TL;DR

> FastDock Vina：AutoDock Vina 的纯 CPU 加速版，可以直接替换原版使用。已测配置下结果与原版逐字节一致，速度几何平均快 1.228×，峰值内存最多降低约 73%。

## Link

- GitHub 仓库：https://github.com/holamochi/fastdock-vina
- 上游 AutoDock Vina：https://github.com/ccsb-scripps/AutoDock-Vina
- 同类项目：
  - QuickVina 2（加速版 AutoDock Vina）：https://github.com/QVina/qvina
  - AutoDock-GPU（GPU 加速的 AutoDock）：https://github.com/forlilab/AutoDock-GPU
  - gnina（基于深度学习打分的分子对接）：https://github.com/gnina/gnina

## 开源许可

本项目遵循 [Apache License 2.0](LICENSE)，保留 AutoDock Vina 的版权与署名。源码来源见 [PROVENANCE](docs/PROVENANCE.md)。
