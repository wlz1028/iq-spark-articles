#[Tutorial]How to setup your first Spark/Scala App with IntelliJ and Maven
Although there are many existing tutorials on Spark/Scala setup with IntelliJ, most people are using SBT instead of Maven. In this tutorial, I will create a simple Spark/Scala App using Maven and IntelliJ. I will also show you how to run and test your App inside IntelliJ. Personally, since this is the first post on my website, I also want to get familiar with Markdown and Wordpress.

## Prerequisites
* IntelliJ and Scala plugin installed (e.g.<https://www.jetbrains.com/idea/download/>)
* You are familiar with Java or Scala (e.g. <https://twitter.github.io/scala_school/>)
* You are familiar with Maven (e.g. <https://maven.apache.org/guides/getting-started/maven-in-five-minutes.html>)
* Source code: https://github.com/wlz1028/iq-first-spark

## Setup Maven settings.xml
First, make sure you have maven installed on you OS. Now, you will need to specify repositories to `${user.home}/.m2/settings.xml`. This allows maven to download artifacts which specified in your pom.xml (e.g. dependencies, plugins). In this tutorial, I will be using Hortonworks Hadoop Distribution (aka HDP). However, you can substitute Cloudera Hadoop Distribution (aka CDH) repo as you wish. 

Here is a sample `${user.home}/.m2/settings.xml` for HDP

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
    <mirrors>
        <mirror>
            <id>hw_central</id>
            <name>Hortonworks Mirror of Central</name>
            <url>http://repo.hortonworks.com/content/groups/public/</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
    </mirrors>
</settings>
```

## Create new Maven project
File -> new -> project  

> Spark 1.6.0 requires Java 7  
> Make sure you have Java 7 JDK installed on you OS  
> Add new JDK version to Project SDK from below window  
	
![newmaven](https://cloud.githubusercontent.com/assets/5523501/17954024/17a1ff0e-6a46-11e6-91d7-042f05913dea.png)

Next

![snip20160824_4](https://cloud.githubusercontent.com/assets/5523501/17954860/db83559e-6a4b-11e6-8bca-3c5b1b791034.png)

Next

![snip20160824_5](https://cloud.githubusercontent.com/assets/5523501/17954876/ff60544e-6a4b-11e6-9039-b30c5ae70430.png)

Enable Auto-Import

![snip20160824_6](https://cloud.githubusercontent.com/assets/5523501/17955206/6810c742-6a4e-11e6-90ce-7e7729eec256.png)

## pom.xml
I will be using HDP's Spark 1.6.0 version. Here is the pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>ca.infoq.spark.tut</groupId>
    <artifactId>first-spark-app</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <spark.hdp.version>1.6.0.2.4.1.1-3</spark.hdp.version>
        <scala.version>2.10.5</scala.version>
        <scala.binary.version>2.10</scala.binary.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_${scala.binary.version}</artifactId>
            <version>${spark.hdp.version}</version>
        </dependency>
    </dependencies>
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>net.alchim31.maven</groupId>
                    <artifactId>scala-maven-plugin</artifactId>
                    <version>3.2.1</version>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>2.0.2</version>
                </plugin>
            </plugins>
        </pluginManagement>
        <plugins>
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>scala-compile-first</id>
                        <phase>process-resources</phase>
                        <goals>
                            <goal>add-source</goal>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>scala-test-compile</id>
                        <phase>process-test-resources</phase>
                        <goals>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <executions>
                    <execution>
                        <phase>compile</phase>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
Your should be able to see dependencies from navigation bar once IntelliJ resolved your pom.xml.

![image](https://cloud.githubusercontent.com/assets/5523501/17955712/208ee7ec-6a52-11e6-8fd5-1f227a8b6cea.png)

##Create Spark/Scala App
Create src/main/scala directory

![snip20160824_7](https://cloud.githubusercontent.com/assets/5523501/17955533/fc496d54-6a50-11e6-83be-e006887514dd.png)

Mark scala/ as `Source Root`

![snip20160824_8](https://cloud.githubusercontent.com/assets/5523501/17955567/38a5da1c-6a51-11e6-981e-ea3e7758c947.png)

Create `ca.infoq.spark.tut` package

![image](https://cloud.githubusercontent.com/assets/5523501/17955609/7d0f2a64-6a51-11e6-8d26-12b4de6b9849.png)

Create a new Spark/Scala app object

![image](https://cloud.githubusercontent.com/assets/5523501/17955647/b51d67ae-6a51-11e6-9476-4a7ad8fe3629.png)

Write you first Spark/Scala app

```scala
package ca.infoq.spark.tut

import org.apache.spark.SparkContext
import org.apache.spark.SparkConf

/**
  * Created by infoq.ca(Edward,Wang) on 2016-08-24.
  */
object SimpleApp extends App {
  // create Spark context with Spark configuration
  val sc = new SparkContext(new SparkConf().setAppName("IQ Simple Spark App").setMaster("local[*]"))
  val rdd = sc.parallelize(Seq(1,2,3,4,5))
  println(rdd.count())
}
```

##Run your app
Right click your Scala file and hit `Run`

![image](https://cloud.githubusercontent.com/assets/5523501/17955935/cd61abb6-6a53-11e6-82c2-ec68105a563d.png)

You will find result in IntelliJ Console

![image](https://cloud.githubusercontent.com/assets/5523501/17955955/f30b0f9c-6a53-11e6-91bb-ae7ffef43341.png)
