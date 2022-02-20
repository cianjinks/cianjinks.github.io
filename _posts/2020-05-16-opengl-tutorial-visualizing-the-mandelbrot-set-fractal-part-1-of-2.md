---
title: OpenGL Tutorial - Visualizing the Mandelbrot Set Fractal - Part 1 of 2
date: 2020-05-16 12:00:00 +0000
categories: [Old]
tags: [opengl, lwjgl, fractal, tutorial]
img_path: /assets/img/post/OpenGLMandelbrot/
---

Welcome to my first blog post/tutorial of many I hope to create on the topic of computer graphics programming using OpenGL. Over the past half a year or so I have been spending a large portion of my spare time teaching myself the ins and outs of the OpenGL api and in doing so have created some projects along the way. In this specific tutorial set I wanted to guide you through creating one of my earlier projects which I completed some time in January of 2020. That being a program which displays a stylised version of the [Mandelbrot Set](https://en.wikipedia.org/wiki/Mandelbrot_set) fractal pattern. I would have loved to have found all of this information condensed in one post when making it and so I hope this will be of benefit to you. In terms of the knowledge required to understand this tutorial, I wrote my programs in Java and would like this to be a tutorial one can learn from scratch with no prior OpenGL experience. I am going to attempt to explain the basic concepts in OpenGL as best I can meaning you will only need some experience with an object oriented programming language to understand it all. I will also regularly supply other sources for reading up on any concepts mentioned throughout.

(If you simply wish to learn how to create the Mandelbrot Set shader please check out [part two](https://cianjinks.github.io/2020-05-31-opengl-tutorial-visualizing-the-mandelbrot-set-part-2-of-2/) of this tutorial)

One last thing before I dive right into the explanations is that a further updated and completed version of this project with all of the source code as well as compiled jars can be found over on my project's github page called [Mandelbrot-Viewer](https://github.com/cianjinks/MandelbrotViewer) so I encourage giving that a look too.

## Getting Started

First things first for those who don't know, what is OpenGL? OpenGL aims to be a cross platform computer graphics api. This means in theory you can use it to communicate with GPUs on various systems and platforms that exist today. _However,_ note that I referred to it as an api. At its core OpenGL is simply a specification of commands that one should be able to send to their GPU to make it perform graphics calculations. It is generally up to third parties to provide the implementations of this api. Luckily for us OpenGL is relatively popular and so pretty much all modern graphics cards support it (although much older graphics cards only support up to specific versions) and there are bindings for it available in a wide variety of programming languages. For this tutorial I will be writing in Java and thus using [LWJGL](https://www.lwjgl.org/) for the OpenGL bindings. LWJGL also comes with some other useful libraries with the most notable for us being GLFW which is used for the creation of an application window which we can then draw to on various operating systems. Since OpenGL is a specification any OpenGL methods that I do use throughout this tutorial should have a largely similar method signature using other bindings and languages making it hopefully no problem for following along.

If you do want to follow this tutorial directly note that I selected the Minimal OpenGL configuration on LWJGL's [customize page](https://www.lwjgl.org/customize) and added the JOML math library addon. I also chose to use Maven for compiling my program and binaries. If you don't know how Maven works I will soon be writing a tutorial about setting up a Maven Java project for OpenGL programming and will use this tutorial as an example.

**EDIT:** That tutorial can now be found [here](https://cianjinks.github.io/2020-05-28-tutorial-using-maven-and-intellij-for-opengl-projects/).

## The Window

To begin our program we are going to want to use GLFW to create a window for our respective operating system. I am using Windows for my OS but everything should work relatively the same cross platform. To do so is relatively simple as we can just head on over to the [LWJGL starter page](https://www.lwjgl.org/guide) and grab their code for starting a window. I chose to place it in a class called `Application`. As for what this code does, it provides three functions named `run`, `init` and `loop` which are run consecutively when the program starts. Our window is created within `init` using `glfwCreateWindow` and some other configuration options are set. Most of these have explanatory comments provided however if you wish to investigate further as to what these various functions do you can check out [GLFW's documentation](https://www.glfw.org/docs/latest/). Do note that examples are provided in C++ not Java. After `glfwCreateWindow` some key callbacks are set up which can be used to detect the pressing of keys. The only one used for the time being is the escape key to close the window. At the moment the only thing we wish to modify here would be the window proportions. For this project I chose to set the width and height to 480x480 (960x960 would also work fine, as would any square size) making for a perfect square which will simplify things later on.

The next area of code we need to worry about is within `loop`. Here you will notice a while loop which waits until the window is closed before completing. This will act as our program's main loop meaning in here we will place code we want to run every frame drawn to the screen (All code here should be placed _before_ the `glfwSwapBuffers` function). Just to make sure that this starter code all works correctly we can go ahead and run the program and should be greeted with a square window colored red due to the `glClearColor` method in `loop`. For reference here is how our code should look so far:

```java
import org.lwjgl.*;
import org.lwjgl.glfw.*;
import org.lwjgl.opengl.*;
import org.lwjgl.system.*;

import java.nio.*;

import static org.lwjgl.glfw.Callbacks.*;
import static org.lwjgl.glfw.GLFW.*;
import static org.lwjgl.opengl.GL11.*;
import static org.lwjgl.system.MemoryStack.*;
import static org.lwjgl.system.MemoryUtil.*;

public class Application {

    // The window handle
    private long window;

    public void run() {
        System.out.println("Hello LWJGL " + Version.getVersion() + "!");

        init();
        loop();

        // Free the window callbacks and destroy the window
        glfwFreeCallbacks(window);
        glfwDestroyWindow(window);

        // Terminate GLFW and free the error callback
        glfwTerminate();
        glfwSetErrorCallback(null).free();
    }

    private void init() {
        // Setup an error callback. The default implementation
        // will print the error message in System.err.
        GLFWErrorCallback.createPrint(System.err).set();

        // Initialize GLFW. Most GLFW functions will not work before doing this.
        if ( !glfwInit() )
            throw new IllegalStateException("Unable to initialize GLFW");

        // Configure GLFW
        glfwDefaultWindowHints(); // optional, the current window hints are already the default
        glfwWindowHint(GLFW_VISIBLE, GLFW_FALSE); // the window will stay hidden after creation
        glfwWindowHint(GLFW_RESIZABLE, GLFW_TRUE); // the window will be resizable

        // Create the window
        window = glfwCreateWindow(960, 960, "Hello World!", NULL, NULL);
        if ( window == NULL )
            throw new RuntimeException("Failed to create the GLFW window");

        // Setup a key callback. It will be called every time a key is pressed, repeated or released.
        glfwSetKeyCallback(window, (window, key, scancode, action, mods) -> {
            if ( key == GLFW_KEY_ESCAPE && action == GLFW_RELEASE )
                glfwSetWindowShouldClose(window, true); // We will detect this in the rendering loop
        });

        // Get the thread stack and push a new frame
        try ( MemoryStack stack = stackPush() ) {
            IntBuffer pWidth = stack.mallocInt(1); // int*
            IntBuffer pHeight = stack.mallocInt(1); // int*

            // Get the window size passed to glfwCreateWindow
            glfwGetWindowSize(window, pWidth, pHeight);

            // Get the resolution of the primary monitor
            GLFWVidMode vidmode = glfwGetVideoMode(glfwGetPrimaryMonitor());

            // Center the window
            glfwSetWindowPos(
                    window,
                    (vidmode.width() - pWidth.get(0)) / 2,
                    (vidmode.height() - pHeight.get(0)) / 2
            );
        } // the stack frame is popped automatically

        // Make the OpenGL context current
        glfwMakeContextCurrent(window);
        // Enable v-sync
        glfwSwapInterval(1);

        // Make the window visible
        glfwShowWindow(window);
    }

    private void loop() {
        // This line is critical for LWJGL's interoperation with GLFW's
        // OpenGL context, or any context that is managed externally.
        // LWJGL detects the context that is current in the current thread,
        // creates the GLCapabilities instance and makes the OpenGL
        // bindings available for use.
        GL.createCapabilities();

        // Set the clear color
        glClearColor(1.0f, 0.0f, 0.0f, 0.0f);

        // Run the rendering loop until the user has attempted to close
        // the window or has pressed the ESCAPE key.
        while ( !glfwWindowShouldClose(window) ) {
            glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

            glfwSwapBuffers(window); // swap the color buffers

            // Poll for window events. The key callback above will only be
            // invoked during this call.
            glfwPollEvents();
        }
    }

    public static void main(String[] args) {
        new Application().run();
    }

}

```
This will produce the following window:

![OpenGL_Tutorial_Mandelbrot_1.png](OpenGL_Tutorial_Mandelbrot_1.png)

## Vertices

Now that we have got the window set up it is nearly time to get to graphics programming. But first there are a few basic concepts that need to be understood. The way in which modern graphics work is that any shape, image or 3d mesh that one wishes to be rendered to a screen is made up of polygons. For more complex work there can sometimes be thousands of said polygons that make up a single shape or model. The most efficient form of polygons for graphics programming are triangles. Each triangle consists of three points and three lines joining all of those points together to form the triangle. Each of these points is what is referred to as a vertex and together they are vertices. However, you should not think of vertices as just the corners/points making up a triangle implying all they can contain is a position. While they do contain a position, in OpenGL these vertices can in fact contain all kinds of data about a given triangle such as color, texture coordinates, lighting information and more.

For our program we will only need to render a simple square to the screen and won't require the vertices to store anything more than their positions. One last thing to note is that, by default, positions in OpenGL range from -1.0 to 1.0 for the x and y coordinates of the screen. With that in mind here is how we would go about creating a square out of triangle polygons which fills the entire window (because the very bottom left in OpenGL is (-1.0, -1.0) and the top right is (1.0, 1.0)):

![OpenGL_Tutorial_Mandelbrot_2.png](OpenGL_Tutorial_Mandelbrot_2.png)

As can be seen all that is needed is two right angled triangles joined together. These can be defined using just four vertices with positions of (-1.0, 1.0), (-1.0, -1.0), (1.0, -1.0) and (1.0, 1.0) so that they fill up the whole screen. In general, when implementing the handling of vertices in code, one would likely wish to create a `Vertex` class to allow the creation of an arbitrary number of vertices each with their unique data. However, in our case we only will be using four vertices so it is easier to just hardcode their position data into an array as follows (this code will be placed in our `loop` method _before_ the actual main window while loop):

```java
// Vertices - Positions
float[] vertices = new float[] {
    -1.0f,  1.0f,   // Vertex 0
    -1.0f, -1.0f,   // Vertex 1
     1.0f, -1.0f,   // Vertex 2
     1.0f,  1.0f    // Vertex 3
};
```

## Vertex Buffers

Now you might be asking yourself how is it that we actually pass this data to our graphics card through OpenGL? This is where the concept of a vertex buffer comes into play. A vertex buffer is simply an area of memory on your gpu which stores information about your various vertices. The way in which we actually place our data there is through the use of a few OpenGL functions which specify how our vertices will be laid out in memory and allow us to place the vertex data into that memory. Before any of that however, there is one more thing we must do in Java specifically which is to transfer our `vertices` array we created above to a `FloatBuffer`. This is common practice when using Java for OpenGL and if you wish to learn why that is I encourage doing some googling or checking out this [stackoverflow post](https://stackoverflow.com/questions/18913001/when-to-use-array-buffer-or-direct-buffer).

```java
// VBO (Vertex Buffer Object)
FloatBuffer vboBuffer = BufferUtils.createFloatBuffer(vertices.length);
for(float vertex : vertices) {
	vboBuffer.put(vertex);
}
vboBuffer.flip();
```

You'll notice that we needed to run `flip` on our buffer at the end. The reasoning for this can be found [here](https://stackoverflow.com/questions/29397407/gldrawbuffers-should-have-flipped-intbuffer).

Now we can finally pass our data to our gpu through some functions provided by OpenGL:

```java
// Pass data to GPU
int positionElementCount = vertices.length / 4;
int vboID = GL30.glGenBuffers();
GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
GL30.glBufferData(GL30.GL_ARRAY_BUFFER, vboBuffer, GL30.GL_STATIC_DRAW);
GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, positionElementCount * Float.BYTES, 0);
GL30.glEnableVertexAttribArray(0);
```

There is quite a bit going on here so I will try to break it down. However I don't want to spend overly long on it so if you are still confused I highly recommend checking out the [docs.gl](http://docs.gl/) website for further reading on what each function does and the various parameters that can be passed to it. (In this tutorial I use OpenGL 3.0 so when checking for functions on docs.gl select their gl3 counterpart).

### glGenBuffers

Here we are telling OpenGL we wish to create a buffer on the gpu where we can store some information. This function will return us an ID in the form of an Integer which we can later use to reference this same buffer.

### glBindBuffer

When we want to modify a buffer we have created we must first bind it. When doing so we specify the type of buffer it is which in our case is a `GL_ARRAY_BUFFER`. If you check out the [docs.gl](http://docs.gl/) page for `glBindBuffer` you'll note it says this type of buffer is used for storing vertex attributes. Vertex attributes are the different pieces of data our vertex stores as mentioned before - position, color, texture coordinates, lighting information and anything else you may want.

### glBufferData

This function is where we actually pass our vertices data to the gpu. We tell it the type of data that we are placing within the buffer which is once again vertex attributes (denoted by `GL_ARRAY_BUFFER`) and then we tell it to use `GL_STATIC_DRAW`. This means the data we pass is going to be used for drawing to the screen and will not be modified again in the future. As you might guess this means we cannot manipulate the vertices on the fly as the program is running which for us is all that we need. If you wish to learn about how one would do such a thing, to allow the moving of vertices and more, I will have another tutorial coming soon which I will link here.

### glVertexAttribPointer

Now that we have provided our vertex data to the gpu how does it know what that data means? We need to explain to it the layout of our memory and what each byte is for. This is where `glVertexAttribPointer` is used. The first parameter which we have passed is 0. This simply tells the GPU this is our first vertex attribute. After that we need to tell it how many pieces of data make up that attribute. In our case we have four positions specified in our array where each position requires an x and a y. This means that each attribute is made up of two pieces of data. The type of this data is then specified next as `GL_FLOAT` in our case because each x and y position is a float. `false` then tells it this data is not normalised. The GPU still needs to know one more thing which is how many bytes each of our position attributes take up. You might wonder why this is necessary since it already knows their type and how many of that type are in each? However, when you have multiple different vertex attributes it needs to know where each one starts. In our case we have two floats worth of bytes which I have calculated by `positionElementCount * Float.BYTES`. Finally the last parameter tells it where the first occurence of this vertex attribute begins in our vertex buffer. Since this is our only vertex attribute it begins at position 0.

Once again a much more detailed explanation of this function is found on its [docs.gl page](http://docs.gl/gl4/glVertexAttribPointer).

### glEnableVertexAttribArray

This last function simply enables our first vertex attribute (index 0).

With that out of the way our vertex buffer is now complete! Here is what our `loop` function should now look like in its entirety:

```java
private void loop() {
        // This line is critical for LWJGL's interoperation with GLFW's
        // OpenGL context, or any context that is managed externally.
        // LWJGL detects the context that is current in the current thread,
        // creates the GLCapabilities instance and makes the OpenGL
        // bindings available for use.
        GL.createCapabilities();

        // Set the clear color
        glClearColor(1.0f, 0.0f, 0.0f, 0.0f);

        // Vertices - Positions
        float[] vertices = new float[] {
                -1.0f,  1.0f,   // Vertex 0
                -1.0f, -1.0f,   // Vertex 1
                 1.0f, -1.0f,   // Vertex 2
                 1.0f,  1.0f    // Vertex 3
        };

        // VBO (Vertex Buffer Object)
        FloatBuffer vboBuffer = BufferUtils.createFloatBuffer(vertices.length);
        for(float vertex : vertices) {
            vboBuffer.put(vertex);
        }
        vboBuffer.flip();

        // Pass data to GPU
        int positionElementCount = vertices.length / 4;
        int vboID = GL30.glGenBuffers();
        GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
        GL30.glBufferData(GL30.GL_ARRAY_BUFFER, vboBuffer, GL30.GL_STATIC_DRAW);
        GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, positionElementCount * Float.BYTES, 0);
	GL30.glEnableVertexAttribArray(0);

        // Run the rendering loop until the user has attempted to close
        // the window or has pressed the ESCAPE key.
        while ( !glfwWindowShouldClose(window) ) {
            glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

            glfwSwapBuffers(window); // swap the color buffers

            // Poll for window events. The key callback above will only be
            // invoked during this call.
            glfwPollEvents();
        }
}
```

## Index Buffers

The next important concept in OpenGL required for us to draw a square are index buffers. So far we have passed to the gpu the positions of our four vertices that are going to make up our square and explained how these positions are laid out in memory. However, we still haven't explained to the gpu how it should make use of this data. How does it know to draw two right angled triangles using these vertex positions? This is where index buffers come into play. We can use an index buffer to specify which vertices make up our triangles by storing some indices within it:

```java
// Indices
int[] indices = new int[] {
	0, 1, 2,
  	2, 3, 0
};
```

In this setup we are saying that one of our right angled triangles is going to be made up of vertex 0, 1 and 2 and the other will be made up of vertex 2, 3 and 0. Note that the order in which these are specified is counter clockwise with respect to the diagram shown in the vertices section of the tutorial.

To pass these indices to the gpu we are going to create an index buffer. This process is much the same as the vertex buffer although slightly simpler:

```java
// IBO (Index Buffer Object)
IntBuffer iboBuffer = BufferUtils.createIntBuffer(indices.length);
for(int index : indices) {
 	iboBuffer.put(index);
}
iboBuffer.flip();

// Pass data to GPU
int iboID = GL30.glGenBuffers();
GL30.glBindBuffer(GL30.GL_ELEMENT_ARRAY_BUFFER, iboID);
GL30.glBufferData(GL30.GL_ELEMENT_ARRAY_BUFFER, iboBuffer, GL30.GL_STATIC_DRAW);
```

One slight difference from the vertex buffer is the use of `GL_ELEMENT_ARRAY_BUFFER` instead of `GL_ARRAY_BUFFER`. This type of buffer is used for storing vertex indices instead of vertex attributes.

With that we are all done with our index buffers and our finalised code should look something like this:

```java
private void loop() {
        // This line is critical for LWJGL's interoperation with GLFW's
        // OpenGL context, or any context that is managed externally.
        // LWJGL detects the context that is current in the current thread,
        // creates the GLCapabilities instance and makes the OpenGL
        // bindings available for use.
        GL.createCapabilities();

        // Set the clear color
        glClearColor(1.0f, 0.0f, 0.0f, 0.0f);

        // Vertices - Positions
        float[] vertices = new float[] {
                -1.0f,  1.0f,   // Vertex 0
                -1.0f, -1.0f,   // Vertex 1
                 1.0f, -1.0f,   // Vertex 2
                 1.0f,  1.0f    // Vertex 3
        };

        // VBO (Vertex Buffer Object)
        FloatBuffer vboBuffer = BufferUtils.createFloatBuffer(vertices.length);
        for(float vertex : vertices) {
            vboBuffer.put(vertex);
        }
        vboBuffer.flip();

        // Pass data to GPU
        int positionElementCount = vertices.length / 4;
        int vboID = GL30.glGenBuffers();
        GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
        GL30.glBufferData(GL30.GL_ARRAY_BUFFER, vboBuffer, GL30.GL_STATIC_DRAW);
        GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, positionElementCount * Float.BYTES, 0);
        GL30.glEnableVertexAttribArray(0);

        // Indices
        int[] indices = new int[] {
            0, 1, 2,
            2, 3, 0
        };

        // IBO (Index Buffer Object)
        IntBuffer iboBuffer = BufferUtils.createIntBuffer(indices.length);
        for(int index : indices) {
            iboBuffer.put(index);
        }
        iboBuffer.flip();

        // Pass data to GPU
        int iboID = GL30.glGenBuffers();
        GL30.glBindBuffer(GL30.GL_ELEMENT_ARRAY_BUFFER, iboID);
        GL30.glBufferData(GL30.GL_ELEMENT_ARRAY_BUFFER, iboBuffer, GL30.GL_STATIC_DRAW);


        // Run the rendering loop until the user has attempted to close
        // the window or has pressed the ESCAPE key.
        while ( !glfwWindowShouldClose(window) ) {
            glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

            glfwSwapBuffers(window); // swap the color buffers

            // Poll for window events. The key callback above will only be
            // invoked during this call.
            glfwPollEvents();
        }
}
```

If we want, we can now go ahead and try drawing our square using one more line of code in our main window loop:

```java
// Run the rendering loop until the user has attempted to close
// the window or has pressed the ESCAPE key.
while ( !glfwWindowShouldClose(window) ) {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

    GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0); //Draw our square

    glfwSwapBuffers(window); // swap the color buffers

    // Poll for window events. The key callback above will only be
    // invoked during this call.
    glfwPollEvents();
}
```

Here we are using `glDrawElements` to draw our vertices based on our index buffer which was a `GL_ELEMENT_ARRAY_BUFFER`. We use `GL_TRIANGLES` mode to specify we want it to take each group of three indices to form a triangle. After than we need to tell it how many indices we have placed in our index buffer up above which is simply the length of our array. Our indices are of course of type Integer and so we specify `GL_UNSIGNED_INT`. The very last parameter is a 0 as our indices start from index 0 in our index buffer.

We can then run our program. If you do get a crash here and are using a different language or OpenGL binding to follow this tutorial it is possible that that OpenGL binding does not come with a default shader so you will have to create one using the next section of the tutorial first. Otherwise using LWJGL you should get a fully white window as your white square now fills the window:

![OpenGL_Tutorial_Mandelbrot_3.png](OpenGL_Tutorial_Mandelbrot_3.png)

Just to be 100% sure that it is working we could modify the position of Vertex 0 to (-0.5, 1.0) which should reveal a slight bit of the original red background as our shape no longer covers that top left portion fully:

![OpenGL_Tutorial_Mandelbrot_4.png](OpenGL_Tutorial_Mandelbrot_4.png)

## Shaders

Some of you may have already heard of shaders before. They are one of the biggest features of OpenGL and are what allow us to customise the look of our square as we please. Eventually we are going to write a shader which visualizes the mandelbrot set however for the time being, as a good introduction to shaders, we will simply write a shader to color our square purple. We won't delve particularly deep into the various features offered by shaders here but if you wish to research further there are numerous resources online for doing so.

The first thing you need to know is that shaders are written in their own programming language called GLSL (GL Shading Language). Don't worry though, it is relatively simple to figure out if you already have experience programming in general. If you do end up wanting to learn much more about it after this tutorial you can check out this [youtube series](https://youtu.be/HIvNePu7UEE?list=PL4neAtv21WOmIrTrkNO3xCyrxg4LKkrF7).

Before we write any shaders of our own we first need to do some set up in our code for handling them. Since they're going to be written in their own language it is common practice to store them in their own files. When we need to pass them to the gpu we will load them in from their respective file as a string. In java I will use a simple `BufferedReader` and it's concept of streams to read in the contents of a file and return them as a string via the function `parseShaderFromFile`. In other languages and on other systems this will likely work differently and you will need to make sure that new line characters are captured accurately as well otherwise your shader will not compile and will produce no errors as we do not have any error handling implemented.

```java
private static String parseShaderFromFile(String filePath) {
    StringBuilder data = new StringBuilder();
    String line = "";
    try {
        BufferedReader reader = new BufferedReader(new InputStreamReader(Application.class.getResourceAsStream(filePath)));
        line = reader.readLine();
        while( line != null )
        {
            data.append(line);
            data.append('\n');
            line = reader.readLine();
        }
    }
    catch(Exception e)
    {
        throw new IllegalArgumentException("Unable to load shader from: " + filePath, e);
    }

    return data.toString();
}
```

With that function setup we can then use it in our code to load in our shaders and send them to our gpu:

```java
// Shaders
int programID = GL30.glCreateProgram();
int vertShaderObj = GL30.glCreateShader(GL30.GL_VERTEX_SHADER);
int fragShaderObj = GL30.glCreateShader(GL30.GL_FRAGMENT_SHADER);
String vertexShader = parseShaderFromFile("/shaders/vert.shader");
GL30.glShaderSource(vertShaderObj, vertexShader);
GL30.glCompileShader(vertShaderObj);
String fragmentShader = parseShaderFromFile("/shaders/frag.shader");
GL30.glShaderSource(fragShaderObj, fragmentShader);
GL30.glCompileShader(fragShaderObj);
GL30.glAttachShader(programID, vertShaderObj);
GL30.glAttachShader(programID, fragShaderObj);
GL30.glLinkProgram(programID);
GL30.glValidateProgram(programID);
GL30.glUseProgram(programID);
```

We do a good bit here so I will try my best to explain. First we must create a program for which to bind our shaders using `glCreateProgram`. There are multiple different types of shaders but by far the main two you should be aware of are Vertex shaders and Fragment shaders. Vertex shaders are run on each vertex of the polygon they are applied to and as a result are able to manipulate the various vertex attributes associated with them. On the other hand fragment shaders are run on every pixel in between these vertices meaning they can be used for texturing, coloring, lighting calculations and more. With that in mind we then use `glCreateShader` to tell the gpu we want a `GL_VERTEX_SHADER` and a `GL_FRAGMENT_SHADER`. This function will return an integer to act as a handler which we can use to reference that same shader again in the future. Next we have to provide this shader to the gpu using `glShaderSource`. Just before doing that you can see we load each shader from their respective file into a string using the `parseShaderFromFile` method. I am storing the shaders in a location relative to my maven project (specifically the resources folder) but your file path may of course be different.

For a majority of the rest of the functions used here you get a basic idea of what they do from reading them and you don't necessarily need to know what they do at a lower level but if you wish to delve deeper check out [docs.gl](http://docs.gl/) once again.

Finally we can begin writing our own shaders. For now we will simply color our square purple but in part two of this tutorial this is where we will write the code within the fragment shader to generate the mandelbrot set.

### Vertex Shader

```glsl
// /shaders/vert.shader
layout(location = 0) in vec2 position;
void main() {
    gl_Position = vec4(position, 0, 1.0);
}
```

This is going to be the basic outline for our vertex shader. The very first line takes in our vertex attribute placed at index 0 which is the position of each of our vertices that we set up way back in our vertex buffer. GLSL supports vectors which are made up of floats and since our position data is stored in groups of 2 floats we are going to store the position of the current vertex being processed by the shader in a vec2 (2 float vector) called `position`. After that we define a `main` which needs to be present in all shaders and provide that position to a special OpenGL variable called `gl_Position`. This variable is what internally determines the position of the current vertex being processed by our vertex shader. The reason for converting our position variable from a vec2 to a vec4 before passing it to `gl_Position` is because OpenGL uses an xyzw coordinate system. An explanation for the reasoning as to why there is an extra w coordinate can be found [here](https://stackoverflow.com/questions/2422750/in-opengl-vertex-shaders-what-is-w-and-why-do-i-divide-by-it).

### Fragment Shader

```glsl
void main() {
    gl_FragColor = vec4(0.3, 0.0, 0.3, 1.0);
}
```

Our fragment shader is even simpler than our vertex shader. All it does is set another built in OpenGL variable, that being `gl_FragColor`, which controls the color of the pixels between our vertices. In this case we are just setting it to a purple color in RGBA. RGBA values range from 0.0 to 1.0 in OpenGL instead of the usual 0 to 255.

With that, we are all done! We can run our program and should see a nice purple window which is our square:

![OpenGL_Tutorial_Mandelbrot_5.png](OpenGL_Tutorial_Mandelbrot_5.png)

This entire process may have seemed overly long and tedious just to produce a simple square but as we explore more in the next part of this tutorial and hopefully in future tutorials it will begin to become clear just how useful and powerful this all is. Once you begin to add layers of abstraction it can simplify things much further as well.

For reference here is the `loop` function in its entirety along with `parseShaderFromFile`:

```java
private void loop() {
    // This line is critical for LWJGL's interoperation with GLFW's
    // OpenGL context, or any context that is managed externally.
    // LWJGL detects the context that is current in the current thread,
    // creates the GLCapabilities instance and makes the OpenGL
    // bindings available for use.
    GL.createCapabilities();

    // Set the clear color
    glClearColor(1.0f, 0.0f, 0.0f, 0.0f);

    // Vertices - Positions
    float[] vertices = new float[] {
            -1.0f,  1.0f,   // Vertex 0
            -1.0f, -1.0f,   // Vertex 1
             1.0f, -1.0f,   // Vertex 2
             1.0f,  1.0f    // Vertex 3
    };

    // VBO (Vertex Buffer Object)
    FloatBuffer vboBuffer = BufferUtils.createFloatBuffer(vertices.length);
    for(float vertex : vertices) {
        vboBuffer.put(vertex);
    }
    vboBuffer.flip();

    // Pass data to GPU
    int positionElementCount = vertices.length / 4;
    int vboID = GL30.glGenBuffers();
    GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
    GL30.glBufferData(GL30.GL_ARRAY_BUFFER, vboBuffer, GL30.GL_STATIC_DRAW);
    GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, positionElementCount * Float.BYTES, 0);
    GL30.glEnableVertexAttribArray(0);

    // Indices
    int[] indices = new int[] {
        0, 1, 2,
        2, 3, 0
    };

    // IBO (Index Buffer Object)
    IntBuffer iboBuffer = BufferUtils.createIntBuffer(indices.length);
    for(int index : indices) {
        iboBuffer.put(index);
    }
    iboBuffer.flip();

    // Pass data to GPU
    int iboID = GL30.glGenBuffers();
    GL30.glBindBuffer(GL30.GL_ELEMENT_ARRAY_BUFFER, iboID);
    GL30.glBufferData(GL30.GL_ELEMENT_ARRAY_BUFFER, iboBuffer, GL30.GL_STATIC_DRAW);

    // Shaders
    int programID = GL30.glCreateProgram();
    int vertShaderObj = GL30.glCreateShader(GL30.GL_VERTEX_SHADER);
    int fragShaderObj = GL30.glCreateShader(GL30.GL_FRAGMENT_SHADER);
    String vertexShader = parseShaderFromFile("/shaders/vert.shader");
    GL30.glShaderSource(vertShaderObj, vertexShader);
    GL30.glCompileShader(vertShaderObj);
    String fragmentShader = parseShaderFromFile("/shaders/frag.shader");
    GL30.glShaderSource(fragShaderObj, fragmentShader);
    GL30.glCompileShader(fragShaderObj);
    GL30.glAttachShader(programID, vertShaderObj);
    GL30.glAttachShader(programID, fragShaderObj);
    GL30.glLinkProgram(programID);
    GL30.glValidateProgram(programID);
    GL30.glUseProgram(programID);

    // Run the rendering loop until the user has attempted to close
    // the window or has pressed the ESCAPE key.
    while ( !glfwWindowShouldClose(window) ) {
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

        GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0); //Draw our square

        glfwSwapBuffers(window); // swap the color buffers

        // Poll for window events. The key callback above will only be
        // invoked during this call.
        glfwPollEvents();
    }
}

private static String parseShaderFromFile(String filePath) {
    StringBuilder data = new StringBuilder();
    String line = "";
    try {
        BufferedReader reader = new BufferedReader(new InputStreamReader(Application.class.getResourceAsStream(filePath)));
        line = reader.readLine();
        while( line != null )
        {
            data.append(line);
            data.append('\n');
            line = reader.readLine();
        }
    }
    catch(Exception e)
    {
        throw new IllegalArgumentException("Unable to load shader from: " + filePath, e);
    }

    return data.toString();
}
```

## Part 2

The [next part](https://cianjinks.github.io/2020-05-31-opengl-tutorial-visualizing-the-mandelbrot-set-part-2-of-2/) of this tutorial will cover some more advanced concepts such as how we can set up a simple camera using user input and an MVP Matrix (Model View Projection). That is also where we will write a fragment shader to visualize the Mandelbrot Set Fractal on our square and discuss some of its limitations. If you've already made it this far thanks so much for reading and I hope I was able to help you learn a bit along the way :D

\- Cian Jinks
