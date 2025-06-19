1、Tomcat核心组件
- Web 容器：完成 Web 服务器的功能。
- Servlet 容器：名字为 catalina，用于处理 Servlet 代码。
- JSP 容器：用于将 JSP 动态网页翻译成 Servlet 代码。

2、Tomcat主要目录说明
- bin：存放启动和关闭 Tomcat 的脚本文件，如 catalina.sh、startup.sh、shutdown.sh 
- conf：存放 Tomcat 服务器的各种配置文件，如主配置文件 server.xml 和 应用默认的部署描述文件 web.xml 
- lib：存放 Tomcat 运行需要的库文件的 jar 包，一般不作任何改动
- logs：存放 Tomcat 执行时的日志
- temp：存放 Tomcat 运行时产生的文件
- webapps：存放 Tomcat 默认的 Web 应用项目资源的目录
- work：Tomcat 的工作目录，存放 Web 应用代码生成和编译文件
