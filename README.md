# 使用 TensorBoard 可视化模型，数据和训练

> 原文： [https://pytorch.org/tutorials/intermediate/tensorboard\_tutorial.html](https://pytorch.org/tutorials/intermediate/tensorboard_tutorial.html)
> 
> 校验者：[yearing1017](https://github.com/yearing1017)

在 [60 分钟闪电战](https://pytorch.org/tutorials/beginner/deep_learning_60min_blitz.html)中，我们向您展示了如何加载数据，如何向定义为`nn.Module`子类的模型提供数据，如何在训练数据上训练该模型，以及在测试数据上对其进行测试。 为了了解发生的情况，我们在模型训练期间打印一些统计信息，以了解训练是否在进行中。 但是，我们可以做得更好：PyTorch 与 TensorBoard 集成，该工具旨在可视化神经网络训练运行的结果。 本教程使用 [Fashion-MNIST 数据集](https://github.com/zalandoresearch/fashion-mnist)说明了其某些功能，可以使用 _torchvision.datasets_ 将其读取到 PyTorch 中。

在本教程中，我们将学习如何：

> 1. 读取数据并进行适当的转换(与先前的教程几乎相同）。
> 2. 设置 TensorBoard。
> 3. 写入 TensorBoard。
> 4. 使用 TensorBoard 检查模型架构。
> 5. 使用 TensorBoard 来创建我们在上一个教程中创建的可视化的替代版本，代码量更少。

具体来说，在第 5 点，我们将看到：

> - 有两种检查训练数据的方法
> - 在训练模型时如何追踪其性能
> - 在训练完成后如何评估模型的性能。

我们将从 [CIFAR-10 教程](https://pytorch.org/tutorials/beginner/blitz/cifar10_tutorial.html)中类似的样板代码开始：

```
# imports
import matplotlib.pyplot as plt
import numpy as np

import torch
import torchvision
import torchvision.transforms as transforms

import torch.nn as nn
import torch.nn.functional as F
import torch.optim as optim

# transforms
transform = transforms.Compose(
    [transforms.ToTensor(),
    transforms.Normalize((0.5,), (0.5,))])

# datasets
trainset = torchvision.datasets.FashionMNIST('./data',
    download=True,
    train=True,
    transform=transform)
testset = torchvision.datasets.FashionMNIST('./data',
    download=True,
    train=False,
    transform=transform)

# dataloaders
trainloader = torch.utils.data.DataLoader(trainset, batch_size=4,
                                        shuffle=True, num_workers=2)

testloader = torch.utils.data.DataLoader(testset, batch_size=4,
                                        shuffle=False, num_workers=2)

# constant for classes
classes = ('T-shirt/top', 'Trouser', 'Pullover', 'Dress', 'Coat',
        'Sandal', 'Shirt', 'Sneaker', 'Bag', 'Ankle Boot')

# helper function to show an image
# (used in the `plot_classes_preds` function below)
def matplotlib_imshow(img, one_channel=False):
    if one_channel:
        img = img.mean(dim=0)
    img = img / 2 + 0.5     # unnormalize
    npimg = img.numpy()
    if one_channel:
        plt.imshow(npimg, cmap="Greys")
    else:
        plt.imshow(np.transpose(npimg, (1, 2, 0)))
```

我们将在该教程中定义一个类似的模型架构，仅需进行少量修改即可说明以下事实：图像现在是一个通道而不是三个通道，是 28x28 而不是 32x32：

```
class Net(nn.Module):
    def __init__(self):
        super(Net, self).__init__()
        self.conv1 = nn.Conv2d(1, 6, 5)
        self.pool = nn.MaxPool2d(2, 2)
        self.conv2 = nn.Conv2d(6, 16, 5)
        self.fc1 = nn.Linear(16 * 4 * 4, 120)
        self.fc2 = nn.Linear(120, 84)
        self.fc3 = nn.Linear(84, 10)

    def forward(self, x):
        x = self.pool(F.relu(self.conv1(x)))
        x = self.pool(F.relu(self.conv2(x)))
        x = x.view(-1, 16 * 4 * 4)
        x = F.relu(self.fc1(x))
        x = F.relu(self.fc2(x))
        x = self.fc3(x)
        return x

net = Net()
```

我们将在之前定义相同的`optimizer`和`criterion`：

```
criterion = nn.CrossEntropyLoss()
optimizer = optim.SGD(net.parameters(), lr=0.001, momentum=0.9)
```

## 1\. TensorBoard 设置

现在，我们将设置 TensorBoard，从`torch.utils`导入`tensorboard`并定义`SummaryWriter`，这是将信息写入 TensorBoard 的关键对象。

```
from torch.utils.tensorboard import SummaryWriter

# default `log_dir` is "runs" - we'll be more specific here
writer = SummaryWriter('runs/fashion_mnist_experiment_1')
```

请注意，仅此行会创建一个`runs/fashion_mnist_experiment_1`文件夹。

## 2.写入 TensorBoard

现在，使用 [make\_grid](https://pytorch.org/docs/stable/torchvision/utils.html#torchvision.utils.make_grid) 将图像写入到 TensorBoard 中，具体来说就是网格。

```
# get some random training images
dataiter = iter(trainloader)
images, labels = dataiter.next()

# create grid of images
img_grid = torchvision.utils.make_grid(images)

# show images
matplotlib_imshow(img_grid, one_channel=True)

# write to tensorboard
writer.add_image('four_fashion_mnist_images', img_grid)
```

现在运行

```
tensorboard --logdir=runs
```

从命令行，然后导航到 [https：// localhost：6006](https://localhost:6006/) 应该显示以下内容。

![intermediate/../../_static/img/tensorboard_first_view.png](img/8b09d6361316e495383ceedf9b8407ea.jpg)

现在您知道如何使用 TensorBoard 了！ 但是，此示例可以在 Jupyter Notebook 中完成-TensorBoard 真正擅长的地方是创建交互式可视化。 接下来，我们将介绍其中之一，并在本教程结束时介绍更多内容。

## 3.使用 TensorBoard 检查模型

TensorBoard 的优势之一是其可视化复杂模型结构的能力。 让我们可视化我们构建的模型。

```
writer.add_graph(net, images)
writer.close()
```

现在刷新 TensorBoard 后，您应该会看到一个“ Graphs”标签，如下所示：

![intermediate/../../_static/img/tensorboard_model_viz.png](img/8f596b99dbb3c262b61db267d5db2d63.jpg)

继续并双击“ Net”以展开它，查看组成模型的各个操作的详细视图。

TensorBoard 具有非常方便的功能，可在低维空间中可视化高维数据，例如图像数据； 接下来我们将介绍。

## 4.在 TensorBoard 中添加一个“投影仪”

我们可以通过 [add\_embedding](https://pytorch.org/docs/stable/tensorboard.html#torch.utils.tensorboard.writer.SummaryWriter.add_embedding) 方法可视化高维数据的低维表示

```
# helper function
def select_n_random(data, labels, n=100):
    '''
    Selects n random datapoints and their corresponding labels from a dataset
    '''
    assert len(data) == len(labels)

    perm = torch.randperm(len(data))
    return data[perm][:n], labels[perm][:n]

# select random images and their target indices
images, labels = select_n_random(trainset.data, trainset.targets)

# get the class labels for each image
class_labels = [classes[lab] for lab in labels]

# log embeddings
features = images.view(-1, 28 * 28)
writer.add_embedding(features,
                    metadata=class_labels,
                    label_img=images.unsqueeze(1))
writer.close()
```

现在，在 TensorBoard 的“投影仪”选项卡中，您可以看到这 100 张图像-每个图像 784 维-向下投影到三维空间中。 此外，这是交互式的：您可以单击并拖动以旋转三维投影。 最后，有两个技巧可以使可视化效果更容易看到：在左上方选择“颜色：标签”，并启用“夜间模式”，这将使图像更容易看到，因为它们的背景是白色的：

![intermediate/../../_static/img/tensorboard_projector.png](img/f4990a0920dff7e4647a23cfc1639a8a.jpg)

现在我们已经彻底检查了我们的数据，接下来让我们展示TensorBoard 如何从训练开始就可以使跟踪模型的训练和评估更加清晰。

## 5.使用 TensorBoard 跟踪模型训练

在前面的示例中，我们仅每 2000 次迭代打印该模型的运行损失。 现在，我们将运行损失记录到 TensorBoard 中，并通过`plot_classes_preds`函数查看模型所做的预测。

```
# helper functions

def images_to_probs(net, images):
    '''
    Generates predictions and corresponding probabilities from a trained
    network and a list of images
    '''
    output = net(images)
    # convert output probabilities to predicted class
    _, preds_tensor = torch.max(output, 1)
    preds = np.squeeze(preds_tensor.numpy())
    return preds, [F.softmax(el, dim=0)[i].item() for i, el in zip(preds, output)]

def plot_classes_preds(net, images, labels):
    '''
    Generates matplotlib Figure using a trained network, along with images
    and labels from a batch, that shows the network's top prediction along
    with its probability, alongside the actual label, coloring this
    information based on whether the prediction was correct or not.
    Uses the "images_to_probs" function.
    '''
    preds, probs = images_to_probs(net, images)
    # plot the images in the batch, along with predicted and true labels
    fig = plt.figure(figsize=(12, 48))
    for idx in np.arange(4):
        ax = fig.add_subplot(1, 4, idx+1, xticks=[], yticks=[])
        matplotlib_imshow(images[idx], one_channel=True)
        ax.set_title("{0}, {1:.1f}%\n(label: {2})".format(
            classes[preds[idx]],
            probs[idx] * 100.0,
            classes[labels[idx]]),
                    color=("green" if preds[idx]==labels[idx].item() else "red"))
    return fig
```

最后，让我们使用与之前教程相同的模型训练代码来训练模型，但是每 1000 批将结果写入 TensorBoard，而不是打印到控制台。 这是通过 [add\_scalar](https://pytorch.org/docs/stable/tensorboard.html#torch.utils.tensorboard.writer.SummaryWriter.add_scalar) 函数完成的。

此外，在训练过程中，我们将生成一幅图像，显示该批次中包含的四幅图像的模型预测与实际结果。

```
running_loss = 0.0
for epoch in range(1):  # loop over the dataset multiple times

    for i, data in enumerate(trainloader, 0):

        # get the inputs; data is a list of [inputs, labels]
        inputs, labels = data

        # zero the parameter gradients
        optimizer.zero_grad()

        # forward + backward + optimize
        outputs = net(inputs)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()

        running_loss += loss.item()
        if i % 1000 == 999:    # every 1000 mini-batches...

            # ...log the running loss
            writer.add_scalar('training loss',
                            running_loss / 1000,
                            epoch * len(trainloader) + i)

            # ...log a Matplotlib Figure showing the model's predictions on a
            # random mini-batch
            writer.add_figure('predictions vs. actuals',
                            plot_classes_preds(net, inputs, labels),
                            global_step=epoch * len(trainloader) + i)
            running_loss = 0.0
print('Finished Training')
```

现在，您可以查看“标量”选项卡，以查看在 15,000 次训练迭代中绘制的运行损失：

![intermediate/../../_static/img/tensorboard_scalar_runs.png](img/afda8238ecd1f547d61be4d155844f68.jpg)

此外，我们可以查看整个学习过程中模型在任意批次上所做的预测。 查看“图像”选项卡，然后在“预测与实际”可视化条件下向下滚动以查看此内容； 这向我们表明，例如，仅经过 3000 次训练迭代，该模型就能够区分出视觉上截然不同的类，例如衬衫，运动鞋和外套，尽管它并没有像后来的训练那样有信心：

![intermediate/../../_static/img/tensorboard_images.png](img/d5ab1f07cb4a9d9200c2a2d3b238340d.jpg)

在之前的教程中，我们研究了模型训练后的每类准确性； 在这里，我们将使用 TensorBoard 绘制每个类的精度回召曲线(很好的解释[在这里](https://www.scikit-yb.org/en/latest/api/classifier/prcurve.html)）。

## 6.使用 TensorBoard 评估经过训练的模型

```
# 1\. gets the probability predictions in a test_size x num_classes Tensor
# 2\. gets the preds in a test_size Tensor
# takes ~10 seconds to run
class_probs = []
class_preds = []
with torch.no_grad():
    for data in testloader:
        images, labels = data
        output = net(images)
        class_probs_batch = [F.softmax(el, dim=0) for el in output]
        _, class_preds_batch = torch.max(output, 1)

        class_probs.append(class_probs_batch)
        class_preds.append(class_preds_batch)

test_probs = torch.cat([torch.stack(batch) for batch in class_probs])
test_preds = torch.cat(class_preds)

# helper function
def add_pr_curve_tensorboard(class_index, test_probs, test_preds, global_step=0):
    '''
    Takes in a "class_index" from 0 to 9 and plots the corresponding
    precision-recall curve
    '''
    tensorboard_preds = test_preds == class_index
    tensorboard_probs = test_probs[:, class_index]

    writer.add_pr_curve(classes[class_index],
                        tensorboard_preds,
                        tensorboard_probs,
                        global_step=global_step)
    writer.close()

# plot all the pr curves
for i in range(len(classes)):
    add_pr_curve_tensorboard(i, test_probs, test_preds)
```

现在，您将看到一个“ PR Curves”选项卡，其中包含每个类别的精度回召曲线。 继续戳一下； 您会发现在某些类别中，模型的“曲线下面积”接近 100％，而在另一些类别中，该面积更低：

![intermediate/../../_static/img/tensorboard_pr_curves.png](img/d15de2be2b754f9a4f46418764232b5e.jpg)

这是 TensorBoard 和 PyTorch 与之集成的介绍。 当然，您可以在 Jupyter Notebook 中完成 TensorBoard 所做的所有操作，但是使用 TensorBoard，默认情况下，您还可以获得交互式视觉效果。