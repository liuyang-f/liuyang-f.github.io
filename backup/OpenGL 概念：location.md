
一句话总结：location 是 CPU 与 Shader 之间定位变量的一种机制。

## 为什么需要 location

shader 中是一些计算公式，计算就需要数据，数据来自 CPU。
数据有很多类型，意味着有很多的变量，数据在 CPU 中是变量，在 GPU 中也是变量。
如何把 CPU 的变量与 shader 的变量进行对应，就需要一种机制。

shader 中的变量：

```glsl
layout(location = 0) in vec3 aPos;
layout(location = 1) in vec3 aColor;
layout(location = 2) in vec2 aUV;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;
```

CPU 中的变量：

```c
// Vertex Attribute
glVertexAttribPointer(0, ...);
glEnableVertexAttribArray(0);

glVertexAttribPointer(1, ...);
glEnableVertexAttribArray(1);

glVertexAttribPointer(2, ...);
glEnableVertexAttribArray(2);

// Uniform
GLint modelLocation =
    glGetUniformLocation(program, "model");

GLint viewLocation =
    glGetUniformLocation(program, "view");

GLint projectionLocation =
    glGetUniformLocation(program, "projection");
```

上面 3 个 attribute 的 location 是 shader 里写死的 0、1、2。
3 个 uniform 没有写 location，编号由链接器分配。
假如这次查询结果刚好是 0、1、2，对应关系就是：

```text
                    Shader Program
                         │
          ┌──────────────┴──────────────┐
          │                             │
    Vertex Inputs                    Uniforms
          │                             │
   Attribute Location              Uniform Location
          │                             │
    ┌─────┼─────┐                 ┌─────┼─────┐
    0     1     2                 0     1     2
    │     │     │                 │     │     │
    ↓     ↓     ↓                 ↓     ↓     ↓
  aPos  aColor  aUV             model  view projection
    ↑     ↑     ↑                 ↑     ↑     ↑
    └─────┴─────┘                 └─────┴─────┘
          VBO                           CPU
```

两套编号各自独立，图里两边都出现 0、1、2，并不冲突。

## 两种类型的 location

Vertex Attribute Location 是“顶点数据入口”的编号。
Uniform Location 是“Uniform 参数”的编号。
两套 Location 独立存在，数字可以重复，但没有任何影响。

```text
Vertex Attribute Location
0 → aPos
1 → aColor
2 → aUV

Uniform Location
0 → model
1 → view
2 → projection
```

## Vertex Attribute Location

顶点数据如何给到 shader？
反过来说，shader 需要使用顶点数据。
按照逆向追踪数据来源的思路，先看 shader：

```glsl
#version 330 core

layout(location = 0) in vec3 aPos;
layout(location = 1) in vec3 aColor;
layout(location = 2) in vec2 aUV;

out vec3 color;
out vec2 uv;

void main()
{
    gl_Position = vec4(aPos, 1.0);

    color = aColor;
    uv = aUV;
}
```

shader 定义了 3 个获取顶点数据的变量：aPos，aColor，aUV；并且分别给它们 location 的值设置为 0,1,2；
那么在 CPU 端必须存在对应的函数，与 0,1,2 进行关联；

接着看 OpenGL 对应函数：

```c
// Location 0 → position
glVertexAttribPointer(
    0,                       // location 0
    3,                       // 3 个 float
    GL_FLOAT,
    GL_FALSE,
    8 * sizeof(float),       // stride
    (void*)0                 // offset
);
glEnableVertexAttribArray(0);

// Location 1 → color
glVertexAttribPointer(
    1,                       // location 1
    3,                       // 3 个 float
    GL_FLOAT,
    GL_FALSE,
    8 * sizeof(float),       // stride
    (void*)(3 * sizeof(float)) // offset
);
glEnableVertexAttribArray(1);

// Location 2 → UV
glVertexAttribPointer(
    2,                       // location 2
    2,                       // 2 个 float
    GL_FLOAT,
    GL_FALSE,
    8 * sizeof(float),       // stride
    (void*)(6 * sizeof(float)) // offset
);
glEnableVertexAttribArray(2);
```

与 0,1,2 关联的分别是 glVertexAttribPointer 函数的第一个参数，glEnableVertexAttribArray 函数的参数。
这 2 个函数记录的是 VAO 的状态。调用时必须先绑定一个 VAO，并且 VBO 已经绑定到 GL_ARRAY_BUFFER。
简单来说，就是记下“VBO 里的数据怎么读”，读出来的结果分别对应 location 0,1,2。

那么 VAO 如何解读 VBO 的数据？
先看 VBO 数据：

```c
// 准备顶点数据
float vertices[] = {
    // position        // color        // UV
     0.0f,  0.5f, 0.0f,  1.0f, 0.0f, 0.0f,  0.5f, 1.0f,
    -0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,  0.0f, 0.0f,
     0.5f, -0.5f, 0.0f,  0.0f, 0.0f, 1.0f,  1.0f, 0.0f
};

// 创建 VBO 并顶点数据关联
GLuint vbo;
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(
    GL_ARRAY_BUFFER,
    sizeof(vertices),
    vertices,
    GL_STATIC_DRAW
);
```

分析顶点数据 vertices 可以发现：

- 格式为 position + color + UV
- position 有 3 个 float，color 有 3 个 float，UV 有 2 个 float，它们加起来一共是 8 个 float
- 这里的排列顺序只决定每段数据的偏移，和 location 编号没有对应关系。location 0 也可以去读 UV，只要 index 和 offset 配在一起

glVertexAttribPointer 连接 vertices 与 shader：

- 第 1 个参数 0,1,2 对应了 shader 中给 aPos、aColor、aUV 设置的 location 值
- 第 2 个参数是每个分类数据对应 vertices 中的数据个数；比如分别从 vertices 中取 3、3、2 个 float，对应 aPos、aColor、aUV；shader 中它们的类型刚好也是 vec3、vec3、vec2
- 第 5 个参数是步进值 stride
  - 步进值决定了取完一组顶点数据后，需要移动多远的距离，取下一组数据
  - vertices 本身是一个按固定格式循环重复的串行数据
  - 当按照 position(3) + color(3) + UV(2) 格式获取到第一个顶点的数据后，需要移动到下一个顶点位置取值
  - 每个顶点有 3+3+2=8 个 float，那么长度就是 8 * sizeof(float)，即步进值
- 第 6 个参数是偏移量 pointer
  - 所谓偏移量是针对一组顶点数据内部而言的，它决定了从一组顶点数据内部哪个位置取值
  - 它是一个字节偏移，类似游标卡尺的卡点，从哪一点读值
  - position 数据是起始的位置，也就是 (void*)0
  - color 数据相对于起始位置，偏离 position(3) 数据的长度，即 (void*)(3 * sizeof(float))
  - UV 数据相对于起始位置，偏移了 position(3) + color(3) 数据的长度，即 (void*)(6 * sizeof(float))

简单来讲，glVertexAttribPointer 记下了从 vertices 取值的规则。真正把数据送进 shader，发生在绘制的时候。

一个完整的链路图：

```text
                  Vertex Shader
              ┌─────────────────┐
              │                 │
location 0 ──→│ aPos            │
location 1 ──→│ aColor          │
location 2 ──→│ aUV             │
              │                 │
              └─────────────────┘
                    ↑
                    │
                  VAO
                    ↑
                    │
                   VBO
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
    position       color         UV
```

## Uniform Location

一次绘制里，uniform 对所有顶点都是同一个值，且数据量少。
顶点 attribute 则是每个顶点各有一份，数据量大。
所以 uniform 从 CPU 到 shader，不需要像顶点数据那样做格式解读。

先看 shader 代码：

```glsl
#version 330 core

layout(location = 0) in vec3 aPos;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main()
{
    gl_Position =
        projection *
        view *
        model *
        vec4(aPos, 1.0);
}
```

可以看到，shader 定义了 3 个 uniform 变量：model、view、projection，它们没有像顶点变量那样设置 location。
`#version 330` 也不能给 uniform 写 layout(location)。这个写法要到 GLSL 4.30。
那么 CPU 如何对应这 3 个 uniform 变量呢？

使用 glGetUniformLocation 获取 uniform location 值：

```c
GLint modelLocation =
    glGetUniformLocation(program, "model");

GLint viewLocation =
    glGetUniformLocation(program, "view");

GLint projectionLocation =
    glGetUniformLocation(program, "projection");
```

很明显，glGetUniformLocation 函数的第 2 个参数直接关联了 shader 中定义的 uniform 变量。
返回值是这个 program 里的编号，换一个 program 就要重新查询。
变量声明了但实际没用到时，链接器会把它优化掉，这里返回 -1。

接着，应该还有对应的变量和函数与 modelLocation、viewLocation、projectionLocation 进行关联才对。
比如 glUniformMatrix4fv 函数：

```c
// 假如存在三个矩阵数据
glm::mat4 model;
glm::mat4 view;
glm::mat4 projection;

// 把矩阵数据与 uniform location 关联
glUseProgram(program);

glUniformMatrix4fv(
    modelLocation,
    1,
    GL_FALSE,
    glm::value_ptr(model)
);

glUniformMatrix4fv(
    viewLocation,
    1,
    GL_FALSE,
    glm::value_ptr(view)
);

glUniformMatrix4fv(
    projectionLocation,
    1,
    GL_FALSE,
    glm::value_ptr(projection)
);
```

glUniformMatrix4fv 写入的是当前正在使用的 program。一个 mat4 只占 1 个 uniform location。
location 为 -1 时，这次调用会被忽略。

此时，一个完整的 uniform location 链路就完成了。
编号由链接器分配。假如这次查到的是：

```text
modelLocation      = 0
viewLocation       = 1
projectionLocation = 2
```

链路就是这样的：

```text
CPU
 │
 ├── glUniformMatrix4fv(0, ...)
 │         ↓
 │       model
 │
 ├── glUniformMatrix4fv(1, ...)
 │         ↓
 │       view
 │
 └── glUniformMatrix4fv(2, ...)
           ↓
       projection
```
