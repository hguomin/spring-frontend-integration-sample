# Spring Boot前后端分离Web应用集成打包

目前在Web应用开发中，为了降低项目或模块间的耦合性，以及应用前端工程化技术，通常会采用“前后端分离”的模式进行项目的开发。

对于基于Spring Boot的Web应用开发，如何将前端项目输出的资源与后端项目一起进行打包以方便部署呢？前端项目的输出包括html/js/css等资源，一般的做法是将前端项目构建输出的资源采用手工或CI的方式复制到后端项目的前端资源目录下（默认位于src\main\resources\static），这种方式无疑显得效率低下。

实际上，通过设置Maven build的相关配置，可以实现对前后端项目输出的自动打包整合，主要有两种方式：
* 利用`maven-resources-plugin`直接复制前端项目生成的资源文件到后端项目中
* 将前端项目作为后端项目的依赖，后端项目构建时利用`spring-boot-maven-plugin`的`repackage`将其所依赖的前端项目的输出文件复制过来

本文将分享如何设置Maven的pom.xml内容以实现上述目标，不涉及前后端项目的具体内容。


## 项目目录结构：  
基于Maven的父子项目结构，我们可以使用以下目录结构创建前后端工程：  
```
\
  |-application :后端项目    
    |-src    
    |-pom.xml  :后端项目的pom.xml
  |-portal :前端项目    
    |-src    
    |-pom.xml  :前端项目的pom.xml   
  |-pom.xml :父项目的pom.xml   
```

其中application是基于Spring boot的后端项目，可使用Spring Initializr进行创建。portal目录下是前端项目，可使用webpack+npm等工具进行创建开发。

## 父项目的pom.xml配置
根据Maven的父子项目结构的配置要求，在`modules`中包含前、后端的项目，具体如下所示：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
    <parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.6.6</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
    
    <groupId>io.devplus</groupId>
    <artifactId>spring-frontend-integration-sample</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-frontend-integration-sample</name>
	<description>Demo project for Spring Boot</description>
    
    <packaging>pom</packaging>

    <!-- 包含前后端子项目 -->
    <modules> 
        <module>portal</module>
        <module>application</module>
    </modules>
</project>
```

## 实现方式一：文件复制，即利用`maven-resources-plugin`直接复制前端项目生成的资源文件

### 前端项目的pom.xml配置
由于前端项目一般采用Webpack+npm等前端工程化工具进行构建开发，因此在这种文件拷贝的方式下pom.xml一般是不需要的，在此只是为了体现其是整个Maven项目的子项目而已，所以其内容比较简单。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.devplus</groupId>
        <artifactId>spring-frontend-integration-sample</artifactId>
        <version>0.0.1-SNAPSHOT</version>
	</parent>

    <groupId>io.devplus</groupId>
    <artifactId>portal</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>Frontend project</name>
	<description>Frontend project for Spring Boot</description>
    
    <packaging>pom</packaging>

</project>
```

### 后端项目的pom.xml配置
后端项目的pom.xml配置除了需要配置其本身的编译构建之外，还需要配置针对前端项目的编译、构建及资源的复制，从而实现前后端资源的整体打包过程。

具体而言，通过使用`frontend-maven-plugin`插件在前端项目目录中安装`node`和`npm`，并调用前端项目的构建命令进行构建以生成前端资源文件，随后利用`maven-resources-plugin`插件将前端资源文件复制到后端项目的前端资源目录下。

后端项目的`pom.xml`的参考配置如下所示，可根据项目实际进行配置。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.devplus</groupId>
        <artifactId>spring-frontend-integration-sample</artifactId>
        <version>0.0.1-SNAPSHOT</version>
	</parent>
	<groupId>io.devplus</groupId>
	<artifactId>application</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>Web application project</name>
	<description>Backend project for Spring Boot</description>

	<packaging>jar</packaging>

	<properties>
		<java.version>8</java.version>
		<node.version>v16.13.1</node.version>
        <npm.version>8.6.0</npm.version>
		<frontend.maven.plugin.version>1.12.1</frontend.maven.plugin.version>
		<portalDir>${project.parent.basedir}/portal</portalDir>
	</properties>
	
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
	<profiles>
		<!-- 2.1 调用前端项目中package.json定义的构建命令，Maven执行时可通过-P指定需要执行的profile: 例如：mvn clean install -P npm-build-dev -->
		<profile>
			<id>npm build production</id>
			<activation>
				<!-- 默认执行这个Profile -->
				<activeByDefault>true</activeByDefault>
				<property>
					<name>npm-build-prod</name>
				</property>
			</activation>
			<build>
				<plugins>
					<plugin>
						<groupId>com.github.eirslett</groupId>
						<artifactId>frontend-maven-plugin</artifactId>
						<version>${frontend.maven.plugin.version}</version>
		
						<configuration>
							<!-- 前端项目的目录 -->
							<workingDirectory>${portalDir}</workingDirectory>
						</configuration>
						
						<executions>
							<!-- 调用前端项目构建命令进行构建 -->
							<execution>
								<id>npm run build</id>
								<goals>
									<goal>npm</goal>
								</goals>
								<phase>compile</phase>
								<configuration>
									<!-- 前端项目的构建命令，需要根据package.json中定义的命令进行设置-->
									<arguments>run build-prod</arguments>
								</configuration>
							</execution>
						</executions>
					</plugin>
				</plugins>
			</build>
		</profile>
		<profile>
			<id>npm build development</id>
			<activation>
				<activeByDefault>false</activeByDefault>
				<property>
					<name>npm-build-dev</name>
				</property>
			</activation>
			<build>
				<plugins>
					<plugin>
						<groupId>com.github.eirslett</groupId>
						<artifactId>frontend-maven-plugin</artifactId>
						<version>${frontend.maven.plugin.version}</version>
	
						<configuration>
							<!-- 前端项目的目录 -->
							<workingDirectory>${portalDir}</workingDirectory>
						</configuration>
						
						<executions>
							<!-- 调用前端项目构建命令进行构建 -->
							<execution>
								<id>npm run build-dev</id>
								<goals>
									<goal>npm</goal>
								</goals>
								<phase>compile</phase>
								<configuration>
									<!-- 前端项目的构建命令，需要根据package.json中定义的命令进行设置-->
									<arguments>run build-dev</arguments>
								</configuration>
							</execution>
						</executions>
					</plugin>
				</plugins>
			</build>
		</profile>
	</profiles>

	<build>
		<plugins>
			<!-- 1. 利用maven-clean-plugin插件在编译前删除前端拷贝过来的文件 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-clean-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <filesets>
                        <fileset>
                            <!-- 后端项目中前端资源文件目录 -->
                            <directory>${baseDir}/src/main/resources/static</directory>
                        </fileset>
                    </filesets>
                </configuration>
			</plugin>
			<!-- 可配置源码和目标的java版本 -->
			<!--plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
			</plugin-->

			<!-- 2. 前端项目构建的配置：安装node和npm、执行npm install下载依赖、以及package.json定义的构建命令 -->
			<plugin>
                <groupId>com.github.eirslett</groupId>
                <artifactId>frontend-maven-plugin</artifactId>
                <!-- Use the latest released version:
                https://repo1.maven.org/maven2/com/github/eirslett/frontend-maven-plugin/ -->
                <version>${frontend.maven.plugin.version}</version>

				<configuration>
					<!-- 前端项目的目录 -->
                    <workingDirectory>${project.parent.basedir}/portal</workingDirectory>

					<nodeVersion>${node.version}</nodeVersion>
                    <npmVersion>${npm.version}</npmVersion>
					
					<!-- node和npm的安装目录 -->
					<installDirectory>${project.parent.basedir}/portal/target/dev-tools</installDirectory> 
                    <!--nodeDownloadRoot>https://npm.taobao.org/mirrors/node/</nodeDownloadRoot>
                    <npmDownloadRoot>https://registry.npm.taobao.org/npm/-/</npmDownloadRoot-->
                    
                </configuration>

                <executions>
					<!-- 本地安装node和npm -->
                    <execution>
                        <id>install node and npm</id>
                        <goals>
                            <goal>install-node-and-npm</goal>
                        </goals>
                        <phase>generate-resources</phase>
                    </execution>
					<!-- 下载npm依赖 -->
                    <execution>
                        <id>npm install</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                        <phase>generate-resources</phase>
                        <configuration>
                            <arguments>install</arguments>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

			<!-- 3. 资源插件从前端项目复制打包好的前端资源文件到后端项目中 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-resources-plugin</artifactId>
                <version>3.2.0</version>
                <executions>
                    <execution>
                        <id>copy frontend resource files</id>
                        <phase>generate-resources</phase>
                        <goals>
                            <goal>copy-resources</goal>
                        </goals>
                        <configuration>
                            <!-- 复制前端资源文件到此文件夹中，可根据项目需要进行自定义设置-->
                            <outputDirectory>${basedir}/src/main/resources/static</outputDirectory>
                            <overwrite>true</overwrite>
                            <resources>
                                <resource>
                                    <!-- 根据项目实际设置前端项目所生成的资源文件目录，将会从此目录中拷贝相应的文件或目录 -->
                                    <directory>${portalDir}/dist</directory>
                                    <includes>
                                        <!-- 根据项目实际指定需要拷贝的前端资源目录或文件 -->
										<include>*.*</include>
                                        <!--include>favicon.ico</include>
                                        <include>index.html</include>
										<include>js/</include-->
                                    </includes>
                                </resource>
                            </resources>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
		</plugins>
	</build>
</project>
```
## 整体项目打包部署
通过上述各pom.xml文件的配置，已经整合了前后端项目的Maven自动打包过程，在构建父项目或者后端项目时，即可将前端项目的资源文件整体打包到后端项目中，从而实现一键打包过程。

## 实现方式二：依赖打包，即在后端项目中利用依赖repackaging来打包前端资源
### 前端项目的pom.xml配置
与前一种实现方式不同，这种实现方式的原理是在前端项目构建完成后，先把生成的前端资源文件通过Maven打包到前端项目Jar包中，然后在后端项目中作为其依赖包引入并通过`spring-boot-maven-plugin`插件的repackaging过程将前端资源最终打包进后端项目中，因此需要在前端项目的build阶段指定打包哪些资源，这可以通过`pom.xml`的`build.resources`中指定。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.devplus</groupId>
        <artifactId>spring-frontend-integration-sample</artifactId>
        <version>0.0.1-SNAPSHOT</version>
	</parent>

    <groupId>io.devplus</groupId>
    <artifactId>portal</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>Frontend project</name>
	<description>Frontend project for Spring Boot</description>
    <properties>
        <node.version>v16.13.1</node.version>
        <npm.version>8.6.0</npm.version>
        <frontend.maven.plugin.version>1.12.1</frontend.maven.plugin.version>
    </properties>

    <profiles>
		<!-- 1.1 调用前端项目中package.json定义的构建命令，Maven执行时可通过-P指定需要执行的profile: 例如：mvn clean install -P npm-build-dev -->
		<profile>
			<id>npm build production</id>
			<activation>
				<!-- 默认执行这个Profile -->
				<activeByDefault>true</activeByDefault>
				<property>
					<name>npm-build-prod</name>
				</property>
			</activation>
			<build>
				<plugins>
					<plugin>
						<groupId>com.github.eirslett</groupId>
						<artifactId>frontend-maven-plugin</artifactId>
						<version>${frontend.maven.plugin.version}</version>
		
						<configuration>
							<!-- 前端项目的目录 -->
							<workingDirectory>${portalDir}</workingDirectory>
						</configuration>
						
						<executions>
							<!-- 调用前端项目构建命令进行构建 -->
							<execution>
								<id>npm run build</id>
								<goals>
									<goal>npm</goal>
								</goals>
								<phase>compile</phase>
								<configuration>
									<!-- 前端项目的构建命令，需要根据package.json中定义的命令进行设置-->
									<arguments>run build-prod</arguments>
								</configuration>
							</execution>
						</executions>
					</plugin>
				</plugins>
			</build>
		</profile>
		<profile>
			<id>npm build development</id>
			<activation>
				<activeByDefault>false</activeByDefault>
				<property>
					<name>npm-build-dev</name>
				</property>
			</activation>
			<build>
				<plugins>
					<plugin>
						<groupId>com.github.eirslett</groupId>
						<artifactId>frontend-maven-plugin</artifactId>
						<version>${frontend.maven.plugin.version}</version>
	
						<configuration>
							<!-- 前端项目的目录 -->
							<workingDirectory>${portalDir}</workingDirectory>
						</configuration>
						
						<executions>
							<!-- 调用前端项目构建命令进行构建 -->
							<execution>
								<id>npm run build-dev</id>
								<goals>
									<goal>npm</goal>
								</goals>
								<phase>compile</phase>
								<configuration>
									<!-- 前端项目的构建命令，需要根据package.json中定义的命令进行设置-->
									<arguments>run build-dev</arguments>
								</configuration>
							</execution>
						</executions>
					</plugin>
				</plugins>
			</build>
		</profile>
	</profiles>

	<build>
        <!-- 0. 指定打包的前端资源文件目录，即前端项目的输出-->
        <resources>
            <resource>
                <directory>${basedir}/dist</directory>
            </resource>
        </resources>
		<plugins>
			<!-- 利用maven-clean-plugin插件在清理文件 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-clean-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <filesets>
                        <fileset>
                            <directory>${basedir}/node_modules</directory>
                        </fileset>
                    </filesets>
                </configuration>
			</plugin>
			<!-- 1. 前端项目构建的配置：安装node和npm、执行npm install下载依赖、以及package.json定义的构建命令 -->
			<plugin>
                <groupId>com.github.eirslett</groupId>
                <artifactId>frontend-maven-plugin</artifactId>
                <!-- Use the latest released version:
                https://repo1.maven.org/maven2/com/github/eirslett/frontend-maven-plugin/ -->
                <version>${frontend.maven.plugin.version}</version>

				<configuration>
					<!-- 前端项目的目录 -->
                    <workingDirectory>${basedir</workingDirectory>

					<nodeVersion>${node.version}</nodeVersion>
                    <npmVersion>${npm.version}</npmVersion>
					<!-- node和npm的安装目录 -->
					<installDirectory>${basedir}/target/dev-tools</installDirectory> 
                </configuration>

                <executions>
					<!-- 本地安装node和npm -->
                    <execution>
                        <id>install node and npm</id>
                        <goals>
                            <goal>install-node-and-npm</goal>
                        </goals>
                        <phase>generate-resources</phase>
                    </execution>
					<!-- 下载npm依赖 -->
                    <execution>
                        <id>npm install</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                        <phase>generate-resources</phase>
                        <configuration>
                            <arguments>install</arguments>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
		</plugins>
	</build>
</project>
```
### 后端项目的pom.xml配置
在这种实现方式下，后端项目只需要引入前端项目作为依赖包，并设置`spring-boot-maven-plugin`插件的repackaging将依赖一并打包即可。通常前端资源会被打包在`target/classes/static`目录下。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.devplus</groupId>
        <artifactId>spring-frontend-integration-sample</artifactId>
        <version>0.0.1-SNAPSHOT</version>
	</parent>
	<groupId>io.devplus</groupId>
	<artifactId>application</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>Web application project</name>
	<description>Backend project for Spring Boot</description>

	<packaging>jar</packaging>

	<properties>
		<java.version>8</java.version>
	</properties>
	
	<dependencies>
		<!-- 引入前端项目作为依赖包之一 -->
		<dependency>
			<groupId>io.devplus</groupId>
			<artifactId>portal</artifactId>
			<version>0.0.1-SNAPSHOT</version>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
	
	<build>
		<plugins>
			<!-- 将项目依赖的包都打包到所生成的jar包中 -->
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<executions>
					<execution>
						<goals>
							<goal>repackage</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
</project>

```
### 整体项目打包部署
在父项目中，执行以下命令即可将前后端项目整体打包，打包后的前端资源通常位于jar包中的`BOOT-INF/classes/static`目录中
```bash
mvn install -DskipTests
```

## 总结
通过以上两种不同的方式，均可以将前端项目生成的资源文件与后端项目进行整体打包部署，在实际项目开发过程中可以根据项目需要灵活选择。