1.以aloha_real为例，系统架构如下：
	相机/关节传感器  →  ROS 驱动节点(ROS1)  →  机器人运行脚本 examples/aloha_real.main
(Realsense/Interbotix)      ↑  /topics             │  (取图像/状态 → 发关节/夹爪命令)
                             │                      │
                             └──────────── gRPC/HTTP ─────→  模型服务 scripts/serve_policy.py
                                               (加载 checkpoint: pi0_base / towel / tupperware ...)


左侧：硬件与ROS驱动（最底层，和机器人/相机直连）；
中间：运行时控制脚本 examples.aloha_real.main（“环境层/控制层”，订阅图像与状态、发控制命令；同时把观测发给“模型服务”，拿回动作）；
右侧：模型服务  scripts/serve_policy.py（加载 checkpoint 做推理）；

2.各部分跑在那里
1）模型（推理）在那里跑 
	进程：（ scripts/serve_policy.py）
	启动：Without Docker：在终端 3 uv run scripts/serve_policy.py --env ALOHA --default_prompt='...'
	位置/机器：可以和机器人同机，也可以在元端GPU上（说明里指向 remote_inference.md，即支持“远程推理”）
	做什么：加载 gs://openpi-assets/checkpoints/... 的 checkpoint（例如 pi0_base、pi0_aloha_towel 等），接收来自运行脚本的相机画面/状态，输出每步关节+夹爪的动作。
2）底层硬件代码（ROS驱动）在那里跑
	进程：roslaunch aloha ros_nodes.launch（ALOHA 驱动、相机发布器等）
	启动：Without Docker：在终端 2 运行 roslaunch aloha ros_nodes.launch
	位置/机器：机器人控制端主机（和机械臂控制器、Realsense 直连的那台）
	做什么：发布四路相机图像（文档让你修改 realsense_publisher.py 以设置相机序列号）
	             暴露手臂/夹爪/关节状态的 ROS 话题；接收来自上层的控制话题（例如 Interbotix/ALOHA 的关节命令）
	             简而言之：和硬件打交道的所有节点都在这里
3）运行时控制脚本（环境/编排）在那里跑
	进程：python -m examples.aloha_real.main
	启动：Without Docker：在终端 1 里创建虚拟环境后运行 python -m examples.aloha_real.main
	位置/机器：既可以和 ROS 驱动同机，也可以同模型服务同机，但需要能访问 ROS 主机的网络（ROS1）
	做什么：订阅 ROS 图像与关节状态（从终端 2 启动的 ROS 驱动那里拿数据）；
	           把观测发给模型服务（终端 3 的 serve_policy.py），拿回动作；
	           把动作转成 ROS 关节/夹爪命令再发回给硬件（通过 ROS 话题）；
	           提供 Reset/场景初始化等流程（与 ALOHA 的 reset/相机/夹爪开合顺序有关）。



3.ros系统中机器人侧（以aloha为例），即服务端的“组件”清单：
	A. 自定义消息与相机发布（ALOHA 包）：
	 1.aloha/msg/RGBGrayscaleImage.msg
		功能：定义你客户端正在订阅的打包图像消息。结构通常是：
		std_msgs/Header header
		sensor_msgs/Image[] images   # images[0]=RGB(bgr8), images[1]=Depth(mono16)（若启用）
		你的 ImageRecorder.image_cb(...) 正是把 images[0] 当 RGB，用 CvBridge 转成 OpenCV。

	 2.aloha_scripts/realsense_publisher.py（或等价的相机发布器）
		功能：按相机序列号打开每台 Realsense，采集 color/depth，组装成上面的 RGBGrayscaleImage 并发布到：
		/cam_high
		/cam_low
		/cam_left_wrist
		/cam_right_wrist
		这是 README 里特别要求“修改以使用相机 serial number”的脚本。
		关键参数：~serial、~topic、~rgb、~depth、~frame_id。

	 3.aloha/launch/ros_nodes.launch（顶层启动器）
		功能：一键拉起四路相机发布器，以及后文的双臂驱动等节点。
		会为四个相机节点分别设定 serial、发布话题名、frame_id 等参数。
		配套：CMakeLists.txt 与 package.xml 要启用消息生成（message_generation、message_runtime 依赖），否则 RGBGrayscaleImage 这个类型在运行时不可用。

	
	
		
		
4.WR1机器人端ros2版本：humble;
  WR1机器人端export ROS_DOMAIN_ID=211;
  按如下方法修改pc端：
	1.网络配置
	确保两台电脑在同一局域网内并能互相ping通
	在两台电脑上设置相同的ROS_DOMAIN_ID（在.bashrc中添加：export ROS_DOMAIN_ID=<相同ID>，ID范围为0-232）
	关闭或配置防火墙以允许ROS2通信（默认使用UDP端口）
	步骤：
		打开 nano ~/.bashrc；
		在文件末尾添加环境变量 export ROS_DOMAIN_ID=211;
		如果使用nano编辑器，按Ctrl+X，然后按Y，最后按Enter；
		使更改生效source ~/.bashrc；
		验证是否生效echo $ROS_DOMAIN_ID；
	2.在新电脑上安装与原电脑相同版本的ROS2