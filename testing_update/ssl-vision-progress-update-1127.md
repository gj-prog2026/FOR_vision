# ssl-vision 测试进度更新

## 结论先行
官方 ssl-vision **能跑起来、能播放真实录像**,但**开箱即用的检测效果几乎是空的**——
相机标定、颜色分类全靠人工配置,官方默认值和我们的场景完全对不上。这其实从侧面
印证了我们今年做 tagvision(自动白线标定 + 环形标记)的必要性。

---

## 已完成

- [x] WSL2 + Ubuntu 环境搭建,解决了 `InstallPackagesUbuntu.sh` 不认 Ubuntu 26.04 的问题(改手动 `apt-get` 装依赖)
- [x] 编译成功(`make` 跑通)
- [x] 下载官方测试数据(`make install_test_data`),解决了缺 `unzip`、路径多一层文件夹的问题(建了个符号链接接上)
- [x] 跑通 GUI,能用 `Read from files` 模式播放官方录像(`Capture ~40fps`)

## 发现的问题

1. **测试数据严重不全**
   - 官方 zip 包里 `rc2022` 这批数据实际只有 **cam0、cam1 两台摄像头**的素材,没有 cam2/cam3
   - 每个场景文件夹只有 **4帧**(`00000~00003.png`),不是完整视频,信息量很有限
   - `rc2022` 的 cam0/cam1 实拍图分别只覆盖**半个场地**(cam0=下半场、cam1=上半场,在
     `go2024` 数据里验证了这个规律),这是真实多摄像头分区布局,不是数据损坏

2. **相机标定(外参)完全没生效**
   - `Camera Calibrator` 里的 `focal length / principal point / q0~q3 / tx,ty,tz`
     全部是出厂占位默认值(比如主点 390/290,但图像实际是 2448×2048,明显对不上)
   - 画面里红色几何参照线和白色真实场地线完全对不上
   - 根因:官方标定需要**人工操作**——手动拖4个 Control Point 到场地角,点
     "Do initial calibration" → "Detect additional points"(内部用 Sobel 算子找边缘)
     → "Do full calibration" 才能标出正确结果,不是自动的

3. **颜色分类线程数 = 0**
   - `Color Threshold → number of threads` 显示 `0`,这一步实际没在跑
   - 导致下游 `Blob Finding` / `Robot Detection` 全部没有输出,画面上看不到任何检测框
   - 颜色LUT本身也需要靠人工用鼠标在画面上点色样本来标定(`YUV Calibrator`),没有
     自动化手段

## 下一步计划

- [ ] 把 `number of threads` 从0改成1,确认颜色分类能不能先跑起来
- [ ] 按官方5步流程手动标定一次相机外参,看红框能不能和真实场地线对齐
- [ ] 尝试用我们自己拍的一段视频(需要先用 ffmpeg 拆成逐帧png)喂给 ssl-vision,
      测试官方算法在我们真实场地光照下的表现——这个需要贴官方彩色色块标记才能触发
      机器人检测,我们机器人目前没贴,这块可能需要先讨论要不要临时贴一套做对照实验
- [ ] 把"官方系统需要大量人工标定才能用"这一点整理成对比材料,作为我们做 tagvision
      自动化方案的论据
