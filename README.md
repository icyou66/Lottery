# 大学生元旦晚会幸运大抽奖

一个基于 Vue3 + TypeScript + Vite 开发的元旦晚会抽奖系统

## 功能特点

- 🎊 元旦晚会主题，红色配色方案
- 🎯 16个奖品，4x4卡片布局
- ✨ 金色高亮轮廓抽奖动画
- ⏱️ 5秒抽奖动画，逐渐减速
- 🎉 中奖弹窗展示
- 📱 响应式设计，支持移动端

## 快速开始

### 安装依赖

```bash
npm install
```

### 启动开发服务器

```bash
npm run dev
```

### 构建生产版本

```bash
npm run build
```

### 预览生产构建

```bash
npm run preview
```

## 技术栈

- Vue 3.4
- TypeScript 5.3
- Vite 5.0
- 原生 CSS 动画

## 项目结构

```
元旦晚会代码/
├── src/
│   ├── App.vue          # 主应用组件
│   ├── main.ts          # 入口文件
│   ├── style.css        # 全局样式
│   └── vite-env.d.ts    # TypeScript 声明文件
├── index.html           # HTML 模板
├── package.json         # 项目配置
├── tsconfig.json        # TypeScript 配置
└── vite.config.ts       # Vite 配置
```

## 使用说明

1. 点击"开始抽奖"按钮启动抽奖
2. 金色高亮轮廓会在5秒内按顺序经过所有卡片
3. 5秒后停止在中奖卡片上
4. 弹窗显示中奖信息
5. 点击遮罩层或"确定"按钮关闭弹窗
6. 可以继续进行下一次抽奖

## 奖品配置

奖品数据在 `App.vue` 中的 `prizes` 数组中配置，每个奖品包含：
- `title`: 奖品标题
- `desc`: 奖品描述

可根据实际需求修改奖品内容。

