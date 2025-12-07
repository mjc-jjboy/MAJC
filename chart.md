flowchart TD
    %% --- 样式定义 (配色方案：专业蓝/灰/绿) ---
    classDef sensor fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,rx:5,ry:5;
    classDef process fill:#ffffff,stroke:#37474f,stroke-width:2px,rx:5,ry:5;
    classDef algorithm fill:#fff3e0,stroke:#ef6c00,stroke-width:1px,stroke-dasharray: 5 5;
    classDef core fill:#ffebee,stroke:#c62828,stroke-width:3px,rx:5,ry:5;
    classDef output fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,rx:5,ry:5;
    classDef storage fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px;

    %% --- 0. 传感器输入层 ---
    subgraph Input ["0. 多源传感器输入"]
        direction LR
        Cam["📷 相机<br/>RGB / 红外 / 事件相机"]:::sensor
        LiDAR["📡 激光雷达<br/>3D LiDAR"]:::sensor
        IMU["🧭 IMU<br/>加速度计+陀螺仪"]:::sensor
    end

    %% --- 1. 预处理层 ---
    subgraph Preproc ["1. 自适应预处理与数据清洗"]
        direction TB
        Sync["⏱️ 硬/软时间同步"]:::process
        subgraph VisionPre ["视觉抗烟雾处理"]
            Dehaze["去雾增强<br/>Dark Channel Prior / AOD-Net"]:::algorithm
            Equali["对比度增强<br/>CLAHE"]:::algorithm
        end
        subgraph Keyframe ["智能关键帧筛选"]
            BlurCheck["模糊度/熵检测"]:::algorithm
            MotionCheck["视差/光流检测"]:::algorithm
            KF_Select{"关键帧<br/>决策"}:::process
        end
    end

    %% --- 2. 前端里程计 ---
    subgraph Frontend ["2. 紧耦合状态估计 (前端)"]
        direction TB
        PreInt["IMU 预积分"]:::process
        FeatExt["特征提取<br/>SuperPoint / ORB"]:::process
        FeatTrack["光流追踪 / 特征匹配"]:::process
        Deskew["点云去畸变"]:::process
        ScanMatch["点云配准<br/>NDT / ICP"]:::process
        Degradation{{"⚠️ 退化检测与权重仲裁<br/>(抗烟雾核心)"}}:::core
        LIO_VIO_Switch["因子图构建<br/>动态协方差调整"]:::process
    end

    %% --- 3. 后端优化与建图 ---
    subgraph Backend ["3. 滑窗优化与稠密建图"]
        direction TB
        SW_BA["📉 滑动窗口优化<br/>Sliding Window BA"]:::core
        Marg["边缘化 Marginalization<br/>保留历史先验"]:::process
        subgraph Loop ["全局一致性"]
            LoopDet["回环检测<br/>BoW / ScanContext"]:::algorithm
            PoseGraph["全局位姿图优化"]:::process
        end
        subgraph Mapping ["地图生成"]
            LocalMap["局部特征地图"]:::storage
            DenseRecon["🧱 稠密重建<br/>TSDF / Octomap"]:::output
            CleanMap["静态地图优化<br/>(滤除烟雾噪点)"]:::output
        end
    end

    %% --- 连接关系 ---
    Input --> Sync
    Sync --> Dehaze --> Equali --> KF_Select
    Sync --> BlurCheck --> KF_Select
    Sync --> MotionCheck --> KF_Select
    Sync --> PreInt
    Sync --> Deskew

    KF_Select --"高质量帧"--> FeatExt --> FeatTrack
    Deskew --> ScanMatch

    FeatTrack --> Degradation
    ScanMatch --> Degradation
    PreInt --> Degradation

    Degradation --"视觉失效: 降权\n激光/IMU: 升权"--> LIO_VIO_Switch
    LIO_VIO_Switch --> SW_BA

    SW_BA --> Marg --> SW_BA
    SW_BA --> LocalMap
    SW_BA --> LoopDet --> PoseGraph

    PoseGraph --> DenseRecon --> CleanMap
