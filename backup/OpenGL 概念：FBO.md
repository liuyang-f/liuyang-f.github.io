
## 什么是 FBO

FBO 是应用程序创建的帧缓冲对象。它不保存像素，只保存附件点。

`glGenFramebuffers` 分配名字，第一次 `glBindFramebuffer` 才建立对象。

- 名字 0：窗口提供的默认帧缓冲
- 渲染到纹理：颜色附件是一张纹理
- 离屏渲染：画到默认帧缓冲以外的目标。附件也可以是 Renderbuffer

## Render to Texture

png 解码成纹理。shader 采样它，片元写入当前帧缓冲。目标是默认帧缓冲时，颜色进后台缓冲，交换缓冲后出现在窗口。

把绘制目标换成挂了纹理的 FBO，片元写入这张纹理。下一趟再把它绑到纹理单元，交给 sampler。

`png → shader(1) → 纹理 A → shader(2) → 纹理 B → … → 窗口`

同一趟绘制里，输入和输出必须是两张纹理。

## 相关函数

创建 FBO：

```cpp
GLuint fbo;
glGenFramebuffers(1, &fbo);

glBindFramebuffer(GL_FRAMEBUFFER, fbo);
```

FBO 不保存像素。准备一张空纹理，接收片元颜色：

```cpp
GLuint texture;

glGenTextures(1, &texture);

glBindTexture(GL_TEXTURE_2D, texture);

glTexImage2D(
    GL_TEXTURE_2D,
    0,
    GL_RGBA8,
    1024,
    1024,
    0,
    GL_RGBA,
    GL_UNSIGNED_BYTE,
    nullptr
);

// 只分配了 level 0，缩小过滤设为 GL_LINEAR
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
```

挂到颜色附件：

```cpp
glFramebufferTexture2D(
    GL_FRAMEBUFFER,
    GL_COLOR_ATTACHMENT0,
    GL_TEXTURE_2D,
    texture,
    0
);
```

绘制前确认帧缓冲完整，并把视口设成纹理大小：

```cpp
glCheckFramebufferStatus(GL_FRAMEBUFFER); // 期望 GL_FRAMEBUFFER_COMPLETE
glViewport(0, 0, 1024, 1024);
```

之后的绘制写入 `texture`。

`glBindFramebuffer(GL_FRAMEBUFFER, 0)` 把之后的绘制改回窗口。`fbo` 和附件都还在。画回窗口前，把 viewport 改回窗口大小。交换缓冲后画面才出现。

## 纹理的两个作用

**作为输出**

绘制目标指向 `fbo`，`GL_COLOR_ATTACHMENT0` 指向 `texture`。

```cpp
glBindFramebuffer(GL_FRAMEBUFFER, fbo);
drawScene();
```

默认帧缓冲的附件由窗口提供。名字 0 上不能调用 `glFramebufferTexture2D`。

**作为输入**

`texture` 绑到某个 Unit 的 `GL_TEXTURE_2D`，`sampler2D` 保存单元编号。

```cpp
glBindFramebuffer(GL_FRAMEBUFFER, 0);

glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, texture);

glUseProgram(program);
glUniform1i(glGetUniformLocation(program, "tex"), 0);
```

`uv` 由顶点着色器传入，一般在 `[0, 1]`。

```glsl
#version 330 core

in vec2 uv;
out vec4 FragColor;

uniform sampler2D tex;

void main()
{
    FragColor = texture(tex, uv);
}
```

`0` 是纹理单元编号。采样前把绘制目标改回窗口，或改到另一张输出纹理的 FBO。

## 经典场景

shader 采样一张纹理，片元写入另一张纹理。

`原始纹理 → shader(1) → 纹理 A → shader(2) → 纹理 B → … → 窗口`

## 如何理解 glBindFramebuffer

它切换之后的绘制写入哪个帧缓冲。

- `0`：窗口的默认帧缓冲。颜色进后台缓冲，交换缓冲后出现在窗口
- 其他名字：对应的 FBO。片元写入当前挂上的附件，可以是纹理或 Renderbuffer
