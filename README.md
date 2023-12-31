# FacePerceptionProject
**A compneuro project about neural activity reflecting face perception**  
![image](https://github.com/Rayz357/FacePerceptionProject/assets/60376633/d8c271a0-ca08-4035-ada8-78538e745fb5)


Step 1:  绘图,每一个电极对面部/房子的平均无噪声响应。为进一步分析确定正确的时间间隔，也就是区分面部/房子所需的时间间隔  
Step 2: 构想出一个指标，这个指标能描述电极区分面部和房子的能力  
Step 3: 在有噪声的面部数据中，得到含不同噪声占比的对面部/房子的平均响应，重新计算选择性指标以确认这些电极确实具有选择性   
Step 4: 基于TP和FN率计算对象的心理测量表现    
Step 5: 现在可以对特定噪声水平的面部图片，评估对象的表现  
Q6: 能否通过在单次试验中，某个通道检测到了响应来预测被试会出现错误判断（而非正确判断）？如果可以，那么说明该通道的神经响应直接反应了对噪声的感知  
Q6->8: 结合多个通道之间的信息。可能要用到sklearn的解码器。估计使用两个信息独立的通道建立的预测模型比起单一通道的预测模型会有多大的提升
Q7: 基于被试的响应，做一个电极的trail-trail变化性。把这个和一个空模型作比较，  

## 数据和预处理
**主题和任务**：有ECoG植入物的受试者在临床环境中进行实验。第一个实验中被动展示了面孔和房屋的图像（dat1），第二个实验中对图像添加噪声，并要求受试者通过按键检测面孔（dat2）。
采样率：始终为1000Hz。  
**滤波和预处理**：ECoG数据经过了特定频率的陷波滤波，然后通过时间进行z-scoring转换，并转换为float16以减小大小。  
**实验1的数据结构**：dat1字典包括了连续电压数据、采样率、刺激开始和结束时间、刺激标识以及大脑表面的3D电极位置。  
以下代码针对dat1['V']的连续电压数据进行了信号处理：
```python
# quick way to get broadband power in time-varying windows
from scipy import signal

V = dat1['V'].astype('float32')
#高通滤波：使用Butterworth高通滤波器，截止频率为50Hz，以消除低于50Hz的频率成分。这有助于消除慢波动和可能的噪声。  
b, a = signal.butter(3, [50], btype='high', fs=1000)
V = signal.filtfilt(b, a, V, 0)
#信号平方：通过将滤波后的信号平方，计算每个采样点的能量。这可以突出信号的振幅变化。
V = np.abs(V)**2
#如果信号具有多个频率成分，则平方操作会导致每个成分与所有其他成分（包括其自身）相乘，从而产生新的频率
#低通滤波：使用Butterworth低通滤波器，截止频率为10Hz，来平滑信号的能量轨迹。这有助于消除高频噪声并保留低频能量变化。  
b, a = signal.butter(3, [10], btype='low', fs=1000)
V = signal.filtfilt(b, a, V, 0)
#归一化：通过除以其平均值来对信号进行归一化，从而使其具有相同的量纲，便于后续分析。 
V = V/V.mean(0)
```
这段代码通过对连续电压信号的滤波和转换，得到了一个可以用于后续分析的信号特征。  
```python
# average the broadband power across all face stimuli and across all house stimuli

nt, nchan = V.shape# 获取时间点的数量nt和通道数nchan
nstim = len(dat1['t_on'])#获取刺激(开始时间)t_on的数量nstim

trange = np.arange(-200, 400)
ts = dat1['t_on'][:, np.newaxis] + trange #每一行代表一个刺激事件，并包括从刺激开始的-200到399的样本索引。通过使用 np.newaxis，这一部分将一维数组转换为二维列向量。 dat1['t_on'] 的形状为 (nstim,)，那么 dat1['t_on'][:, np.newaxis] 的形状将是 (nstim, 1)。  
#trange 是一个包括 -200 到 399 的整数的一维数组。通过将上述二维列向量与这个一维数组相加，使用 NumPy 的广播规则，我们实际上将 trange 的值加到了 dat1['t_on'] 中的每一个元素上。这样，每一行都会加上同样的 trange 值。结果是一个二维数组 ts，其中每一行代表一个刺激事件，并且包括了从刺激开始的 -200 到 399 的样本索引。  
V_epochs = np.reshape(V[ts, :], (nstim, 600, nchan))#结果是一个三维数组 V_epochs，每个元素 (i, j, k) 包括第 i 个刺激事件在时间 j 的第 k 个通道的电压值

V_house = (V_epochs[dat1['stim_id'] <= 50]).mean(0)#在刺激的维度上求平均，也就是将所有刺激索引的相同第二第三维相加求平均，也就是对每个相同时间，相同通道的平均响应
V_face = (V_epochs[dat1['stim_id'] > 50]).mean(0)
# 可以得到不同通道在时间间隔内的响应曲线

# let's find the electrodes that distinguish faces from houses
plt.figure(figsize=(20, 10))
for j in range(50):#遍历50个通道
  ax = plt.subplot(5, 10, j+1)
  plt.plot(trange, V_house[:, j])
  plt.plot(trange, V_face[:, j])
  plt.title('ch%d'%j)
  plt.xticks([-200, 0, 200])
  plt.ylim([0, 4])
plt.show()# 发现46号电极对面部图像刺激有明显响应，而对房屋图像没有；43号电极对房屋图像刺激有明显响应
#可视化特定电极（在这里是电极 46）对所有刺激的响应。响应被组织为图像，并按照刺激 ID 进行排序，使得房屋和脸部刺激分别聚在一起
# let's look at all the face trials for electrode 46 that has a good response to faces
# we will sort trials by stimulus id (1-50 is houses, 51-100 is faces)
plt.subplot(1, 3, 1)
isort = np.argsort(dat1['stim_id'])#dat1['stim_id']原本是乱序的，这里对刺激序号进行小到大排序
plt.imshow(V_epochs[isort, :, 46].astype('float32'),#纵轴每一行代表一个刺激（序号），横轴为时间，颜色深浅代表46号电极响应
           aspect='auto', vmax=7, vmin=0,
           cmap='magma')
plt.colorbar()
plt.show()
#类似的，对43号电极进行绘制
```

### Step 2: selectivity index
```latex
selectivity index=
mean response to faces-mean response to houses/mean response to faces+mean response to houses
```
这个指数的值范围从-1到1，其中0表示电极对面孔和房屋的响应完全相同，正值表示对面孔的更强响应，负值表示对房屋的更强响应。  
​

Step 1&2完成  
绘制出了43，46号电极在不同噪声分布时的dat2响应曲线  
![image](https://github.com/Rayz357/FacePerceptionProject/assets/60376633/751361c5-13f7-49a8-a660-1b7e7d929742)
## Daily Progress:
8-11
- 确定数据集和大致问题
- TD: 弄懂数据集基本操作

## References:
- Miller, K. J., Hermes, D., Pestilli, F., Wig, G. S., and Ojemann, J. G. (2017). Face percept formation in human ventral temporal cortex. Journal of neurophysiology 118(5): 2614-2627. doi: 10.1152/jn.00113.2017
- Miller, K. J., Hermes, D., Witthoft, N., Rao, R. P., and Ojemann, J. G. (2015). The physiology of perception in human temporal lobe is specialized for contextual novelty. Journal of neurophysiology 114(1): 256-263. doi: 10.1152%2Fjn.00131.2015
- Miller, K. J., Schalk, G., Hermes, D., Ojemann, J. G., and Rao, R. P. (2016). Spontaneous decoding of the timing and content of human object perception from cortical surface recordings reveals complementary information in the event-related potential and broadband spectral change. PLoS computational biology 12(1): e1004660. doi: 10.1371/journal.pcbi.1004660
## 文献阅读
- Miller, K. J., Hermes, D., Pestilli, F., Wig, G. S., and Ojemann, J. G. (2017). Face percept formation in human ventral temporal cortex. Journal of neurophysiology 118(5): 2614-2627. doi: 10.1152/jn.00113.2017
- 这项研究探讨了人类脑部的颞下皮层（ventral temporal cortex）是如何在观看面部和其他物体时产生选择性活动的。特别是，研究集中于后颞中回褶皱（posterior fusiform gyrus）区域内的面部选择性区域。用宽带脑电图（broadband electrocorticographic）变化测量, 这些区域在观看噪声退化图像（面部和房屋）。发现当图像噪声增加到超过知觉阈值时，这些面部选择性区域的皮层活动参数减少。当噪声水平高于知觉阈值，以及对于房屋刺激时，活动保持在基线水平。这些发现表明，这些面部选择性区域可能是从简单视觉特征中提取面部感知的活动区域。这些区域存在于人类颞下视觉流（ventral visual stream）中面部感知形成的拓扑结构内，前者是由非选择性活动在早期视觉区域pericalcarine及与颞下皮层的后知觉亚区的全或无活动相协调。这种拓扑组织为面部感知的解剖学提供了生理基础，解释了颞叶损伤后不同的知觉缺陷。总之，这项研究探讨了脑部如何处理和识别面部，以及这些过程是如何受到图像质量（例如噪声水平）影响的。
