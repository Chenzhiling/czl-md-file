# Maven插件使用

## 1. 前言

记录下自己使用maven插件的经历

## 2. maven-scala-plugin

使用该插件将同时存在java和scala代码的项目进行打包

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.scala-tools</groupId>
            <artifactId>maven-scala-plugin</artifactId>
            <version>2.15.2</version>
            <executions>
                <execution>
                    <goals>
                        <!--编译源码-->
                        <goal>compile</goal>
                        <!--编译测试源码-->
                        <goal>testCompile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

## 3. maven-surfire-plugin

使用该插件打包时可以跳过测试文件

```xml
<build>
    <plugins>
        <!--打包跳过测试-->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>2.7</version>
            <configuration>
                <skipTests>true</skipTests>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 4. maven-dependency-plugin

使用该插件可将项目用到的第三方依赖也进行打包

- copy-dependencies

```xml
<build>
    <plugins>
        <!--拷贝依赖-->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <version>3.0.2</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
     </plugins>
</build>
```

- copy

```xml
<build>    
    <plugins>
        <!--拷贝依赖-->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <version>3.0.2</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>copy</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <excludeTransitive>false</excludeTransitive>
                <stripVersion>false</stripVersion>
                <artifactItems>
                    <!--将这个依赖放入storage目录下-->
                    <dependency>
                        <groupId>org.czl.study</groupId>
                        <artifactId>object-ceph</artifactId>
                        <version>${project.version}</version>
                        <outputDirectory>${project.build.directory}/storage</outputDirectory>
                    </dependency>
                </artifactItems>
            </configuration>
     </plugin>
</build>
```



## 5. maven-assembly-plugin

使用该插件配合设定的assembly.xml文件将项目打成自己想要的格式

```xml
 <build>
     <plugins>
         <!--assembly插件，打到assembly文件夹下-->
         <plugin>
             <groupId>org.apache.maven.plugins</groupId>
             <artifactId>maven-assembly-plugin</artifactId>
             <version>3.1.1</version>
             <!--执行打包-->
             <executions>
                 <execution>
                     <!--名字随意-->
                     <id>CZL-assembly</id>
                     <!--package操作上-->
                     <phase>package</phase>
                     <goals>
                         <!--只进行一次-->
                         <goal>single</goal>
                     </goals>
                 </execution>
             </executions>
             <configuration>
                 <!--true finalName属性值作为打包文件的前缀，assembly文件中的id作为紧随其后的名称。false不与id连用-->
                 <appendAssemblyId>false</appendAssemblyId>
                 <descriptors>
                     <descriptor>assembly.xml</descriptor>
                 </descriptors>
             </configuration>
         </plugin>
      </plugins>
</build>
```

## 6. assembly.xml

```xml
<assembly>
    <!--添加到生成文件名称的后缀符 例如${artifactId}-${id}.tar.gz-->
    <id>bin</id>
    <formats>
        <!--支持的打包格式-->
        <format>tar.gz</format>
    </formats>
    <!--将依赖打入指定文件夹-->
    <dependencySets>
        <dependencySet>
            <unpack>false</unpack>
            <!--符合runtime作用范围的依赖会被打包进去-->
            <scope>runtime</scope>
            <!--是否把当前pom项目添加到依赖文件夹下-->
            <useProjectArtifact>true</useProjectArtifact>
            <!--将项目所有依赖包拷贝到lib目录下-->
            <outputDirectory>lib</outputDirectory>
            <excludes>
                <!--排除以下jar包-->
                <exclude>javax.servlet:servlet-api</exclude>
            </excludes>
        </dependencySet>
        <!--将这个包放入plugins目录下-->
        <dependencySet>
            <outputDirectory>plugins</outputDirectory>
            <includes>
                <include>org.czl:study</include>
            </includes>
        </dependencySet>
    </dependencySets>


    <fileSets>
        <fileSet>
            <directory>src/assembly/bin</directory>
            <outputDirectory>bin</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
        <!--将target/lib下的依赖放入lib-->
        <fileSet>
            <directory>${project.build.directory}/lib</directory>
            <outputDirectory>lib</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
        <fileSet>
            <directory>src/assembly/logs</directory>
            <outputDirectory>logs</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
        <fileSet>
            <directory>src/assembly/temp</directory>
            <outputDirectory>temp</outputDirectory>
            <fileMode>0777</fileMode>
        </fileSet>
        <fileSet>
            <directory>src/assembly/client</directory>
            <outputDirectory>client</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
        <fileSet>
            <directory>src/assembly/script</directory>
            <outputDirectory>script</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
        <fileSet>
            <directory>src/assembly/plugins</directory>
            <outputDirectory>plugins</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
        <!--将resources资源下配置文件打包放入到conf文件夹下-->
        <fileSet>
            <directory>src/main/resources</directory>
            <outputDirectory>conf</outputDirectory>
            <fileMode>0755</fileMode>
            <includes>
                <include>*.yml</include>
                <include>*.properties</include>
            </includes>
        </fileSet>
        <!--根目录下的configutation放入到conf文件夹下-->
        <fileSet>
            <directory>${project.parent.parent.basedir}/configuration</directory>
            <outputDirectory>bin/conf</outputDirectory>
            <fileMode>0755</fileMode>
            <includes>
                <include>*.properties</include>
            </includes>
        </fileSet>
    </fileSets>
</assembly>
```

