# 概念笔记

按主题一篇，篇幅不求长。**只写自己真正用上、或被坑过的部分**，不做文档复述。

## 计划中的主题

- [ ] [`colcon-workspace.md`](colcon-workspace.md) —— colcon、overlay/underlay、rosdep
- [ ] [`ros2-control.md`](ros2-control.md) —— `controller_manager` / `hardware_interface` / 资源管理
- [ ] [`pluginlib.md`](pluginlib.md) —— C++ 类如何按名字被加载
- [ ] [`rmw-dds-qos.md`](rmw-dds-qos.md) —— RMW 抽象层、DDS 与 QoS
- [ ] [`zenoh.md`](zenoh.md) —— Zenoh 是什么，替代 DDS 做 MCU 接入的取舍
- [ ] [`micro-ros.md`](micro-ros.md) —— Micro-ROS 架构，与 zenbedded 的对比

## 写作约定

1. 开头写清**它解决什么问题**，再写机制，最后写**我怎么用上 / 被坑的**
2. 引用官方文档链接，而不是复述内容
3. 每篇末尾留一行"待验证"，列出还没搞清的点——它们是最诚实的部分
