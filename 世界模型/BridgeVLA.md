- **双阶段训练：**
    1. **2D热图预训练 (2D Heatmap Pre-training)：** 赋予 VLM 主干网络预测2D热图的能力。
    2. **3D动作微调 (3D Action Fine-tuning)：** 将预训练好的模型应用于3D机器人操作任务。
- **统一2D空间对齐 (Unified 2D Space Alignment)：** 这是 BridgeVLA 的核心，通过以下两个关键机制实现：
    - **3D输入到多视角2D图像投影：** 将3D点云观察投影到多个正交视角的2D图像，使其与 VLM 主干网络的2D图像输入对齐。
    - **2D热图作为动作预测输出：** 统一输入观察和输出动作到一致的2D图像空间，特别用于平移动作 (translational action) 的预测。
      
#### **step1 2D热图预训练 (2D Heatmap Pre-training)**

- **目的：** 弥补 VLM 预训练时预测 token 序列与下游任务预测热图之间的不兼容性，使 VLM 具备空间感知的热图预测能力。
- **数据：** 使用大规模物体检测数据集 (例如 RoboPoint 的120K物体检测分割数据)。
- **输入：** 图像 + 描述目标物体的文本提示。
- **输出：** 2D热图，高亮显示图像中目标物体的位置。
- **热图构建 (Ground-truth Heatmap)：**
    - 对于每个感兴趣的物体，根据其边界框中心 xcxc​ 和概率阈值 pminpmin​ 构建一个概率图 P(x)P(x) (高斯分布)。
    - 所有物体的概率图通过平均和归一化融合，得到最终的 ground-truth 热图 Hgt(x)Hgt(x)。
    - Hst(x)={P(x)if P(x)≥pmin0otherwiseHst(x)={P(x)0​if P(x)≥pmin​otherwise​
    - Hgt(x)=Havg(x)Σx∈ΩHavg(x)Hgt(x)=Σx∈Ω​Havg​(x)Havg​(x)​, 其中 Havg(x)=1NΣi=1NHist(x)Havg​(x)=N1​Σi=1N​Hist​(x)。
- **模型架构：** 使用 PaliGemma 作为 VLM 主干 (SigLIP 视觉编码器 + Gemma transformer)。
- **预测过程：** VLM 输出的图像 token 经过重新排列形成空间特征网格，然后通过凸上采样模块 (convex upsampling block) 转换为与输入图像分辨率相同的热图。
- **损失函数：** 采用交叉熵损失 (cross-entropy loss) 训练模型，以预测热图来定位图像中所有感兴趣物体的位置。
- **可扩展性：** 该方法具有高度可扩展性，原则上可以利用任何可被公式化为热图预测任务 (如关键点检测、语义分割) 的视觉-语言数据集。

#### step2 3D动作微调 (3D Action Fine-tuning)**

- **输入转换：** 从 RGB-D 图像重建场景点云。为了与 VLM 的2D图像输入对齐，将点云渲染为**三个正交投影图像**（顶视图、前视图、右视图），作为 VLM 主干的输入。
- **动作预测：**
    - **平移动作 (Translational Actions)：** 预训练的 VLM 主干针对这三张投影图像生成2D热图。这些热图将反投影回3D点云网格，得分最高的3D点确定了末端执行器在下一关键帧的平移位置。
    - **旋转、夹爪和碰撞标志 (Rotation, Gripper, Collision)：** 通过对每个投影图像的输出 token 进行最大池化 (max-pooling) 得到全局特征，并从每个视图的热图峰值提取局部特征。这些特征被拼接后，通过 MLP 预测欧拉角 (离散化为72个 bin)、二值夹爪动作和碰撞标志。
- **粗到细 (Coarse-to-fine) 策略：** 像先前工作一样，BridgeVLA 采用粗到细的策略来提高动作预测精度。在初始预测后，模型会在预测平移点周围裁剪一个长方体，并在裁剪后的点云上进行第二次前向传播以获取更精确的动作。
- **损失函数 (Training Loss)：** 微调的总损失由四部分组成： L=Ltrans+Lrot+Lgripper+LcollisionL=Ltrans​+Lrot​+Lgripper​+Lcollision​
    - LtransLtrans​：用于热图预测的交叉熵损失，监督平移动作。
    - LrotLrot​：用于旋转预测的交叉熵损失 (欧拉角离散化)。
    - LgripperLgripper​ 和 LcollisionLcollision​：用于夹爪动作和碰撞避免的二值交叉熵损失。
- **几何鲁棒性：** 训练过程中对点云和 ground-truth 动作联合应用随机刚体变换，以增强模型的几何鲁棒性。
  
  ![[Pasted image 20250921133900.png]]