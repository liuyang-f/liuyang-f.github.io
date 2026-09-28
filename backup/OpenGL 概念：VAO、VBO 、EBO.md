
## 简介
- VBO（Vertex Buffer Object）
  - 用于存放顶点数据
  - 顶点坐标、UV、法线、颜色等都可以放在里面
  - 缓冲里是字节，具体类型由后面的属性设置来说明
  - 按顶点顺序绘制时，使用 glDrawArrays
- EBO（Element Buffer Object）
  - 用于存放索引
  - 索引告诉 GPU 按什么顺序复用 VBO 中的顶点
  - 一个索引对应一个完整顶点，顶点数据仍在 VBO 中
  - 按索引绘制时，使用 glDrawElements
- VAO（Vertex Array Object）
  - 不存放顶点数据
  - 记录属性怎么从 VBO 中取值，以及使用哪个 VBO、哪个 EBO
  - Core Profile 中，设置属性、绑定 EBO、执行绘制，都要先绑定 VAO
  - 设置属性的函数是 glVertexAttribPointer

## VBO 是数据容器
一个 3D 模型的顶点坐标、UV、法线、颜色，要在 GPU 中保存下来，VBO 就承担了这个责任。
它的本质是一段缓冲。下面这份顶点坐标用 float 存放：
```
float vertices[] = {
    // position
     0.0f,  0.5f, 0.0f,
    -0.5f, -0.5f, 0.0f,
     0.5f, -0.5f, 0.0f
};
```

接着使用 OpenGL 函数把数据放到 GPU 中：
```
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

这里是 3 个顶点，一共 9 个 float。要画出它们，还要先绑定 VAO，并用 `glVertexAttribPointer` 把 position 指到这个 VBO，见后面的 VAO 一节。绘制时，count 填顶点个数：
```
glDrawArrays(GL_TRIANGLES, 0, 3);
```

`glDrawArrays` 按这个顺序取 VAO 中已经启用的顶点属性。

## EBO 是为了高效使用 VBO 中的数据

一句话总结：VBO 存顶点，EBO 存顶点索引。

假如我们需要绘制 2 个三角形。每个三角形有 3 个顶点，不用索引时，VBO 中要放 6 个顶点。
这两个三角形共用一条边，6 个顶点里有 2 个是重复的。
使用 EBO 后，VBO 中只放 4 个顶点。EBO 保存的是这些顶点的编号，用编号就可以引用 VBO 中的整个顶点。
这样就实现了数据的复用，这就是 EBO 的作用。

比如这样 4 个顶点：
```
vertices = {
    A, // 索引 0
    B, // 索引 1
    C, // 索引 2
    D  // 索引 3
};
```

使用索引后：
```
unsigned int indices[] = {
    0, 1, 2, // 三角形 ABC
    0, 2, 3  // 三角形 ACD
};
```

对应的 OpenGL 函数。`GL_ELEMENT_ARRAY_BUFFER` 的绑定会记在当前 VAO 里，所以先绑定 VAO：
```
glBindVertexArray(vao);

GLuint ebo;

glGenBuffers(1, &ebo);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);

glBufferData(
    GL_ELEMENT_ARRAY_BUFFER,
    sizeof(indices),
    indices,
    GL_STATIC_DRAW
);

glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, nullptr);
```

`6` 是索引个数。`GL_UNSIGNED_INT` 和上面的 `unsigned int` 一致。`nullptr` 表示从索引缓冲的开头读取。绘制时，VAO 中的顶点属性也要已经设置好。

VBO + EBO 的组合关系：
```
VBO
 ├── Vertex 0 = A
 ├── Vertex 1 = B
 ├── Vertex 2 = C
 └── Vertex 3 = D

EBO
 ├── 0
 ├── 1
 ├── 2
 ├── 0
 ├── 2
 └── 3
```

索引 0 取到的是顶点 A。顶点里如果还有 UV、法线，会和坐标一起被取出来。

## VAO 是用来解释 VBO 数据的

很多时候，VBO 中的数据不仅仅是顶点坐标，还有很多其他类型。
需要一种方法来告诉 GPU 如何使用 VBO 中的数据，并把这份说明保存下来，这就是 VAO 的职责。

假如 VBO 中数据为：
```
float vertices[] = {
    // position          // UV
     0.0f,  0.5f, 0.0f,  0.5f, 1.0f,
    -0.5f, -0.5f, 0.0f,  0.0f, 0.0f,
     0.5f, -0.5f, 0.0f,  1.0f, 0.0f
};
```
既有 position 数据，也有 UV 数据，实际上 vertices 中每一行的数据格式是 `X Y Z U V`。
就 vertices 的第一行数据来说，OpenGL 需要知道：
```
0 ~ 2 → position
3 ~ 4 → UV
```
后面每一行都是一样的格式。

先绑定 VAO，再把这份 VBO 绑到 `GL_ARRAY_BUFFER`。`glVertexAttribPointer` 会把这一刻绑定的 VBO 记到对应属性上。之后换成别的 `GL_ARRAY_BUFFER`，该属性仍然读取原来的 VBO。
```
GLuint vao;

glGenVertexArrays(1, &vao);
glBindVertexArray(vao);

glBindBuffer(GL_ARRAY_BUFFER, vbo);
```

先获取 `0 ~ 2 → position` 的数据：
```
glVertexAttribPointer(
    0,
    3,
    GL_FLOAT,
    GL_FALSE,
    5 * sizeof(float),
    (void*)0
);

glEnableVertexAttribArray(0);
```
它表示，location 0 取 3 个 float，作为 position。

再获取 `3 ~ 4 → UV` 的数据：
```
glVertexAttribPointer(
    1,
    2,
    GL_FLOAT,
    GL_FALSE,
    5 * sizeof(float),
    (void*)(3 * sizeof(float))
);

glEnableVertexAttribArray(1);
```
它表示，location 1 取 2 个 float，作为 UV。location 要和 shader 里的 `layout(location = N)` 一致。

## 为什么需要 VAO

Core Profile 中，设置顶点属性、绑定 EBO、执行绘制，都要有一个已绑定的 VAO。一个模型用一个 VAO，就可以把这套配置留住。

假如有 `模型 A 模型 B 模型 C` 3 个模型数据，它们的数据格式也不一样：
```
模型 A：
position + UV

模型 B：
position + normal + UV

模型 C：
position + color
```
那么 OpenGL 需要知道：每个模型的 VBO 数据怎么对应到 shader 的 location，又该用哪一个 VBO 和 EBO。VAO 把这些配置保存起来。格式相同的两个模型，只要 VBO 或 EBO 不同，也各用一个 VAO。

使用模型 A 的数据就是：
```
glBindVertexArray(vao_A);
glDrawElements(GL_TRIANGLES, indexCountA, GL_UNSIGNED_INT, nullptr);
```

使用模型 B 的数据就是：
```
glBindVertexArray(vao_B);
glDrawElements(GL_TRIANGLES, indexCountB, GL_UNSIGNED_INT, nullptr);
```

模型没有索引时，改为调用 `glDrawArrays`，count 填该模型的顶点个数。

## 总结

为了存储数据，OpenGL 引入了 VBO 的概念；数据可以是顶点坐标、UV、法线、颜色等，具体类型在设置属性时声明。
为了避免 VBO 存放重复的顶点，OpenGL 引入了 EBO 的概念；使用顶点索引可以重复使用 VBO 的数据。
为了记住 VBO 中的数据怎么读取、读取哪一个缓冲，OpenGL 引入了 VAO 的概念。

一句话总结：VBO 是数据池，EBO 给 VBO 瘦身，VAO 给 VBO 翻译。
