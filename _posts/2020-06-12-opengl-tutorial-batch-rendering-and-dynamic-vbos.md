---
title: OpenGL Tutorial - Batch Rendering and Dynamic VBOs
date: 2020-06-12 12:00:00 +0000
categories: [Old]
tags: [opengl, lwjgl, tutorial]
img_path: /assets/img/post/OpenGLBatchRendering/
---
Hello and welcome to my second ever tutorial on computer graphics programming using OpenGL. My [previous tutorials](https://cianjinks.github.io/posts/opengl-tutorial-visualizing-the-mandelbrot-set-fractal-part-1-of-2/) covered many of the basics of OpenGL such as VBOs, IBOs, Shaders and MVP Matrices by implementing them in a Mandelbrot Set Fractal application. In that tutorial series we only ended up drawing a single quad to the screen which we then applied our Mandelbrot Fragment Shader to. If you remember, when supplying our data to our VBO using `glBufferData`, we specified for it to use `GL_STATIC_DRAW`. This meant that whatever data we placed there would not be changed again in the future. This data being the vertex attributes of our vertices, which in our case was their position.

As a result, we are unable to modify the position of our quad's vertices on the fly. So to get around this when we wanted to "move" our quad, we instead moved the "camera" which we created. Meaning the original vertex positions of our quad never changed. This works great when all we have is one quad however what if you wanted to have multiple quads in the same Vertex Buffer and move them all in different directions independent of each other? That's where dynamic vertex buffers and batch rendering comes into play.

## Draw Calls

Before we begin talking about Batch Rendering and making our Vertex Buffers dynamic we need to grasp the idea of a draw call. Once all of our various vertex attributes and indices are placed into their respective buffers on the GPU we then use the following function in our main loop to draw everything to the screen:

```java
GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0);
```

(If you don't understand the specific parameters of this function check out the [mandelbrot tutorial part 1](https://cianjinks.github.io/posts/opengl-tutorial-visualizing-the-mandelbrot-set-fractal-part-1-of-2/) or read about it on [docs.gl](http://docs.gl/))

This function is called a "Draw Call" as we are telling the GPU to use the data we have given it to draw stuff to the screen. One way in which we could draw two quads with different positions would be to use multiple draw calls, moving our "camera" using an MVP Matrix in between like so (this code is incomplete but represents the idea):

```java
mvpmatrix.translate(0.5, 0);
int mat4location = GL30.glGetUniformLocation(programID, "u_MVP");
GL30.glUniformMatrix4fv(mat4location, false, mvpmatrix);

GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0);

mvpmatrix.translate(-0.5, 0);
int mat4location = GL30.glGetUniformLocation(programID, "u_MVP");
GL30.glUniformMatrix4fv(mat4location, false, mvpmatrix);

GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0);
```

This would in fact work. When we run such a program we should see two quads on our screen with different positions. What we are in fact actually doing is drawing the same quad twice, at two different camera positions. This works fine and all but I'm sure you can already think of some of the major limitations of this approach. For example we would never be able to color the two quads differently as they both are using the same shaders. Not only that, but using multiple Draw Calls is quite intensive on the GPU when you start to draw much more things. Therefore, it is best practicec to batch as much as possible together into a single draw call.

## Batch Rendering

To demonstrate how to implement batch rendering we are going to start a new project in which we render four quads together in one draw call with different positions each and also different colors. Just like in the Mandelbrot Tutorials I am going to be using [LWJGL](https://www.lwjgl.org/) for my OpenGL bindings and therefore programming in Java. If you wish to follow along directly with this tutorial I wrote another on how to setup a project exactly like mine [here](https://cianjinks.github.io/posts/tutorial-using-maven-and-intellij-for-opengl-projects/). To get started I will create a file called `Application.java` and paste in the starter code from LWJGL's [starter page](https://www.lwjgl.org/guide). If you're wondering what this code does I once again already discussed it in part one of the Mandelbrot Tutorial Series.

(Before writing any new OpenGL code I will simply set a few variables such as the window resolution to 480x480 as well as title to "Batch Rendering Tutorial")

To start we are going to define the positions of the vertices of our four quads in a simple float array:

```java
float[] vertices = new float[] {
  // Quad 1 - Top Left
  -0.75f,  0.75f,
  -0.75f,  0.25f,
  -0.25f,  0.25f,
  -0.25f,  0.75f,

  // Quad 2 - Bottom Left
  -0.75f, -0.25f,
  -0.75f, -0.75f,
  -0.25f, -0.75f,
  -0.25f, -0.25f,

  // Quad 3 - Bottom Right
  0.25f, -0.25f,
  0.25f, -0.75f,
  0.75f, -0.75f,
  0.75f, -0.25f,

  // Quad 4 - Top Right
  0.25f, 0.75f,
  0.25f, 0.25f,
  0.75f, 0.25f,
  0.75f, 0.75f
};
```

We then need to setup a VBO just like normal:

```java
// VBO (Vertex Buffer Object)
FloatBuffer vboBuffer = BufferUtils.createFloatBuffer(vertices.length);
for(float vertex : vertices) {
  vboBuffer.put(vertex);
}
vboBuffer.flip();

// Pass data to GPU
int positionElementCount = vertices.length / 16;
int vboID = GL30.glGenBuffers();
GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
GL30.glBufferData(GL30.GL_ARRAY_BUFFER, vboBuffer, GL30.GL_STATIC_DRAW);
GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, positionElementCount * Float.BYTES, 0);
GL30.glEnableVertexAttribArray(0);
```
For our IBO we are going to need to specify indices for all four quads in our integer array like so:

```java
// Indices
int[] indices = new int[] {
  // Quad 1
  0, 1, 2,
  2, 3, 0,

  // Quad 2
  4, 5, 6,
  6, 7, 4,

  // Quad 3
  8, 9, 10,
  10, 11, 8,

  // Quad 4
  12, 13, 14,
  14, 15, 12
};
```

And then pass them to the GPU using an IBO same as normal again:

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

Just to be sure that there is a default shader present we will create one ourselves:

**Application.java**
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

**/shaders/vert.shader**
```glsl
#version 410 core

layout(location = 0) in vec2 position;

void main() {
    gl_Position = vec4(position, 0.0f, 1.0f);
}
```

**/shaders/frag.shader**
```glsl
#version 410 core

void main() {
    gl_FragColor = vec4(0.5f, 1.0f, 0.5, 1.0f);
}
```

This should give our quads a green tint. We can now add a draw call to our program and go ahead and run it:

```java
GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0);
```

![OpenGL_Tutorial_Batch_Rendering_1.png](OpenGL_Tutorial_Batch_Rendering_1.png)

That's it! We've batched together four separate quads into the one draw call. Each has their own unique vertex position attributes laid out in our `vertices` array and unique indices in our `indices` array. However, there are still quite a few problems with this implementation. How do we color the quads differently? They currently all use the same shader. How do we move the quads after their data has been sent to the GPU using a VBO? Our VBO is still specified as `GL_STATIC_DRAW`. That's what we are going to cover next to make Batch Rendering work just the same as using multiple draw calls.

Here is the source code for `Application.java` which we have written so far:

```java
import org.lwjgl.*;
import org.lwjgl.glfw.*;
import org.lwjgl.opengl.*;
import org.lwjgl.system.*;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.nio.*;

import static org.lwjgl.glfw.Callbacks.*;
import static org.lwjgl.glfw.GLFW.*;
import static org.lwjgl.opengl.GL11.*;
import static org.lwjgl.system.MemoryStack.*;
import static org.lwjgl.system.MemoryUtil.*;

public class Application {

  // The window handle
  private long window;
  private static final int WINDOW_WIDTH = 960 / 2;
  private static final int WINDOW_HEIGHT = 960 / 2;

  public void run() {

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
    window = glfwCreateWindow(WINDOW_WIDTH, WINDOW_HEIGHT, "Batch Rendering Tutorial", NULL, NULL);
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

    float[] vertices = new float[] {
      // Quad 1 - Top Left
      -0.75f,  0.75f,
      -0.75f,  0.25f,
      -0.25f,  0.25f,
      -0.25f,  0.75f,

      // Quad 2 - Bottom Left
      -0.75f, -0.25f,
      -0.75f, -0.75f,
      -0.25f, -0.75f,
      -0.25f, -0.25f,

      // Quad 3 - Bottom Right
      0.25f, -0.25f,
      0.25f, -0.75f,
      0.75f, -0.75f,
      0.75f, -0.25f,

      // Quad 4 - Top Right
      0.25f, 0.75f,
      0.25f, 0.25f,
      0.75f, 0.25f,
      0.75f, 0.75f
      };

    // VBO (Vertex Buffer Object)
    FloatBuffer vboBuffer = BufferUtils.createFloatBuffer(vertices.length);
    for(float vertex : vertices) {
      vboBuffer.put(vertex);
    }
    vboBuffer.flip();

    // Pass data to GPU
    int positionElementCount = vertices.length / 16;
    int vboID = GL30.glGenBuffers();
    GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
    GL30.glBufferData(GL30.GL_ARRAY_BUFFER, vboBuffer, GL30.GL_STATIC_DRAW);
    GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, positionElementCount * Float.BYTES, 0);
    GL30.glEnableVertexAttribArray(0);

    // Indices
    int[] indices = new int[] {
      // Quad 1
      0, 1, 2,
      2, 3, 0,

      // Quad 2
      4, 5, 6,
      6, 7, 4,

      // Quad 3
      8, 9, 10,
      10, 11, 8,

      // Quad 4
      12, 13, 14,
      14, 15, 12
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

      GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0); //Draw our squares

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

  public static void main(String[] args) {
    new Application().run();
  }

}
```

## Batch Rendering - Colors

Let's start by coloring our quads differently. You may remember in my earlier tutorial I discussed how vertices can contain all kinds of vertex attributes, not just position. This is where we can make use of that. We can store the color of each vertex as an attribute within our `vertices` array and then tell the GPU the position, size, etc of this attribute. Here is what our new vertices array will be:

```java
float[] vertices = new float[] {
  // Quad 1 - Top Left (Black)
  -0.75f,  0.75f, 1.0f, 1.0f, 1.0f,
  -0.75f,  0.25f, 1.0f, 1.0f, 1.0f,
  -0.25f,  0.25f, 1.0f, 1.0f, 1.0f,
  -0.25f,  0.75f, 1.0f, 1.0f, 1.0f,

  // Quad 2 - Bottom Left (White)
  -0.75f, -0.25f, 0.0f, 0.0f, 0.0f,
  -0.75f, -0.75f, 0.0f, 0.0f, 0.0f,
  -0.25f, -0.75f, 0.0f, 0.0f, 0.0f,
  -0.25f, -0.25f, 0.0f, 0.0f, 0.0f,

  // Quad 3 - Bottom Right (Green)
  0.25f, -0.25f, 0.0f, 1.0f, 0.0f,
  0.25f, -0.75f, 0.0f, 1.0f, 0.0f,
  0.75f, -0.75f, 0.0f, 1.0f, 0.0f,
  0.75f, -0.25f, 0.0f, 1.0f, 0.0f,

  // Quad 4 - Top Right (Blue)
  0.25f, 0.75f, 0.0f, 0.0f, 1.0f,
  0.25f, 0.25f, 0.0f, 0.0f, 1.0f,
  0.75f, 0.25f, 0.0f, 0.0f, 1.0f,
  0.75f, 0.75f, 0.0f, 0.0f, 1.0f
};
```

Now the composition of each quad looks like this:

![OpenGL_Tutorial_Batch_Rendering_2.png](OpenGL_Tutorial_Batch_Rendering_2.png)

The GPU still does not know that we have placed this data there and presumes every two floats is position data so we are going to need to change this in our VBO by adding a new vertex attribute pointer using `glVertexAttribPointer`:

```java
// Pass data to GPU
int positionElementCount = 2;
int colorElementCount = 3;
int stride = (positionElementCount * Float.BYTES) + (colorElementCount * Float.BYTES);
int vboID = GL30.glGenBuffers();
GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
GL30.glBufferData(GL30.GL_ARRAY_BUFFER, vboBuffer, GL30.GL_STATIC_DRAW);
GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, stride, 0);
GL30.glVertexAttribPointer(1, colorElementCount, GL_FLOAT, false, stride, positionElementCount * Float.BYTES);
GL30.glEnableVertexAttribArray(0);
GL30.glEnableVertexAttribArray(1);
```

There are only a few slight differences this time. Firstly, we need to provide a new value for the stride for both of our vertex attributes. The stride is simply how many bytes one vertex takes up so we calculate it above. Then finally the last parameter has been changed. This parameter is an offset into our vertices data, this offset being the beginning of that vertex attribute. Our color attribute begins two floats in from the start (after the first position attribute) so we set it to two floats worth of bytes.

We can now go ahead and use this new color data within our shaders. First we take in the data inside of our vertex shader in the same way we did for the position data. Then we create a varying to pass the color data over to the fragment shader where we can actually apply it to our quad:

```glsl
#version 410 core

layout(location = 0) in vec2 position;
layout(location = 1) in vec3 color;

out vec3 v_Color;

void main() {
    gl_Position = vec4(position, 0.0f, 1.0f);
    v_Color = color;
}
````

(Note the color data is 3 floats long so it is taken is as a vec3)

Now we can go ahead and use this data in our fragment shader to color the quad using `gl_FragColor`:

```glsl
#version 410 core

in vec3 v_Color;

void main() {
    gl_FragColor = vec4(v_Color, 1.0f);
}
```

If everything is working correctly we should now be able to run our code and see the following output:

![OpenGL_Tutorial_Batch_Rendering_3.png](OpenGL_Tutorial_Batch_Rendering_3.png)

As you can see we have now successfully batch rendered four separate quads with different colors for each. The final challenge we may run into is moving the quads independent of one another (ie, without using an MVP Matrix as this will move all of them together). For this we need some way in which to update our VBO while the programming is running.

## Dynamic VBOs

As mentioned a few times previously when we passed data into a Vertex Buffer Object we told it that this data would never change using `GL_STATIC_DRAW`. This meant we could never update the vertex positions to move around our quad. However, now that we have implemented Batch Rendering we know why it is necessary to be able to do exactly that. Therefore we can create a Dynamic VBO using `GL_DYNAMIC_DRAW`. To demonstrate this we are going to make a new quad get added to the screen while the program is running and also modify the position of one of the other quads.

First let's go ahead and make our VBO dynamic. Insead of passing data directly to `glBufferData` we can  instead allocate the amount of memory we are going to use. Here we do that by allocating the size of our `vboBuffer` FloatBuffer in bytes:

```java
// Pass data to GPU
int positionElementCount = 2;
int colorElementCount = 3;
int stride = (positionElementCount * Float.BYTES) + (colorElementCount * Float.BYTES);
int vboID = GL30.glGenBuffers();
GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
GL30.glBufferData(GL30.GL_ARRAY_BUFFER, vboBuffer.capacity() * Float.BYTES, GL30.GL_DYNAMIC_DRAW);
GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, stride, 0);
GL30.glVertexAttribPointer(1, colorElementCount, GL_FLOAT, false, stride, positionElementCount * Float.BYTES);
GL30.glEnableVertexAttribArray(0);
GL30.glEnableVertexAttribArray(1);
```

To test if our buffer is now dynamic we are going to create a keybind to move the position of our top left quad upwards. First we will create an instance variable called `aKey` and then update it within the `glfwSetKeyCallback`:

```java
private boolean aKey = false;
```

```java
glfwSetKeyCallback(window, (window, key, scancode, action, mods) -> {
  if ( key == GLFW_KEY_ESCAPE && action == GLFW_RELEASE ) {
    glfwSetWindowShouldClose(window, true); // We will detect this in the rendering loop
  }

  if ( key == GLFW_KEY_A && action == GLFW_PRESS) {
    aKey = true;
  }
  if ( key == GLFW_KEY_A && action == GLFW_RELEASE) {
    aKey = false;
  }
});
```

Now we can poll for this key press within our main rendering loop and update the top left quad's position using a function called `glBufferSubData`. This function is commonly used with dynamic VBOs as it simply replaces the data in the buffer instead of recreating the buffer entirely like `glBufferData` would. (Note the indices in our vertices `ArrayList` for this quad's y values are 1, 6, 11 and 16):

```java
while ( !glfwWindowShouldClose(window) ) {
  glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

  // If A Key is held down
  if(aKey) {
    vboBuffer.put(1, vboBuffer.get(1) + 0.02f);
    vboBuffer.put(6, vboBuffer.get(6) + 0.02f);
    vboBuffer.put(11, vboBuffer.get(11) + 0.02f);
    vboBuffer.put(16, vboBuffer.get(16) + 0.02f);
  }

  GL30.glBufferSubData(GL30.GL_ARRAY_BUFFER, 0, vboBuffer);

  GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0); //Draw our squares

  glfwSwapBuffers(window); // swap the color buffers
  // Poll for window events. The key callback above will only be
  // invoked during this call.
  glfwPollEvents();
}
```

That should be it, we can now go ahead and run our program and hold down the A key. If everything has worked correctly the quad should slide off the screen as we update its position every frame:

![OpenGL_Tutorial_Batch_Rendering_4.gif](OpenGL_Tutorial_Batch_Rendering_4.gif)

Finally we can try adding a fifth quad using a timer to demonstrate increasing the size of our dynamic buffer on the fly. First let's define our quad in a separate float array outside of our main render loop:

```java
float[] quad5 = new float[] {
  // Quad 5 - Middle (Orange)
  -0.25f,  0.25f, 1.0f, 0.65f, 0.0f,
  -0.25f, -0.25f, 1.0f, 0.65f, 0.0f,
   0.25f, -0.25f, 1.0f, 0.65f, 0.0f,
   0.25f,  0.25f, 1.0f, 0.65f, 0.0f
  };
```

We are also going to need to add some indices for this quad to our original indices array:

```java
// Indices
int[] indices = new int[] {
  // Quad 1
  0, 1, 2,
  2, 3, 0,

  // Quad 2
  4, 5, 6,
  6, 7, 4,

  // Quad 3
  8, 9, 10,
  10, 11, 8,

  // Quad 4
  12, 13, 14,
  14, 15, 12,

  // Quad 5
  16, 17, 18,
  18, 19, 16
};
```

One final change we need to make before writing the code that displays the quad is increasing the allocated memory of our VBO on the GPU and our `FloatBuffer`:

```java
// VBO (Vertex Buffer Object)
FloatBuffer vboBuffer = BufferUtils.createFloatBuffer(vertices.length + quad5.length);
for(float vertex : vertices) {
  vboBuffer.put(vertex);
}
vboBuffer.flip();

// Pass data to GPU
int positionElementCount = 2;
int colorElementCount = 3;
int stride = (positionElementCount * Float.BYTES) + (colorElementCount * Float.BYTES);
int vboID = GL30.glGenBuffers();
GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
GL30.glBufferData(GL30.GL_ARRAY_BUFFER, (vboBuffer.capacity() * Float.BYTES) + (quad5.length * Float.BYTES), GL30.GL_DYNAMIC_DRAW);
GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, stride, 0);
GL30.glVertexAttribPointer(1, colorElementCount, GL_FLOAT, false, stride, positionElementCount * Float.BYTES);
GL30.glEnableVertexAttribArray(0);
GL30.glEnableVertexAttribArray(1);
```

Here you can see I simply made use of `quad5.length` when creating the `FloatBuffer` as well as when creating the `GL_ARRAY_BUFFER`. Now we can finally add some code to our main render loop to add the square:

```java
int timer = 0;
// Run the rendering loop until the user has attempted to close
// the window or has pressed the ESCAPE key.
while ( !glfwWindowShouldClose(window) ) {
  glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

  // If A Key is held down
  if(aKey) {
    vboBuffer.put(1, vboBuffer.get(1) + 0.02f);
    vboBuffer.put(6, vboBuffer.get(6) + 0.02f);
    vboBuffer.put(11, vboBuffer.get(11) + 0.02f);
    vboBuffer.put(16, vboBuffer.get(16) + 0.02f);
  }

  timer++;
  // After 200 frames
  if(timer == 200) {
    // Regenerate our vboBuffer with the new quad
    vboBuffer.clear();
    for(float vertex : vertices) {
      vboBuffer.put(vertex);
    }
    for(float vertex : quad5) {
      vboBuffer.put(vertex);
    }
    vboBuffer.flip();
  }

  GL30.glBufferSubData(GL30.GL_ARRAY_BUFFER, 0, vboBuffer);

  GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0); //Draw our squares

  glfwSwapBuffers(window); // swap the color buffers
  // Poll for window events. The key callback above will only be
  // invoked during this call.
  glfwPollEvents();
}
```

Here I use a timer to add the quad after 200 frames. Due to the way Java's Buffers work we end up having to recreate our `vboBuffer` after the timer has completed, adding the new quad to it. If all is working correctly the quad will appear shortly after running the program:

![OpenGL_Tutorial_Batch_Rendering_5.gif](OpenGL_Tutorial_Batch_Rendering_5.gif)

From all of this it might be apparent that for larger projects it would be best to abstract the idea of a quad and vertex into their own classes and generate the `vboBuffer` using these to make it much easier to create a quad and add it to the screen. But for now this way of doing things works for showing off the general concept of Batch Rendering and using Dynamic VBOs. Here is the entirety of the source code so far:

**Application.java**
```java
import org.lwjgl.*;
import org.lwjgl.glfw.*;
import org.lwjgl.opengl.*;
import org.lwjgl.system.*;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.nio.*;
import java.util.ArrayList;
import java.util.Collections;

import static org.lwjgl.glfw.Callbacks.*;
import static org.lwjgl.glfw.GLFW.*;
import static org.lwjgl.opengl.GL11.*;
import static org.lwjgl.system.MemoryStack.*;
import static org.lwjgl.system.MemoryUtil.*;

public class Application {

  // The window handle
  private long window;
  private static final int WINDOW_WIDTH = 960 / 2;
  private static final int WINDOW_HEIGHT = 960 / 2;

  private boolean aKey = false;

  public void run() {

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
    window = glfwCreateWindow(WINDOW_WIDTH, WINDOW_HEIGHT, "Batch Rendering Tutorial", NULL, NULL);
    if ( window == NULL )
      throw new RuntimeException("Failed to create the GLFW window");

    // Setup a key callback. It will be called every time a key is pressed, repeated or released.
    glfwSetKeyCallback(window, (window, key, scancode, action, mods) -> {
      if ( key == GLFW_KEY_ESCAPE && action == GLFW_RELEASE ) {
        glfwSetWindowShouldClose(window, true); // We will detect this in the rendering loop
      }

      if ( key == GLFW_KEY_A && action == GLFW_PRESS) {
        aKey = true;
      }
      if ( key == GLFW_KEY_A && action == GLFW_RELEASE) {
        aKey = false;
      }
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

    float[] vertices = new float[] {
      // Quad 1 - Top Left (Black)
      -0.75f,  0.75f, 1.0f, 1.0f, 1.0f,
      -0.75f,  0.25f, 1.0f, 1.0f, 1.0f,
      -0.25f,  0.25f, 1.0f, 1.0f, 1.0f,
      -0.25f,  0.75f, 1.0f, 1.0f, 1.0f,

      // Quad 2 - Bottom Left (White)
      -0.75f, -0.25f, 0.0f, 0.0f, 0.0f,
      -0.75f, -0.75f, 0.0f, 0.0f, 0.0f,
      -0.25f, -0.75f, 0.0f, 0.0f, 0.0f,
      -0.25f, -0.25f, 0.0f, 0.0f, 0.0f,

       // Quad 3 - Bottom Right (Green)
       0.25f, -0.25f, 0.0f, 1.0f, 0.0f,
       0.25f, -0.75f, 0.0f, 1.0f, 0.0f,
       0.75f, -0.75f, 0.0f, 1.0f, 0.0f,
       0.75f, -0.25f, 0.0f, 1.0f, 0.0f,

       // Quad 4 - Top Right (Blue)
       0.25f, 0.75f, 0.0f, 0.0f, 1.0f,
       0.25f, 0.25f, 0.0f, 0.0f, 1.0f,
       0.75f, 0.25f, 0.0f, 0.0f, 1.0f,
       0.75f, 0.75f, 0.0f, 0.0f, 1.0f
    };

    float[] quad5 = new float[] {
       // Quad 5 - Middle (Orange)
       -0.25f,  0.25f, 1.0f, 0.65f, 0.0f,
       -0.25f, -0.25f, 1.0f, 0.65f, 0.0f,
        0.25f, -0.25f, 1.0f, 0.65f, 0.0f,
        0.25f,  0.25f, 1.0f, 0.65f, 0.0f
    };

    // VBO (Vertex Buffer Object)
    FloatBuffer vboBuffer = BufferUtils.createFloatBuffer(vertices.length + quad5.length);
    for(float vertex : vertices) {
      vboBuffer.put(vertex);
    }
    vboBuffer.flip();

    // Pass data to GPU
    int positionElementCount = 2;
    int colorElementCount = 3;
    int stride = (positionElementCount * Float.BYTES) + (colorElementCount * Float.BYTES);
    int vboID = GL30.glGenBuffers();
    GL30.glBindBuffer(GL30.GL_ARRAY_BUFFER, vboID);
    GL30.glBufferData(GL30.GL_ARRAY_BUFFER, (vboBuffer.capacity() * Float.BYTES) + (quad5.length * Float.BYTES), GL30.GL_DYNAMIC_DRAW);
    GL30.glVertexAttribPointer(0, positionElementCount, GL_FLOAT, false, stride, 0);
    GL30.glVertexAttribPointer(1, colorElementCount, GL_FLOAT, false, stride, positionElementCount * Float.BYTES);
    GL30.glEnableVertexAttribArray(0);
    GL30.glEnableVertexAttribArray(1);

    // Indices
    int[] indices = new int[] {
      // Quad 1
      0, 1, 2,
      2, 3, 0,

      // Quad 2
      4, 5, 6,
      6, 7, 4,

      // Quad 3
      8, 9, 10,
      10, 11, 8,

      // Quad 4
      12, 13, 14,
      14, 15, 12,

      // Quad 5
      16, 17, 18,
      18, 19, 16
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


    int timer = 0;
    // Run the rendering loop until the user has attempted to close
    // the window or has pressed the ESCAPE key.
    while ( !glfwWindowShouldClose(window) ) {
      glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

      // If A Key is held down
      if(aKey) {
        vboBuffer.put(1, vboBuffer.get(1) + 0.02f);
        vboBuffer.put(6, vboBuffer.get(6) + 0.02f);
        vboBuffer.put(11, vboBuffer.get(11) + 0.02f);
        vboBuffer.put(16, vboBuffer.get(16) + 0.02f);
      }

      timer++;
      // After 200 frames
      if(timer == 200) {
        // Regenerate our vboBuffer with the new quad
        vboBuffer.clear();
        for(float vertex : vertices) {
          vboBuffer.put(vertex);
        }
        for(float vertex : quad5) {
          vboBuffer.put(vertex);
        }
        vboBuffer.flip();
      }

      GL30.glBufferSubData(GL30.GL_ARRAY_BUFFER, 0, vboBuffer);

      GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0); //Draw our squares

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

  public static void main(String[] args) {
    new Application().run();
  }

}
```

**/shaders/vert.shader**
```glsl
#version 410 core

layout(location = 0) in vec2 position;
layout(location = 1) in vec3 color;

out vec3 v_Color;

void main() {
    gl_Position = vec4(position, 0.0f, 1.0f);
    v_Color = color;
}
```

**/shaders/frag.shader**
```glsl
#version 410 core

in vec3 v_Color;

void main() {
    gl_FragColor = vec4(v_Color, 1.0f);
}
```

## Dynamic IBOs

When we added the fifth quad you might have wondered how we could make our IBO dynamic as we simply hardcoded it up above. Doing so is the exact same as for the VBO, maybe even slighter simpler. Firstly we will make a loop using some simple maths to generate our indices for us:

```java
// Indices
int indicesPerQuad = 6;
int quads = 5;
int[] indices = new int[indicesPerQuad * quads];
int offset = 0;
for(int i = 0; i < indices.length; i += 6) {
  indices[i + 0] = 0 + offset;
  indices[i + 1] = 1 + offset;
  indices[i + 2] = 2 + offset;

  indices[i + 3] = 2 + offset;
  indices[i + 4] = 3 + offset;
  indices[i + 5] = 0 + offset;

  offset += 4;
}

// IBO (Index Buffer Object)
IntBuffer iboBuffer = BufferUtils.createIntBuffer(indicesPerQuad * quads);
for(int index : indices) {
  iboBuffer.put(index);
}
iboBuffer.flip();
```

Then we make our IBO dynamic using `GL_DYNAMIC_DRAW` and allocate memory for our indices:

```java
// Pass data to GPU
int iboID = GL30.glGenBuffers();
GL30.glBindBuffer(GL30.GL_ELEMENT_ARRAY_BUFFER, iboID);
GL30.glBufferData(GL30.GL_ELEMENT_ARRAY_BUFFER, iboBuffer.capacity() * Float.BYTES, GL30.GL_DYNAMIC_DRAW);
```

Now in our main render loop, just like before we pass in our indices using `glBufferSubData`:

```java
int timer = 0;
// Run the rendering loop until the user has attempted to close
// the window or has pressed the ESCAPE key.
while ( !glfwWindowShouldClose(window) ) {
  glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

  // If A Key is held down
  if(aKey) {
    vboBuffer.put(1, vboBuffer.get(1) + 0.02f);
    vboBuffer.put(6, vboBuffer.get(6) + 0.02f);
    vboBuffer.put(11, vboBuffer.get(11) + 0.02f);
    vboBuffer.put(16, vboBuffer.get(16) + 0.02f);
  }

  timer++;
  // After 200 frames
  if(timer == 200) {
    // Regenerate our vboBuffer with the new quad
    vboBuffer.clear();
    for(float vertex : vertices) {
      vboBuffer.put(vertex);
    }
    for(float vertex : quad5) {
      vboBuffer.put(vertex);
    }
    vboBuffer.flip();
  }

  GL30.glBufferSubData(GL30.GL_ARRAY_BUFFER, 0, vboBuffer);
  GL30.glBufferSubData(GL30.GL_ELEMENT_ARRAY_BUFFER, 0, iboBuffer);

  GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0); //Draw our squares

  glfwSwapBuffers(window); // swap the color buffers
  // Poll for window events. The key callback above will only be
  // invoked during this call.
  glfwPollEvents();
}
```

With that our program is complete! It will work the exact same as before except this time we could add more indices on the fly if necessary too, just like for our vertices.

## Conclusion

Thanks so much for reading this tutorial, I hoped it helped you out somewhat and if you could please leave some feedback in the comments I'd really appreciate it :D
I recently completed a project using in which I used Batch Rendering and Dynamic VBOs and IBOs to visualize sorting algorithms. If you want to check that out to see how these techniques can be applied, and also abstracted, it can be found on my github [here](https://github.com/cianjinks/Sorting-Visualizer).
