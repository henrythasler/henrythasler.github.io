---
title: Hello GLSL
subtitle: GLSL Shader App with Qt
tags: [opengl, qt]

---

In this tutorial we will extend the previous tutorial and make use of a [Shader](http://en.wikipedia.org/wiki/Shader) written in [GLSL](http://en.wikipedia.org/wiki/GLSL) to fill our spinning quad. GLSL is a C-like language you can use to execute code on your [GPU](http://en.wikipedia.org/wiki/Graphics_processing_unit) (aka graphics card). More sophisticated effects can be achieved with this technique.

The point of this tutorial is to show how Qt supports shader programs and how to use them in your application.

Sourcecode: [Hello_GLSL.zip](/files/Hello_GLSL.zip)

![](/img/blog/Tut02-AppWindow.png)

## Using GLSL Shaders with Qt

Qt already provides two classes to work with shaders. We just have to add some function to use them.

The first class we need is [QGLShaderProgram](http://doc.qt.nokia.com/QGLShaderProgram.html).  It provides a container for our shaders and comes with functions to  link and use the shader programs while rendering OpenGL objects. The  second class is [QGLShader](http://doc.qt.nokia.com/QGLShader.html).  This class supports loading and compiling various (vertex, geometric,  fragment) shaders written win GLSL. To use these classes include  <QGLShader>.

We extend the RenderThread with a function to load a vertex and  fragment shaders given in two separate files. The filenames are passed  as two QString parameters.

```c++
    void QGLRenderThread::LoadShader(QString vshader, QString fshader)
    {
    if(ShaderProgram)
        {
        ShaderProgram->release();
        ShaderProgram->removeAllShaders();
        }
    else ShaderProgram = new QGLShaderProgram;

    if(VertexShader)
        {
        delete VertexShader;
        VertexShader = NULL;
        }

    if(FragmentShader)
        {
        delete FragmentShader;
        FragmentShader = NULL;
        }

    // load and compile vertex shader
    QFileInfo vsh(vshader);
    if(vsh.exists())
        {
        VertexShader = new QGLShader(QGLShader::Vertex);
        if(VertexShader->compileSourceFile(vshader))
            ShaderProgram->addShader(VertexShader);
        else qWarning() << "Vertex Shader Error" << VertexShader->log();
        }
    else qWarning() << "Vertex Shader source file " << vshader << " not found.";

    // load and compile fragment shader
    QFileInfo fsh(fshader);
    if(fsh.exists())
        {
        FragmentShader = new QGLShader(QGLShader::Fragment);
        if(FragmentShader->compileSourceFile(fshader))
            ShaderProgram->addShader(FragmentShader);
        else qWarning() << "Fragment Shader Error" << FragmentShader->log();
        }
    else qWarning() << "Fragment Shader source file " << fshader << " not found.";

    if(!ShaderProgram->link())
        {
        qWarning() << "Shader Program Linker Error" << ShaderProgram->log();
        }
    else ShaderProgram->bind();
    }
```

After some cleanup in case there was already a shader program active, we load first the vertex, then the fragment shader program and compile   them. When we are successful the shaders are linked to our  ShaderProgram  and thats it.

One good thing when using the Qt functions is that you get the error messages from the GLSL compiler for shader debugging purposes   (Shader->log()).

After that we can load our basic shaders

```c++
    void QGLRenderThread::run()
    {
        GLFrame->makeCurrent();
        GLInit();
        LoadShader("./Basic.vsh", "./Basic.fsh");
    //...snip...
```

The sample shader programs are pretty easy (=dull). I won't go into details here.

Vertex Shader:

```glsl
    void main(void)
    {
      gl_Position    = ftransform();
    }
```

Fragment Shader:

```glsl
    void main(void)
    {
      gl_FragColor = vec4(1.);
    }
```

## Conclusion

We have shown how to include GLSL-Shaders with Qt and render a simple white quad. More to follow.