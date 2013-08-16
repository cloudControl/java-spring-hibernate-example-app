#Deploying a Spring/MVC/Hibernate example application

In this tutorial we're going to show you how to create example Spring/MVC/Hibernate application using [Spring Roo](http://www.springsource.org/spring-roo), integrate it with [MySQLs Add-on](https://www.cloudcontrol.com/add-ons/mysqls), deploy it on [embedded Jetty server](http://jetty.codehaus.org/jetty/) and run on [cloudControl](https://www.cloudcontrol.com/). Check out the [buildpack-java](https://github.com/cloudControl/buildpack-java) for supported features.

##Prerequisites
 * [cloudControl user account](https://github.com/cloudControl/documentation/blob/master/Platform%20Documentation.md#user-accounts)
 * [cloudControl command line client](https://github.com/cloudControl/documentation/blob/master/Platform%20Documentation.md#command-line-client-web-console-and-api)
 * [git](https://help.github.com/articles/set-up-git)
 * [J2SE JDK/JVM](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
 * [Spring Roo](http://www.springsource.org/spring-roo)
 * [Maven3](http://maven.apache.org/download.html)

##Creating a sample application using Spring Roo

Download Spring Roo, extract it and use `bin/roo.sh` script to create `Petclinic` example application:

~~~bash
$ mkdir APP_NAME; cd APP_NAME;
$ roo.sh script --file clinic.roo
~~~

Generate data source configuration for [Hibernate](http://www.hibernate.org/) / MySQL

~~~bash
$ roo.sh persistence setup --provider HIBERNATE --database MYSQL
~~~

###Prepare to run on Jetty

For a fast and easy way to run your app, without having to install and administer a Jetty server, use the [Jetty Runner](http://wiki.eclipse.org/Jetty/Howto/Using_Jetty_Runner) - add to build plugins:

~~~xml
...
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>2.3</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>copy</goal>
            </goals>
            <configuration>
                <artifactItems>
                    <artifactItem>
                        <groupId>org.mortbay.jetty</groupId>
                        <artifactId>jetty-runner</artifactId>
                        <version>7.4.5.v20110725</version>
                        <destFileName>jetty-runner.jar</destFileName>
                    </artifactItem>
                </artifactItems>
            </configuration>
        </execution>
    </executions>
</plugin>
...
~~~

###Adjust data source configuration to MySQLs Add-on

Go to application context configuration file: `src/main/resources/META-INF/spring/applicationContext.xml` and modify datasource `username`, `password` and `url` properties to use credentials provided by MySQLs Add-on:

~~~xml
<property name="url" value="jdbc:mysql://${MYSQLS_HOSTNAME}:${MYSQLS_PORT}/${MYSQLS_DATABASE}"/>
<property name="username" value="${MYSQLS_USERNAME}"/>
<property name="password" value="${MYSQLS_PASSWORD}"/>
~~~

###Defining the process type
CloudControl uses a `Procfile` to know how to start your process. Create a file called Procfile:

~~~
web: java $JAVA_OPTS -jar target/dependency/jetty-runner.jar --port $PORT target/*.war
~~~

###Initializing git repository
Initialize a new git repository in the project directory and commit the files you have just created.

~~~bash
$ git init
$ git add pom.xml Procfile src
$ git commit -am "Initial commit"
~~~

##Pushing and deploying your app
Choose a unique name (from now on called APP_NAME) for your application and create it on the cloudControl platform:

~~~bash
$ cctrlapp APP_NAME create java
~~~

Push your code to the application's repository:

~~~bash
$ cctrlapp APP_NAME/default push

-----> Receiving push
-----> Installing OpenJDK 1.6...done
-----> Installing settings.xml... done
-----> executing /srv/tmp/buildpack-cache/.maven/bin/mvn -B -Duser.home=/srv/tmp/builddir -Dmaven.repo.local=/srv/tmp/buildpack-cache/.m2/repository -s /srv/tmp/buildpack-cache/.m2/settings.xml -DskipTests=true clean install
       [INFO] Scanning for projects...
       [INFO]
       [INFO] ------------------------------------------------------------------------
       [INFO] Building APP_NAME 1.0-SNAPSHOT
       [INFO] ------------------------------------------------------------------------
       ...
       [INFO] Packaging webapp
       [INFO] Assembling webapp [petclinic] in [/srv/tmp/builddir/target/petclinic-0.1.0.BUILD-SNAPSHOT]
       [INFO] Processing war project
       [INFO] Copying webapp resources [/srv/tmp/builddir/src/main/webapp]
       [INFO] Webapp assembled in [365 msecs]
       [INFO] Building war: /srv/tmp/builddir/target/petclinic-0.1.0.BUILD-SNAPSHOT.war
       [INFO] WEB-INF/web.xml already added, skipping
       [INFO]
       [INFO] --- maven-dependency-plugin:2.3:copy (default) @ petclinic ---
       ...
       [INFO] ------------------------------------------------------------------------
       [INFO] BUILD SUCCESS
       [INFO] ------------------------------------------------------------------------
       [INFO] Total time: 3:38.174s
       [INFO] Finished at: Thu Jan 24 10:16:16 UTC 2013
       [INFO] Final Memory: 20M/229M
       [INFO] ------------------------------------------------------------------------
-----> Building image
-----> Uploading image (84M)

To ssh://APP_NAME@cloudcontrolled.com/repository.git
 * [new branch]      master -> master
~~~

Create MySQLs Add-on:

~~~bash
$ cctrlapp APP_NAME/default addon.add mysqls.PLAN
~~~

Deploy your app (increase container size to meet high memory consumption by spring framework):

~~~bash
$ cctrlapp APP_NAME/default deploy --size=6
~~~

**Congratulations, you should now be able to reach your application at http://APP_NAME.cloudcontrolled.com.**
