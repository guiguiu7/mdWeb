### 本网站使用Docsify 和 Cloud Flare Pages进行搭建

前提：需要先安装node环境

1. 全局安装docsify-cli

   ```bash
   npm i docsify-cli -g
   ```

2. 初始化文档目录

   ```bash
   docsify init ./md
   ```

3. 运行服务

   ```bash
   docsify serve
   ```



文档默认页面文件：

- _coverpage.md：网站封面
- _README.md：网站首页
- _sidebar.md：网站侧边栏



在`index.htm`l中进行配置：

##### 通用配置

```html
<script>
    window.$docsify = {
        name: '我的文档',
        repo: 'https://github.com/guiguiu7/mdWeb',
        coverpage: true,         // 如果不需要封面页可关闭
        themeColor: '#42b983',
        // 启用自动侧边栏
        autoSidebar: true,
        // 侧边栏文档目录
        loadSidebar: true,

        subMaxLevel: 2,
        alias: {
            "/.*/_sidebar.md": "/_sidebar.md",
        },
        // 跳转后自动到顶部
        auto2top: true,

        // --------------

        // 页面右侧toc
        toc: {
            tocMaxLevel: 2,
            target: "h2, h3, h4, h5, h6",
        },

        // 全文搜索
        search: {
            depth: 6,
            noData: "没有搜到!",
            placeholder: "搜索...",
            // 避免搜索索引冲突,同一域下的多个网站之间
            namespace: "website-1",
        },
        // 底部导航
        pagination: {
            previousText: "上一页",
            nextText: "下一页",
            crossChapter: true,
            crossChapterText: true,
        },
        // 字数统计
        count: {
            countable: true,
            position: "top",
            margin: "10px",
            float: "right",
            fontsize: "0.9em",
            color: "red",
            language: "chinese",
            localization: {
                words: "",
                minute: "",
            },
            isExpected: true,
        },
    };
</script>

<!-- docsify -->
<script src="https://cdn.jsdelivr.net/npm/docsify@4/lib/docsify.min.js"></script>

<!-- 代码高亮  https://cdn.jsdelivr.net/npm/prismjs@1/components/ -->
<script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-bash.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-python.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-cmake.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-java.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-csharp.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-docker.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-powershell.min.js"></script>

<!-- 多tab支持 -->
<script src="https://cdn.jsdelivr.net/npm/docsify-tabs@1/dist/docsify-tabs.min.js"></script>

<!-- 代码复制 -->
<script src="https://cdn.jsdelivr.net/npm/docsify-copy-code@2/dist/docsify-copy-code.min.js"></script>

<!-- 底部 上一页 / 下一页 -->
<script src="https://cdn.jsdelivr.net/npm/docsify-pagination@2/dist/docsify-pagination.min.js"></script>

<script src="https://cdn.jsdelivr.net/npm/docsify@4/lib/plugins/external-script.min.js"></script>

<script src="https://cdn.jsdelivr.net/npm/docsify@4/lib/plugins/ga.min.js"></script>

<!-- 全文搜索 -->
<script src="https://cdn.jsdelivr.net/npm/docsify@4/lib/plugins/search.js"></script>

<!-- 图片放大缩小 -->
<script src="https://cdn.jsdelivr.net/npm/docsify@4/lib/plugins/zoom-image.min.js"></script>

<!-- 字数统计 -->
<script src="https://cdn.jsdelivr.net/npm/docsify-count@latest/dist/countable.min.js"></script>

<!-- 侧边栏目录折叠 -->
<script src="https://cdn.jsdelivr.net/npm/docsify-sidebar-collapse/dist/docsify-sidebar-collapse.min.js"></script>

<!-- 页面右侧 TOC -->
<script src="https://cdn.jsdelivr.net/npm/docsify-plugin-toc@1.1.0/dist/docsify-plugin-toc.min.js"></script>

<!-- emoji -->
<script src="https://cdn.jsdelivr.net/npm/docsify/lib/plugins/emoji.min.js"></script>
```



自动目录生成使用[Docsify-Build-Sidebar](https://github.com/xxxxue/Docsify-Build-Sidebar)

- 在 Github 页面右侧的 "Releases" 中下载编译好的软件
- 修改 Config 中的 `HomePath` 为 `自己电脑上的目标根目录`
- 双击 exe 执行

