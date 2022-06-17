---
title: Creating a CMake Project for cross platform graphics programming with OpenGL
date: 2022-06-17 12:00:00 +0000
categories: [Graphics]
tags: [opengl, cmake, tutorial, imgui]
img_path: /assets/img/post/CMakeProject/
---
A while back I found myself switching to Linux as my main development environment for college work. Up until that point I had been spending sporadic free time learning graphics programming on Windows using Visual Studio and wanted to port over some of my projects. I thought CMake would be the best route as it was extensively used by other graphics projects on Github and is well maintained.

However, it took me quite a while to figure out how to get a project setup with all of my usual libraries (GLFW, GLM, ImGui, more), as there was no clear tutorial I could find. Now that I've had it working for some time, I thought I'd take a stab at writing my own. If you're not interested in reading how it all is setup, and just would like a project template, checkout this [Github template](https://github.com/cianjinks/CMake-OpenGL-Template).

## Plan

For this tutorial I am imagining a hypothetical scenario where I want to start a new graphics programming project which I can run on both Linux and Windows. My project is going to use OpenGL for graphics and various other libraries:

- [ImGui](https://github.com/ocornut/imgui) for GUI
- [GLFW](https://www.glfw.org/) for cross platform window management
- [GLM](https://github.com/g-truc/glm) for math
- [stb_image](https://github.com/nothings/stb/blob/master/stb_image.h) for texture loading

With the information in this article (and perhaps a little more research) you should be able to understand how to add more of your own libraries, support MacOS and other platforms and potentially setup a similar project for Vulkan, DirectX, Metal, etc programming.

## Project Structure

Our project structure is going to look like this:

```
CMake-Graphics-Project
├── CMakeLists.txt
├── dependencies
│   ├── glad
│   ├── glfw
│   ├── glm
│   ├── imgui
│   └── stb_image
├── README.md
├── resources
└── src
    └── Main.cpp
```

We start off by creating our main project folder called `Cmake-Graphics-Project`. We then also create a few subfolders:

- `dependencies` is where we will put the libraries for our project
- `resources` is where we will put resources like textures, shaders, etc
- `src` is where our source code will be

For now I will keep these folders empty, except a `Main.cpp` file in `src` with the following contents:

```cpp
#include <iostream>

int main()
{
    std::cout << "Hello World!" << std::endl;
}
```
{: file='/src/Main.cpp'}

I have also added a simple `README.md` for Github.

## Initial CMake Setup

The last and most important file of the initial setup is the `CMakeLists.txt`. This is the file which configures our CMake project. I will start by creating a simple one so that we can compile our test Main program while we have no dependencies or resources. Before we start you will need to install [CMake](https://cmake.org/download/) and a C and C++ compiler (if you have Visual Studio installed on Windows you already have a compiler). If you are on Windows also make sure to tick to add CMake to PATH so that it can be used from the command line. Linux should do this automatically.

With CMake installed we can check if it's working from the command line like so:

```console
$ cmake --version
cmake version 3.23.2
CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

Now that it's working we can start writing our `CMakeLists.txt`. The first few lines of nearly every CMakeLists define the CMake minimum version and declare a project with name. In this case we are naming it `CMakeGraphicsProject` but it could be anything you want:

```cmake
cmake_minimum_required(VERSION 3.3)
project(CMakeGraphicsProject)
```
{: file='CMakeLists.txt'}

On the next lines we are then going to define our C++ standard. To do this we use the CMake command `set(<variable>, <value>)` to set some builtin variables:

```cmake
# C++ Standard
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED True)
```
{: file='CMakeLists.txt'}

Finally we need to define an executable to compile and add our code (`.cpp`) and header files (`.h`) to it. We can use a neat trick to gather all `.cpp` and `.h` files under `./src` into one variable called `source`. Then we just need to add the `source` variable to the executable.

```cmake
# Set `source` variable to all .h and .cpp files in `src`
file(GLOB_RECURSE source CONFIGURE_DEPENDS "src/*.h" "src/*.cpp")

add_executable(CMakeGraphicsProject ${source})
```
{: file='CMakeLists.txt'}

That's it for now. We should be able to build and run our simple project next.

## Initial CMake Building

Before we build our executable we need to do some more CMake setup. Create a folder called `build`, change directory into it and run `cmake ../` from a commandline. On Linux I do this all from the commandline:

```console
$ mkdir build
$ cd ./build
$ cmake ../
-- The C compiler identification is GNU 12.1.0
-- The CXX compiler identification is GNU 12.1.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/hobbes/Documents/Graphics/CMake-Graphics-Project/build
```

This is where things differ between Linux and Windows. On Windows you will now find a Visual Studio solution file (`.sln`) in the `./build` folder which you can open in Visual Studio and use as normal (build, run, debug, etc). However on Linux we need one more command to build the executable.

```console
$ cmake --build ./
[ 50%] Building CXX object CMakeFiles/CMakeGraphicsProject.dir/src/Main.cpp.o
[100%] Linking CXX executable CMakeGraphicsProject
[100%] Built target CMakeGraphicsProject

$ ./CMakeGraphicsProject
Hello World!
```

## Setting up for OpenGL

So far our CMake project will let us program C++ with standard libraries, but this means no OpenGL support. To include OpenGL headers in our project, we can use some built-in CMake features. Add the following lines just under the `project(CMakeGraphicsProject)` line of our `CMakeLists.txt`:

```cmake
# OpenGL
set(OpenGL_GL_PREFERENCE GLVND)
find_package(OpenGL REQUIRED)
include_directories(src ${OPENGL_INCLUDE_DIRS})
```
{: file='CMakeLists.txt'}

Then also add this line under the `add_executable` line:

```cmake
target_link_libraries(CMakeGraphicsProject ${OPENGL_LIBRARIES})
```
{: file='CMakeLists.txt'}

These lines tell CMake to include and link basic built-in system OpenGL functionality. However, we still do not have modern OpenGL support nor an easy cross platform way to create a window. To get these working we are going to need to introduce two dependencies. `GLAD` for modern OpenGL bindings and `GLFW` for window management.

### Setting up GLAD

GLAD is a C library which provides bindings for modern OpenGL functionality. You can grab the latest version from the GLAD [generator website](https://glad.dav1d.de/). Just pick what version of OpenGL you want, hit "Generate", and you should get a file called `glad.zip`. This ZIP should contain two folders: `include` and `src`.

If you haven't already, create a folder called `glad` under `dependencies` and place these two folders in there like so:

```
dependencies
    └── glad
        ├── include
        └── src
```

Now that we've got our GLAD dependency, how do we add it to the main CMake project? We need to make a subproject. For all of our dependencies we are going to create a CMake subproject by placing a `CMakeLists.txt` file in each dependency's folder. In this case we need to create `/dependencies/glad/CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.3)
project(Glad)

add_library(glad include/glad/glad.h src/glad.c)
target_include_directories(glad PUBLIC include/)
```
{: file='/dependencies/glad/CMakeLists.txt'}

Same as before we define our minimum cmake version and project name. However, we don't want to build an executable for this project because it is meant to be a library. Therefore, we don't add the source and header files in the same way we did before. We instead add a library (`add_library`) called `glad` with the header files and source files from the subfolders. Depending on if there are new updates to GLAD, your folder structure from the ZIP file will be different and you may need to change this line slightly.

On the last line we specify include directories for the library `glad`.

We have now created a CMake subproject for GLAD and just need to add it to our main `CMakeLists.txt` like so:

```cmake
cmake_minimum_required(VERSION 3.3)
project(CMakeGraphicsProject)

# OpenGL
set(OpenGL_GL_PREFERENCE GLVND)
find_package(OpenGL REQUIRED)
include_directories(src ${OPENGL_INCLUDE_DIRS})

# C++ Standard
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# Subprojects
add_subdirectory(dependencies/glad)

# Set `source` variable to all .h and .cpp files in `src`
file(GLOB_RECURSE source CONFIGURE_DEPENDS "src/*.h" "src/*.cpp")

add_executable(CMakeGraphicsProject ${source})
target_link_libraries(CMakeGraphicsProject ${OPENGL_LIBRARIES} glad) # <Added GLAD>
```
{: file='CMakeLists.txt'}

Note that we add the `add_subdirectory` line and then update our `target_link_libraries`. The latter tells CMake what libraries to link into our final executable for `CMakeGraphicsProject`.

### Setting up GLFW

First we need to download the latest version of [GLFW](https://www.glfw.org/). Same as before, we need to extract the zip file contents into our `/dependencies/glfw` folder so we end up with a structure like so:

```
dependencies
    └── glfw
        ├── CMake
        │   └── modules
        ├── deps
        │   ├── glad
        │   ├── mingw
        │   └── vs2008
        ├── docs
        │   └── html
        │       └── search
        ├── examples
        ├── include
        │   └── GLFW
        ├── src
        └── tests
```

This structure may look slightly different for you if GLFW changes, but it should be generally similar.

Luckily for us, GLFW comes with an already configured `CMakeLists.txt` subproject file. Therefore, we just need to add the subproject to our main `CMakeLists.txt` and update our `target_link_libraries`:

```cmake
# Subprojects
add_subdirectory(dependencies/glad)
add_subdirectory(dependencies/glfw)
.
.
.
target_link_libraries(CMakeGraphicsProject ${OPENGL_LIBRARIES} glad glfw) # <Added GLFW>
```
{: file='CMakeLists.txt'}

### Building & Running

Now that we have GLAD and GLFW added to our project, we are going to want to modify our `Main.cpp` file to test them out. I am basing this test code off of the example code from [GLFW's documentation](https://www.glfw.org/documentation.html) and have simply added code to initialise GLAD for testing as well:

```cpp
#include <glad/glad.h>
#include <GLFW/glfw3.h>

int main(void)
{
    GLFWwindow *window;

    /* Initialize the library */
    if (!glfwInit())
        return -1;

    /* Create a windowed mode window and its OpenGL context */
    window = glfwCreateWindow(640, 480, "Hello World", NULL, NULL);
    if (!window)
    {
        glfwTerminate();
        return -1;
    }

    /* Make the window's context current */
    glfwMakeContextCurrent(window);

    /* Initialize GLAD */
    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
        return -1;

    /* Loop until the user closes the window */
    while (!glfwWindowShouldClose(window))
    {
        /* Render here */
        glClear(GL_COLOR_BUFFER_BIT);
        glClearColor(0.0f, 1.0f, 0.0f, 1.0f);

        /* Swap front and back buffers */
        glfwSwapBuffers(window);

        /* Poll for and process events */
        glfwPollEvents();
    }

    glfwTerminate();
    return 0;
}
```
{: file='/src/Main.cpp'}

Now we can build and run our project. On Windows we would do this within the VS solution (remember to set the startup project to CMakeGraphicsProject before hitting run!), but on Linux I will use the same command as before:

```console
$ cmake --build ./build
<This produces a large output as it builds our executable and the new libraries>
$ ./build/CMakeGraphicsProject
```

![GLFW_Window.png](GLFW_Window.png)

If all works correctly, when we run our executable a green GLFW window should appear, as seen above. If you get as far as building your executable without any issue, but when you run the program nothing happens then maybe try adding some debug print statements to make sure it's not exiting early with a `return -1`!

Our current CMake setup is enough to get started graphics programming with OpenGL, however when I am working on a project I tend to use a few more libraries so we are going to set those up next.

## Adding Extra Libraries

The three last libraries I am going to add to this project are [ImGui](https://github.com/ocornut/imgui) for easy GUI setup, [stb_image](https://github.com/nothings/stb/blob/master/stb_image.h) for texture loading and [GLM](https://github.com/g-truc/glm) for mathematics. ImGui is significantly more complex to add to a CMake project so we are going to start with `stb_image` and `GLM`.

### Setting up stb_image

`stb_image` is a single header library for loading textures (png, jpg, etc) into memory for use with OpenGL. To start we need to download the [stb_image.h](https://github.com/nothings/stb/blob/master/stb_image.h) header and setup a dependencies folder for it. The folder will have the following structure:

```
stb_image
    ├── CMakeLists.txt
    ├── include
    │   └── stb_image.h
    └── stb_image_impl.c
```

As you can see I have added a file called `stb_image_impl.c` (stb_image implementation). This file simply includes the `stb_image.h` header and defines `STB_IMAGE_IMPLEMENTATION` so that we can build it as a static library using CMake:

```c
#define STB_IMAGE_IMPLEMENTATION
#include <stb_image.h>
```
{: file='/dependencies/stb_image/stb_image_impl.c'}

Our `CMakeLists.txt` looks nearly identical to the one we created for `GLAD` because we once again just need to create a static library with `.c` and `.h` files:

```cmake
cmake_minimum_required(VERSION 3.3)
project(STB_IMAGE)

add_library(stb_image include/stb_image.h stb_image_impl.c)
target_include_directories(stb_image PUBLIC include/)
```
{: file='/dependencies/stb_image/CMakeLists.txt'}

With the subproject dependency now created we just need to add it to our main `CMakeLists.txt` in the usual way:

```cmake
# Subprojects
add_subdirectory(dependencies/glad)
add_subdirectory(dependencies/glfw)
add_subdirectory(dependencies/stb_image)
.
.
.
target_link_libraries(CMakeGraphicsProject ${OPENGL_LIBRARIES} glad glfw stb_image) # <Added stb_image>
```
{: file='CMakeLists.txt'}

### Setting up GLM

`GLM` is a math library for OpenGL which implements various useful datastructures like vectors and matrices, as well as various operations for doing calculations using those datastructures. To get started we need to download the latest release ZIP from [Github](https://github.com/g-truc/glm/releases/). `GLM`'s structure is nearly identical to `GLFW`'s and so the process for adding it to our CMake project is also basically the same.

First we extract it to the folder `dependencies/glm` so that we get a structure like the following:

```
glm
    ├── cmake
    │   └── glm
    ├── doc
    │   ├── api
    │   │   └── search
    │   ├── manual
    │   └── theme
    ├── glm
    │   ├── detail
    │   ├── ext
    │   ├── gtc
    │   ├── gtx
    │   └── simd
    ├── test
    │   ├── bug
    │   ├── cmake
    │   ├── core
    │   ├── ext
    │   ├── gtc
    │   ├── gtx
    │   └── perf
    └── util
```

Then since it already has a `CMakeLists.txt` subproject setup for us we just need to add it to our main `CMakeLists.txt`:

```cmake
# Subprojects
add_subdirectory(dependencies/glad)
add_subdirectory(dependencies/glfw)
add_subdirectory(dependencies/stb_image)
add_subdirectory(dependencies/glm)
.
.
.
target_link_libraries(CMakeGraphicsProject ${OPENGL_LIBRARIES} glad glfw stb_image glm) # <Added GLM>
```
{: file='CMakeLists.txt'}

### Setting up ImGui

`ImGui` is a library that makes OpenGL GUI incredibly quick and easy to implement. It allows us to add all kinds of UI buttons, sliders, settings, etc with very few lines of code. It is highly confiurable and works with a large number of graphics pipelines so it will take some fiddling to get it setup in our CMake project.

As per usual, the  first thing we need to do is head over to the ImGui Github [releases page](https://github.com/ocornut/imgui/releases/) and download the latest release. At the time of writing there is no specific release ZIP, so you need to just download the source code ZIP. With it download we need to extract the contents into `dependencies/imgui` so that we end up with a structure like so:

```
imgui
    ├── backends
    │   └── vulkan
    ├── docs
    ├── examples
    │   ├── example_*
    ├── imconfig.h
    ├── imgui.cpp
    ├── imgui_demo.cpp
    ├── imgui_draw.cpp
    ├── imgui.h
    ├── imgui_internal.h
    ├── imgui_tables.cpp
    ├── imgui_widgets.cpp
    ├── imstb_rectpack.h
    ├── imstb_textedit.h
    ├── imstb_truetype.h
    ├── LICENSE.txt
    └── misc
        ├── cpp
        ├── debuggers
        ├── fonts
        ├── freetype
        ├── README.txt
        └── single_file
```

As you can see, ImGui does not come with a `CMakeLists.txt` file and this is because it can support so many different platforms and graphics pipelines that they expect you to simply include the headers and source files that you need from it.

Therefore, in our main `CMakeLists.txt` we are going to pull in various useful header files from ImGui for GLFW and OpenGL3 support as well as general ImGui sources and headers. Just after where we define our C++ standard place the following:

```cmake
# ImGui
set(imgui-directory dependencies/imgui)
set(imgui-source ${imgui-directory}/imconfig.h
	${imgui-directory}/imgui.h
	${imgui-directory}/imgui.cpp
	${imgui-directory}/imgui_draw.cpp
	${imgui-directory}/imgui_internal.h
	${imgui-directory}/imgui_widgets.cpp
	${imgui-directory}/imstb_rectpack.h
	${imgui-directory}/imstb_textedit.h
	${imgui-directory}/imstb_truetype.h
    ${imgui-directory}/imgui_tables.cpp
	${imgui-directory}/imgui_demo.cpp
	${imgui-directory}/backends/imgui_impl_glfw.cpp
	${imgui-directory}/backends/imgui_impl_opengl3.cpp
)
```
{: file='CMakeLists.txt'}

This sets variables `imgui-directory` and `imgui-source` which we can then use like so:

```cmake
add_executable(CMakeGraphicsProject ${source} ${imgui-source})
target_link_libraries(CMakeGraphicsProject ${OPENGL_LIBRARIES} glad glfw stb_image glm) # <Unchanged>
target_include_directories(CMakeGraphicsProject PRIVATE ${imgui-directory}) # <Added this line>
```
{: file='CMakeLists.txt'}

Now when we compile and link our executable it will pull in the ImGui `.h` and `.cpp` files shown above.

### Building & Running

After all of that configuring let's try modifying `Main.cpp` to test out the new libraries we added. Here is a new `Main.cpp` which tries some vector math, loads an image, and renders the ImGui demo window inside our GLFW window:

```cpp
#include "imgui.h"
#include "imgui_internal.h"
#include "backends/imgui_impl_glfw.h"
#include "backends/imgui_impl_opengl3.h"

#include <glad/glad.h>
#include <GLFW/glfw3.h>

#include <glm/glm.hpp>
#include <stb_image.h>

#include <iostream>

int main(void)
{
    GLFWwindow *window;

    /* Initialize the library */
    if (!glfwInit())
        return -1;

    /* Create a windowed mode window and its OpenGL context */
    window = glfwCreateWindow(640, 480, "Hello World", NULL, NULL);
    if (!window)
    {
        glfwTerminate();
        return -1;
    }

    /* Make the window's context current */
    glfwMakeContextCurrent(window);

    /* Initialize GLAD */
    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
        return -1;

    /* Vector Math */
    glm::vec3 v1(1.0, 2.0, 3.0);
    glm::vec3 v2(10.0, 11.0, 2.0);
    glm::vec3 v = v1 + v2;
    std::cout << "Vector Result: " << v.x << " " << v.y << " " << v.z << std::endl;

    /* Load an Image */
    int w, h, nrChannels;
    unsigned char *image = stbi_load("/home/hobbes/Documents/Graphics/CMake-Graphics-Project/resources/test_image.png", &w, &h, &nrChannels, STBI_default);
    if (!image)
        return -1;
    std::cout << "Width: " << w << " | Height: " << h << " | Channels: " << nrChannels << std::endl;

    /* Initialize ImGui */
    IMGUI_CHECKVERSION();
    ImGui::CreateContext();
    ImGuiIO &io = ImGui::GetIO();
    ImGui::StyleColorsDark();

    ImGui_ImplGlfw_InitForOpenGL(window, true);
    ImGui_ImplOpenGL3_Init("#version 460 core");

    /* Loop until the user closes the window */
    while (!glfwWindowShouldClose(window))
    {
        /* ImGui Frame */
        ImGui_ImplOpenGL3_NewFrame();
        ImGui_ImplGlfw_NewFrame();
        ImGui::NewFrame();
        ImGui::ShowDemoWindow();
        ImGui::Render();

        /* Render here */
        glClear(GL_COLOR_BUFFER_BIT);
        glClearColor(0.0f, 1.0f, 0.0f, 1.0f);

        /* ImGui Render */
        ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());

        /* Swap front and back buffers */
        glfwSwapBuffers(window);

        /* Poll for and process events */
        glfwPollEvents();
    }

    glfwTerminate();
    return 0;
}
```
{: file='/src/Main.cpp'}

![ImGui_Window.png](ImGui_Window.png)

If everything goes as planned you should see the usual GLFW window with a demo ImGui window inside which you can mess around with, as well as some console output to show that GLM and stb_image are working.

You might be wondering why I am using a global filepath for the `.png` image. We'll discuss that now in the final section.

## Resources

The final piece of the puzzle to complete our CMake project is our resources folder. This folder sits in the top level of our project and is where I would put any textures, shaders, text files, json files, etc that I might want to use within my code. However, depending on where you use cmake to build your project the executable will move around the place. This means that using relative file paths within the code will likely not work. To solve this issue, we want our resources folder to always end up in the same folder as the executable. This will allow us to use filepaths like "resources/shaders/test.glsl" from within our code. But how do we do this?

We _could_ tell CMake to copy our resources folder to the executable output directory like so:

```cmake
# Copy Resources
file(COPY resources/ DESTINATION ${PROJECT_BINARY_DIR}/resources/)
```

However, the issue with this approach is when you start having gigabytes of resources in your project it will eat up a lot of disk space. This started to happen to me recently so I came up with a better solution to create a symbolic link or symlink to the main resources folder and place it beside the executable. CMake does not have an easy way to do this but with a bit of searching I found:

```cmake
# Symlink Resources
add_custom_command(TARGET CMakeGraphicsProject PRE_BUILD COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_CURRENT_SOURCE_DIR}/resources $<TARGET_FILE_DIR:CMakeGraphicsProject>/resources)
```

A symlink is essentially a folder that points to another folder. They are enabled by default on both Windows and Linux. If you place this at the bottom of your `CMakeLists.txt` then instead of loading the texture in `Main.cpp` from "/home/hobbes/Documents/Graphics/CMake-Graphics-Project/resources/test_image.png", you can just load it like so:

```cpp
unsigned char *image = stbi_load("./resources/test_image.png", &w, &h, &nrChannels, STBI_default);
```

## Wrap

And that's it. We now have what I would consider a fully configured CMake project for cross platform graphics programming. I found it quite a headache when I initially tried to get a CMake project working for this purpose so if this guide was able to help you solve at least one problem you've been having then it was worth writing!

If you would like to checkout the finished working version of this project it can be found on Github at [this link](https://github.com/cianjinks/CMake-OpenGL-Template). This version has a few extra files like a README and linux/windows setup scripts. It also has some extra CMake settings for address sanitizer and GLFW flags.

### Extra Notes

If you don't want people to have to set the startup project in Visual Studio when they first build your project, you can set it automatically in CMake by adding the following anywhere in your `CMakeLists.txt`:

```cmake
# Visual Studio Startup Project
if(WIN32)
	set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT CMakeGraphicsProject)
endif()
```
{: file='CMakeLists.txt'}