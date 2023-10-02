# Faces-HQ-Retouching-Base-BaidunetDIsk
#### [代码连接(https://aistudio.baidu.com/projectdetail/6714032)  

百度网盘AI大赛-美颜祛斑祛痘AB榜第一，决赛创新性太低，无名次(AB榜指标比现在第一指标高0.2db)    
个人感觉相比其他队伍还是有不小的创新，各位看官可自评自取    

# 图像处理挑战赛:美颜祛斑祛痘方案
# 一、赛题介绍
#### 背景
伴随颜值经济迅速崛起，人像美颜美肤逐渐成为应用最广、需求最大的技术之一。相较于互娱场景的磨皮美颜，广告级、影楼级的精细化美颜美肤给算法带来了更高的要求与挑战。本赛题希望参赛选手能研究出祛除斑痘瑕疵并保留皮肤纹理质感等高精度、低耗时的算法，助力解决广告及影楼级修图的痛难点，大幅提升工作效率与修图效果。           
![](https://ai-studio-static-online.cdn.bcebos.com/434c7b30dc1145e98be3d7ae647d2eb02aa6c796b05349b3aac365d91b664538)
#### 要求
机器配置：V100，显存15G，内存10G；    
模型大于10MB或单张图片耗时>50ms，决赛中的性能分数记0分；    

# 二、方案解析
#### 退化分析
![](https://ai-studio-static-online.cdn.bcebos.com/65e73abad2c844e497a72070960bc9dbc8213451558f4988a89bac9b7d416d52)
本赛题任务是属于图像对图像的修复任务：    
1.相比与去模糊，去摩尔纹等，退化存在严重的不均衡；    
2.相比于去水印，去遮挡等，背景区域也存在一定的退化；     
因此整个任务需要在退化过程的模拟上构造新的思路：背景区域修复和斑点区域修复，整个退化可用如下过程进行数学建模：    
**J(x) = K*I(x) + D(x) + I(x)  **
 
其中K代表全局退化，D代表严重的斑块损伤，I代表高清输入，J代表输出   

#### 网络设计
![](https://ai-studio-static-online.cdn.bcebos.com/b2325baf621c4a7db39b67e601281ed15b7526332a2c41808e376b0bf7850aaf)
网络整体结构为经典的编解码结构，再该结构的基础上做了如下改进:    
改进一：采用更深层次的降采样和上采样，获取更大感受野，捕获更多的信息以区分不同的退化等级。    
改进二：进行三级输入，为网络注入更多的信息。    
改进三：网络的最后一层不直接预测rgb图，而是根据退化建模进行参数预测来还原图像。    
网络设计参考论文：       
1.gUNet: https://arxiv.org/pdf/2209.11448v1.pdf    
网络的设计模型<10M，模型推理<50ms进行设计。   

具体的模块实现可参考论文。
#### 损失函数
采用多个损失函数进行监督，不同的损失函数在不同阶段进行逐级监督和还原，共用到如下损失函数：    
L1 Loss    
Color Loss    
PSNR Loss    
MSSSIMLoss    

# 三、训练策略
整个训练分为三个阶段，每个阶段针对不同的优化目标，逐级实现最终的优化目的。
#### 阶段一
使用大的学习率5e-4和大batch size 8 和patch size 480，使用L1 Loss和Color Loss进行网络优化，快速达到初步美颜的效果。该阶段训练结束后，实现初步的人脸的修复和色彩稳定。
#### 阶段二
使用小学习率5e-5进行finetune，同时仅使用左右翻转的数据增强以保持人脸先验信息。在保证第一阶段效果的前提下，通过使用PSNRLoss达到线上指标的提升。相比第一阶段，线上指标接近0.1的提升。
#### 阶段三
使用小学习率2e-5，使用Score Loss(PSNRLoss+MSSSIMLoss)进行finetune，进一步提升线上score，有小数点后近两位的提升。
  
# 三、其他尝试
### 不同类型的修复backbone：
累积尝试了十多种图像修复的结构仅尝试修改，最终基于gUNet的结构在速度和精度上达到最优的平衡，因此在gUNet上进行深度改进。
### 使用基于mask估计的方式：   
尝试使用单阶段mask估计方式，花费了不少时间，但是效果不佳。
### 提取mask区域进行mask偏移的数据增强：
提取mask区域，进行上下，左右等角度的偏移，生成更多的退化数据进行网络训练，实验了两次，没提升，后弃用。
### 进行随机resize的数据增强：
实验了两次，感觉提升很小，后弃用。
### 提取mask区域进行损失函数监督：
将像素区域差异在30以上的视为退化严重部分进行损失函数监督，未有提升。
```
            mask_loss, 难样本挖掘, 使用ssim损失,或者使用最新版本paddle,使用sfft 求损失
            mask_gt = paddle.abs(tensor_ref - tensor_damage) * 255
            # mask_gt[mask_gt>30] = 255 # 像素值差异在30以上，像素值保留，进行约束
            mask_gt[mask_gt<30] = 0 # 认为像素值差异在30以上，难学习，多加一个loss约束
            mask_pred = paddle.abs(out_tensor - tensor_damage) * 255
            mask_pred[mask_gt< 30] = 0
```

# 五、代码结构
/home/aistudio/work/test_code：      推理代码，包含最优模型      
/home/aistudio/work/train_code：     训练代码    
/home/aistudio/data             部分数据
训练时，逐级执行   
python train_stage1.py     
python train_stage3.py     
python train_stage2.py      
#### 在下面执行了代码，仅为展示训练过程的验证性训练
#### 默认参数在24G显存下可运行，OOM可尝试调整参数

 
