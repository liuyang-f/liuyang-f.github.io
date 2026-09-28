

## 纹理

- **什么是纹理**

  纹理是 GPU 上的图像对象。png 先解码成像素，再上传。

- **为什么需要纹理**

  shader 按坐标采样图像，例如颜色、法线。

- **一张 png 如何交给 shader**

  - 先把 png 解码成像素
  - 再把像素从 CPU 传到 GPU
  - shader 按坐标采样

## OpenGL 中的函数

- `glGenTextures`：分配纹理名字（texture）
- `glActiveTexture`：选中纹理单元（`GL_TEXTURE0`）
- `glBindTexture`：把纹理对象绑到该单元的 `GL_TEXTURE_2D`
- `glTexImage2D`：把像素写入当前绑定的纹理对象
- `glGetUniformLocation`：取得 shader 里 `tex` 的 location
- `glUniform1i`：让 `tex` 指向纹理单元 0

数据流向：

`pixels → Texture Object → 绑到某个 Unit 的 GL_TEXTURE_2D → sampler 保存单元编号`

调用顺序：

`glActiveTexture → glBindTexture → glTexImage2D → glUseProgram → glGetUniformLocation → glUniform1i`

`glUniform1i` 写的是单元编号。

## 一些问题

- **1. 为什么需要纹理对象**
  - 纹理对象保存一份图像和它的采样参数
  - `glBindTexture` 之后，后续 `glTex*` 都作用在这个对象上
  - 只用一张图时，可以绑默认名字 0
  - 多张图就各生成一个对象。`glGenTextures` 分配名字，第一次 bind 才建立对象

- **2. `GL_TEXTURE0` 与 `GL_TEXTURE_2D`**
  - `GL_TEXTURE0` 是纹理单元，还有 `GL_TEXTURE1`、`GL_TEXTURE2`
  - `GL_TEXTURE_2D` 是单元上的 target，还有 `GL_TEXTURE_1D`、`GL_TEXTURE_CUBE_MAP`
  - 关系是 `unit[单元][target] = 纹理对象`
  - 纹理对象第一次绑定后，target 就固定了

- **3. `GL_TEXTURE0` 与 `glUniform1i` 的 0**
  - 都表示单元 0，传入的数不同
  - `glActiveTexture` 传 `GL_TEXTURE0`（`0x84C0`），表示正在修改单元 0
  - `glUniform1i` 传整数 `0`，表示 shader 从单元 0 采样
  - 单元 i：`glActiveTexture(GL_TEXTURE0 + i)`，`glUniform1i(loc, i)`

- **4. `glActiveTexture` 不是激活纹理的意思**
  - 就纹理来说，它多了一个概念：纹理单元（类似纹理数组），Active 就是选中具体哪一个纹理单元
  - 像 Buffer、VAO Object，它们就不需要 Active 函数；它们只有 `glGenBuffers` → `glBindBuffer`，`glGenVertexArrays` → `glBindVertexArray`

- **5. OpenGL 状态机**

```text
上下文
  Texture Unit          glActiveTexture 选中
    GL_TEXTURE_2D       glBindTexture
      Texture Object    glGenTextures
  Buffer Target         glBindBuffer
    Buffer Object       glGenBuffers
  VAO                   glBindVertexArray
    VAO Object          glGenVertexArrays
  FBO                   glBindFramebuffer
    Attachment
      Texture / Renderbuffer
  Program               glUseProgram
    glCreateShader → 编译 → glCreateProgram → attach → link
```

FBO 附件和纹理单元是两条绑定。

## C++ 代码

program 已链接，vao 已绑好索引缓冲，pixels 是解码后的紧密 RGBA8。

```cpp
// ===========================
// 1. 创建 Texture Object
// ===========================

GLuint texture;
glGenTextures(1, &texture);


// ===========================
// 2. 选择 Texture Unit 0
// ===========================

glActiveTexture(GL_TEXTURE0);


// ===========================
// 3. 把 Texture Object
//    绑定到 Unit 0
// ===========================

glBindTexture(GL_TEXTURE_2D, texture);


// ===========================
// 4. 上传图片
// ===========================

glTexImage2D(
    GL_TEXTURE_2D,
    0,
    GL_RGBA,
    width,
    height,
    0,
    GL_RGBA,
    GL_UNSIGNED_BYTE,
    pixels
);

// 只上传了 level 0，缩小过滤设为 GL_LINEAR
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);


// ===========================
// 5. Shader 中的 sampler
//    指向 Unit 0
// ===========================

glUseProgram(program);

GLint texLocation =
    glGetUniformLocation(program, "tex");

glUniform1i(texLocation, 0);


// ===========================
// 6. 绘制
// ===========================

glBindVertexArray(vao);

glDrawElements(
    GL_TRIANGLES,
    6,
    GL_UNSIGNED_INT,
    nullptr
);
```

## shader 代码

uv 由顶点着色器传入，一般在 [0, 1]。

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
