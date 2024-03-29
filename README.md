# datasetq - A heap queue dataset sampler

<div>
<img src="https://badgen.net/badge/python/3.6+/blue">
<img src="https://badgen.net/github/license/Brandhsu/datasetq">
<img src="https://badgen.net/badge/code%20style/black?color=black">
</div>

A heap queue dataset sampler for loss-based priority sampling.

## Introduction

Typical machine learning datasets return samples in a fixed or random order independent of label quality. In contrast, `datasetq` considers label quality when returning samples. A priority queue (heapq) is used to automatically retrieve and or discard the most problematic samples (largest loss) as the model is training. Furthermore, `datasetq` gives individuals the ability to find difficult or poorly labeled samples based on how frequently a model visits it during training.

Using `datasetq` is the same as using any other machine learning dataset and additionally keeps track of the following parameters during model training.

1. id: unique id for a sample
2. loss: loss value for a sample
3. visits: frequency a sample is visited during training

## Implementation

By default, all of the data will be visited during the first epoch (can be configured up to the nth epoch), thereon after, the heap will be used to prioritize samples to train. The heap is a memory lookup table to the underlying dataset and will order samples based on an associated loss value. The intuition here is that by sampling based on loss, the model will spend more training iterations on samples that it struggles with. However, this makes the following assumption, that the model should optimize for large loss samples over low loss samples. It is often the case that large losses do not necessarily correlate to difficult examples; they can just be labeled poorly. To address this glaring issue, a naive solution is implemented: if a sample is visited too often, above a certain threshold, remove it from the training set.

Each sample in the dataset is given a unique immutable `id` which corresponds to its global position. The number of times a sample is trained on is also monitored which may be useful for debugging the model, training, or underlying dataset.

The goal of loss-based priority sampling is to mine for poorly labeled data, reduce training time, and improve overall model performance.

## Results

These experiments were carried out between March - April 2022.

To keep it brief, it works on the MNIST datasets but doesn't work on ImageNet, so in other words, it doesn't work **yet** 😒.

| Dataset         | Training Script                                       | Training Summary                                            | Training Visualization                                                  |
| --------------- | ----------------------------------------------------- | ----------------------------------------------------------- | ----------------------------------------------------------------------- |
| MNIST           | [mnist.sh](benchmarks/mnist/mnist.sh)                 | [mnist/summary.ipynb](benchmarks/mnist/summary.ipynb)       | [mnist/visualization.ipynb](benchmarks/mnist/visualization.ipynb)       |
| Kuzushiji-MNIST | [kmnist.sh](benchmarks/mnist/kmnist.sh)               | [mnist/summary.ipynb](benchmarks/mnist/summary.ipynb)       |                                                                         |
| Fashion-MNIST   | [fashion-mnist.sh](benchmarks/mnist/fashion_mnist.sh) | [mnist/summary.ipynb](benchmarks/mnist/summary.ipynb)       |                                                                         |
| ImageNet        | [imagenet.sh](benchmarks/imagenet/imagenet.sh)        | [imagenet/summary.ipynb](benchmarks/imagenet/summary.ipynb) | [imagenet/visualization.ipynb](benchmarks/imagenet/visualization.ipynb) |

```python
# Minor note regarding the configurations tested (random sampling is the same as loss-priority sampling with steps=1)
{"sampler": False, "shuffle": True} == {"sampler": True, "steps": 1, "shuffle": True}
```

## Installation

This package works with Python 3.6+.

```shell
$ pip install -e .
```

## Examples

An example of using `datasetq` with PyTorch.

<table>
<tr>
<th>Datasetq</th>
<th>PyTorch</th>
</tr>
<tr>
<td>

```python
import tqdm
import torch
import torch.nn as nn
import torch.nn.functional as F
import torch.optim as optim
from torch.optim.lr_scheduler import StepLR
from torchvision import datasets, transforms
from torch.utils.data import DataLoader

from datasetq.dataset import decorate_with_indices
from datasetq.sampler import HeapqSampler


class Net(nn.Module):
    def __init__(self):
        super(Net, self).__init__()
        self.conv1 = nn.Conv2d(1, 32, 3, 1)
        self.conv2 = nn.Conv2d(32, 64, 3, 1)
        self.dropout1 = nn.Dropout(0.25)
        self.dropout2 = nn.Dropout(0.5)
        self.fc1 = nn.Linear(9216, 128)
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
        x = self.conv1(x)
        x = F.relu(x)
        x = self.conv2(x)
        x = F.relu(x)
        x = F.max_pool2d(x, 2)
        x = self.dropout1(x)
        x = torch.flatten(x, 1)
        x = self.fc1(x)
        x = F.relu(x)
        x = self.dropout2(x)
        x = self.fc2(x)
        output = F.log_softmax(x, dim=1)
        return output


def train_step(model, data, target, device):
    data = torch.FloatTensor(data).to(device)
    target = torch.LongTensor(target).to(device)

    output = model(data)
    loss = F.nll_loss(output, target, reduction="none")

    return loss


def train(model, train_loader, batch_size, optimizer, device):
    model.train()
    for batch in tqdm.tqdm(train_loader):
        index, (data, target) = batch
        loss = train_step(model, data, target, device)

        optimizer.zero_grad()
        torch.mean(loss).backward()
        optimizer.step()

        train_loader.sampler.update(index.tolist(), {"loss": loss.tolist()})


# --- Configs
device = "cpu"
epochs = 14
lr = 1.0
gamma = 0.7
batch_size = 64
root = "."

# --- Data
transform = transforms.Compose(
    [transforms.ToTensor(), transforms.Normalize((0.1307,), (0.3081,))]
)
datasets.MNIST = decorate_with_indices(datasets.MNIST)
train_dataset = datasets.MNIST(root, train=True, download=True, transform=transform)
sampler = HeapqSampler(train_dataset)
train_loader = DataLoader(train_dataset, batch_size=batch_size, sampler=sampler)

# --- Model
model = Net().to(device)
optimizer = optim.Adadelta(model.parameters(), lr=lr)

# --- Training
scheduler = StepLR(optimizer, step_size=1, gamma=gamma)
for epoch in range(1, epochs + 1):
    train(model, train_loader, batch_size, optimizer, device)
    scheduler.step()
```

</td>
<td>

```python
import tqdm
import torch
import torch.nn as nn
import torch.nn.functional as F
import torch.optim as optim
from torch.optim.lr_scheduler import StepLR
from torchvision import datasets, transforms
from torch.utils.data import DataLoader

# from datasetq.dataset import decorate_with_indices
# from datasetq.sampler import HeapqSampler


class Net(nn.Module):
    def __init__(self):
        super(Net, self).__init__()
        self.conv1 = nn.Conv2d(1, 32, 3, 1)
        self.conv2 = nn.Conv2d(32, 64, 3, 1)
        self.dropout1 = nn.Dropout(0.25)
        self.dropout2 = nn.Dropout(0.5)
        self.fc1 = nn.Linear(9216, 128)
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
        x = self.conv1(x)
        x = F.relu(x)
        x = self.conv2(x)
        x = F.relu(x)
        x = F.max_pool2d(x, 2)
        x = self.dropout1(x)
        x = torch.flatten(x, 1)
        x = self.fc1(x)
        x = F.relu(x)
        x = self.dropout2(x)
        x = self.fc2(x)
        output = F.log_softmax(x, dim=1)
        return output


def train_step(model, data, target, device):
    data = torch.FloatTensor(data).to(device)
    target = torch.LongTensor(target).to(device)

    output = model(data)
    loss = F.nll_loss(output, target, reduction="none")

    return loss


def train(model, train_loader, batch_size, optimizer, device):
    model.train()
    for batch in tqdm.tqdm(train_loader):
        data, target = batch
        loss = train_step(model, data, target, device)

        optimizer.zero_grad()
        torch.mean(loss).backward()
        optimizer.step()

        # train_loader.sampler.update(index.tolist(), {"loss": loss.tolist()})


# --- Configs
device = "cpu"
epochs = 14
lr = 1.0
gamma = 0.7
batch_size = 64
root = "."

# --- Data
transform = transforms.Compose(
    [transforms.ToTensor(), transforms.Normalize((0.1307,), (0.3081,))]
)
# datasets.MNIST = decorate_with_indices(datasets.MNIST)
train_dataset = datasets.MNIST(root, train=True, download=True, transform=transform)
# sampler = HeapqSampler(train_dataset)
train_loader = DataLoader(train_dataset, batch_size=batch_size)

# --- Model
model = Net().to(device)
optimizer = optim.Adadelta(model.parameters(), lr=lr)

# --- Training
scheduler = StepLR(optimizer, step_size=1, gamma=gamma)
for epoch in range(1, epochs + 1):
    train(model, train_loader, batch_size, optimizer, device)
    scheduler.step()
```

</td>
</tr>
</table>

## Limitations

- Assumes each dataset sample is independent of one another
- Assumes large losses correlate to poorly labeled samples

## Citation

If you find this work useful, please consider citing it.

```
@misc{hsu2022datasetq,
  title        = {A heap queue dataset sampler for loss-based priority sampling},
  author       = {Brandhsu},
  howpublished = {https://github.com/Brandhsu/datasetq},
  year         = {2022}
}
```
