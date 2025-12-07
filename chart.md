graph TD
    %% 定义样式
    classDef input fill:#f9f9f9,stroke:#333,stroke-width:2px;
    classDef process fill:#e1f5fe,stroke:#0277bd,stroke-width:2px;
    classDef algo fill:#fff9c4,stroke:#fbc02d,stroke-dasharray: 5 5;
    classDef core fill:#ffebee,stroke:#c62828,stroke-width:4px;
    classDef output fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;

    %% --- 输入层 ---
    subgraph Sensors [0. 多源传感器输入]
        direction LR
        Cam[RGB/红外相机]:::input
        LiDAR[激光雷达]:::input
        IMU[惯性测量单元]:::input
    end

    %% --- 第一阶段：预处理 ---
    subgraph Phase1 [1. 自适应关键帧提取与硬核预处理]
        direction TB
        
        %% 同步
        Sync[时间戳硬同步/软同步]:::process
        
        %% 图像增强分支
        subgraph ImageProc [视觉抗烟雾增强]
            Dehaze[去雾算法<br/>(Dark Channel Prior / AOD-Net)]:::algo
            Contrast[直方图均衡化 CLAHE]:::algo
        end
        
        %% 关键帧策略
        subgraph KeyframeStrategy [智能关键帧策略]
            MotionCheck[视差/光流检测]:::algo
            BlurCheck[图像模糊度检测]:::algo
            Select[关键帧筛选]:::process
        end
        
        Cam --> Sync
        LiDAR --> Sync
        IMU --> Sync
        
        Sync --> Dehaze --> Contrast --> Select
        Sync --> MotionCheck --> Select
    end

    %% --- 第二阶段：前端里程计 ---
    subgraph Phase2 [2. 紧耦合状态估计 (抗烟雾核心)]
        direction TB
        
        %% 预积分
        PreInt[IMU预积分<br/>(处理高频运动)]:::process
        
        %% 视觉前端
        subgraph VisionFront [视觉前端]
            FeatExt[特征提取<br/>(SuperPoint/ORB)]:::algo
            FeatTrack[光流/特征匹配]:::algo
        end
        
        %% 激光前端
        subgraph LidarFront [激光前端]
            PointProcess[点云去畸变]:::algo
            ScanMatch[点云配准 NDT/ICP]:::algo
        end
        
        %% 核心：退化检测与权重调整
        DegradationCheck{{环境退化检测<br/>(烟雾浓度/特征点丢失率)}}:::core
        
        %% 因子图构建
        FactorGraph[局部因子图构建]:::process
        
        Sync --> PreInt
        Select --> FeatExt --> FeatTrack
        Sync --> PointProcess --> ScanMatch
        
        FeatTrack --> DegradationCheck
        ScanMatch --> DegradationCheck
        
        PreInt --> FactorGraph
        DegradationCheck --"动态调整协方差"--> FactorGraph
    end

    %% --- 第三阶段：后端优化与建图 ---
    subgraph Phase3 [3. 增量式滑窗优化与稠密建图]
        direction TB
        
        %% 优化求解
        BA[滑动窗口优化 (Sliding Window BA)]:::process
        Marg[边缘化 (Marginalization)<br/>保留历史约束]:::algo
        
        %% 全局一致性
        LoopClose[回环检测 (BoW / Scan Context)]:::process
        PoseGraph[全局位姿图优化]:::process
        
        %% 建图
        DenseMap[稠密重建 (TSDF/Octomap)]:::output
        LocalMap[局部地图维护]:::process
        
        FactorGraph --> BA
        BA --> Marg --> BA
        BA --> LocalMap
        BA --> LoopClose --> PoseGraph
        PoseGraph --> DenseMap
    end

    %% 连接各阶段
    Sensors --> Phase1
    Phase1 --> Phase2
    Phase2 --> Phase3
