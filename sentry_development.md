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

   此类含有的信息如下：

   1. 当前状态

   ```cpp
   enum Mode
   {
     idle,
     auto_aim,
     small_buff,
     big_buff,
     outpost
   };
   ```
	如果是烧饼，还有这些
   ```cpp
   enum ShootMode
   {
     left_shoot,
     right_shoot,
     both_shoot
   };
   ```

   还有弹速，无人机还有一个`double ft_angle`

   imu的信息
   
   ```cpp
   struct IMUData
   {
       Eigen::Quaterniond q;
       std::chrono::steady_clock::time_point timestamp;
   };
   ```
   
   还有管理imu数据的线程队列的`ThreadSafeQueue`.
   
   

