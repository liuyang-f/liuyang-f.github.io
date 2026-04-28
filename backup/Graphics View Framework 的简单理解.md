## 作用
  - 是QT提供的一个由View、Scene、Item组成的可以绘制大量组件的2D绘图框架；它支持缩放、旋转以及时间传递等功能。
  - 使用场景：比如实现一个类似windows上的画图工具，由于需要批量生成、处理一系列图形控件，可以考虑使用这个框架实现。

## 组成
  - Item：是一个个图形基本单元类型。
  - Scene：本身不可见，是一个存放Item的容器。
  - View：是一个可视化的widget类型，可用于显示Scene中的内容。多个View可以使用同一个Scene。

## QGraphicsScene
  QGraphicsScene 本身不可见，可以存放Item，是一个图形场景的集合，通过和View关联后，能够把一些列Item显示到不同的View窗口上。
  - 继承关系图：QGraphicsScene ==》 QObject
  - 管理大量的Item。
  - 把event传递到每一个Item。
  - 管理Item状态，比如Item的选择和焦点处理。
  ```
  QGraphicsScene scene;
  QGraphicsRectItem *rect = scene.addRect(QRectF(0, 0, 100, 100));
  QGraphicsItem *item = scene.itemAt(50, 50, QTransform());
  ```
  
## QGraphicsView
  QGraphicsView 是一个显示窗口，有大小位置属性，专门用于显示Scene保存的Item。
  - 继承关系图：QGraphicsView ==》QAbstractScrollArea ==》 QFrame ==》 QWidget
  - 带有滚动区域和滚动条。
  - 可以接收键盘和鼠标的输入事件。
  ```
  QGraphicsScene scene;
  myPopulateScene(&scene);
  QGraphicsView view(&scene);
  view.show();
  ```
  
## QGraphicsItem
  - QGraphicsItem 是 QGraphics 家族中图形单元的基类
  - Graphics View 提供的典型图形的Item有：
    - 矩形：QGraphicsRectItem 
	- 椭圆：QGraphicsEllipseItem
	- 文本项目：QGraphicsTextItem
	- 自定义：QGraphicsItem
  - QGraphicsItem 支持的功能：
    - 鼠标移动、点击、释放、双击、悬停、滚轮等events
	- 键盘和按键输入events
	- 分组，通过父子关系和QGraphicsItemGroup进行分组
	- Drag和drop events
	- 碰撞检测（Collision detection）