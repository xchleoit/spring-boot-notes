# 8. 部署

## 8.1 部署到应用服务器

首先，我们构建一个war包：

```
apply plugin: 'war'

war {
    baseName = 'readinglist'
    version = '0.0.1-SNAPSHOT'
}
```

这样就能打成war包了，但目前这个war包没什么用，因为既没有包含web.xml也没有一个servlet initializer来enable Spring MVC的DispatcherServlet。这时候就需要用到SpringBootServletInitializer了，它是Spring的WebApplicationInitializer的一个实现，除了能配置DispatcherServlet，还能找到Filter、Servlet、ServletContextInitializer类型的beans，把它们绑定到servlet容器：

```
package readinglist;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.context.web.SpringBootServletInitializer;

public class ReadingListServletInitializer extends SpringBootServletInitializer {
  @Override
  protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
    return builder.sources(Application.class);
  }
}
```

configure方法里source了一个配置类，这个配置类就是Spring Boot的主配置来和启动类，实际上更简洁的做法是让Application类继承SpringBootServletInitializer就好了。

然后构建：

```
gradle build
```

war包就会在build/libs下面了，把它部署到Tomcat服务器就好了，当然，你仍然可以用java -jar运行：

```
java -jar readinglist-0.0.1-SNAPSHOT.war
```

## 8.2 生产数据库

开发的时候我们可以用自动配置的内置H2数据库，不过生产环境你就得用MySQL这样的数据库了：

```
---
spring:
  profiles: prod
  datasource:
    url: jdbc:mysql://localhost:3306/readinglist?useUnicode=true&characterEncoding=utf8
    username: ${MYSQL_USER}
    password: ${MYSQL_PASSWORD}
```

用户名和密码可以设置在系统环境变量里，防止密码暴露在代码里，要启用这段配置，要设置spring.profiles.active为prod，比较方便的做法是设置环境变量：

```
export SPRING_PROFILES_ACTIVE=prod
```

如果你使用Hibernate（JPA）和内置的H2数据库，Spring Boot会默认配置Hibernate自动创建数据库表，更具体地，它会设置Hibernate的hibernate.hbm2ddl.auto为create-drop，表明当Hibernate的SessionFactory创建完之后创建表，关闭的时候删除表。不过如果不使用内置的H2数据库，Spring Boot什么也不会做，表不会被创建，因此查询数据库的时候会报错，所以你要显示设置spring.jpa.hibernate.ddl-auto为create，create-drop或update，尽管设置为update看起来不错，不过在生产中并不推荐这么做，更好的做法是使用数据库迁移工具，Spring Boot为两种流行的工具提供了自动配置：

* [Flyway](http://flywaydb.org)
* [Liquibase](http://www.liquibase.org)

**使用Flyway**

使用SQL编写脚本，脚本都有版本号，Flyway会按顺序自动执行这些脚本，并将执行状态记录到数据库中防止重复执行。把脚本放在classpath根路径的/db/migration目录下（src/main/resources/db/migration），脚本命名方式是大写的“V”打头+一个版本号+双下划线+描述脚本用途的名字.sql，如V1\_\_init.sql，当然你需要设置spring.jpa.hibernate.ddl-auto为none使得Hibernate不会自动创建表，最后引入flyway包即可：

```
compile("org.flywaydb:flyway-core")
```

当你启动应用的时候，flyway会根据schema_version表（会自动创建）中的脚本执行记录来执行db/migration中的脚本。

**使用Liquibase**

虽然Flyway用起来非常简单，不过用SQL脚本导致换一种数据库就无法工作了，Liquibase提供多种格式来编写脚本，包括XML、YAML和JSON，当然了，SQL也是支持的，引入Liquibase包：

```
compile("org.liquibase:liquibase-core")
```

默认情况下，liquibase的所有迁移脚本都写在/db/changelog目录下的db.changelog-master.yaml文件里，里面的每一块changeSet都有一个唯一id（不一定要是数字，任何文本都可以），脚本执行历史保存在databaseChangeLog表中，你可以修改默认的脚本位置：

```
liquibase:
  change-log: classpath:/db/changelog/db.changelog-master.xml
```

## 8.3 云端部署

### 8.3.1 部署到Cloud Foundry

Cloud Foundry是Pivotal公司的一个PaaS（Platform as a Service）平台，该公司是Spring生态系统的支持公司。

我们将要把应用部署到Pivotal Web Services（[PWS](http://run.pivotal.io)），是Pivotal的一个公共Cloud Foundry平台。上去注册，有60天的试用期，从 https://console.run.pivotal.io/tools 下载并安装cf命令行工具，首先得登录：

```
$ cf login -a https://api.run.pivotal.io
```

部署：

```
$ cf push sbia-readinglist -p build/libs/readinglist.war
```

第一个参数是Cloud Foundry上的应用名称，并且会是应用的子域名 http://sbia-readinglist.cfapps.io。所以得确保唯一，不过你可以使用--random-route来随机生成一个子域名：

```
$ cf push sbia-readinglist -p build/libs/readinglist.war --random-route
```

不仅仅是WAR包，你也可以提供可执行JAR包，甚至是通过Spring Boot CLI运行的未编译的Groovy脚本。

重启应用：

```
$ cf restart
```

Cloud Foundry提供一系列服务，可以从marketplace找到，比如MySQL等，PostgreSQL服务在上面叫做elephantsql，有不同的套餐可以选择，查看套餐：

```
$ cf marketplace -s elephantsql
```

我们选择免费的turtle套餐，使用如下命令创建一个数据库服务：

```
$ cf create-service elephantsql turtle readinglistdb
```

服务创建完毕后，需要绑定到我们的应用：

```
$ cf bind-service sbia-readinglist readinglistdb
```

绑定服务只不过是通过VCAP_SERVICES环境变量来提供服务的连接信息，它并不会改变应用本身。我们不必修改应用，而是使用restage命令：

```
$ cf restage sbia-readinglist
```

cf restage命令使得Cloud Foundry重新部署应用并重新读取VCAP_SERVICES值。

### 8.3.2 部署到Heroku

Heroku使用不同的方式部署应用，它为你的应用维护Git仓库，每次你push代码，它会构建和部署应用。

首先你需要初始化你的项目目录作为Git仓库：

```
$ git init
```

这样就能使得Heroku命令行工具为你的项目自动添加远程Heroku Git仓库，然后使用apps:create命令在Heroku中建立应用：

```
$ heroku apps:create sbia-readinglist
```

上面的命令指定了项目的名称是sbia-readinglist，这个名字会作为Git仓库的名字以及应用的子域名，所以要确保名字唯一，或者你可以留空，这样Heroku会帮你生成一个唯一的名字。

apps:create命令会创建一个远程Git仓库 https://git.heroku.com/sbia-readinglist.git ，并且会在你的本地项目的Git配置中添加一个名为“heroku”的远程引用（可以用git remote -v查看），这样就可以用git命令push到Heroku了。

Heroku需要你提供一个名为Procfile的文件来告诉它如何运行应用，对于我们的reading-list应用来说，我们需要告诉Heroku用java命令来运行WAR包，假设我们用Gradle来构建，则需要在Procfile有一行：

```
web: java -Dserver.port=$PORT -jar build/libs/readinglist.war
```

Maven则是：

```
web: java -Dserver.port=$PORT -jar target/readinglist.war
```

你需要设置server.port为Heroku分配的端口（由$PORT变量提供）。

对于Gradle应用来讲，当Heroku试着构建应用的时候，它会执行stage任务，所以你需要在build.gradle中加入：

```
task stage(dependsOn: ['build']) {
}
```

它仅仅依赖了build任务，以便用stage任务来触发build。

你可能需要指定构建应用时的Java版本，最方便的方法是在项目根目录下建一个system.properties文件来设置：

```
java.runtime.version=1.7
```

接下来就可以push到Heroku了：

```
$ git commit -am "Initial commit"
$ git push heroku master
```

代码提交之后，Heroku会用Maven或Gradle构建应用，然后用Procfile中的指令运行，没问题的话你就可以访问应用了，比如 https://sbia-readinglist.herokuapp.com

然后，我们可以创建并绑定到一个PostgreSQL服务：

```
$ heroku addons:add heroku-postgresql:hobby-dev
```

这里我们选用了heroku-postgresql服务的免费的hobby-dev套餐，现在PostgreSQL已经创建并绑定到我们的应用了，并且Heroku会自动重启应用确保绑定成功。但此刻我们看/health接口，发现还是在使用内置的H2数据库，那是因为H2的自动配置仍然生效，我们并没有告诉Spring Boot去使用PostgreSQL。

一种选择是设置spring.datasource.\*属性，我们可以使用如下命令查看数据库连接信息：

```
$ heroku addons:open waking-carefully-3728
```

其中waking-carefully-3728是我们的数据库实例的名字。这个命令会在浏览器中打开一个页面，里面有详细的数据库连接信息。

但是有一种更简单的方式，那就是使用Spring Cloud Connectors，它能和Cloud Foundry和Heroku一起工作，来发现绑定到应用的服务并自动配置应用使用这些服务。

我们只需要把它加入到依赖：

```
compile("org.springframework.boot:spring-boot-starter-cloud-connectors")
```

Spring Cloud Connectors只有在“cloud” profile激活的时候才会工作，在Heroku中激活“cloud” profile：

```
$ heroku config:set SPRING_PROFILES_ACTIVE="cloud"
```

然后就是push代码：

```
$ git commit -am "Add cloud connector"
$ git push heroku master
```

然后等应用启动完，再看/health接口，就会发现database已经变成了PostgreSQL。