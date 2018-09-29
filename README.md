# 文档协作说明

该仓库是用来生成 gitbook 文档用的，请严格按照以下说明来添加资料

- Gitbook 访问地址(会自动发布)：https://terark.gitbooks.io/terarkdb-manual/content/
- Terark 官网访问地址（会自动发布）：http://terark.com/docs/terarkdb-manual/
- 因为 push 后会触发编译发布逻辑，请不要频繁的 push 代码


## 目录结构说明
- en/  英文文档路径
- zh-hans/  中文文档路径
  - SUMMARY.md  文档目录，该文件中的目录需要引用同一目录下的其他 md 文档
  - xxx.md 每个独立的文档
- images/ 所有的图片均统一放在此处（包括多语言）
  - 每个文档所使用的图片，请单独创建文件夹，如 `images/ycsb_benchmark/xxx.svg`

  
## 添加新文档步骤

1. 请首先在 zh-hans/SUMMARY.md(中文) 中找到合适的位置把新文档添加到目录中
2. 在 zh-hans/ 目录下创建新文档
3. 新的文档 push 到仓库后，可以通过 gitbook 或官网地址来查看效果
