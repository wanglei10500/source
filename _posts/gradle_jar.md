---
title: 利用gradle生成可执行jar包
tags:
 - gradle
 - jar
categories: 经验分享
---
## 用gradle生成可执行jar包
gradle使用一种基于Groovy的特定领域语言(DSL)来声明项目设置。开发中遇到应用场景需要开发一个非Web工程的可执行jar包，又便于项目管理方便持续集成。所以找了一些关于用gradle生成可执行jar包的方法。
使生成的jar包可执行重要的是MANIFEST.MF文件，需要Main-Class和Class-Path两个属性。
Main-Class:程序的启动类，也就是main方法的所在类.
Class-Path:该属性我们应该将项目依赖的jar包路径配置在此。

关于Class-Path大概有两种情况。
1、手动引入的lib文件夹下的jar包
2、通过Gradle管理的依赖
对于1此部分的配置应为
```
jar {
    manifest {
        attributes 'Main-Class': 'com.myClass'
        attributes 'Class-Path': new File(libPath).list().findAll {
            it.endsWith('.jar') }.collect { "$libPath/$it" }.join(' ')
    }
}
```
第二种方式所有的配置如下
```

jar {
    manifest {
        attributes("Implementation-Title": project.name)
        attributes("Implementation-Version": version)
        attributes("Main-Class": "com.myClass")
    }
}
 
task addDependToManifest << {
    if (!configurations.runtime.isEmpty()) {
        jar.manifest.attributes('Class-Path': ". lib/" + configurations.runtime.collect { it.name }.join(' lib/'))
    }
}
jar.dependsOn += addDependToManifest 
 
task copyDependencies(type: Copy) {
  from configurations.runtime
  destinationDir = file('build/libs/lib')
}
jar.dependsOn += copyDependencies
```
用gradle构建后 在工程的build/libs下可以找到生成的jar包
java -jar xxx-1.0.jar 命令可执行jar包

## 参考文章
参考很多文章后，发现这篇博文介绍的非常详细，并且已经成功解决我的问题。
http://www.cnblogs.com/yongtao/p/4104526.html
