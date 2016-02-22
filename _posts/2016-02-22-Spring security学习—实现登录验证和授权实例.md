最近要把公司APP后台管理系统的安全部分改用Spring security实现（原来用shiro实现）。由于对spring框架以及spring-security的理解不充分，就花了不少时间研究，也看了不少别人的代码。这里通过一个自己写的spring-security样例来回顾总结一下自己当时理解不对的地方。

##### 样例程序说明：



##### 接下来就来一步一步构建这个样例：

1. 需要安装的程序: IDEA, MAVEN, MYSQL；
2. 在IDEA创建新MAVEN项目Spring-security，创建完新项目后项目目录如下： ![image]()

3. 编辑pom.xml文件，构建maven依赖，源代码如下，直接复制粘贴就好：

   ```XML
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>

       <groupId>org.springframework</groupId>
       <artifactId>gs-securing-web</artifactId>
       <version>0.1.0</version>

       <parent>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-parent</artifactId>
           <version>1.3.2.RELEASE</version>
       </parent>

       <dependencies>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-thymeleaf</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-security</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.security</groupId>
               <artifactId>spring-security-web</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.security</groupId>
               <artifactId>spring-security-config</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-data-jpa</artifactId>
           </dependency>
           <dependency>
               <groupId>mysql</groupId>
               <artifactId>mysql-connector-java</artifactId>
               <scope>runtime</scope>
           </dependency>
           <dependency>
               <groupId>mysql</groupId>
               <artifactId>mysql-connector-java</artifactId>
               <version>5.1.38</version>
               <scope>runtime</scope>
           </dependency>
       </dependencies>

       <properties>
           <java.version>1.8</java.version>
       </properties>
   ​

       <build>
           <plugins>
               <plugin>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-maven-plugin</artifactId>
               </plugin>
           </plugins>
       </build>

       <repositories>
           <repository>
               <id>spring-releases</id>
               <name>Spring Releases</name>
               <url>https://repo.spring.io/libs-release</url>
           </repository>
       </repositories>
       <pluginRepositories>
           <pluginRepository>
               <id>spring-releases</id>
               <name>Spring Releases</name>
               <url>https://repo.spring.io/libs-release</url>
           </pluginRepository>
       </pluginRepositories>

   </project>
   ```

   后续的过程中可能会碰到各种can't resolve xxx的情况，可能就是maven还没来得及把需要的jar包从网上下下来，可以尝试在IDEA中右击->Maven->Reimport，应该可以解决；

4. ​

   ​