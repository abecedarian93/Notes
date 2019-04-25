#### Maven常见插件
###### 插件作用为定义构建逻辑,常见插件有：
- maven-compiler-plugin(编译插件)
- maven-resources-plugin(资源插件)
- maven-surefire-plugin(测试插件)
- maven-clean-plugin(清除插件)
- maven-war-plugin(打包插件)
- maven-dependency-plugin(依赖插件)
- maven-scala-plugin(scala插件)
- build-helper-maven-plugin(自定义编译插件)
- maven-assembly-plugin(定制化打包插件)
- maven-shade-plugin(可执行包插件)

###### maven-compiler-plugin 编辑插件
```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.1</version>
    <configuration>
       <source>1.8</source>  <!-- 源代码使用jdk1.8 -->
       <target>1.8</target>  <!-- 使用jvm1.8编译目标代码 -->
       <encoding>UTF-8</encoding> <!-- 编码方式UTF-8 -->
    </configuration>
</plugin>
```

###### maven-resources-plugin 资源插件
```
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-dependency-plugin</artifactId>
  <version>2.8</version> 
  <executions>
      <execution>
          <id>copy</id>
          <phase>package</phase>
          <goals>
              <goal>copy-dependencies</goal>
          </goals>
          <configuration>
              <outputDirectory>./target/classes/lib</outputDirectory>   <!-- 把所有依赖的jar拷到./target/classes/lib下 -->
          </configuration>
      </execution>
  </executions>
</plugin>
```

###### maven-scala-plugin scala插件
```
<plugin>
    <groupId>org.scala-tools</groupId>
    <artifactId>maven-scala-plugin</artifactId>
    <version>2.15.0</version>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
                <goal>testCompile</goal>
            </goals>
            <configuration>
                <args>
                    <arg>-dependencyfile</arg>
                    <arg>${project.build.directory}/.scala_dependencies</arg>
                </args>
            </configuration>
        </execution>
    </executions>
</plugin>
```

####### build-helper-maven-plugin 自定义编译插件
````
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>build-helper-maven-plugin</artifactId>
    <version>1.1</version>
    <executions>
        <execution>
            <id>add-source</id>
            <phase>generate-sources</phase>
            <goals>
                <goal>add-source</goal>
            </goals>
            <configuration>
                <sources>
                    <source>src/main/code</source>  <!-- 指定编译代码路径 -->
                </sources>
            </configuration>
        </execution>
    </executions>
</plugin>
````

###### maven-assembly-plugin 定制化打包插件
````
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>  <!-- 包后缀名 -->
        </descriptorRefs>
        <archive>
            <manifest>
                <mainClass></mainClass>
            </manifest>
        </archive>
    </configuration>
    <executions>
        <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
````

###### maven-shade-plugin 可执行包插件
````
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>2.4.3</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <relocations>
            <relocation>
                <pattern>org.apache.hadoop.hive.ql.exec</pattern>  <!-- 原始类名 -->
                <shadedPattern>org.shade.hadoop.hive.ql.exec</shadedPattern><!--  打包后替换的类名 -->
            </relocation>
        </relocations>

        <shadedArtifactAttached>true</shadedArtifactAttached>
        <shadedClassifierName>jar-with-dependencies</shadedClassifierName>  <!-- 包后缀名 -->
    </configuration>
</plugin>
`````