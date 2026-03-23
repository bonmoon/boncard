# 手机卡片管家 / CardVault

一个偏高保真、可直接运行的单文件 PWA 原型，用来管理课程卡、会员卡和订阅续费。

当前仓库内容以 `index.html` 原型为主，配套 `PRD.md` 作为后续工程化实现说明。也就是说，这里现在不是完整的 React + Vite 项目，而是一个已经补齐核心交互的静态 demo。

## 当前已实现

- Apple Wallet 风格的卡片堆叠展示
- 课程卡 / 会员卡 / 订阅 三种卡片类型
- 新增、编辑、归档、恢复、彻底删除
- 搜索、筛选、临期提醒中心
- 本地存储与 JSON 备份导入导出
- 中文优先的本地 AI 问答逻辑
- `manifest.json`、`sw.js`、应用图标等基础 PWA 文件

## 文件说明

- [index.html](/Users/bonmoon/Downloads/手机卡片管家/index.html): 主应用原型，包含 UI、样式和交互逻辑
- [PRD.md](/Users/bonmoon/Downloads/手机卡片管家/PRD.md): 产品需求与后续 React 化架构参考
- [manifest.json](/Users/bonmoon/Downloads/手机卡片管家/manifest.json): PWA 配置
- [sw.js](/Users/bonmoon/Downloads/手机卡片管家/sw.js): 离线缓存用的基础 Service Worker

## 运行方式

直接双击 `index.html` 就可以预览大部分功能。

如果要测试 PWA 安装和 Service Worker，建议在当前目录启动一个本地静态服务，例如：

```bash
python3 -m http.server 4173
```

然后访问 [http://localhost:4173/index.html](http://localhost:4173/index.html)。

## 下一步建议

- 按 [PRD.md](/Users/bonmoon/Downloads/手机卡片管家/PRD.md) 迁移到 React + Vite + Zustand
- 接入真实的 Google Sheets 双层存储
- 接入 DeepSeek 或其他模型的在线 AI 助手
- 增加图片裁剪、声音资源与真正的通知调度
