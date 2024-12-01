## 數位研公式サイト  
本项目为數位研官方网站代码仓库  
网页通过[持续集成](https://hexo.io/zh-cn/docs/github-pages)发布为artifact, 请将博文以markdown格式存储至dev分支  

### 维护笔记
* 当前网站主题适配头图4:3, 建议像素960x720或文件大小<=100kb  
* 如果做错了什么, 就`git reset --soft HEAD`回滚到最后一个完好的提交, 修改完再`git push origin dev --force`  
* markdown页首的文章属性块分隔符不需要追加换行符  