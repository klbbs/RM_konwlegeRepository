sentry

工具

1. tools::Exiter()

   整个程序只能开一个，多开会报错，接受终端的指令"ctrl+c"，终止程序

2. tools::Plotter()

   输出debug信息到PlotJuggler, 端口9870,采用的AF_INET，所以能够跨机器调试，但是目前项目似乎没有使用这个能力

3. tools::Recoder()

   记录图像帧及其对应的云台姿态

IO

1. io::ROS2()

   创建发布和订阅节点，不采用ros2,无视咯

2. io::CBoard()

   接收串口信息，发生指令。

  	初始化串口通信，获取串口信息。
  	
  	接收两种信息:
  	
  	1. 位姿信息 x,y,z,w四元数信息
  	2. 射击信息:射速，射击模式，ft_angle
  	
  	发送命令信息:
  	
  	​	是否开火，yaw,pitch等
  	
  	此外还有同步数据用的四元数插值
  	
  	
