# storm-spr-remote
storm整合springboot，远程模式（生产模式）。

二、storm-remote整合Springboot

在架构与启动顺序上，与storm-local的整合模式很不一样。

storm-remote构架，springboot运行在storm容器（平台）中，就像是springboot运行在tomcat中一样回事：



storm-remote模式中组件的启动顺序是，先启动storm，接着在storm中启动SpringBoot。这个与Local模式的启动顺序完全相反。



程序启动后（或者说是提交运程作业后），接着开始storm运算：



storm-remote模式整合SpringBoot示例，业务运算与storm-local一样，不一样的地方是，什么时候初始化Spring BeanFactory。

我们应该注意到，Storm的每个节点都是实现了Serializable接口，在生产环境中（也就是remote模式）每个节点可能会在不同的进程（不同的JVM）中被反序列化。Storm-remote整合SpringBoot时，是SpringBoot在Storm容器中被初始化并被管理，每个进程（这个进程就是Storm）有自己的Spring BeanFactory。

综上所述，我们已经知道，storm-remote中，第一个节点都是独立的了，不可能通过一个JVM（再次说明，一个JVM就是一个进程）共享数据，我们必须在每个节点初始化时，同时初始化一个Spring BeanFactory；因此，相对于storm-local的整合SpringBoot的方式，我们只要着重改造初始化SpringBoot的时机即可。

Storm-remot源码请参考：https://github.com/banat020/storm-spr-remote

2.1）改造SpringBoot的启动程序与Main程序。

2.1.1）SpringBoot的启动程序

package com.banling.stormspr;
 
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Import;
 
import java.util.concurrent.atomic.AtomicBoolean;
 
@SpringBootApplication
@Import(com.banling.stormspr.config.SpringContext.class)
public class StormSprApplication {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(StormSprApplication.class);
	
	private static AtomicBoolean flag=new AtomicBoolean(false);
 
	public synchronized  static void run(String... args) {
		if(flag.compareAndSet(false, true)) {
			LOGGER.info("SpringBoot is starting");
			
			SpringApplication springApplication = new SpringApplication(StormSprApplication.class);
	        //忽略Spring启动信息日志
	        springApplication.setLogStartupInfo(false);
	        springApplication.run(args);
 
			LOGGER.info("SpringBoot launched");
		}
	}
 
}
 
2.1.2）Main程序

package com.banling.stormspr;
 
import com.banling.stormspr.numcount.RemoteStorm;
 
public class StartStorm {
	public static void main(String[] args) {
		RemoteStorm.start();
	}
}
 
2.2）（mop.xml文件）改造资源引用与打包方式。

2.2.1）修改原

<parent>

        <groupId>org.springframework.boot</groupId>

        <artifactId>spring-boot-starter-parent</artifactId>

        <version>2.0.6.RELEASE</version>

        <relativePath/>

    </parent>

为

<dependencyManagement>

        <dependencies>

            <dependency>

                <groupId>org.springframework.boot</groupId>

                <artifactId>spring-boot-dependencies</artifactId>

                <version>2.0.6.RELEASE</version>

                <type>pom</type>

                <scope>import</scope>

            </dependency>

        </dependencies>

    </dependencyManagement>

2.2.2）在<plugins>节点中增加下列plugin

<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.6</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                            <classpathPrefix>lib/</classpathPrefix>
                            <mainClass>com.banling.stormspr.StartStorm</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
            <!-- 用Shade打包 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.1.1</version>
                <dependencies>
                    <dependency>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-maven-plugin</artifactId>
                        <version>2.0.6.RELEASE</version>
                    </dependency>
                </dependencies>
                <configuration>
                    <keepDependenciesWithProvidedScope>true</keepDependenciesWithProvidedScope>
                    <createDependencyReducedPom>true</createDependencyReducedPom>
                    
                    <filters>
                        <filter>
                            <artifact>*:*</artifact>
                            <excludes>
                                <exclude>META-INF/*.SF</exclude>
                                <exclude>META-INF/*.DSA</exclude>
                                <exclude>META-INF/*.RSA</exclude>
                            </excludes>
                        </filter>
                    </filters>
                     
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        
                        <configuration>
                            <transformers>
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>META-INF/spring.handlers</resource>
                                </transformer>
                                <transformer
                                        implementation="org.springframework.boot.maven.PropertiesMergingResourceTransformer">
                                    <resource>META-INF/spring.factories</resource>
                                </transformer>
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>META-INF/spring.schemas</resource>
                                </transformer>
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer" />
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>com.banling.stormspr.StartStorm</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                        
                    </execution>
                </executions>
            </plugin>
2.3)在所有运算结点中增加初始化SpringBoot

也就是，在Spout中的open方法中，在Bolt中的prepare方法中，初始化SpringBoot：

StormSprApplication.run();

2.4）测试效果

2.4.1）向生产环境提交作业：

$STORM_HOME/bin/storm jar storm-spr-remote-0.0.1-SNAPSHOT.jar com.banling.stormspr.StartStorm



在supervisor机器上，在目录$STORM_HOME/logs可查看执行日志。
