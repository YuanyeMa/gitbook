# pytorch+tensorboardX 

简单总结一下tensorboardX的用法。

```python
import numpy as np 
from tensorboardX import SummaryWriter

write = SummaryWriter()

for epoch in range(100):
    write.add_scalar('scale/test', np.random.rand(), epoch)
    write.add_scalars('scale/scales_test', {'xsinx':epoch*np.sin(epoch), 'xcosx' : epoch*np.cos(epoch)}, epoch)

write.close()
```

在终端中运行上边代码:`python ./tensorboard_scales.py`，查看代码目录下多了一个`runs`的目录，里边保存的就是代码运行时的数据。  

![01](/images/tensorboardX/01.png)

再使用`tensorboard --logdir runs`,在浏览器地址栏输入`ip_address:6006`就能看到图像。

![04](/images/tensorboardX/04.png)

此时再次运行代码，再查看目录结构，会发现又多了一些目录，这是因为tensorboard会保存几个历史版本。这个功能很好用，可以通过点击左下角的复选框勾选要对比的曲线，进而对比模型的性能。

![02](/images/tensorboardX/02.png)

试试`graph`

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from tensorboardX import SummaryWriter


class Net1(nn.Module):
    def __init__(self):
        super(Net1, self).__init__()
        self.conv1 = nn.Conv2d(1, 10, kernel_size=5)
        self.conv2 = nn.Conv2d(10, 20, kernel_size=5)
        self.conv2_drop = nn.Dropout2d()
        self.fc1 = nn.Linear(320, 50)
        self.fc2 = nn.Linear(50, 10)
        self.bn = nn.BatchNorm2d(20)

    def forward(self, x):
        x = F.max_pool2d(self.conv1(x), 2)
        x = F.relu(x) + F.relu(-x)
        x = F.relu(F.max_pool2d(self.conv2_drop(self.conv2(x)), 2))
        x = self.bn(x)
        x = x.view(-1, 320)
        x = F.relu(self.fc1(x))
        x = F.dropout(x, training=self.training)
        x = self.fc2(x)
        x = F.softmax(x, dim=1)
        return x


dummy_input = torch.rand(13, 1, 28, 28)

model = Net1()
with SummaryWriter(comment='Net1') as w:
    w.add_graph(model, (dummy_input,))
```

在运行的时候报错了。

![error](/images/tensorboardX/error.png)

上网搜索了一番说是`pytorch`版本的问题，为了跑写好的强化学习的代码，我用的是比较老的`0.4.0`版本的，回头有机会测一下`pytorch 1.0+`的看可不可以用。

## 总结一下TensorboardX的用法

- 声明一个`SummaryWriter()`对象，`log_dir`参数能指定log文件保存的路径。
- 然后使用`add_xxx()`函数向log中添加内容，`xxx`可以是`scalar`, `image`, `figure`, `histogram`, `audio`, `text`, `graph`, `onnx_graph`, `embedding`, `pr_curve`, `mesh`, `hyper-parameters` and `video`，[详情参见](https://github.com/lanpa/tensorboardX)或者[tensorboardX](https://tensorboardx.readthedocs.io/en/latest/tensorboard.html)。
- 关闭`SummaryWriter()`对象
- 使用`tensorboard --logdir`命令启动`tensorboard`
- 在浏览器里查看输出。