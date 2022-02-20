---
title: OpenGL Tutorial - Visualizing the Mandelbrot Set Fractal - Part 2 of 2
date: 2020-05-31 12:00:00 +0000
categories: [Old]
tags: [opengl, lwjgl, fractal, tutorial]
img_path: /assets/img/post/OpenGLMandelbrot/
---
This is part 2 of my tutorial series on Visualizing the Mandelbrot Set Fractal using OpenGL. This part will cover the creation of a camera for zooming in and out and moving around as well as the fragment shader responsible for generating the fractal. If you are interested in how to draw a square to the screen with shaders applied to it, check out [part one](https://cianjinks.github.io/posts/opengl-tutorial-visualizing-the-mandelbrot-set-fractal-part-1-of-2/). This part is going to continue from where we left off in part one (a simple square colored purple using shaders).

## The Camera

### MVP Matrix

The first thing to learn when it comes to creating our camera is the basics of what an MVP Matrix or Model View Projection Matrix is. When you have a 3D space or game and want to implement a camera you are going to need some way to convert what the camera sees to a 2D screen. This is where the MVP Matrix comes into play. Using some matrix multiplication and tricks one can easily convert rendered scenes into 2D screen space. An MVP Matrix is actually made up of three matrices multiplied together. Model * View * Projection. In this tutorial we are going to only be dealing with the Projection Matrix of the MVP Matrix as that is all that we need. If you do wish to research further (such as for a 3D project) [here](http://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/) is a great tutorial on the topic.

The Projection Matrix is the last part in the process of converting a 3D object to 2D screen space coordinates. However, because all we have is simply a 2D scene we don't need to deal with converting our object to 2D coordinates as it already is. Therefore we can simply use the Projection Matrix to "move" our object around. In reality, what we are really doing is moving our _camera_ that looks at our object around. We are not modifying any of the vertex position attributes of our square. We can also make the object appear closer or further away from the screen by scaling our cameras view. With that all in mind lets begin creating our Projection Matrix.

You may remember back at the start of part 1 we added the JOML addon to our LWJGL configuration. This addon is a great Java math library designed specifically for OpenGL which will provide us with many classes and functions for handling matrices and matrix math. In fact, it even provides some handy functions for generating our projection matrix for us:

```java
// Projection Matrix
private Matrix4f pmatrix = new Matrix4f().ortho(-2.0f, 2.0f, -2.0f, 2.0f, -1.0f, 1.0f);
```

Much of this code is based off of JOML's [README](https://github.com/JOML-CI/JOML). The first thing we do is create a new 4\*4 square matrix filled with floats. This is going to be our projection matrix. We then apply an orthographic projection to it using `ortho()`. The parameters for this are going to be our camera views bounds. The first -2.0f and 2.0f correspond to the horizontal coordinates of our screen. If you remember from part one the default OpenGL coordinates range from -1.0f to 1.0f. In our case this change is simply making it range from -2.0f to 2.0f. The next -2.0f and 2.0f correspond to the same thing except vertically. Finally the -1.0f, 1.0f is for the z-direction. This doesn't necessarily matter to us as we aren't going to be moving anything in that direction so I simply set it to the OpenGL default. Note that I am declaring this as an instance variable at the top of our program as it is going to need to be accessible by all areas of our program so that we can modify it using user input.

Before that, however, we should first try applying it to our square. But how do we do that? What we want to do is multiply this matrix by our gl_Position in our fragment shader. Therefore we need some way to pass this matrix into our shader. The passing of information from the CPU to shaders is done via something called uniforms.

### Uniforms

To create a uniform on the cpu side we are going to need to two simple OpenGL functions:

```java
// Camera
try (MemoryStack stack = MemoryStack.stackPush()) {
  FloatBuffer projection = pmatrix.get(stack.mallocFloat(4 * 4));
  int mat4location = GL30.glGetUniformLocation(programID, "u_MVP");
  GL30.glUniformMatrix4fv(mat4location, false, projection);
}
```

Note that this example uses correct LWJGL memory management practices as mentioned on the JOML [README](https://github.com/JOML-CI/JOML) by first placing our matrix in a FloatBuffer before passing it as the data for our uniform. If you wish to read the documentation for a `Matrix4f` it can be found [here](https://joml-ci.github.io/JOML/apidocs/org/joml/Matrix4f.html).

The first of these OpenGL functions is `glGetUniformLocation`. This takes in the shader's program ID which already exists in our program as well what you wish to call this uniform's variable. In my case I like to use the convention of "u_Something" when naming uniforms. This function then returns us an Integer which we can use to reference this uniform later on. Next we use `glUniformMatrix4fv` and pass it this Integer. This tells OpenGL we want this uniform to be a 4\*4 square matrix of floats. After that we pass it `false` so that it does not transpose the matrix (this can be necessary as other graphics apis like DirectX use different matrix ordering in memory) and finally we pass in our FloatBuffer which contains our projection matrix. This code should be placed in the main loop of the program, as we will want to be able to update the matrix to move the camera around every frame. If you want to learn more about any of these OpenGL functions, just like in [part one](https://cianjinks.github.io/posts/opengl-tutorial-visualizing-the-mandelbrot-set-fractal-part-1-of-2/), I highly recommend checking out [docs.gl](http://docs.gl/).

Finally we are going to head on over to our Vertex Shader called `vert.shader` and add in a small bit of extra code to make use of this projection matrix:

```glsl
layout(location = 0) in vec2 position;

uniform mat4 u_MVP;

void main() {
    gl_Position = u_MVP * vec4(position, 0, 1.0);
}
```

As you can see, at the top we take in the uniform using its variable name we chose above and then multiply it by the position of each of our vertices before passing them to `gl_Positon`.

If we now go ahead and run our program it should produce the following window:

![OpenGL_Tutorial_Mandelbrot_6.png](OpenGL_Tutorial_Mandelbrot_6.png)

As you can see, we have now changed the window's scale from its original -1.0f to 1.0f to -2.0f to 2.0f. However, we have left the coordinates of our squares vertices the same and so now it only takes up a quarter of the screen in the center. The reasoning for making our screen coordinates range from -2.0f to 2.0f on both axises will become apparent once we start discussing the Mandelbrot Set calculations.

Here is how our `Application.java` should look now:

```java
import org.joml.Matrix4f;
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

    // Projection Matrix
    private Matrix4f pmatrix = new Matrix4f().ortho(-2.0f, 2.0f, -2.0f, 2.0f, -1.0f, 1.0f);

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
        window = glfwCreateWindow(960 / 2, 960 / 2, "Hello World!", NULL, NULL);
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

            // Camera
            try (MemoryStack stack = MemoryStack.stackPush()) {
                FloatBuffer projection = pmatrix.get(stack.mallocFloat(4 * 4));
                int mat4location = GL30.glGetUniformLocation(programID, "u_MVP");
                GL30.glUniformMatrix4fv(mat4location, false, projection);
            }

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

    public static void main(String[] args) {
        new Application().run();
    }

}

```

### Panning the Camera

To pan our camera around we are going to start taking some user input. Inside of our `init` function we can declare some key call backs using glfw, in fact one is already defined for us to close the window when pressing escape. If you check out glfw's [input guide](https://www.glfw.org/docs/3.3/input_guide.html) you'll note that the options available for `action` are `GLFW_PRESS`, `GLFW_RELEASE` and `GLFW_REPEAT`. As it turns out none of these allow for the pressing and holding of a key to move our camera around. Instead we are going to set up some booleans which are true only when the key is pressed, and then are false when it gets released. We can then poll these booleans within our main loop to move our camera around. Here is a quick example of how we would do so.

We will declare some booleans at the top of our `Application.java`:

```java
// Camera Panning Input
private boolean upArrow = false;
private boolean downArrow = false;
private boolean rightArrow = false;
private boolean leftArrow = false;
```

Then we will modify these booleans in our key callback when their respective key is pressed or released:

```java
glfwSetKeyCallback(window, (window, key, scancode, action, mods) -> {
  if ( key == GLFW_KEY_ESCAPE && action == GLFW_RELEASE ) {
    glfwSetWindowShouldClose(window, true); // We will detect this in the rendering loop
  }

  // Up Arrow Key
  if (key == GLFW_KEY_UP && action == GLFW_PRESS) {
    upArrow = true;
  }
  if(key == GLFW_KEY_UP && action == GLFW_RELEASE) {
    upArrow = false;
  }

  // Down Arrow Key
  if (key == GLFW_KEY_DOWN && action == GLFW_PRESS) {
    downArrow = true;
  }
  if(key == GLFW_KEY_DOWN && action == GLFW_RELEASE) {
    downArrow = false;
  }

  // Left Arrow Key
  if (key == GLFW_KEY_LEFT && action == GLFW_PRESS) {
    leftArrow = true;
  }
  if(key == GLFW_KEY_LEFT && action == GLFW_RELEASE) {
    leftArrow = false;
  }

  // Right Arrow Key
  if (key == GLFW_KEY_RIGHT && action == GLFW_PRESS) {
    rightArrow = true;
  }
  if(key == GLFW_KEY_RIGHT && action == GLFW_RELEASE) {
    rightArrow = false;
  }
});
```

With that all setup its time to move the camera. In our main program loop we will check the value of these booleans every frame and if one is held we will pan our camera in that direction:

```java
float cameraSpeed = 0.0125f;

// Run the rendering loop until the user has attempted to close
// the window or has pressed the ESCAPE key.
while ( !glfwWindowShouldClose(window) ) {
  glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

  // Camera Panning
  if(upArrow) {
    // Translate the projection matrix by the distance cameraSpeed in the positive y direction
    pmatrix.translate(new Vector3f(0.0f, cameraSpeed, 0.0f));
  }
  if(downArrow) {
    // Translate the projection matrix by the distance cameraSpeed in the negative y direction
    pmatrix.translate(new Vector3f(0.0f, -cameraSpeed, 0.0f));
  }
  if(rightArrow) {
    // Translate the projection matrix by the distance cameraSpeed in the positive x direction
    pmatrix.translate(new Vector3f(cameraSpeed, 0.0f, 0.0f));
  }
  if(leftArrow) {
    // Translate the projection matrix by the distance cameraSpeed in the negative x direction
    pmatrix.translate(new Vector3f(-cameraSpeed, 0.0f, 0.0f));
  }

  // Camera
  try (MemoryStack stack = MemoryStack.stackPush()) {
    FloatBuffer projection = pmatrix.get(stack.mallocFloat(4 * 4));
    int mat4location = GL30.glGetUniformLocation(programID, "u_MVP");
    GL30.glUniformMatrix4fv(mat4location, false, projection);
  }

  GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0); //Draw our square

  glfwSwapBuffers(window); // swap the color buffers

  // Poll for window events. The key callback above will only be
  // invoked during this call.
  glfwPollEvents();
}
```

Here we are using JOML's `translate` function to move our projection matrix by a 3 float vector. The 3 floats in this vector represent the x, y and z direction. For example when we press the up arrow key we move the projection matrix by the distance cameraSpeed specified above in the positive y direction.

If you now go ahead and run the program you can pan the camera using the arrow keys and it should be nice and smooth.

### Zooming

Zooming is relatively easy to implement using our projection. Lets first set up some more booleans for polling the zooming keys being held down. This will be just the same as it was for panning.

Here are our instance variables:

```java
// Zooming
private boolean zoomingIn = false;
private boolean zoomingOut = false;
```

Then we update them in the key callback, same as before:

```java
// Zooming In (Z Key)
if (key == GLFW_KEY_Z && action == GLFW_PRESS) {
  zoomingIn = true;
}
if (key == GLFW_KEY_Z && action == GLFW_RELEASE) {
  zoomingIn = false;
}

// Zooming Out (X Key)
if (key == GLFW_KEY_X && action == GLFW_PRESS) {
  zoomingOut = true;
}
if (key == GLFW_KEY_X && action == GLFW_RELEASE) {
  zoomingOut = false;
}
```
Now we can finally use these in our main loop to zoom in and out. Luckily for us, JOML provides quite a few different scaling functions. The one we are going to use is called `scaleLocal`. This scaling function will apply the scaling _after_ any vector translation has been done (ie, after the camera's position is taken into account). This works great for us as it means we can zoom the camera towards the center of it's view no matter where it is positioned.

Here is how we are going to implement this zooming in our main loop:

```java
// Zooming
if(zoomingIn) {
  pmatrix.scaleLocal(1.05f); // Zoom In
} else if(zoomingOut) {
  pmatrix.scaleLocal(1 / 1.05f); // Zoom Out
}
```
If you now go ahead and run the program you'll see you can now zoom in and out after panning the camera and all works smoothly. One problem does remain however. We aren't modifying our panning speed based on how much we are zoomed in. This means after you zoom in a great distance panning feels insanely fast and jumps you across the screen. To counteract this we are going to keep track of how far the camera is zoomed and scale the panning speed based off of this value. Here is how I did that:

```java
float basePanningSpeed = 0.0125f;
float currentPanningSpeed = 0.0125f;
float zoomAmount = 1.0f;
float zoomSpeed = 1.05f;

// Run the rendering loop until the user has attempted to close
// the window or has pressed the ESCAPE key.
while ( !glfwWindowShouldClose(window) ) {
  glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

  // Camera Panning
  if(upArrow) {
    // Translate the projection matrix by the distance cameraSpeed in the positive y direction
    pmatrix.translate(new Vector3f(0.0f, currentPanningSpeed, 0.0f));
  }
  if(downArrow) {
    // Translate the projection matrix by the distance cameraSpeed in the negative y direction
    pmatrix.translate(new Vector3f(0.0f, -currentPanningSpeed, 0.0f));
  }
  if(rightArrow) {
    // Translate the projection matrix by the distance cameraSpeed in the positive x direction
    pmatrix.translate(new Vector3f(currentPanningSpeed, 0.0f, 0.0f));
  }
  if(leftArrow) {
    // Translate the projection matrix by the distance cameraSpeed in the negative x direction
    pmatrix.translate(new Vector3f(-currentPanningSpeed, 0.0f, 0.0f));
  }

  // Zooming
  if(zoomingIn) {
    zoomAmount = zoomAmount * zoomSpeed; // Update the zoomAmount
    currentPanningSpeed = basePanningSpeed / zoomAmount; // Update camera panning speed

    pmatrix.scaleLocal(zoomSpeed); // Zoom In

  } else if(zoomingOut) {
    zoomAmount = zoomAmount * (1 / zoomSpeed); // Update the zoomAmount
    currentPanningSpeed = basePanningSpeed / zoomAmount; // Update camera panning speed

    pmatrix.scaleLocal(1 / zoomSpeed); // Zoom Out

  }

  // Camera
  try (MemoryStack stack = MemoryStack.stackPush()) {
    FloatBuffer projection = pmatrix.get(stack.mallocFloat(4 * 4));
    int mat4location = GL30.glGetUniformLocation(programID, "u_MVP");
    GL30.glUniformMatrix4fv(mat4location, false, projection);
  }

  GL30.glDrawElements(GL30.GL_TRIANGLES, indices.length, GL_UNSIGNED_INT, 0); //Draw our square

  glfwSwapBuffers(window); // swap the color buffers

  // Poll for window events. The key callback above will only be
  // invoked during this call.
  glfwPollEvents();
}
```

What I have done is created a variable called zoomAmount which keeps track of how far in we are currently zoomed. Therefore each time we zoom in I have to not only scale the projection matrix by zoomSpeed but also scale zoomAmount by zoomSpeed likewise. I can then use this zoomAmount to generate a new panningSpeed based off of the basePanningSpeed. With that in place we are done with the camera! For reference here is the entirety of `Application.java` with a working camera implemented:

```java
import org.joml.Matrix4f;
import org.joml.Vector3f;
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

    // Projection Matrix
    private Matrix4f pmatrix = new Matrix4f().ortho(-2.0f, 2.0f, -2.0f, 2.0f, -1.0f, 1.0f);

    // Camera Panning Input
    private boolean upArrow = false;
    private boolean downArrow = false;
    private boolean rightArrow = false;
    private boolean leftArrow = false;

    // Zooming
    private boolean zoomingIn = false;
    private boolean zoomingOut = false;

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
        window = glfwCreateWindow(960 / 2, 960 / 2, "Hello World!", NULL, NULL);
        if ( window == NULL )
            throw new RuntimeException("Failed to create the GLFW window");

        // Setup a key callback. It will be called every time a key is pressed, repeated or released.
        glfwSetKeyCallback(window, (window, key, scancode, action, mods) -> {
            if ( key == GLFW_KEY_ESCAPE && action == GLFW_RELEASE ) {
                glfwSetWindowShouldClose(window, true); // We will detect this in the rendering loop
            }

            // Up Arrow Key
            if (key == GLFW_KEY_UP && action == GLFW_PRESS) {
                upArrow = true;
            }
            if(key == GLFW_KEY_UP && action == GLFW_RELEASE) {
                upArrow = false;
            }

            // Down Arrow Key
            if (key == GLFW_KEY_DOWN && action == GLFW_PRESS) {
                downArrow = true;
            }
            if(key == GLFW_KEY_DOWN && action == GLFW_RELEASE) {
                downArrow = false;
            }

            // Left Arrow Key
            if (key == GLFW_KEY_LEFT && action == GLFW_PRESS) {
                leftArrow = true;
            }
            if(key == GLFW_KEY_LEFT && action == GLFW_RELEASE) {
                leftArrow = false;
            }

            // Right Arrow Key
            if (key == GLFW_KEY_RIGHT && action == GLFW_PRESS) {
                rightArrow = true;
            }
            if(key == GLFW_KEY_RIGHT && action == GLFW_RELEASE) {
                rightArrow = false;
            }

            // Zooming In (Z Key)
            if (key == GLFW_KEY_Z && action == GLFW_PRESS) {
                zoomingIn = true;
            }
            if (key == GLFW_KEY_Z && action == GLFW_RELEASE) {
                zoomingIn = false;
            }

            // Zooming Out (X Key)
            if (key == GLFW_KEY_X && action == GLFW_PRESS) {
                zoomingOut = true;
            }
            if (key == GLFW_KEY_X && action == GLFW_RELEASE) {
                zoomingOut = false;
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

        float basePanningSpeed = 0.0125f;
        float currentPanningSpeed = 0.0125f;
        float zoomAmount = 1.0f;
        float zoomSpeed = 1.05f;

        // Run the rendering loop until the user has attempted to close
        // the window or has pressed the ESCAPE key.
        while ( !glfwWindowShouldClose(window) ) {
            glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

            // Camera Panning
            if(upArrow) {
                // Translate the projection matrix by the distance cameraSpeed in the positive y direction
                pmatrix.translate(new Vector3f(0.0f, currentPanningSpeed, 0.0f));
            }
            if(downArrow) {
                // Translate the projection matrix by the distance cameraSpeed in the negative y direction
                pmatrix.translate(new Vector3f(0.0f, -currentPanningSpeed, 0.0f));
            }
            if(rightArrow) {
                // Translate the projection matrix by the distance cameraSpeed in the positive x direction
                pmatrix.translate(new Vector3f(currentPanningSpeed, 0.0f, 0.0f));
            }
            if(leftArrow) {
                // Translate the projection matrix by the distance cameraSpeed in the negative x direction
                pmatrix.translate(new Vector3f(-currentPanningSpeed, 0.0f, 0.0f));
            }

            // Zooming
            if(zoomingIn) {
                zoomAmount = zoomAmount * zoomSpeed; // Update the zoomAmount
                currentPanningSpeed = basePanningSpeed / zoomAmount; // Update camera panning speed

                pmatrix.scaleLocal(zoomSpeed); // Zoom In

            } else if(zoomingOut) {
                zoomAmount = zoomAmount * (1 / zoomSpeed); // Update the zoomAmount
                currentPanningSpeed = basePanningSpeed / zoomAmount; // Update camera panning speed

                pmatrix.scaleLocal(1 / zoomSpeed); // Zoom Out

            }

            // Camera
            try (MemoryStack stack = MemoryStack.stackPush()) {
                FloatBuffer projection = pmatrix.get(stack.mallocFloat(4 * 4));
                int mat4location = GL30.glGetUniformLocation(programID, "u_MVP");
                GL30.glUniformMatrix4fv(mat4location, false, projection);
            }

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

    public static void main(String[] args) {
        new Application().run();
    }

}
```

## The Mandelbrot Set Fractal

If you aren't already aware of how the Mandelbrot Set works I will attempt to give a brief overview here but I recommend checking out some resources much better at explaining it than me such as [this numberphile video](https://www.youtube.com/watch?v=NGMRB4O922I) or [the wikipedia page](https://en.wikipedia.org/wiki/Mandelbrot_set).

The Mandelbrot Set is a set of complex numbers which when drawn on the complex plane produces a very distinct pattern. This pattern is known as a fractal because you can infinitely zoom into the complex plane and it will produce different patterns as you continue to zoom forever. The Mandelbrot Set is one of the more popular fractals because it is simple enough to understand and produces some incredible patterns when colored using a good coloring function. The main goal of this tutorial series was to create this fractal using a shader in OpenGL and explore the infinite patterns it produces by panning around and zooming in and out.

To get started we first need a tiny bit of maths. I believe wikipedia's explanation is best:

![OpenGL_Tutorial_Mandelbrot_7.png](OpenGL_Tutorial_Mandelbrot_7.png)

Since we are dealing with complex numbers we can simply use the 2D coordinates of our screen as the complex plane. You may remember from part one that our fragment shader is run on every pixel in between the vertices of our square so this is perfect for us. Our 2D coordinates are stored in `vec2`'s in OpenGL making treating them as complex numbers easy. One of the basic properties of the Mandelbrot Set is that it is compact, meaning it is all contained within a radius 2 circle around the origin of the complex plane. Now it should make sense why we made the bounds of our camera range from -2.0 to 2.0. A point is considered to belong to the set if, after a certain number of iterations of the function listed above, the magnitude of the complex number never becomes greater than 2. This number of iterations is something we can control within our code and the more iterations you check for every pixel, the more precise the mandelbrot set image. However, this does come at the cost of performance as there are more calculations being done per pixel.

Hopefully my explanation was somewhat clear but if not I once again recommend the tutorials linked above. Finally we can get to generating the set ourselves. First off we should begin by making our square fill the entire -2.0 to 2.0 screen size by replacing the vertices positions array with the following:

```java
// Vertices - Positions
float[] vertices = new float[] {
  		-2.0f,  2.0f,   // Vertex 0
  		-2.0f, -2.0f,   // Vertex 1
  		 2.0f, -2.0f,   // Vertex 2
  		 2.0f,  2.0f    // Vertex 3
};
```
Now that our square fills the screen we can get to writing our fragment shader.

### The Fragment Shader

To begin writing our fragment shader we are going to need some way to reference the current pixel position the fragment shader is being applied to. To do this we will pass the position from the vertex shader to the fragment shader. The passing of data between shaders is done using something called a varying. Varying's can be declared using the `out` and `in` keyword. Here is how we do this and a quick test to make sure it works:

**Vertex Shader**

```glsl
layout(location = 0) in vec2 position;

uniform mat4 u_MVP;

out vec2 v_Pos;

void main() {
    gl_Position = u_MVP * vec4(position, 0, 1.0);
    v_Pos = position;
}
```

**Fragment Shader**

```glsl
in vec2 v_Pos;

void main() {
    if(v_Pos.x > 0.5) {
        gl_FragColor = vec4(0.3, 0.0, 0.3, 1.0);
    } else {
        gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);
    }
}
```

As you can see I have created a varying called `v_Pos` in the vertex shader (I like to use the convention of a v_Something for varying) and declared it as going `out` from the vertex shader. I then set it equal to the position in the main function. In the fragment shader I take `in` this varying and then use it in a quick test to draw the right quarter of the square as purple and the rest black. If everything works correctly the square will look like this:

![OpenGL_Tutorial_Mandelbrot_8.png](OpenGL_Tutorial_Mandelbrot_8.png)

Now that that is set up we can begin implementing the Mandelbrot Set's function. If you remember the function requires the squaring of an imaginary number. I could not find any built in OpenGL functions for doing this using a `vec2` so I've gone ahead and wrote my own:

```glsl
vec2 squareImaginary(vec2 imaginaryNum) {
    vec2 imaginaryResult;
    imaginaryResult.x = (imaginaryNum.x * imaginaryNum.x) - (imaginaryNum.y * imaginaryNum.y);
    imaginaryResult.y = 2 * imaginaryNum.x * imaginaryNum.y;
    return imaginaryResult;
}
```

(If you want to get better at using GLSL I recommend [this](https://www.youtube.com/watch?v=HIvNePu7UEE&list=PL4neAtv21WOmIrTrkNO3xCyrxg4LKkrF7) video series which I mentioned in [part one](https://cianjinks.github.io/posts/opengl-tutorial-visualizing-the-mandelbrot-set-fractal-part-1-of-2/))

Then in our main line we can use it to generate a black and white version of the Mandelbrot Set:

```glsl
in vec2 v_Pos;

vec2 squareImaginary(vec2 imaginaryNum) {
    vec2 imaginaryResult;
    imaginaryResult.x = (imaginaryNum.x * imaginaryNum.x) - (imaginaryNum.y * imaginaryNum.y);
    imaginaryResult.y = 2 * imaginaryNum.x * imaginaryNum.y;
    return imaginaryResult;
}

void main() {
    vec2 c, z;
    c = v_Pos;      // c starts as the current position
    z = c;          // We can skip one iteration of the function where z = 0 as it is uneccessary if we simply set z = c

    vec3 color = vec3(0.0, 0.0, 0.0); // The default color will be black
    int iterations = 100; // 100 is a good number of iterations
    for(int i = 0; i < iterations; i++) {
        // fc(z) = z^2 + c - Here we iterate over this function until it either becomes greater than 2 or not
        vec2 result = squareImaginary(z) + c;
        if(length(result) > 2.0) {
            // This is a point not in the mandelbrot set - Change color to white
            color = vec3(1.0, 1.0, 1.0);
            break;
        }
        z = result;
    }

    // Color the point
    gl_FragColor = vec4(color, 1.0);
}
```

You can already pan around and zoom in and potentially see some cool patterns but since it is just black and white it may not look too great.

![OpenGL_Tutorial_Mandelbrot_9.png](OpenGL_Tutorial_Mandelbrot_9.png)

### Coloring Function

If you have ever seen videos or pictures of the Mandelbrot Set it likely was colored in some way to make it look much better and make the patterns more impressive. To achieve this effect we are going to use a coloring function. If you wish to learn about lots of various coloring methods and how they work check out [this](https://www.math.univ-toulouse.fr/~cheritat/wiki-draw/index.php/Mandelbrot_set) web page on the topic.

One coloring technique mentioned on that website is "escape time based coloring". With this technique instead of coloring pixels not within the set white if its length becomes >2 we will color the pixel based on the number of iterations it took to become >2. To implement this we simply have to make one small modification to our main function inside the fragment shader:

```glsl
for(int i = 0; i < iterations; i++) {
    // fc(z) = z^2 + c - Here we iterate over this function until it either becomes greater than 2 or not
    vec2 result = squareImaginary(z) + c;
   	if(length(result) > 2.0) {
        // This is a point not in the mandelbrot set - Change color based on iterations
        color = colorFunc(i);
        break;
    }
    z = result;
}
```

Here i is our number of iterations so we are going to pass this value to a special color function which returns a color based on it. There are many ways we could define such a color function but I chose to use HSV values where the Hue and Value are based on the number of iterations. This HSV value is then converted to RGB and returned by the function.

```glsl
vec3 colorFunc(int iter) {
    // Color in HSV - Tweak these values to your liking and for different coloring effects
    vec3 color = vec3(0.012*iter , 1.0, 0.2+.4*(1.0+sin(0.3*iter)));

    // Convert from HSV to RGB
    // Taken from: http://lolengine.net/blog/2013/07/27/rgb-to-hsv-in-glsl
    vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
    vec3 m = abs(fract(color.xxx + K.xyz) * 6.0 - K.www);
    return vec3(color.z * mix(K.xxx, clamp(m - K.xxx, 0.0, 1.0), color.y));
}
```

Our full fragment shader for generating the Mandelbrot Set is now complete! Here is it in its entirety:

```glsl
in vec2 v_Pos;

vec2 squareImaginary(vec2 imaginaryNum) {
    vec2 imaginaryResult;
    imaginaryResult.x = (imaginaryNum.x * imaginaryNum.x) - (imaginaryNum.y * imaginaryNum.y);
    imaginaryResult.y = 2 * imaginaryNum.x * imaginaryNum.y;
    return imaginaryResult;
}

vec3 colorFunc(int iter) {
    // Color in HSV - Tweak these values to your liking and for different coloring effects
    vec3 color = vec3(0.012*iter , 1.0, 0.2+.4*(1.0+sin(0.3*iter)));

    // Convert from HSV to RGB
    // Taken from: http://lolengine.net/blog/2013/07/27/rgb-to-hsv-in-glsl
    vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
    vec3 m = abs(fract(color.xxx + K.xyz) * 6.0 - K.www);
    return vec3(color.z * mix(K.xxx, clamp(m - K.xxx, 0.0, 1.0), color.y));
}

void main() {
    vec2 c, z;
    c = v_Pos;      // c starts as the current position
    z = c;          // We can skip one iteration of the function where z = 0 as it is uneccessary if we simply set z = c

    vec3 color = vec3(0.0, 0.0, 0.0); // The default color will be black
    int iterations = 100; // 100 is a good number of iterations
    for(int i = 0; i < iterations; i++) {
        // fc(z) = z^2 + c - Here we iterate over this function until it either becomes greater than 2 or not
        vec2 result = squareImaginary(z) + c;
        if(length(result) > 2.0) {
            // This is a point not in the mandelbrot set - Change color based on iterations
            color = colorFunc(i);
            break;
        }
        z = result;
    }

    // Color the point
    gl_FragColor = vec4(color, 1.0);
}
```

Running the program and exploring around will produce all kinds of incredible fractal patterns (If you haven't already you could switch the program resolution from 480x480 to 960x960 now as it will look much better):

![OpenGL_Tutorial_Mandelbrot_10.png](OpenGL_Tutorial_Mandelbrot_10.png)

![OpenGL_Tutorial_Mandelbrot_11.png](OpenGL_Tutorial_Mandelbrot_11.png)

![OpenGL_Tutorial_Mandelbrot_12.png](OpenGL_Tutorial_Mandelbrot_12.png)


## Limitations

Now that we've finally completed the project I wanted to discuss some of the limitations of the implementation and perhaps lay out some future tasks for you to try.

You may have already noticed that if you zoom in far enough on the set it will begin to break down and look more and more pixelated until you cannot zoom anymore.

![OpenGL_Tutorial_Mandelbrot_13.png](OpenGL_Tutorial_Mandelbrot_13.png)

The reason for this is due to the fact that OpenGL handles everything in 32-bit floats. We use these floats for our vertices data, our camera position and within our shader code. This means that as you zoom in more and more you start to deal with smaller and smaller numbers until they are too small to be represented by a float. One quick improvement for this can be done by adding this line to the top of the fragment shader:

```glsl
precision highp float;
```

However, this change will be quite minute. Instead the best way to improve this would be to switch from using floats to doubles. OpenGL does in fact support the use of doubles for both vertex attributes and uniforms as of a somewhat recent update. As a possible task to test what you've learned you could try implementing this program from scratch using doubles. [This update](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_gpu_shader_fp64.txt) adds double uniforms and [this one](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_vertex_attrib_64bit.txt) adds support for double vertex attributes and each will detail the required functions. One thing to note is we used JOML Matrices of type float, `Matrix4f`, but you can use doubles by doing `Matrix4d`. If you do manage to get this working please share it with me in the comments!

One final challenge would be to create an infinite zoom. Some online resources detail how to do this using a lot of vector math or OpenCL (OpenGL for computing tasks) but I will leave research of this up to you.

## Conclusion

That's it for the tutorial series! If you came all the way from the beginning, thanks so much for reading all the way through and even if you didn't thanks so much for reading in general. I hope my explanations were clear and well understood. If you did note any problems leave a comment so I can possibly make some additions or changes in the future. As I mentioned in [part one](https://cianjinks.github.io/posts/opengl-tutorial-visualizing-the-mandelbrot-set-fractal-part-1-of-2/) if you want to check out my original version of this project it can be found on my [github](https://github.com/cianjinks/MandelbrotViewer). (This version has an implemented UI and settings and as well as that a lot of concepts covered here such as the Camera are abstracted into their own classes)

\- Cian Jinks

## Full Source Code

### Application.java

```java
import org.joml.Matrix4f;
import org.joml.Vector3f;
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

    // Projection Matrix
    private Matrix4f pmatrix = new Matrix4f().ortho(-2.0f, 2.0f, -2.0f, 2.0f, -1.0f, 1.0f);

    // Camera Panning Input
    private boolean upArrow = false;
    private boolean downArrow = false;
    private boolean rightArrow = false;
    private boolean leftArrow = false;

    // Zooming
    private boolean zoomingIn = false;
    private boolean zoomingOut = false;

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
        window = glfwCreateWindow(960 / 2, 960 / 2, "Hello World!", NULL, NULL);
        if ( window == NULL )
            throw new RuntimeException("Failed to create the GLFW window");

        // Setup a key callback. It will be called every time a key is pressed, repeated or released.
        glfwSetKeyCallback(window, (window, key, scancode, action, mods) -> {
            if ( key == GLFW_KEY_ESCAPE && action == GLFW_RELEASE ) {
                glfwSetWindowShouldClose(window, true); // We will detect this in the rendering loop
            }

            // Up Arrow Key
            if (key == GLFW_KEY_UP && action == GLFW_PRESS) {
                upArrow = true;
            }
            if(key == GLFW_KEY_UP && action == GLFW_RELEASE) {
                upArrow = false;
            }

            // Down Arrow Key
            if (key == GLFW_KEY_DOWN && action == GLFW_PRESS) {
                downArrow = true;
            }
            if(key == GLFW_KEY_DOWN && action == GLFW_RELEASE) {
                downArrow = false;
            }

            // Left Arrow Key
            if (key == GLFW_KEY_LEFT && action == GLFW_PRESS) {
                leftArrow = true;
            }
            if(key == GLFW_KEY_LEFT && action == GLFW_RELEASE) {
                leftArrow = false;
            }

            // Right Arrow Key
            if (key == GLFW_KEY_RIGHT && action == GLFW_PRESS) {
                rightArrow = true;
            }
            if(key == GLFW_KEY_RIGHT && action == GLFW_RELEASE) {
                rightArrow = false;
            }

            // Zooming In (Z Key)
            if (key == GLFW_KEY_Z && action == GLFW_PRESS) {
                zoomingIn = true;
            }
            if (key == GLFW_KEY_Z && action == GLFW_RELEASE) {
                zoomingIn = false;
            }

            // Zooming Out (X Key)
            if (key == GLFW_KEY_X && action == GLFW_PRESS) {
                zoomingOut = true;
            }
            if (key == GLFW_KEY_X && action == GLFW_RELEASE) {
                zoomingOut = false;
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

        // Vertices - Positions
        float[] vertices = new float[] {
                -2.0f,  2.0f,   // Vertex 0
                -2.0f, -2.0f,   // Vertex 1
                 2.0f, -2.0f,   // Vertex 2
                 2.0f,  2.0f    // Vertex 3
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

        float basePanningSpeed = 0.0125f;
        float currentPanningSpeed = 0.0125f;
        float zoomAmount = 1.0f;
        float zoomSpeed = 1.05f;

        // Run the rendering loop until the user has attempted to close
        // the window or has pressed the ESCAPE key.
        while ( !glfwWindowShouldClose(window) ) {
            glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // clear the framebuffer

            // Camera Panning
            if(upArrow) {
                // Translate the projection matrix by the distance cameraSpeed in the positive y direction
                pmatrix.translate(new Vector3f(0.0f, currentPanningSpeed, 0.0f));
            }
            if(downArrow) {
                // Translate the projection matrix by the distance cameraSpeed in the negative y direction
                pmatrix.translate(new Vector3f(0.0f, -currentPanningSpeed, 0.0f));
            }
            if(rightArrow) {
                // Translate the projection matrix by the distance cameraSpeed in the positive x direction
                pmatrix.translate(new Vector3f(currentPanningSpeed, 0.0f, 0.0f));
            }
            if(leftArrow) {
                // Translate the projection matrix by the distance cameraSpeed in the negative x direction
                pmatrix.translate(new Vector3f(-currentPanningSpeed, 0.0f, 0.0f));
            }

            // Zooming
            if(zoomingIn) {
                zoomAmount = zoomAmount * zoomSpeed; // Update the zoomAmount
                currentPanningSpeed = basePanningSpeed / zoomAmount; // Update camera panning speed

                pmatrix.scaleLocal(zoomSpeed); // Zoom In

            } else if(zoomingOut) {
                zoomAmount = zoomAmount * (1 / zoomSpeed); // Update the zoomAmount
                currentPanningSpeed = basePanningSpeed / zoomAmount; // Update camera panning speed

                pmatrix.scaleLocal(1 / zoomSpeed); // Zoom Out

            }

            // Camera
            try (MemoryStack stack = MemoryStack.stackPush()) {
                FloatBuffer projection = pmatrix.get(stack.mallocFloat(4 * 4));
                int mat4location = GL30.glGetUniformLocation(programID, "u_MVP");
                GL30.glUniformMatrix4fv(mat4location, false, projection);
            }

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

    public static void main(String[] args) {
        new Application().run();
    }

}
```

### /shaders/vert.shader

```glsl
layout(location = 0) in vec2 position;

uniform mat4 u_MVP;

out vec2 v_Pos;

void main() {
    gl_Position = u_MVP * vec4(position, 0, 1.0);
    v_Pos = position;
}
```

### /shaders/frag.shader

```glsl
precision highp float;

in vec2 v_Pos;

vec2 squareImaginary(vec2 imaginaryNum) {
    vec2 imaginaryResult;
    imaginaryResult.x = (imaginaryNum.x * imaginaryNum.x) - (imaginaryNum.y * imaginaryNum.y);
    imaginaryResult.y = 2 * imaginaryNum.x * imaginaryNum.y;
    return imaginaryResult;
}

vec3 colorFunc(int iter) {
    // Color in HSV - Tweak these values to your liking and for different coloring effects
    vec3 color = vec3(0.012*iter , 1.0, 0.2+.4*(1.0+sin(0.3*iter)));

    // Convert from HSV to RGB
    // Taken from: http://lolengine.net/blog/2013/07/27/rgb-to-hsv-in-glsl
    vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
    vec3 m = abs(fract(color.xxx + K.xyz) * 6.0 - K.www);
    return vec3(color.z * mix(K.xxx, clamp(m - K.xxx, 0.0, 1.0), color.y));
}

void main() {
    vec2 c, z;
    c = v_Pos;      // c starts as the current position
    z = c;          // We can skip one iteration of the function where z = 0 as it is uneccessary if we simply set z = c

    vec3 color = vec3(0.0, 0.0, 0.0); // The default color will be black
    int iterations = 100; // 100 is a good number of iterations
    for(int i = 0; i < iterations; i++) {
        // fc(z) = z^2 + c - Here we iterate over this function until it either becomes greater than 2 or not
        vec2 result = squareImaginary(z) + c;
        if(length(result) > 2.0) {
            // This is a point not in the mandelbrot set - Change color based on iterations
            color = colorFunc(i);
            break;
        }
        z = result;
    }

    // Color the point
    gl_FragColor = vec4(color, 1.0);
}
```
