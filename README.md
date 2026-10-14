# robotics-journey

从软件工程转向**机器人 / 具身智能**的公开技术记录：概念笔记、真实排障、基准测试与实践产出。

> 本仓库是**技术过程记录**，不是简历。方向定位：实时控制与系统集成（ROS 2 + Zephyr/RTOS + `ros2_control`）。

## 当前进度

| 阶段 | 周次 | 主题 | 状态 |
|---|---|---|---|
| 一 | W1–W8 | 焊接底层与生态（Zephyr + Zenoh/Micro-ROS + `ros2_control`） | 🟨 进行中 |
| 二 | W9–W20 | 机器人核心算法（Gazebo + Nav2 + 自定义插件） | ⬜ 未开始 |
| 三 | W21–W32 | 杀手级项目（Zephyr + ROS 2 移动机器人实时导航） | ⬜ 未开始 |

_最后更新：2026-10（W1）_

## 仓库导览

| 目录 | 内容 |
|---|---|
| [`notes/`](notes/) | 概念笔记：ros2_control、pluginlib、RMW/DDS、Zenoh |
| [`troubleshooting/`](troubleshooting/) | 真实排障记录（现象 → 诊断 → 根因 → 验证） |
| [`benchmarks/`](benchmarks/) | 性能对比：延迟 / 抖动 / 内存，含采集脚本 |
| [`weekly/`](weekly/) | 每周复盘 |
| [`assets/`](assets/) | 架构图、截图、演示链接 |

## 值得一读

- [WSL2 内 GitHub 不可达：从 hosts 劫持到中间人证书链](troubleshooting/2026-10-wsl-github-network.md)

## 技术栈

`ROS 2 Jazzy` · `Ubuntu 24.04 (WSL2)` · `C++17` · `Zephyr RTOS` · `Zenoh` · `ros2_control` · `Nav2` · `Gazebo`

## 说明

本仓库只包含个人学习产物。所属雇主的代码、文档与内部信息不在其中，也不会出现在其中。
