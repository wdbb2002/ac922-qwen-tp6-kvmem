> 以下教程适用于完整离线交付包，命令须在该包根目录执行。本 GitHub 仓库当前仅提供文档及性能结果，尚未提供教程引用的 tools、fixtures、verification 和 image 载荷。文中的运行状态描述为交付验收当时的记录。

# 同款AC922千问27B / vLLM / KVMem有限验收精简包

> **收尾状态：有限范围复现完成，非全场景稳定发布。** v5单路256K及预热后的暖四路256K严格通过；独立冷四路触发NUMA8保护并安全停止（最低2.782GiB<3GiB），未完成输出，不能称合格。5K/30K/100K六组合没有本版成绩，八组合性能矩阵未完成。没有后台验收继续运行。

本包适用同款POWER9/ppc64le、6×V100-SXM2 16GB、驱动535.216.03。它含完整OCI运行环境、模型/tokenizer、原生库和编译缓存；不含宿主机操作系统/驱动安装。原OCI 37406064640字节，SHA256 a449d472d64724c8b9ab6719f34d4989b9225091328147cdea86b5ea2b4a46b9。精简包使用原OCI自带kv64375编译缓存，无需另行下载历史缓存。

当前配置是原冻结OCI加明确版本化strict-host-shift-kv64375-v5。GPU共享KV池596523 tokens。v4严格CPU放置后590697池暖四路发生一次抢占，因此v5恢复原64MiB/卡容量；未改变模型数学和评分。单请求上限262144 tokens，长测试输入261120加1024输出预算。四份256K历史借助CPU KVMem存储，不是四份KV同时全部常驻GPU。保留同native8数学库/精度/选择页预算；六worker内存使用严格CPU节点绑定，只有GPU2 worker的host内存目标改8，CPU亲和不变。没有采用会影响本机CUDA初始化的全局CPU-only cpuset，也没有采用可能向GPU NUMA fallback的soft interleave。

原namespace完整缓存与18个AOT model由镜像提供。AOT结果复用；每个新进程仍需重新捕获CUDA Graph。本版已通过和失败证据均在verification，PERFORMANCE_RESULTS.json明确列出缺项；不能用历史成绩补齐本版空项。

## 安装

先在整包根目录执行`python3 verify_bundle.py`。准备本地ext4/xfs：容器存储至少120GiB空余，临时解包至少50GiB；同盘建议220GiB以上空余。不要把graphstore直接设在NTFS共享盘；可把整包复制到服务器已有大盘后安装。需要宿主机已有Podman、同驱动和六个GPU设备、物理CPU NUMA0/8。安装器硬件门禁不为其他机型作保证。

```bash
sudo python3 tools/install_ac922_strict_capacity_v5_portable_20261025.py --bundle "$PWD" --root /data/ac922-qwen27-reproduction --load-only
```

root必须全新，父目录已存在且位于ext4/xfs。这一步全量校验原镜像、创建停止P2容器、自动应用三个钉SHA的版本payload，复用原镜像缓存；不先运行旧配置、不启动GPU、不安装开机服务。原OCI不修改，原字节备份和应用receipt保存在root/overlays/strict-host-shift-kv64375-v5。

确认GPU无人使用、8000端口空闲，在一个终端启动：

```bash
sudo python3 tools/ac922_portable_manage.py --root /data/ac922-qwen27-reproduction run --profile p2
```

另一终端等待并严格审计：

```bash
sudo python3 tools/ac922_portable_manage.py --root /data/ac922-qwen27-reproduction ready
sudo python3 tools/audit_portable_runtime_strict_capacity_v5_20261025.py --root /data/ac922-qwen27-reproduction --output /data/ac922-qwen27-reproduction/audit.json
```

审计包括62源、原生库真实映射、18缓存实际加载、596523池及六worker实际CPU匿名页/绑定策略。先做小规模真实代码验证：

```bash
sudo python3 tools/benchmark_portable_real_code_20261025.py --root /data/ac922-qwen27-reproduction --fixture "$PWD/fixtures/real_code_5120.json" --concurrency 1 --output /data/ac922-qwen27-reproduction/test-c1-5k
```

已验证路径：在全新P2进程先预热，再测暖四路256K（仍须保持探针原NUMA保护）：

```bash
sudo python3 tools/benchmark_portable_long_original_20261025.py --root /data/ac922-qwen27-reproduction --fixture "$PWD/fixtures/original_256k_261120.json" --concurrency 4 --allow-four-256k --phase prime-warm --output /data/ac922-qwen27-reproduction/test-prime-warm-256k
sudo python3 tools/ac922_portable_manage.py --root /data/ac922-qwen27-reproduction stop
```

本次冷四路没有通过：独立P2虽然启动与严格审计通过，但冷四路计算过程中NUMA8降至2917440KiB，探针按原门槛停止精确自有容器；NUMA0仍8149760KiB，合计约10.55GiB。这是CPU节点放置余量不足，不能改成放宽保护/宣称内存总量耗尽。冷四路暂不作为可用交付能力；不要在生产中直接并发四个未预热256K请求。完整冷四路修复、剩余六组合和断电恢复未完成。

DEV_MODE保持0，不启用/reset开发接口。输出目录必须全新，不覆盖失败证据。精简包只保留5K冒烟输入和256K验收输入；30K/100K数据留在原研发包，本包不提供其测试入口数据；本包没有伪造其性能数字。256K C1直接复用最终prime，暖4复用本次完整结果。未实施的CPU页迁移方案没有写入运行镜像或安装器。

## 数据口径和限制

prefill直接读取引擎local_compute实际计算token，和local_cache_hit/external_kv_transfer做守恒对账。主速度为实际计算token/发送至最后首token墙钟，含调度和首次decode；另存引擎各请求prefill耗时总和，不能冒称纯GPU kernel吞吐。逻辑输入吞吐与缓存命中分开。decode使用所有请求共同重叠区间合计token/s，四路要求实际running4、零抢占、完整输出和自然EOS。每个原长请求严格验证4个源码事实、128执行案例和8个模型自拟测试。有限fixture通过不代表所有模型智力/任务百分之百通过，也不等同于稀疏选择历史与dense输出逐值等价；ZSTD存储压缩本身无损。

每秒保护物理CPU0/8各至少3GiB、合计至少8GiB；不足明确停止本次精确标记容器。GPU NUMA250..255不算CPU内存预算。P1对照曾事实4/4与执行128/128正确，但自拟测试7/8，不能称合格回滚；未重测挑通过。早期版本的首chunk/启动显存OOM和NUMA保护失败均保留，当前不宣称历史故障已全部定位或任意启动100%成功。

停止仅本root标记的容器：

```bash
sudo python3 tools/ac922_portable_manage.py --root /data/ac922-qwen27-reproduction stop
```

本次已测路径为load-only加手动run/stop，不提供已验断电恢复、宿主机重启或开机自启保证。verification只保留导出/内容校验和最终v5成功、失败报告，没有旧版本成绩。


## 本次实际结果

|负载|实际prefill token/s（计算token/TTFT墙钟）|共同decode token/s|质量/抢占|
|---|---:|---:|---|
|单路261120冷输入|1594.927|149.923|全对/0抢占|
|四路261120预热后|1640.914|260.779|全对/0抢占/真实running4|
|四路261120独立冷启动|未合格，无完整速度|未合格|NUMA8保护停止|

暖四路490752实际计算、553728外部前缀命中；逻辑吞吐3492.399不是实际计算吞吐。单路223输出，暖四路868输出。九请求中目前只完成prime+warm共5请求；这5请求20事实/640执行/40自拟测试全部通过。服务已停止，GPU无人占用，8000无监听。安装器无论是否写load-only都只安装校验，永不自动启动或安装自启。


## 精简包说明

只使用上文唯一v5安装入口，tools里的其余文件是依赖组件，不必逐个运行。已移除旧安装器、历史补丁、kv6375缓存、历史日志、重复报告、未完成的30K/100K输入和Python缓存。原研发包留在原目录。

评分器的基础组件已封入tools，读取路径改为包内相邻文件，相应SHA锁定同步更新；实际评分程序和门槛未改变。这是便携性修复，没有重跑GPU性能，不将原报告冒称改包后的新机验收。

先读本文件；查看全部保留文件用途见文件清单.md。直接复制整个精简目录给朋友即可，不依赖原研发目录。安装器需要全量读取镜像进行SHA校验；不要用Python -O运行安装、管理与验收工具。
