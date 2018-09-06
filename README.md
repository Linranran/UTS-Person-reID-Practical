# UTS-Person-reID-Practical
By [Zhedong Zheng](http://zdzheng.xyz/)

**This repo is no longer maintained. You may check the latest version at [here](https://github.com/layumi/Person_reID_baseline_pytorch/tree/master/tutorial).**

This is a [University of Technology Sydney](https://www.uts.edu.au) computer vision practical, authored by Zhedong Zheng.
The practical explores the basic of learning pedestrian features. In this pratical, we will learn to build a simple person re-ID system step by step. Any suggestion is welcomed.

![](https://github.com/layumi/Person_reID_baseline_pytorch/blob/master/show.png)

Person re-ID can be viewed as an image retrieval problem. Given one query image in Camera **A**, we need to find the images of the same person in other Cameras. The key of the person re-ID is to find a discriminative representation of the person. Many recent works apply deeply learned model to extract visual features, and achieve the state-of-the-art performance.

## Prerequisites
- Python 3.6
- GPU Memory >= 6G
- Numpy
- Pytorch 0.3+ (http://pytorch.org/)
- Torchvision from the source
```
git clone https://github.com/pytorch/vision
cd vision
python setup.py install
```

## Getting started
Check the Prerequisites. The download links for this practical are:

- Code: [Practical-Baseline](https://github.com/layumi/Person_reID_baseline_pytorch)
- Data: [Market-1501](http://188.138.127.15:81/Datasets/Market-1501-v15.09.15.zip)

## Part 1: Training
### Part 1.1: Prepare Data Folder (`python prepare_data.py`)
You may notice that the downloaded folder is organized as:
```
├── Market/
│   ├── bounding_box_test/          /* Files for testing (candidate images pool)
│   ├── bounding_box_train/         /* Files for training 
│   ├── gt_bbox/                    /* We do not use it 
│   ├── gt_query/                   /* Files for multiple query testing 
│   ├── query/                      /* Files for testing (query images)
│   ├── readme.txt
```
Open and edit the script `prepare.py` in the editor. Change the fifth line in `prepare.py` to your download path, such as `\home\zzd\Download\Market`. Run this script in the terminal.
```bash
python prepare.py
```
We create a subfolder called `pytorch` under the download folder. 
```
├── Market/
│   ├── bounding_box_test/          /* Files for testing (candidate images pool)
│   ├── bounding_box_train/         /* Files for training 
│   ├── gt_bbox/                    /* We do not use it 
│   ├── gt_query/                   /* Files for multiple query testing 
│   ├── query/                      /* Files for testing (query images)
│   ├── readme.txt
│   ├── pytorch/
│       ├── train/                   /* train 
│           ├── 0002
|           ├── 0007
|           ...
│       ├── val/                     /* val
│       ├── train_all/               /* train+val      
│       ├── query/                   /* query files  
│       ├── gallery/                 /* gallery files  
```

In every subdir, such as `pytorch/train/0002`, images with the same ID are arranged in the folder.
Now we have successfully prepared the data for `torchvision` to read the data. 

```diff
+ Quick Question. How to recognize the images of the same ID?
```

### Part 1.2: Build Neural Network (`model.py`)
We can use the pretrained networks, such as `AlexNet`, `VGG16`, `ResNet` and `DenseNet`. Generally, the pretrained networks help to achieve a better performance, since it perserves some good visual patterns from ImageNet[1].

In pytorch, we can easily import them by two lines. For example,
```python
from torchvision import models
model = models.resnet50(pretrained=True)
```
You can simply check the structure of the model by:
```python
print(model)
```

But we need to modify the networks a little bit. There are 751 classes (different people) in Market-1501, which is different with 1,000 classes in ImageNet. So here we change the model to use our classifier.

```python
import torch
import torch.nn as nn
from torchvision import models

# Define the ResNet50-based Model
class ft_net(nn.Module):
    def __init__(self, class_num = 751):
        super(ft_net, self).__init__()
        #load the model
        model_ft = models.resnet50(pretrained=True) 
        # change avg pooling to global pooling
        model_ft.avgpool = nn.AdaptiveAvgPool2d((1,1))
        self.model = model_ft
        self.classifier = ClassBlock(2048, class_num) #define our classifier.

    def forward(self, x):
        x = self.model.conv1(x)
        x = self.model.bn1(x)
        x = self.model.relu(x)
        x = self.model.maxpool(x)
        x = self.model.layer1(x)
        x = self.model.layer2(x)
        x = self.model.layer3(x)
        x = self.model.layer4(x)
        x = self.model.avgpool(x)
        x = torch.squeeze(x)
        x = self.classifier(x) #use our classifier.
        return x
```

```diff
+ Quick Question. Why we use AdaptiveAvgPool2d? What is the difference between the AvgPool2d and AdaptiveAvgPool2d?
+ Quick Question. Does the model have parameters now? How to intialize the parameter in the new layer?
```
More details are in `model.py`. You may check it later, after you have gone through this practical.

### Part 1.3: Training (`python train.py`)
OK. Now we have prepared the training data and defined model structure.
Before we start training, the last thing is how to read data and their labels from the prepared folder.
Using `torch.utils.data.DataLoader`, we can obtain two iterators `dataloaders['train']` and `dataloaders['val']` to read data and label.
```python
image_datasets = {}
image_datasets['train'] = datasets.ImageFolder(os.path.join(data_dir, 'train'),
                                          data_transforms['train'])
image_datasets['val'] = datasets.ImageFolder(os.path.join(data_dir, 'val'),
                                          data_transforms['val'])

dataloaders = {x: torch.utils.data.DataLoader(image_datasets[x], batch_size=opt.batchsize,
                                             shuffle=True, num_workers=8) # 8 workers may work faster
              for x in ['train', 'val']}
dataset_sizes = {x: len(image_datasets[x]) for x in ['train', 'val']}
```

Let's train the model.
Yes. It's only about 20 lines. Make sure you can understand every line of the code.
```python
            # Iterate over data.
            for data in dataloaders[phase]:
                # get a batch of inputs
                inputs, labels = data
                now_batch_size,c,h,w = inputs.shape
                if now_batch_size<opt.batchsize: # skip the last batch
                    continue
                # print(inputs.shape)
                # wrap them in Variable, if gpu is used, we transform the data to cuda.
                if use_gpu:
                    inputs = Variable(inputs.cuda())
                    labels = Variable(labels.cuda())
                else:
                    inputs, labels = Variable(inputs), Variable(labels)

                # zero the parameter gradients
                optimizer.zero_grad()

                #-------- forward --------
                outputs = model(inputs)
                _, preds = torch.max(outputs.data, 1)
                loss = criterion(outputs, labels)

                #-------- backward + optimize -------- 
                # only if in training phase
                if phase == 'train':
                    loss.backward()
                    optimizer.step()
```
```diff
+ Quick Question. Why we need optimizer.zero_grad()? What happens if we remove it?
+ Quick Question. The dimension of the outputs is batchsize*751. Why?
```
Every 10 training epoch, we save a snapshot and update the loss curve.
```python
                if epoch%10 == 9:
                    save_network(model, epoch)
                draw_curve(epoch)
```

## Part 2: Extracting feature (`python test.py`)
In this part, we load the network weight (we just trained) to extract the visual feature of every image.
We need to import the model structure and then load the weight to the model.
```python
model_structure = ft_net(751)
model = load_network(model_structure)
```
For every query and gallery image, we extract the feature by simply forward the data.
```python
outputs = model(input_img) 
# ---- L2-norm Feature ------
ff = outputs.data.cpu()
fnorm = torch.norm(ff, p=2, dim=1, keepdim=True)
ff = ff.div(fnorm.expand_as(ff))
```
```diff
+ Quick Question. Why we flip the test image horizontally when testing? How to fliplr in pytorch?
+ Quick Question. Why we L2-norm the feature?
```

## Part 3: Evaluation
Yes. Now we have the feature of every image. The only thing we need to do is matching the images by the feature.

We sort the predicted similarity score.
```python
query = qf.view(-1,1)
# print(query.shape)
score = torch.mm(gf,query) # Cosine Distance
score = score.squeeze(1).cpu()
score = score.numpy()
# predict index
index = np.argsort(score)  #from small to large
index = index[::-1]
```

Note that there are two kinds of images we do not consider as right-matching images.
* Junk_index1 is the index of mis-detected images which contains the body parts.

* Junk_index2 is the index of images which is of the same identity in the same cameras.

```python
    query_index = np.argwhere(gl==ql)
    camera_index = np.argwhere(gc==qc)
    # The images of the same identity in different cameras
    good_index = np.setdiff1d(query_index, camera_index, assume_unique=True)
    # Only part of body is detected. 
    junk_index1 = np.argwhere(gl==-1)
    # The images of the same identity in same cameras
    junk_index2 = np.intersect1d(query_index, camera_index)
```

We can use the function `compute_mAP` to obtain the final result.
```python
CMC_tmp = compute_mAP(index, good_index, junk_index)
```

## Part 4: A simple visualization (`python demo.py`)
In fact, it is similar to the `evaluate.py`. We just added the visualization part.
Index is the predicted ranking list.
```python
try: # Visualize Ranking Result 
    # Graphical User Interface is needed
    fig = plt.figure(figsize=(16,4))
    ax = plt.subplot(1,11,1)
    ax.axis('off')
    imshow(query_path,'query')
    for i in range(10): #Show top-10 images
        ax = plt.subplot(1,11,i+2)
        ax.axis('off')
        img_path, _ = image_datasets['gallery'].imgs[index[i]]
        label = gallery_label[index[i]]
        imshow(img_path)
        if label == query_label:
            ax.set_title('%d'%(i+1), color='green') # true matching
        else:
            ax.set_title('%d'%(i+1), color='red') # false matching
        print(img_path)
except RuntimeError:
    for i in range(10):
        img_path = image_datasets.imgs[index[i]]
        print(img_path[0])
    print('If you want to see the visualization of the ranking result, graphical user interface is needed.')
```

[1] Deng, Jia, Wei Dong, Richard Socher, Li-Jia Li, Kai Li, and Li Fei-Fei. "Imagenet: A large-scale hierarchical image database." In Computer Vision and Pattern Recognition, 2009. CVPR 2009. IEEE Conference on, pp. 248-255. Ieee, 2009.


