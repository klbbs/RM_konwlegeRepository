一些需要了解的项目自制的yolo类。

1. auto_aim::YOLOBase

   yolo基类，只包含两个虚函数，要求继承此类的必须实现.

   1. detect: 需要图像帧和帧序号，返回装甲板结构体信息列表(list)，装甲板结构体在armor.cpp
   2. postprocess:同样需要图像帧和帧序号，还需要`Mat &output`，`double scale`.返回装甲板结构体信息列表(list).

   具体看子类实现。

2. auto_aim::YOLO

   是不同YOLO类的封装，统一封装为YOLO类，调用detect和postprocess时实际上调用内部具体的yolobase指针所指向的各种yolo类

   初始化：配置文件路径，是否启用debug(默认启用)

   有一个独占指针，类型为YOLOBase,会根据初始化过的配置文件，决定加载哪个yolo类,存在`YOLO5`,`YOLO8`,`YOLO11`这些基类派生类,调用YOLO类

猜测不同yolo类应该实现都差不多，所以这里挑选YOLO11类来看

3. YOLO11
   1. 

