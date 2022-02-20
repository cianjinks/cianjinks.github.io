---
title: Tutorial - Using Maven and IntelliJ for OpenGL Projects
date: 2020-05-28 12:00:00 +0000
categories: [Old]
tags: [opengl, lwjgl, maven, tutorial]
img_path: /assets/img/post/MavenTutorial/
---
This tutorial will quickly cover how one can use Maven along with IntelliJ to get started with graphics programming using OpenGL. I originally wrote this so people could use it to directly follow along with my [Visualising the Mandelbrot Set Tutorial Series](https://cianjinks.github.io/posts/opengl-tutorial-visualizing-the-mandelbrot-set-fractal-part-1-of-2/) but I believe it is a great way to setup any OpenGL project in java. I use IntelliJ as my main IDE for programming Java and so that is what this tutorial will use, however others like eclipse will have an almost identical process with the only difference being the importing of the project.

**Prerequisite:** Install [Maven](https://maven.apache.org/download.cgi) for your specific OS before following this tutorial.

## The pom.xml

The most important part of creating a Maven project is the pom.xml configuration file. In here we will define all kinds of things related to our project such as the libraries it depends on, various settings, information about the author and much more. This file and the source code for our project is _all_ that we will need to compile our java application in the future making using Maven incredibly simple.

In terms of our pom.xml file for OpenGL programming we are going to obtain it from the [LWJGL customize page](https://www.lwjgl.org/customize). As I mentioned in the Mandelbrot Tutorial, LWJGL is a great java binding for many OpenGL functions. To get a fully configured pom.xml file with all required LWJGL dependencies we can simply choose the "Maven" option under "Mode" and it will generate one for us which we can download from the bottom of the page. For the Mandelbrot Set tutorial series I chose the Minimal OpenGL preset, the JOML math library addon and version 3.2.3 (the latest at the time). With those options selected our pom.xml file should look something like the following:

```xml
<properties>
  <lwjgl.version>3.2.3</lwjgl.version>
  <joml.version>1.9.24</joml.version>
  <lwjgl.natives>natives-windows</lwjgl.natives>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.lwjgl</groupId>
      <artifactId>lwjgl-bom</artifactId>
      <version>${lwjgl.version}</version>
      <scope>import</scope>
      <type>pom</type>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl</artifactId>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-assimp</artifactId>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-glfw</artifactId>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-openal</artifactId>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-opengl</artifactId>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-stb</artifactId>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl</artifactId>
    <classifier>${lwjgl.natives}</classifier>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-assimp</artifactId>
    <classifier>${lwjgl.natives}</classifier>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-glfw</artifactId>
    <classifier>${lwjgl.natives}</classifier>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-openal</artifactId>
    <classifier>${lwjgl.natives}</classifier>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-opengl</artifactId>
    <classifier>${lwjgl.natives}</classifier>
  </dependency>
  <dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-stb</artifactId>
    <classifier>${lwjgl.natives}</classifier>
  </dependency>
  <dependency>
    <groupId>org.joml</groupId>
    <artifactId>joml</artifactId>
    <version>${joml.version}</version>
  </dependency>
</dependencies>

```

This pom.xml is actually incomplete. LWJGL has simply provided us with the various dependency declarations but we still are missing any actual information about our project. If we head on over to Maven's [POM reference](https://maven.apache.org/pom.html) page we can take a look at the Quick Overview section. Here you will see some examples of simple configuration options. Before any of that, however, the most important thing to take note of is the `<project></project>` block which must encapsulate any pom.xml file. We will start by adding that to our file with their settings:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

	<properties>
		<lwjgl.version>3.2.3</lwjgl.version>
		<joml.version>1.9.24</joml.version>
		<lwjgl.natives>natives-windows</lwjgl.natives>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.lwjgl</groupId>
				<artifactId>lwjgl-bom</artifactId>
				<version>${lwjgl.version}</version>
				<scope>import</scope>
				<type>pom</type>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<dependencies>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-assimp</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-glfw</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-openal</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-opengl</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-stb</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-assimp</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-glfw</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-openal</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-opengl</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-stb</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.joml</groupId>
			<artifactId>joml</artifactId>
			<version>${joml.version}</version>
		</dependency>
	</dependencies>
</project>
```

With that in place all we need to do is specify some information about our project. The most important ones are `<groupId>`, `<artifactId>` and `<version>`. The `<groupId>` is simply going to be a unique identifier for projects by you. Generally people set this to a domain that they own or something else unique to them. `<artifactId>` has the same concept except it's unique to the specific project the pom.xml file is used for. Finally, `<version>` is the just version of your project. Some other options you may also want to configure might include `<licenses>`, `<developers>` or `<url>`. If you want to read up on what these are used for, once again you can check out the [POM reference](https://maven.apache.org/pom.html) page's Quick Overview section. With these extra options in our pom.xml it will now look something like this:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

	<groupId>io.github.cianjinks</groupId>
    <artifactId>MandelbrotViewerTutorial</artifactId>
    <version>1.1</version>

	<developers>
        <developer>
            <name>Cian Jinks</name>
            <email>cjinks99@gmail.com</email>
        </developer>
    </developers>

	<properties>
		<lwjgl.version>3.2.3</lwjgl.version>
		<joml.version>1.9.24</joml.version>
		<lwjgl.natives>natives-windows</lwjgl.natives>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.lwjgl</groupId>
				<artifactId>lwjgl-bom</artifactId>
				<version>${lwjgl.version}</version>
				<scope>import</scope>
				<type>pom</type>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<dependencies>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-assimp</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-glfw</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-openal</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-opengl</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-stb</artifactId>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-assimp</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-glfw</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-openal</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-opengl</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.lwjgl</groupId>
			<artifactId>lwjgl-stb</artifactId>
			<classifier>${lwjgl.natives}</classifier>
		</dependency>
		<dependency>
			<groupId>org.joml</groupId>
			<artifactId>joml</artifactId>
			<version>${joml.version}</version>
		</dependency>
	</dependencies>
</project>
```

## Maven with IntelliJ

With our pom.xml all set up it finally comes time to start a new project using IntelliJ. To do so we simply create a new project as normal, except this time we choose the Maven option:

![OpenGL_MavenTutorial_1.PNG](OpenGL_MavenTutorial_1.PNG)

Once the project is created you will see it comes with a generated pom.xml file. Here we can just paste in our own pom.xml file we have created above and import the changes. This will automatically download all of the required dependencies for use within our project. We can now create a new java file under `src/main/java` called whatever we want (In my case I just made `Application.java` with a main function). This is where all of our packages and code will go and we can import any of our libraries downloaded by our pom.xml into classes here. Importing and using some random functions from OpenGL and JOML you can see no errors are thrown and all is working:

![OpenGL_MavenTutorial_2.PNG](OpenGL_MavenTutorial_2.PNG)

Running our program in IntelliJ works the very same as usual. Simply create a run configuration of type `Application` and specify the entry file and you are all good to go.

## Compiling to JAR file

**NOTE:** For this section of the tutorial I replaced `Application.java` with the `Application.java` that we left off with in Part 1 of the [Mandelbrot Set Tutorial Series](https://cianjinks.github.io/posts/opengl-tutorial-visualizing-the-mandelbrot-set-fractal-part-1-of-2/) and brought our shaders over into the resources folder. This means we are now essentially working as if this project is the one from the tutorial.

When you want to distribute your java project it is common to package it all into a single jar file which can then be ran by people cross platform. Maven makes this incredibly easy to accomplish as it comes with some inbuilt commands for doing so. You can run any of these commands from the `Maven` tab on the right side of the IntelliJ window:

![OpenGL_MavenTutorial_3.PNG](OpenGL_MavenTutorial_3.PNG)

Running `install` will usually generate a folder named `target` in the root directory of your project and in here will be the compiled jar file for your project. _However_ if you try running this as is, you will run into errors similar to these:

```
Source option 5 is no longer supported. Use 6 or later.
Target option 1.5 is no longer supported. Use 1.6 or later.
```

To fix these we are going to have to make a few more modifications to our pom.xml file. First off we need to specify 3 new properties:

```xml
<properties>
	<lwjgl.version>3.2.3</lwjgl.version>
    <joml.version>1.9.20</joml.version>
    <lwjgl.natives>natives-windows</lwjgl.natives>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

These properties simply specify what version of java we want our project to be compiled with. Once added, `install` will now function correctly and produce our jar file in `target`. If you go ahead and try to run this jar file you will encounter yet another error. To make this work we require a simple plugin to package all of our dependencies together and specify an entry file for our program jar. Here is the xml for that plugin:

```xml
<build>
  <plugins>
    <plugin>
      <artifactId>maven-assembly-plugin</artifactId>
      <executions>
        <execution>
          <id>make-assembly</id>
          <phase>package</phase>
          <goals>
            <goal>single</goal>
          </goals>
        </execution>
      </executions>
      <configuration>
        <archive>
          <manifest>
            <!--Here we point to our main class based on our package.-->
            <!--In my class I will simply put Application as my main class is called Application-->
            <!--and is not within any packages.-->
            <mainClass>com.packagename.class</mainClass>
          </manifest>
        </archive>
        <descriptorRefs>
          <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
        <appendAssemblyId>false</appendAssemblyId>
      </configuration>
    </plugin>
  </plugins>
</build>
```

If we go ahead and add this to our pom.xml everything should now work perfectly. We can compile a jar file of our project using Maven's `install` and run it using any machine with java installed. Here is how my final pom.xml looks:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.hobbesos</groupId>
    <artifactId>MandelbrotViewerTutorial</artifactId>
    <version>1.1</version>

    <url>https://github.com/cianjinks/MandelbrotViewer</url>

    <licenses>
        <license>
            <name>MIT License</name>
            <url>http://www.opensource.org/licenses/mit-license.php</url>
        </license>
    </licenses>

    <developers>
        <developer>
            <name>Cian Jinks</name>
            <email>cjinks99@gmail.com</email>
        </developer>
    </developers>

    <properties>
        <lwjgl.version>3.2.3</lwjgl.version>
        <joml.version>1.9.20</joml.version>
        <lwjgl.natives>natives-windows</lwjgl.natives>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.lwjgl</groupId>
                <artifactId>lwjgl-bom</artifactId>
                <version>${lwjgl.version}</version>
                <scope>import</scope>
                <type>pom</type>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.lwjgl</groupId>
            <artifactId>lwjgl</artifactId>
        </dependency>
        <dependency>
            <groupId>org.lwjgl</groupId>
            <artifactId>lwjgl-glfw</artifactId>
        </dependency>
        <dependency>
            <groupId>org.lwjgl</groupId>
            <artifactId>lwjgl-opengl</artifactId>
        </dependency>
        <dependency>
            <groupId>org.lwjgl</groupId>
            <artifactId>lwjgl</artifactId>
            <classifier>${lwjgl.natives}</classifier>
        </dependency>
        <dependency>
            <groupId>org.lwjgl</groupId>
            <artifactId>lwjgl-glfw</artifactId>
            <classifier>${lwjgl.natives}</classifier>
        </dependency>
        <dependency>
            <groupId>org.lwjgl</groupId>
            <artifactId>lwjgl-opengl</artifactId>
            <classifier>${lwjgl.natives}</classifier>
        </dependency>
        <dependency>
            <groupId>org.joml</groupId>
            <artifactId>joml</artifactId>
            <version>${joml.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>Application</mainClass>
                        </manifest>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <appendAssemblyId>false</appendAssemblyId>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## Building your project from source

If you wish to distribute your project you can of course just send out the JAR file but if you want people to be able to build your project using your source code it is also quite simple. All you have to provide them is the `pom.xml` file and the folder called `src` from your project root directory. You can see an example of this on my github for the [Mandelbrot Viewer](https://github.com/cianjinks/MandelbrotViewer). They can then simply import the pom.xml file into their respective IDE or compile the JAR file from the command line using the command `mvn clean package` in the project folder.

With that, this tutorial is complete! You should now be able to quickly get setup when you want to start programming using OpenGL through Java and can easily package your program to distribute however you like. I hope you were able to learn something from this and will possibly use Maven for future projects of your own. If you wish to explore more of Maven's capabilities as there are many you can check out its [website](https://maven.apache.org/). Thanks so much for reading!

\- Cian Jinks
