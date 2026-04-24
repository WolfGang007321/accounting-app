# 记账本

一款简洁好看的记账网页应用，帮你记录每天的收支，掌握财务状况。

**在线访问：** https://unique-chaja-52fd9c.netlify.app

---

## 功能

- 记收入 / 记支出
- 月度统计：收入、支出、结余
- 支出分布饼图，直观看到钱花哪儿了
- 月份切换，回顾历史记录
- 月预算设置，超支警告
- 累计结余，看整体财务状况
- 编辑 / 删除记录
- 手机端适配

---

## 技术栈

| 技术 | 说明 |
|------|------|
| Vue 3 | 前端框架 |
| Vite | 构建工具 |
| Chart.js | 饼图图表 |
| localStorage | 本地数据存储 |

---

## 本地运行

```bash
npm install
npm run dev
```

---

## 部署

```bash
npm run build
```

构建完成后，`docs/` 目录即为可部署的文件。

---

## 项目结构

```
├── src/
│   ├── App.vue      # 主应用组件
│   ├── main.js      # 入口文件
│   └── style.css    # 全局样式
├── docs/            # 打包后的静态文件
├── public/          # 静态资源
├── index.html       # HTML 入口
└── vite.config.js   # Vite 配置
```
