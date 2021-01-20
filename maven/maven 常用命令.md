# maven 常用命令

实际项目中经常用`maven`构建我们的项目

记录一下常用的`maven`命令

`mvn clean`  **清空target产生的项目**

`mvn compile`  **编译源代码**

`mvn install` **在本地`repository`中安装`jar`（包含`mvn compile mvn package`然后上传到本地仓库）**

`mvn deploy `  **上传到私服（包含`mvn install` 然后上传到私服）**

`mvn package ` **打包**

`mvn test` **运行测试**

`mvn site` **产生site**

`mvn test-compile` **编译测试代码**

`mvn -Dtest package`**只打包不测试**

`mvn jar:jar` **只打jar包**

`mvn test -skipping compile -skipping test-compile`**只测试不编译 也不测试编译** 

`mvn source.jar` **源码打包**

