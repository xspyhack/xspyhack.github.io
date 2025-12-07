## Cherry Blessing

个人技术博客源码仓库，基于 Hexo 构建。

### 快速开始

```bash
# 1. 克隆源码（新设备）
git clone git@github.com:xspyhack/xspyhack.github.io.git -b source
cd xspyhack.github.io

# 2. 安装依赖
npm install

# 3. 本地预览
npx hexo server
# 访问 http://localhost:4000
```

### 写作流程

```bash
# 创建新文章
npx hexo new "文章标题"

# 本地预览
npx hexo server

# 生成静态文件
npx hexo generate

# 部署到 master 分支（自动发布到 GitHub Pages）
npx hexo deploy
```

### 分支说明

- **source**: 源码分支（默认分支），存储 Markdown 文章和配置
- **master**: 部署分支，存储生成的静态网站，由 `hexo deploy` 自动推送

### 多设备同步

```bash
# 设备 A：提交更改
git add .
git commit -m "新文章：XXX"
git push origin source

# 设备 B：同步更改
git pull origin source
npm install  # 如果 package.json 有变化
```

### 注意事项

- 始终在 `source` 分支工作
- `public/` 和 `.deploy_git/` 已被忽略，不会提交
- 部署前确保本地生成成功：`npx hexo generate`
