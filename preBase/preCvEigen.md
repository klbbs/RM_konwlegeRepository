# Opencv and Eigen 装甲板解算相关库函数



## CV库:

#### eigen2cv(Eigen::Matrix,cv::Mat)

​	顾名思义，将Eigen库里面的Matrix矩阵转换为cv库里面的矩阵Mat
​	需要注意的是如果Eigen矩阵是按列优先存储的话，转换后会变成其转置

#### PNP解算函数

```cpp
bool cv::solvePnP(
    InputArray objectPoints,
    InputArray imagePoints,
    InputArray cameraMatrix,
    InputArray distCoeffs,
    OutputArray rvec,
    OutputArray tvec,
    bool useExtrinsicGuess = false,
    int flags = SOLVEPNP_ITERATIVE
);
```

​	参数
​	objectPoints: 目标在3d世界里的一些真实点,3D点集
​	imagePoints: 那些点对应的在图像里面的位置,2D点集
​	cameraMatrix: 相机内参矩阵
$$
\begin{bmatrix}{}
f_x & 0 & c_x \\
0 & f_y & c_y \\
0 & 0 & 1
\end{bmatrix}
$$
​	distCoeffs: 相机畸变数组
$$
\begin{array}{}
(k1, k2, p1, p2, k3)
\end{array}
$$
​	rvec: 输出的旋转向量 
​	tvec: 输出的平移向量
​	bool useExtrinsicGuess = false: 是否使用初始的rvec和tvec进行推测
​	flags: 求解方法，这里默认是cv::SOLVEPNP_IPPE,此方法专注平面目标，如果3d点不共面不可靠

#### void cv::Rodrigues(InputArray src,OutputArray dst,OutputArray jacobian = noArray());

​	将旋转向量转换为旋转矩阵
​	参数:
​	src: 输入的旋转向量，通常为1\*3或3\*1
​	dst: 输出的旋转矩阵, 3\*3

​	

​	

## Eigen库:

#### Eigen::Matrix3d::Identity()
3d表示3\*3,Identity表示单位矩阵，代表创建一个3\*3的单位矩阵,类型为Matrix3d

#### Eigen::Matrix<double, 3, 3, Eigen::RowMajor>
构造一个3\*3的矩阵，数据类型double，存储方式行为主(默认是列为主，内存为行连续)



