flowchart TD

%% ===================== 0. 多源数据输入 + 同步 =====================

subgraph S0[0. 多源数据输入与系统调度]
direction TB

A0[相机: RGB/IR/鱼眼] --> A4
A1[激光雷达: 16/32/64/128 线] --> A4
A2[IMU: 加速度/角速度] --> A4
A3[可选传感器: GPS / UWB / Wheel Odometry] --> A4

A4[时间同步模块<br>时间戳补偿 + 多传感器校准] --> A5
A5[数据缓存与队列管理<br>RingBuffer + Drop Frame 策略] --> A6
A6 --> S1
end


%% ===================== 1. 预处理模块 =====================

subgraph S1[1. 数据预处理与增强]
direction LR

subgraph S1A[图像增强链]
B0[亮度/模糊/噪声检测] --> B1[低光/过曝修正<br>多曝光融合]
B0 --> B2[图像去雾<br>Dark Channel / AOD-Net / LiDAR 约束]
B0 --> B3[锐化/去模糊<br>DeblurGAN / UNet]
end

subgraph S1B[点云预处理链]
B4[点云时间畸变校正<br>基于 IMU 积分] --> B5
B5[点云滤波<br>VoxelGrid / ROR] --> B6
B6[提取边缘点/平面点] --> B7
end

subgraph S1C[IMU 预处理链]
B8[IMU 噪声标定<br>Allan Variance] --> B9
B9[IMU 预积分] --> B10
end

B1 --> C0
B2 --> C0
B3 --> C0
B7 --> C0
B10 --> C0

end


%% ===================== 2. 前端里程计 =====================

subgraph S2[2. 多模态前端里程计]
direction TB

subgraph S2A[视觉前端]
C1[特征提取<br>SuperPoint/ORB/FAST] --> C2
C2[特征跟踪<br>KLT/光流金字塔] --> C3
C3[结构化特征匹配<br>双向匹配 + RANSAC] --> C7
end

subgraph S2B[激光前端]
C4[ICP / NDT 配准] --> C5
C5[Scan-to-Map 配准<br>LOAM/LIO-SAM] --> C7
end

subgraph S2C[语义辅助]
C6[语义分割<br>YOLO/Mask R-CNN] --> C8
C8[动态物体过滤<br>光流 + 深度残差] --> C7
end

C7[视觉/激光/语义融合匹配] --> C9
C9[初步位姿估计] --> S3
end


%% ===================== 3. 多传感器融合（前端-后端桥接） =====================

subgraph S3[3. 多传感器融合定位模块]
direction TB

D0[构建残差项<br>视觉/LiDAR/IMU/时间同步残差] --> D1
D1[外参在线标定<br>Cam-LiDAR / Cam-IMU] --> D2
D2[鲁棒核激活<br>Huber / Cauchy / Tukey] --> D3
D3[前端优化输出<br>位姿 + 不确定度协方差] --> S4

end


%% ===================== 4. 滑动窗口后端 =====================

subgraph S4[4. 滑动窗口优化（后端 BA）]
direction LR

E0[Sliding Window BA<br>优化变量：位姿/特征点/IMU Bias/外参] --> E1
E1[求解器<br>Levenberg-Marquardt / Dogleg] --> E2
E2[边缘化<br>Schur Complement + Hessian 保稀疏] --> E3
E3 --> S5

end


%% ===================== 5. 回环检测 & 全局优化 =====================

subgraph S5[5. 回环检测与全局一致性]
direction LR

F0[回环检测 1：视觉 BoW] --> F3
F1[回环检测 2：ScanContext（LiDAR）] --> F3
F2[回环检测 3：语义一致性匹配] --> F3

F3[回环候选验证<br>重投影误差 + ICP 几何一致性] --> F4
F4[全局 Pose Graph 优化<br>SE3/Sim3 优化] --> S6

end


%% ===================== 6. 地图模块（稀疏 + 稠密 + 语义） =====================

subgraph S6[6. 地图构建与输出]
direction TB

subgraph S6A[稀疏地图]
G0[三角化局部地图点] --> G1
G1[闭环后全局修正] --> G4
end

subgraph S6B[稠密建图]
G2[TSDF 融合] --> G3
G3[Mesh 重建<br>Marching Cubes] --> G4
end

subgraph S6C[语义地图]
G5[语义标签融合] --> G6
G6[动态物体清除] --> G4
end

G4[最终地图输出<br>稀疏 + 稠密 + 语义融合]
end


%% ===================== END =====================