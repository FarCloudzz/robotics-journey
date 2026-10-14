# ROS 2 环境不完整：同一个包，两种失败的测试

- **日期**：2026-10
- **环境**：Ubuntu 24.04 (WSL2) + ROS 2 Jazzy + colcon
- **对象**：`zenbedded_hardware_interface`（[`ros-controls/zenbedded`](https://github.com/ros-controls/zenbedded)）
- **结论**：两次失败的**症状不同**，但都能由"运行命令的那个 shell 里 ROS 环境不完整"解释

## 症状一：动态库找不到

```console
$ colcon test --packages-select zenbedded_hardware_interface
...
test_interface_schema: error while loading shared libraries:
libzenohc.so: cannot open shared object file: No such file or directory
-- run_test.py: return code 127
...
Summary: 17 tests, 1 error, 1 failure, 0 skipped
```

## 症状二：Python 模块找不到

换一个终端复跑，同一个包换了另一种失败：

```console
ModuleNotFoundError: No module named 'ament_cmake_test'
```

两个测试包装脚本都在 `run_test.py` 的 `import` 阶段就挂了——**连被测二进制都没启动**。

## 前提：`colcon test` 报 `finished` 不代表通过

两次输出里都有这一行：

```
Finished <<< zenbedded_hardware_interface [1.46s]
```

区别只在后面跟不跟 `[ with test failures ]`。**只读 `finished` 或退出码，会把失败当成通过。** 唯一的判据是：

```bash
colcon test-result --verbose
```

## 排查过程（含走过的弯路）

先按"动态库路径问题"排查，得到三组事实：

1. **库存在**：`/opt/ros/jazzy/opt/zenoh_cpp_vendor/lib/libzenohc.so`
2. **`ldconfig -p` 不知道它** —— `/opt` 下的路径本就不在系统链接器缓存里，属于正常
3. **消费方的 RUNPATH 是正确的**：

```console
$ readelf -d build/zenbedded_hardware_interface/libzenbedded_hardware_interface.so
 NEEDED:  ... libzenohc.so ...
 RUNPATH: /opt/ros/jazzy/lib : /opt/ros/jazzy/lib/x86_64-linux-gnu
          : /opt/ros/jazzy/opt/zenoh_cpp_vendor/lib
```

第 3 条与"找不到库"的现象**直接矛盾**，于是先前的两个解释都被观察否定：

- ~~"`DT_RUNPATH` 不可传递，所以传递依赖找不到"~~ —— 语义本身没错，但**不是本次原因**
- ~~"vendor 包缺少 `ld_library_path` 环境钩子，属于打包缺陷"~~ —— **错**。`source` ROS 之后该目录**本来就在** `LD_LIBRARY_PATH` 里

一度据此准备给上游提 issue，**没有提交是正确的**：按错误结论提交会被维护者当场纠正。

### 弯路的核心教训

当时在拿**两个不同 shell** 的运行结果互相推理：

- 一次已 `source /opt/ros/jazzy/setup.bash`
- 一次没有
- 中间还夹了一次只影响单个终端的 `export LD_LIBRARY_PATH`

**输入不可比时，推理越严谨，错得越远。**

## 真正的判据

在当前**执行命令的这个 shell** 里直接确认环境，而不是猜测：

```bash
echo "ROS_DISTRO=[$ROS_DISTRO]"
echo "$LD_LIBRARY_PATH" | tr ':' '\n' | grep -i zenoh
```

`source /opt/ros/jazzy/setup.bash` 之后，第二条会输出：

```
/opt/ros/jazzy/opt/zenoh_cpp_vendor/lib
```

—— 这个目录**本来就在 ROS 环境里**。因此问题不在任何包的打包方式，而在环境是否完整加载。

## 解决

```bash
source /opt/ros/jazzy/setup.bash

cd ~/zephyr_zenoh_ws
colcon test --packages-select zenbedded_hardware_interface
colcon test-result --verbose
```

```console
Finished <<< zenbedded_hardware_interface [4.96s]
Summary: 30 tests, 0 errors, 0 failures, 0 skipped
```

写入 `~/.bashrc`，让每个新终端自动具备完整环境：

```bash
grep -q 'ros/jazzy/setup.bash' ~/.bashrc || \
  echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc
```

## 症状对照表

| 缺失的部分 | 症状 |
|---|---|
| `PYTHONPATH` | `ModuleNotFoundError: No module named 'ament_cmake_test'`（包装脚本 import 即失败） |
| `LD_LIBRARY_PATH` | `libzenohc.so: cannot open shared object file`（rc=127，二进制启动失败） |

两者都是**"ROS 环境不完整"**的不同表现——不是代码缺陷，也不是任何包的打包问题。

## 尚未确定的部分

失败那一刻的 shell 究竟缺了哪一项，**没有拿到证据**（当时没有打印 `LD_LIBRARY_PATH`，且两次运行来自不同环境）。因此本文不给出"因为 A 所以 B"的结论，只记录**可复现的判据与修复方式**。

## 可复用的规则

1. **先锁环境，再分析行为。** 任何 ROS/colcon 报错，第一步是在**执行该命令的那个 shell** 里确认 `ROS_DISTRO` 与 `LD_LIBRARY_PATH`。
2. **不要跨 shell 比较结果。** 那是两套不同的环境，比较得出的结论不会成立。
3. **`colcon test` 的 `finished` 不是判据**，`colcon test-result --verbose` 才是。
4. 把"可以报上游 bug"当作结论之前，先确认现象能在**干净且已 source** 的环境里稳定复现。
