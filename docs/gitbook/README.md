---
description: Powered by ContinualAI
---

# Avalanche: an End-to-End Framework for Continual Learning Research

![](.gitbook/assets/avalanche_logo_with_clai.png)

**Avalanche** is an _end-to-end Continual Learning research_ framework based on [**Pytorch**](https://pytorch.org/), born within [**ContinualAI**](https://www.continualai.org/) with the unique goal of providing a **shared** and **collaborative** open-source **codebase** for _fast prototyping_, _training_ and _reproducible_ _evaluation_ of continual learning algorithms. 

Avalanche can help _Continual Learning_ researchers in several ways:

* _Write less code, prototype faster & reduce errors_
* _Improve reproducibility_
* _Improve modularity and reusability_
* _Increase code efficiency, scalability & portability_
* _Augment impact and usability of your research products_

The framework is organized in three main modules:

* **`Benchmarks`**: This module maintains a uniform API for data handling: mostly generating a stream of data from one or more datasets. It contains all the major CL benchmarks \(similar to what has been done for [torchvision](https://pytorch.org/docs/stable/torchvision/index.html)\).
* **`Training`**: This module provides all the necessary utilities concerning model training. This includes simple and efficient ways of implement new _continual learning_ strategies as well as a set pre-implemented CL baselines and state-of-the-art algorithms you will be able to use for comparison!
* **`Evaluation`**: This modules provides all the utilities and metrics that can help evaluate a CL algorithm with respect to all the factors we believe to be important for a continually learning system. It also includes advanced logging and plotting features, including native [Tensorboard](https://www.tensorflow.org/tensorboard) support.

_Avalanche_ is one of the first experiments of a **End-to-end Research Framework** for reproducible m_achine learning_ research where you can find _benchmarks_, _algorithms_ and _evaluation protocols_ **in the same place**.  
  
Let's make it together 👫 a wonderful ride! 🎈

Check out _how you code changes_ when you start using _Avalanche_! 👇

{% tabs %}
{% tab title="With Avalanche" %}
```python
import torch
from torch.nn import CrossEntropyLoss
from torch.optim import SGD

from avalanche.benchmarks.classic import PermutedMNIST
from avalanche.evaluation import EvalProtocol
from avalanche.evaluation.metrics import ACC
from avalanche.extras.models import SimpleMLP
from avalanche.training.strategies import Naive

# Config
device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")
n_tasks = 5
n_classes = 10
train_ep = 2
mb_size = 32

# model
model = SimpleMLP(num_classes=n_classes)

# CL Benchmark Creation
perm_mnist = PermutedMNIST(n_tasks)

# Train & Test
optimizer = SGD(model.parameters(), lr=0.001, momentum=0.9)
criterion = CrossEntropyLoss()
evaluation_protocol = EvalProtocol(metrics=[ACC(n_classes)]
cl_strategy = Naive(
    model, 'classifier', SGD(model.parameters(), lr=0.001, momentum=0.9),
    CrossEntropyLoss(), train_mb_size=mb_size, train_epochs=train_ep,
    test_mb_size=mb_size, evaluation_protocol=evaluation_protocol, device=device)

results = []
for task in perm_mnist:
    cl_strategy.train(task, num_workers=4)
    results.append(cl_strategy.test(task))
```
{% endtab %}

{% tab title="Without Avalanche" %}
```python
import torch
from torch.nn import CrossEntropyLoss
from torch.optim import SGD
from torchvision import transforms
from torchvision.datasets import MNIST
from torchvision.transforms import ToTensor, RandomCrop

# Config
device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")
n_tasks = 5
n_classes = 10
train_ep = 2
mb_size = 32

# model
class SimpleMLP(nn.Module):

    def __init__(self, num_classes=10, input_size=28*28):
        super(SimpleMLP, self).__init__()

        self.features = nn.Sequential(
            nn.Linear(input_size, 512),
            nn.ReLU(inplace=True),
            nn.Dropout(),
        )
        self.classifier = nn.Linear(512, num_classes)
        self._input_size = input_size

    def forward(self, x):
        x = x.contiguous()
        x = x.view(x.size(0), self._input_size)
        x = self.features(x)
        x = self.classifier(x)
        return x
model = SimpleMLP(num_classes=n_classes)

# CL Benchmark Creation
list_train_dataset = []
list_test_dataset = []
rng_permute = np.random.RandomState(seed)
train_transform = transforms.Compose([
    RandomCrop(28, padding=4),
    ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))
])
test_transform = transforms.Compose([
    ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))
])

# for every incremental step
for _ in range(n_tasks):
    # choose a random permutation of the pixels in the image
    idx_permute = torch.from_numpy(rng_permute.permutation(784)).type(torch.int64)

    # add the permutation to the default dataset transformation
    train_transform_list = train_transform.transforms.copy()
    train_transform_list.append(
        transforms.Lambda(lambda x: x.view(-1)[idx_permute].view(1, 28, 28))
    )
    new_train_transform = transforms.Compose(train_transform_list)

    test_transform_list = test_transform.transforms.copy()
    test_transform_list.append(
        transforms.Lambda(lambda x: x.view(-1)[idx_permute].view(1, 28, 28))
    )
    new_test_transform = transforms.Compose(test_transform_list)

    # get the datasets with the constructed transformation
    permuted_train = MNIST(root='./data/mnist',
                           download=True, transform=train_transformation)
    permuted_test = MNIST(root='./data/mnist',
                    train=False,
                    download=True, transform=test_transformation)
    list_train_dataset.append(permuted_train)
    list_test_dataset.append(permuted_test)

# Train
optimizer = SGD(model.parameters(), lr=0.001, momentum=0.9)
criterion = CrossEntropyLoss()

for task_id, train_dataset in enumerate(list_train_dataset):

    train_data_loader = DataLoader(
        train_dataset, num_workers=num_workers, batch_size=train_mb_size)
    
    for ep in range(train_ep):
        for iteration, (train_mb_x, train_mb_y) in enumerate(train_data_loader):
            optimizer.zero_grad()
            train_mb_x = train_mb_x.to(device)
            train_mb_y = train_mb_y.to(device)

            # Forward
            logits = model(train_mb_x)
            # Loss
            loss = criterion(logits, train_mb_y)
            # Backward
            loss.backward()
            # Update
            optimizer.step()

# Test
acc_results = []
for task_id, test_dataset in enumerate(list_test_dataset):
    
    train_data_loader = DataLoader(
        train_dataset, num_workers=num_workers, batch_size=train_mb_size)
    
    correct = 0
    for iteration, (test_mb_x, test_mb_y) in enumerate(test_data_loader):

        # Move mini-batch data to device
        test_mb_x = test_mb_x.to(device)
        test_mb_y = test_mb_y.to(device)

        # Forward
        test_logits = model(test_mb_x)

        # Loss
        test_loss = criterion(test_logits, test_mb_y)

        # compute acc
        correct += (test_mb_y.eq(test_logits.long())).sum()
    
    acc_results.append(len(test_dataset)/correct)
```
{% endtab %}
{% endtabs %}

## 🚦 Getting Started

We know that learning a new tool _may be tough at first_. This is why we made _Avalanche_ as easy as possible to learn with a set of resources that will help you along the way.  
  
For example, you may start with our _**5-minutes**_ **guide** that will let you acquire the basics about _Avalanche_ and how you can use it in your research project:

We have also prepared for you a large set of _**examples & snippets**_ you can plug-in directly into your code and play with:

Having completed these two sections, you will already feel with _superpowers_ ⚡, this is why we have also created an **in-depth tutorial** that will cover all the aspect of _Avalanche_ in details and make you a true _Continual Learner_! 👨‍🎓️

## 🗂️ Maintained by ContinualAI Research

![](.gitbook/assets/continualai_research_logo.png)

_Avalanche_ is the flagship open-source collaborative project of [**ContinuaAI**](https://www.continualai.org/#home): _a non profit research organziation and the largest open community on Continual Learning for AI_. __

The _Avalanche_ project is maintained by the collaborative research team [_**ContinualAI Research \(CLAIR\)**_](https://www.continualai.org/research/)_._ We are always looking for new _awesome members_, so check out our official website if you want to learn more about us and our activities.

Learn more about the [_**Avalanche Team and all the people who made it great**_](contacts-and-links/the-team.md)_!_
