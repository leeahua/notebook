## springboot + jacoco + sonar + sonarqube+maven+idea 集成

## 第一步  设置sonarqube 的maven环境

1.1 idea里面确认你的maven的setting文件位置

 File-setting-maven

![image-20210804114758564](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804114758564.png) 

1.2 在你的maven setting.xml配置文件中添加sonarqube 服务的配置信息

```
<settings>
    <profiles>
       <!-- sonarqube 配置开始 -->
        <profile>
               <id>sonar</id>
               <activation>
                       <activeByDefault>true</activeByDefault>
               </activation>
               <properties>
                        <!-- sonarqube服务地址 目前公司使用的是七号服务器-->
                       <sonar.host.url>http://10.1.120.7:9001</sonar.host.url>
                       <!-- sonarqube的用户名 对应的是内网系统的用户名 demo: p_landyhli -->
                       <sonar.login>用户名</sonar.login>
                       <!-- sonarqube的用户名对应的密码-->
                       <sonar.password>填自己的密码</sonar.password>
               </properties>
            </profile>
         <!-- sonarqube 配置结束 -->
   </profiles>
 </settings>
```

1.3  在项目中添加jacoco 和相关组件,在pom中添加如下maven插件

```
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.22.2</version>
            </plugin>
            <plugin>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>2.22.2</version>
            </plugin>
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.5</version>
                <executions>
                    <execution>
                        <id>default-prepare-agent</id>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>default-report</id>
                        <goals>
                            <goal>report</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

完整的pom.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.autopai</groupId>
    <artifactId>sonar-demo-clean</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>sonar-demo-clean</name>
    <description>sonar-demo-clean</description>

    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>2.3.7.RELEASE</spring-boot.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>2.3.7.RELEASE</version>
                <configuration>
                    <mainClass>com.autopai.sonar.SonarDemoCleanApplication</mainClass>
                </configuration>
                <executions>
                    <execution>
                        <id>repackage</id>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.22.2</version>
                <configuration>
                    <excludes>
                        <exclude>**/*Entity.java</exclude>
                    </excludes>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>2.22.2</version>
            </plugin>
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.5</version>
                <executions>
                    <execution>
                        <id>default-prepare-agent</id>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>default-report</id>
                        <goals>
                            <goal>report</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>

```

1.3 项目勾选sonar的profile

![image-20210804115558879](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804115558879.png)

## 第二步 编写测试用例

2.1 测试实体类HelloDTO.java

```java
package com.autopai.sonar.dto;

/**
 * 测试实体类
 * @author lyh
 * @date 2021/8/3 17:42
 */
public class HelloDTO {

    private String name;

    private Integer age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "HelloDTO{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```

2. 2 测试实体类HelloEntity.java

```java
package com.autopai.sonar.dto;

/**
 * 测试实体类
 * @author lyh
 * @date 2021/8/3 17:42
 */
public class HelloDTO {

    private String name;

    private Integer age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "HelloDTO{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```



2.3  业务调用类Hello.java

```java
package com.autopai.sonar.busi;

import com.autopai.sonar.dto.HelloDTO;
import com.autopai.sonar.entity.HelloEntity;

/**
 * hello 业务处理类
 * @author lyh
 * @date 2021/8/3 17:44
 */
public class Hello {

    private static final String HELLO_TAG = "jack";

    public String sayHello(HelloDTO helloDTO){
        if(helloDTO == null){
            return null;
        }
        if(HELLO_TAG.equals(helloDTO.getName())){
            return helloDTO.getAge()+"";
        }
        return helloDTO.getName();
    }

    public String sayEntityHello(HelloEntity helloDTO){
        if(helloDTO == null){
            return null;
        }
        if(HELLO_TAG.equals(helloDTO.getName())){
            return helloDTO.getAge()+"";
        }
        return helloDTO.getName();
    }
}

```

2.4  添加单元测试 idea中打开Hello类，单击类名右键->Generate->test  勾选需要生成测试用例的方法

![image-20210804120358609](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804120358609.png)

2.5 编写单元测试

```java
package com.autopai.sonar.busi;

import com.autopai.sonar.dto.HelloDTO;
import com.autopai.sonar.entity.HelloEntity;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class HelloTest {

    @Test
    void sayHello() {
        Hello hello = new Hello();
        HelloDTO helloDTO = new HelloDTO();
        helloDTO.setAge(12);
        helloDTO.setName("test");
        assertEquals(hello.sayHello(helloDTO), helloDTO.getName());
        assertNull(hello.sayHello(null));
        HelloDTO helloDTOTag = new HelloDTO();
        helloDTOTag.setAge(14);
        helloDTOTag.setName("jack");
        assertEquals(hello.sayHello(helloDTOTag), helloDTOTag.getAge()+"");
    }


    @Test
    void sayEntityHello(){
        Hello hello = new Hello();
        HelloEntity helloEntity = new HelloEntity();
        helloEntity.setAge(12);
        helloEntity.setName("test");
        assertEquals(hello.sayEntityHello(helloEntity), helloEntity.getName());
        assertNull(hello.sayHello(null));
        HelloEntity helloEntityTag = new HelloEntity();
        helloEntityTag.setAge(14);
        helloEntityTag.setName("jack");
        assertEquals(hello.sayEntityHello(helloEntityTag), helloEntityTag.getAge()+"");
    }
}
```

2.6 项目目录如下

![image-20210804120506480](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804120506480.png)

2.7 执行单元测试：

```
mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent install
```

2.8 单元测试执行完毕，生成本地单元测试报告位置

![image-20210804120756382](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804120756382.png)

2.9 本地单元测试报告信息

![image-20210804120830876](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804120830876.png)



## 第三步 将测试报告上传到sonarqube

3.1 在sonarqube上添加项目

![image-20210804121246646](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804121246646.png)

注意：项目标识要记住，后面指令要用



3.2 创建访问令牌

![image-20210804121506423](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804121506423.png)

3.2 分析项目

![image-20210804121530072](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804121530072.png)



3.3 你可以用这个指令推送报告，也可以直接推报告上去，因为我们在maven的setting配置里面已经配置了sonarqube的用户名和密码 

推送报告：

```
#按照setting里面配置走 其实用这个就可以了
mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent install sonar:sonar -Dsonar.projectKey=soanr-demo-2

#想指定自己的sonarqube地址 可以加-Dsoanr.host.url  也可以指定用户名和密码 
/**
mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent install -Dmaven.test.failure.ignore=true sonar:sonar -Dsonar.projectKey=autopai-data-#collection_autopai-tlc_AXsGARWiO_brSYGkwUsP -Dsonar.host.url=http://localhost:9000 -Dsonar.login=admin -Dsonar.password=123456
**/
mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent install sonar:sonar -Dsonar.projectKey=soanr-demo-2 -Dsonar.host.url=http://10.1.120.7:9001
```



报告地址：

![image-20210804122207386](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804122207386.png)

报告内容：

![image-20210804122237268](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210804122237268.png)



好了，下面就没有了。





## 多模块项目集成

 ###  1. 在最顶级项目pom.xml 添加 如下配置

```
<profiles>
		<profile>
			<id>coverage</id>
			<activation>
				<activeByDefault>true</activeByDefault>
			</activation>
			<build>
				<plugins>
					<plugin>
						<groupId>org.apache.maven.plugins</groupId>
						<artifactId>maven-surefire-plugin</artifactId>
						<configuration>
							<skipTests>false</skipTests>
						</configuration>
					</plugin>
					<plugin>
						<groupId>org.jacoco</groupId>
						<artifactId>jacoco-maven-plugin</artifactId>
						<version>0.8.5</version>
						<executions>
							<execution>
								<id>default-prepare-agent</id>
								<goals>
									<goal>prepare-agent</goal>
								</goals>
							</execution>
							<execution>
								<id>default-report</id>
								<goals>
									<goal>report</goal>
								</goals>
							</execution>
						</executions>
					</plugin>
				</plugins>
			</build>
		</profile>
	</profiles>
```

### 2. 勾选profile

![image-20210805115441434](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210805115441434.png)



### 3. 在你要执行命令的项目下执行如下命令：

```
mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent install -Dmaven.test.failure.ignore=true sonar:sonar
```

### 4. 查看sonarqube的报告

![image-20210805115609048](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210805115609048.png)

![image-20210805115623338](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20210805115623338.png)

然后 就没有了。