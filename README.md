# utools-ghostty

在 [uTools](https://u.tools) 中快速启动 [Ghostty](https://ghostty.org) 终端（macOS）。

## 功能

- 关键词 `ghostty` / `gt` 唤起插件
- 回车或点击即可打开 Ghostty

## 开发

1. 打开 uTools「开发者工具」
2. 选择「导入插件」，选中本目录的 `plugin.json`
3. 修改代码后插件会自动重载

## 目录结构

```
├── logo.png          插件图标
├── plugin.json       插件清单
└── src/
    ├── index.html    插件主界面
    └── preload.js    Node 预加载脚本
```

## 打包发布

在 uTools 开发者工具中点击「打包」，生成 `.upx` 后上传至插件市场。
