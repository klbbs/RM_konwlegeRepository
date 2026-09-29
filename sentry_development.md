sentry

工具

1. tools::Exiter()

   整个程序只能开一个，多开会报错，接受终端的指令"ctrl+c"，终止程序

2. tools::Plotter()

   输出debug信息到PlotJuggler, 端口9870,采用的AF_INET，所以能够跨机器调试，但是目前项目似乎没有使用这个能力

3. tools::Recoder()

   记录图像帧及其对应的云台姿态
   
4. 



