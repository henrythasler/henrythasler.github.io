---
title: Hello Quad
subtitle: basic OpenGL App with Qt
tags: [opengl, qt]

---

In this first tutorial we will create a simple (I mean really simple...)  Qt program to render [OpenGL graphics](http://en.wikipedia.org/wiki/Opengl) (spinning quad). I will extend the programs you find in the [Qt boxes demo](http://doc.qt.nokia.com/demos-boxes.html) and [Qt OpenGL examples](http://doc.qt.nokia.com/examples-opengl.html) with a separate  thread that does the actual redering to be independent  (in terms of  framerate) from any GUI interaction or other Qt events.

Sourcecode: [Tut_01_OpenGL_Setup.zip](/files/Tut_01_OpenGL_Setup.zip) (Qt-Project files).

The result of this tutorial looks like this:

![](/img/blog/Tut01-AppWindow.png)

This tutorial was mainly inspired by the Qt newsletter [Glimpsing the Third Dimension](http://doc.trolltech.com/qq/qq06-glimpsing.html).

## Before you start

Make sure your have a working installation of Qt. I use 4.7.0 (Aug 2010) but the latest version should work as well.

*   Qt Installation ([qt.nokia.com/downloads](http://qt.nokia.com/downloads))
I use the windows version of Qt so I will focus on that   platform.  In general the code should work on other platforms too. Let me  know if  there is a problem

## Overview

First we need to subclass the [QGLWidget](http://doc.trolltech.com/qglwidget.html) to get the OpenGL functionality. Then we have to create a new thread based on [QThread](http://doc.trolltech.com/qthread.html) and tell the thread to do some rendering. Finally we embed our specially designed [QGLWidget](http://doc.trolltech.com/qglwidget.html) derivate into our [main window](http://doc.trolltech.com/qmainwindow.html).

## Adding required Project files

Create a new "Qt Gui Application". When you're done your will see the following default project tree.

![](/img/blog/Tut01-ProjectSetup.png)

Since we want to use OpenGL we need to add the opengl module to our   project. Open the Project file "Tut_01_OpenGL_Setup.pro" with a double   click and add the "opengl" keyword to the 'QT +=' statement as shown   below:

```bash
    #-------------------------------------------------
    #
    # Project created by QtCreator 2011-04-25T21:36:17
    #
    #-------------------------------------------------

    QT       += core gui opengl

    TARGET = Tut_01_OpenGL_Setup
    TEMPLATE = app

    SOURCES += main.cpp
            mainwindow.cpp 
        myglframe.cpp 
        myrenderthread.cpp

    HEADERS  += mainwindow.h 
        myglframe.h 
        myrenderthread.h

    FORMS    += mainwindow.ui
```

Add a new class (right-click project root) and choose  "C++  Class". Name the new class something like "MyGLFrame". Set the  Base  class property to "QGLWidget". Type information is inherited from [QWidget](http://doc.qt.nokia.com/qwidget.html). After that 2 new files (myglframe.h and myglframe.cpp) will appear in your project tree.

![](/img/blog/Tut01-NewGLWidget.png)

You need to add one more class named "MyRenderThread" based on [QThread](http://doc.trolltech.com/qthread.html).  Type information is inherited from QObject. Another two files   (myrenderthread.h and myrenderthread.cpp) will appear in your project.   The final project structure should look like this:

![](/img/blog/Tut01-ProjectFinal.png)

## The Render Thread in Detail

The header file contains the prototypes of the functions used by the  thread. Note that the constructor is called with a pointer to the OpenGL  context.

The "resizeViewport()" is used to tell the thread that the OpenGL  frame has changed its size and the viewport has to be modified as well  with a "GLResize()" call. "stop()" simply stops the rendering.

The thread's main function is "run()". At first it is nessessary to  claim the OpenGL rendering context so the thread can do the actual  rendering. The GLInit() function can is used to set all OpenGL-specific  properties. In this example is't just the background color since the  default settings of all other properties are fine for now.

The rendering loop is placed in a while-statement until the rendering  is disabled by a call to "stop()" which sets the doRendering-variable  to false. In this while-loop we check if a resize-event has occured and  eventually change the projection matrix. After that the frame is cleared  and the inentity matrix is loaded and our rendering function is called  with "paintGL()". Make sure you swap the buffers to make the rendered  contents visible.

To save some CPU/GPU power we add the following [msleep()](http://doc.qt.nokia.com/qthread.html#msleep) statement which does nothing but release the CPU for 16 milliseconds to  do other stuff than render our dull quad. 16ms gives us about 60 FPS  which is more than enough to get a smooth animation.

```c++
    void MyRenderThread::run()
    {
        GLFrame->makeCurrent();
        GLInit();

        while (doRendering)
            {
            if (doResize)
                {
                GLResize(w, h);
                doResize = false;
                }
            glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
            glLoadIdentity();

            paintGL(); // render actual frame

            FrameCounter++;
            GLFrame->swapBuffers();

            msleep(16); // wait 16ms => about 60 FPS
            }
    }
```

This is the code to resize our OpenGL window accordingly. Check out [Nehe's OpenGL tutorials](http://nehe.gamedev.net/data/lessons/lesson.asp?lesson=01) for details.

```c++
    void MyRenderThread::GLResize(int width, int height)
    {
        glViewport(0, 0, width, height);
        glMatrixMode(GL_PROJECTION);

        glLoadIdentity();
        gluPerspective(45.,((GLfloat)width)/((GLfloat)height),0.1f,1000.0f);

        glMatrixMode(GL_MODELVIEW);
        glLoadIdentity();
    }
```

This is our simple render function. Is draws four colored (glColor3f)  vertices (glVertex3f) and combines them to a quad (glBegin). The quad is  moved somewhat away from the camera (glTranslatef) and rotated  (glRotatef) 1 degree around the z-axis every frame. Thats it.

```c++
    void MyRenderThread::paintGL(void)
    {
        glTranslatef(0.0f, 0.0f, -5.0f);
        glRotatef(FrameCounter,0.0f,0.0f,1.0f);
        glBegin(GL_QUADS);
            glColor3f(1.,1.,0.); glVertex3f(-1.0, -1.0,0.0);
            glColor3f(1.,1.,1.); glVertex3f(1.0, -1.0,0.0);
            glColor3f(1.,0.,1.); glVertex3f(1.0, 1.0,0.0);
            glColor3f(1.,0.,0.); glVertex3f(-1.0, 1.0,0.0);
        glEnd();
    }
```

## The OpenGL-Widget

To get an OpenGL rendering context we had to subclass the QGLWidget  (myglframe). This class contains just our rendering thread and does the  interaction with it.

Please make sure to define the destructor (~MyGLFrame) and also the  paintEvent. Otherwise you will get linker errors and error messages  during runtime.

The constructor initializes the RenderThread and disables the automatic buffer swapping (we do this in our thread, remember?).

```c++
    MyGLFrame::MyGLFrame(QWidget *parent) :
        QGLWidget(parent),
        RenderThread(this)
    {
        setAutoBufferSwap(false);
    }
```

To assure a clean shutdown of our thread, you must stop and wait for it before exit.

```c++
    void MyGLFrame::stopRenderThread(void)
    {
        RenderThread.stop();
        RenderThread.wait();
    }
```

## The Main Window

In the MainWindow we create a new instance of our MyGLFrame and assign it to the CentralWidget ([setCentralWidget](http://doc.qt.nokia.com/qmainwindow.html#setCentralWidget)). After initializing the thread (initRenderThread) we are good to go.

```c++
    MainWindow::MainWindow(QWidget *parent) :
        QMainWindow(parent),
        ui(new Ui::MainWindow)
    {
        ui->setupUi(this);
        GLFrame = new MyGLFrame();
        setCentralWidget(GLFrame);
        GLFrame->initRenderThread();
    }

    MainWindow::~MainWindow()
    {
        delete GLFrame;
        delete ui;
    }

    void MainWindow::closeEvent(QCloseEvent *evt)
    {
        GLFrame->stopRenderThread();
        QMainWindow::closeEvent(evt);
    }
```

## Conclusion

We have implemented a simple OpenGL render thread with Qt. This  framework can be extended as you like and will also be used in further  tutorials. Hope you like it
