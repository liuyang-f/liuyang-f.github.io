
一句话总结：PBO 可以存放图像数据，是 CPU 与 GPU 之间像素数据的中转站。

## 什么是 PBO

PBO 全称 Pixel Buffer Object
它是一块由 GL 管理的内存，作为 CPU 与 GPU 之间的缓存。单个 PBO 只负责中转；两个以上的 PBO 轮流读写，CPU 与 GPU 才能并行处理数据。

PBO 与 Texture
  - 它们的共同点是，存放像素数据
  - PBO 是 GL 管理的缓冲区，用来搬运像素；Texture 存放的图像在 GPU 上，给 Shader 使用
  - PBO 连接 CPU 与 GPU，数据可以是：data → PBO → Texture → shader；也可以是：帧缓冲中的图像 → PBO → data。图像在 Texture 上，或在挂到 FBO 的 Renderbuffer 上；读屏幕时，图像在默认帧缓冲上
  - Texture 既可以是 shader 的输入端，也可以是 shader 的输出端：输入是 data → Texture → shader，输出是 shader 经 FBO 画进 Texture
PBO 与 VBO
  - 它们都是 GL 里的 Buffer Object，内存由驱动管理，用来存放数据
  - PBO 存放像素数据，VBO 存放顶点数据
PBO 与 FBO
  - 从名字来看，它们很像；但是就功能而言，不是一个层次的东西
  - FBO 用来解决 GPU 绘制的图像画到哪里：把 Texture（或 Renderbuffer）挂上去再画；画到屏幕用的是默认帧缓冲。FBO 自己不保存像素，要挂上图像才能画
  - PBO 用来在 CPU 与 GPU 之间传递图像数据，它衔接 CPU 交出的像素，和 GPU 帧缓冲里的图像


## PBO 连接了 CPU 与 GPU
- 数据流向
  - 先看数据从 CPU 到 GPU：`读取图像数据 → PBO → Texture → shader`
  - 再看数据从 GPU 到 CPU：`帧缓冲中的图像 → PBO → 读回图像数据`
  - 图像在 Texture 上，或在挂到 FBO 的 Renderbuffer 上；读屏幕时，图像在默认帧缓冲上
- PACK 与 UNPACK
  - 像素排成一份紧密的包裹，PBO 用来装这份包裹
  - 数据从 CPU 到 GPU 就是解包（UNPACK）：从包裹里取出像素交给 Texture，GPU 才能用
  - 数据从 GPU 到 CPU 就是打包（PACK）：把帧缓冲里的像素装进包裹，CPU 再来保存

## 为什么需要 PBO
首先没有 PBO 的情况下，数据无论从 CPU 到 GPU，还是从 GPU 到 CPU 都是可以传递的。
也就是说，PBO 有传递数据的能力，但它的重要意义在其他方面。

在 CPU 的多线程编程中，有一个双 buffer 的处理机制：
- 线程 A 从 buffer-1 读取数据
- 同时线程 B 往 buffer-2 写入数据
- 交换 buffer-1 与 buffer-2 的读写关系
- 线程 A 读到的就是线程 B 刚写好的那份数据
- 如此往复，实现数据的并行处理，提高效率

没错，CPU 与 GPU 之间也需要高效的处理数据，这就是 PBO 存在的意义。
- CPU → GPU
  - 像素数据写入 PBO1
  - 同时 GPU 从 PBO2 读取数据，送进 Texture，这是上一轮 CPU 写好的像素
  - 交换 PBO1 与 PBO2 的读写关系
  - 如此往复，并发执行

- GPU → CPU
  - GPU 把帧缓冲里的像素写入 PBO1
  - 同时 CPU 从 PBO2 获取像素数据，这份数据是上一轮写好的
  - 交换 PBO1 与 PBO2 的读写关系
  - 如此往复，并发执行

当然根据具体的场景需求，也可以是 3 个，4 个，n 个 PBO。
多备几个，是为了让上一轮的数据有时间准备好，不挡住当前这一轮。
从这个角度而言，PBO 是 CPU 与 GPU 之间的数据缓存空间。
