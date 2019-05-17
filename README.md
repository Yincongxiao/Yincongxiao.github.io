## This repository is for backup of hexo blogs.
目前使用`hexo-git-backup`来备份hexo源文件,方便迁移和协同合作.
如果想在别的电脑上更新博客需要进行以下步骤:
#### 环境配置
* 确认电脑是否有ssh权限,是否有git,npm,node环境
* 安装hexo工具套件:
     - 安装hexo: `sudo npm install -g hexo`
     - 安装hexo 发布工具:`npm install hexo-deployer-git --save`
     - 安装hexo 备份工具: `npm install hexo-git-backup --save`
* 在空文件夹下克隆仓库`git clone https://github.com/Yincongxiao/Yincongxiao.github.io.git `,克隆完毕后切换到hexo_source分支. `git checkout hexo_source`
* `npm install`


#### 更新文章

* 在主目录下`hexo new post "文章名字"`
* 生成html文件: `hexo g`
* 发布: `hexo d`
* 备份: `hexo b`
* 在需要情况下可以使用`hexo clean` 清除缓存,`hexo s`可以通过[localhost:4000](localhost:4000)本地预览.
