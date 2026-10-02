一些需要了解的项目自制的io和工具类。

1. `io::SocketCAN`

   作用：串口通信。

   接收串口名字，回调函数，设置了守护线程，如果程序没结束但是读取线程结束了，会尝试重新打开读取线程

    	1. 打开指定串口，设置回调函数，初始化文件描述符(-1)
    	2. epoll等待，读到数据后用回调函数(此函数在cboard.cpp)解析
    	3. 关闭套接字

2. `tools::ThreadSafeQueue`

   其实就是单独开一个线程的，处理好锁的queue队列，队列满时自动弹出(也可以替换为别的回调函数)，push,pop等都进行了锁处理。

3. `io::Command`

   想要发出去的命令.只是一个结构体，内容如下
   ```cpp
   struct Command
   {
     bool control;
     bool shoot;
     double yaw;
     double pitch;
     double horizon_distance = 0;  //无人机专有
   };
   ```

   