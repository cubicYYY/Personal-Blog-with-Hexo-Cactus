# 基于Hexo cactus theme的个人博客

原cactus主题疏于维护，自行魔改部分内容
下文是一些技术备忘。

## 魔改

- 字体：霞鹜文楷阅读版
  - 修改`themes/cactus/_config.yml`中的cdn部分
  - 修改`themes/cactus/layout/_partial/styles.ejs` 以导入该CSS
  - 修改`themes/cactus/source/css/_variables.styl`中的`font-family`
- 修复了issue: [Post page navigation disappears when scrolling down](https://github.com/probberechts/hexo-theme-cactus/issues/378)

     ```diff
     - var topDistance = menu.offset().top;
     + var topDistance = document.documentElement.scrollTop;
     ```
