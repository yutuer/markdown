## maven 打包配置

包括版本修改.  jar包分离.  jar包打入到一起

```xml
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <!-- 打包jar文件时，配置manifest文件，加入lib包的jar依赖 -->
            <!--<plugin>-->
                <!--<groupId>org.apache.maven.plugins</groupId>-->
                <!--<artifactId>maven-jar-plugin</artifactId>-->
                <!--<configuration>-->
                    <!--<archive>-->
                        <!--<manifest>-->
                            <!--<addClasspath>true</addClasspath>-->
                            <!--<classpathPrefix>circleExcelCheck/</classpathPrefix>-->
                            <!--<mainClass>CheckCheckBootStrap</mainClass>-->
                        <!--</manifest>-->
                    <!--</archive>-->
                    <!--<excludes>-->
                        <!--<exclude>dict.properties</exclude>-->
                        <!--<exclude>log4j2-test.xml</exclude>-->
                        <!--<exclude>classes/**</exclude>-->
                    <!--</excludes>-->
                <!--</configuration>-->
            <!--</plugin>-->
            <!--&lt;!&ndash; 拷贝依赖的jar包到lib目录 &ndash;&gt;-->
            <!--<plugin>-->
                <!--<groupId>org.apache.maven.plugins</groupId>-->
                <!--<artifactId>maven-dependency-plugin</artifactId>-->
                <!--<executions>-->
                    <!--<execution>-->
                        <!--<id>copy</id>-->
                        <!--<phase>package</phase>-->
                        <!--<goals>-->
                            <!--<goal>copy-dependencies</goal>-->
                        <!--</goals>-->
                        <!--<configuration>-->
                            <!--&lt;!&ndash; ${project.build.directory}是maven变量，内置的，表示target目录,如果不写，将在跟目录下创建/lib &ndash;&gt;-->
                            <!--<outputDirectory>${project.build.directory}/circleExcelCheck</outputDirectory>-->
                            <!--&lt;!&ndash; excludeTransitive:是否不包含间接依赖包，比如我们依赖A，但是A又依赖了B，我们是否也要把B打进去 默认不打&ndash;&gt;-->
                            <!--<excludeTransitive>false</excludeTransitive>-->
                            <!--&lt;!&ndash; 复制的jar文件去掉版本信息 &ndash;&gt;-->
                            <!--&lt;!&ndash;<stripVersion>true</stripVersion>&ndash;&gt;-->
                        <!--</configuration>-->
                    <!--</execution>-->
                <!--</executions>-->
            <!--</plugin>-->
            <!--&lt;!&ndash; 解决资源文件的编码问题 &ndash;&gt;-->
            <!--<plugin>-->
                <!--<groupId>org.apache.maven.plugins</groupId>-->
                <!--<artifactId>maven-resources-plugin</artifactId>-->
                <!--<configuration>-->
                    <!--<encoding>UTF-8</encoding>-->
                <!--</configuration>-->
            <!--</plugin>-->
            <!--&lt;!&ndash; 打包source文件为jar文件 &ndash;&gt;-->
            <!--<plugin>-->
                <!--<artifactId>maven-source-plugin</artifactId>-->
                <!--<version>2.1</version>-->
                <!--<configuration>-->
                    <!--<attach>true</attach>-->
                <!--</configuration>-->
                <!--<executions>-->
                    <!--<execution>-->
                        <!--<phase>compile</phase>-->
                        <!--<goals>-->
                            <!--<goal>jar</goal>-->
                        <!--</goals>-->
                    <!--</execution>-->
                <!--</executions>-->
            <!--</plugin>-->


            <plugin>
                <finalName>circleCheck</finalName>
                
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>

                        <configuration>
                            <!--<minimizeJar>true</minimizeJar>-->
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>log4j2-test.xml</exclude>
                                        <exclude>dict.properties</exclude>

                                        <exclude>classes/excel-game-check.jar</exclude>
                                        <exclude>classes/mmo.core-1.jar</exclude>
                                        <exclude>classes/script.jar</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>com.game2sky.ExcelDataOpera</mainClass>
                                    <manifestEntries>
                                        <Premain-Class>com.game2sky.reload.ClassReloader</Premain-Class>
                                        <Can-Redefine-Classes>true</Can-Redefine-Classes>
                                        <Can-Retransform-Classes>true</Can-Retransform-Classes>
                                    </manifestEntries>
                                </transformer>
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>META-INF/spring.handlers</resource>
                                </transformer>
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>META-INF/spring.schemas</resource>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```

