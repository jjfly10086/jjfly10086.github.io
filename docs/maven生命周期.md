## 生命周期
maven
    clean   清理target目录
    compile 编译class到target目录
    package 编译class到target目录 + 打包到target目录 //常用于项目打包
    install 编译class到target目录 + 打包到target目录 + 生成的jar包安装到本地maven仓库
    deploy  编译class到target目录 + 打包到target目录 + 生成的jar包安装到本地maven仓库 + 发布到远程仓库
