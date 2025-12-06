# App Template

这是一个应用模板目录，用于快速创建新的应用页面。

## 如何使用

1. 复制整个 `app-template` 目录：
   ```bash
   cp -r source/apps/app-template source/apps/your-app-codename
   ```

2. 修改内容：
   - `index.html` - 更新应用名称、描述、App Store 链接
   - `style.css` - 修改配色方案
   - `terms.html` - 根据应用功能编写用户协议
   - `privacy.html` - 根据应用功能编写隐私政策
   - `assets/app-icon.png` - 替换为应用图标

3. 更新 `/apps/index.html` 添加新应用卡片

4. 部署：
   ```bash
   npx hexo clean
   npx hexo generate
   npx hexo deploy
   ```

## 目录说明

- `index.html` - 应用主介绍页
- `style.css` - 独立样式文件
- `terms.html` - 用户协议
- `privacy.html` - 隐私政策
- `assets/` - 图片资源目录（放置 app-icon.png）
