# C语言教程网站

这是一个基于MkDocs构建的C语言教程网站，包含从基础到进阶的完整C语言学习内容。

## 📋 项目特性

- 🎯 **循序渐进的学习路径**：从Hello World到高级特性
- 📱 **响应式设计**：支持手机、平板和桌面设备
- 🌙 **主题切换**：支持浅色和深色主题
- 🔍 **搜索功能**：全文搜索支持
- 📝 **代码高亮**：语法高亮和代码复制功能
- 🌐 **中文界面**：完全中文化的学习体验

## 🚀 快速开始

### 环境要求

- Python 3.6+
- pip 包管理器

### 安装步骤

1. **克隆仓库**
   ```bash
   git clone git@github.com:LiYijin/npcpp-tutorial.git
   cd npcpp-tutorial
   ```

2. **安装依赖**
   ```bash
   # 使用系统pip
   pip install mkdocs mkdocs-material

   # 如果使用conda环境
   ~/anaconda3/bin/pip install mkdocs mkdocs-material
   ```

3. **启动本地服务器**
   ```bash
   # 基本启动
   mkdocs serve

   # 指定端口和地址（推荐）
   mkdocs serve --dev-addr=0.0.0.0:8000

   # 如果使用conda环境
   ~/anaconda3/bin/mkdocs serve --dev-addr=0.0.0.0:8000
   ```

4. **访问网站**

   打开浏览器访问：http://localhost:8000 或 http://0.0.0.0:8000

## 🛠️ 开发指南

### 本地开发

1. **启动开发服务器**
   ```bash
   mkdocs serve
   ```
   开发服务器支持实时预览，修改文件后网站会自动刷新。

2. **指定开发端口**
   ```bash
   mkdocs serve --dev-addr=0.0.0.0:8000
   ```

3. **启用详细日志**
   ```bash
   mkdocs serve --verbose
   ```

### 构建静态网站

```bash
# 构建生产版本
mkdocs build

# 构建的静态文件位于 site/ 目录中
```

### 部署

部署到GitHub Pages：

```bash
mkdocs gh-deploy
```

## 📁 项目结构

```
npcpp-tutorial/
├── docs/                    # 文档源文件
│   ├── index.md            # 首页
│   ├── 基础语法/            # 基础语法教程
│   │   ├── index.md
│   │   ├── 第一个程序.md
│   │   ├── 变量与数据类型.md
│   │   ├── 运算符.md
│   │   ├── 输入输出.md
│   │   ├── 控制流程.md
│   │   └── 快速开始.md
│   ├── 进阶特性/            # 进阶特性教程
│   │   └── index.md
│   ├── 实践示例/            # 实践示例（待补充）
│   └── 参考资源/            # 参考资源（待补充）
├── mkdocs.yml              # MkDocs配置文件
└── README.md               # 项目说明文件
```

## 🎨 自定义配置

### 主题配置

项目使用Material for MkDocs主题，支持以下自定义：

- 主题切换（浅色/深色）
- 导航结构
- 代码高亮
- 搜索功能

配置文件：`mkdocs.yml`

### 添加新内容

1. 在`docs/`目录下创建新的Markdown文件
2. 在`mkdocs.yml`的`nav`部分添加导航链接
3. 运行`mkdocs serve`预览效果

## 🐛 常见问题

### 1. 依赖安装失败

```bash
# 使用国内镜像源
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple mkdocs mkdocs-material
```

### 2. 端口被占用

```bash
# 使用其他端口
mkdocs serve --dev-addr=0.0.0.0:8080
```

### 3. 权限问题

```bash
# 使用用户权限安装
pip install --user mkdocs mkdocs-material
```

## 📚 教程内容

### 基础语法
- [第一个程序](基础语法/第一个程序.md) - Hello World和程序结构
- [变量与数据类型](基础语法/变量与数据类型.md) - 基本数据类型详解
- [运算符](基础语法/运算符.md) - 算术、关系、逻辑运算符
- [输入输出](基础语法/输入输出.md) - printf和scanf使用
- [控制流程](基础语法/控制流程.md) - 条件语句和循环
- [快速开始](基础语法/快速开始.md) - 实用示例代码

### 进阶特性
- [函数](进阶特性/index.md) - 模块化编程（待补充）
- [数组](进阶特性/index.md) - 数据集合处理（待补充）
- [指针](进阶特性/index.md) - 内存操作（待补充）
- [结构体](进阶特性/index.md) - 自定义数据类型（待补充）
- [文件操作](进阶特性/index.md) - 文件读写（待补充）

## 🤝 贡献指南

欢迎贡献内容！请遵循以下步骤：

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启 Pull Request

## 📄 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情。

## 🔗 相关链接

- [MkDocs 官方文档](https://www.mkdocs.org/)
- [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/)
- [C 语言参考手册](https://zh.cppreference.com/w/c)

## 👨‍💻 作者

- **Li Yijin** - [GitHub](https://github.com/LiYijin)

---

⭐ 如果这个项目对您有帮助，请给个星标支持一下！