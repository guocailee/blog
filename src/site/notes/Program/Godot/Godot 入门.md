---
{"dg-publish":true,"permalink":"/Program/Godot/Godot 入门/","noteIcon":"","created":"2026-01-24T11:47:43.799+08:00"}
---

# Godot 引擎入门

Godot 是一个免费、开源的游戏引擎，支持 2D 和 3D 游戏开发。

## 主要特点

- **完全开源**：MIT 许可证，无商业限制
- **轻量级**：引擎仅 70-100MB
- **跨平台**：支持 Windows、macOS、Linux、Android、iOS 等
- **多语言支持**：GDScript（官方语言）、C#、C++ 等
- **节点系统**：基于节点的架构，灵活的组合方式

## 引擎版本

- Godot 4.x：最新稳定版，支持 Vulkan 渲染
- Godot 3.x：成熟稳定版本，基于 OpenGL

## 核心概念

### 节点 (Node)
Godot 中一切皆节点，节点是游戏对象的基本单位。

### 场景 (Scene)
场景是节点的集合，可以保存、加载和实例化。场景是 Godot 的组织方式。

### 资源 (Resource)
资源是可复用的数据，如纹理、音频、脚本等。

## 工作流程

1. **创建节点**：添加需要的节点
2. **组织场景**：将节点组成场景树
3. **编写脚本**：使用 GDScript 为节点添加行为
4. **保存场景**：保存为可复用的场景文件
5. **实例化**：在主场景中实例化子场景

## 学习资源

- [官方文档](https://docs.godotengine.org)
- [官方教程](https://docs.godotengine.org/en/stable/tutorials)
- [Godot 示例项目](https://github.com/godotengine/godot-demo-projects)
