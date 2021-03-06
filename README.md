# Project-Details
## Credit Card Related
The whole pipeline is divided into 2 parts: detection and recognition. It's not an end to end pipeline. Megvii will use the self-designed deep learning framework called MegaBrain rather than Pytorch or Tensorflow. So I implemented all the network architechture from scratch mentioned below.
### Detection
In detection part, I implemented [RetinaNet](https://arxiv.org/abs/1708.02002) for detecting the credit card. It's because we need to consider the efficiency of the model after deployment on edge-devices. One-stage method is faster than two-stage method. Here I will introduce the details of RetinaNet.
1. Focal Loss: After the One-stage method obtains the feature map, it will produce dense target proposal regions, and only a small part of these proposal regions are real targets, which causes the classic imbalance of positive and negative training samples in machine learning. It often causes the final calculated training loss to be dominated by negative samples that account for the absolute majority but contain little information (consider 1 positive target vs 1000 negative targets). The key information provided by a small number of positive samples cannot play a normal role in the commonly used training loss, so it cannot be a loss that can provide correct guidance for model training. But after the Two-tage method obtains the proposal, its proposal area is much smaller than the proposal area generated by One-stage, so it will not cause serious category imbalance.
Focal Loss is very simple, that is, a factor is added to the original cross-entropy loss function to make the loss function pay more attention to hard examples. In actual use, the paper proposes to add a alpha balance factor which can produce a slight accuracy increase, the formula is shown below:

<p align="center">
  <img src="https://latex.codecogs.com/gif.latex?FL_%7B%28p_%7Bt%7D%29%7D%3D-%5Calpha%20_%7Bt%7D%281-p_%7Bt%7D%29%5E%7B%5Cgamma%20%7Dlog%28p_%7Bt%7D%29">
</p>

2. Architechture: 
    1. Backbone: ResNet50+FPN: We all know that [ResNet](https://arxiv.org/abs/1512.03385) used residual block to solve gradient vanish problem. Then we are able to make the network deeper and trainable. [FPN](https://arxiv.org/abs/1612.03144) network concentrated more on different scale. Feature extraction for images of each scale can produce multi-scale feature representations, and feature maps of all levels have strong semantic information, even including some high-resolution feature maps. Here is the structure: ![](https://pic4.zhimg.com/80/v2-71dde3ef14448ed166c609498fb2800f_720w.jpg)
    2. Details of FPN: The FPN network consists of three parts: bottom-up path (from image to high-level features), top-down path and side connection. The bottom-up path is composed of the ResNet network, and the feature map of the top-down path is generated using the output of the feature activation layer of the last residual block of each stage of ResNet. M5 is generated by C5 using 1x1 convolution, which is mainly used for dimensionality reduction, and then P5 is obtained through 3x3 convolution. M4 is M5 for 2 times upsampling + C4 is generated by 1x1 convolution (point-wise adding), and then P4 is obtained by 3x3 convolution....Then we get feature pyramid.
    3. cls+reg network: After the feature pyramid is obtained, two sub-networks (classification network + detection location regression) are used for each layer of the feature pyramid. These two sub-networks are modified by the RPN network.
    	- Similar to the RPN network, anchors are also used to generate proposals. Each layer of the feature pyramid corresponds to an anchor area. In order to produce a more dense coverage, three area ratios ![](https://www.zhihu.com/equation?tex=%5Cleft%5C%7B+2%5E0%2C+2%5E%5Cfrac%7B1%7D%7B2%7D%2C2%5E%5Cfrac%7B2%7D%7B3%7D+%5Cright%5C%7D) are added (that is, the area corresponding to the current anchor is multiplied by the corresponding ratio to form three scales), and then the anchors length and width ratio is still ![](https://www.zhihu.com/equation?tex=%5Cleft%5C%7B+1%3A2%2C+1%3A1%2C+2%3A1+%5Cright%5C%7D), so each layer of the feature pyramid corresponds to 9 kinds of anchors.
    	- The classification network of the original RPN network only distinguishes two types of foreground and background, here is changed to the number of target categories K.
### Recognition
In recognition part, I modified and implemented [Spatial Transformer Network](https://arxiv.org/abs/1506.02025) to align photo taken from extreme angle and inserted the STN moudule in the piepline. Here I will introduce the STN:

1. Localization Net: The Localisation net input is a feature map. After a number of convolution or fullly connection operations, a regression layer returns the output transformation parameter ??. The dimension of ?? depends on whether the specific transformation type selected by the network is affine transformation or projection transformation. The value of ?? determines the "magnitude" of the spatial transformation selected by the network. According to the paper's introduction, Localization Net can use FC layer or Conv layer to generate the transform matrix. After several architechture experiments, I found that 2 Conv layer will have the best performance. So it's Conv1+MaxPool+Relu+Conv2+MaxPool+Relu+FC1+FC2(regression layer for affine transformation).

2. Grid generator: For each position of the output Feature map, we perform an affine transformation on it to find the spatial position corresponding to the input Feature map. So far, if the output of this step is an integer value (which is often impossible), in other words, if the transformed coordinates can exactly correspond to some spatial positions of the original image, then the task is completed. The input image is successively determined after the Localisation Net and the Grid generator, the spatial transformation method and the mapping relationship. The above operation does not directly operate on the feature map, but calculates the feature position to find the corresponding relationship between input and output. The calculation is discrete, that is, when the feature position changes slightly, the output cannot be solved. This makes back propagation impossible.

3. Sampler: After the above two steps, each pixel on the output feature map will correspond to a certain pixel position of the input feature map through spatial transformation, but because the feature score cannot be calculated for the partial derivative of the feature position, we need to construct a a kind of position->score mapping, and the mapping has the property of derivation, so as to meet the conditions of back propagation. Here the author used bilinear interpolation. algorithm. If the origin position is not located right on the interger point, it will fill it with the 4 points near the location with bilinear interpolation.
<p align="center">
  <img src="https://pic3.zhimg.com/80/v2-382c6532831b8a53ec0fa227acd1199a_720w.jpg">
</p>

4.Architechture:

![](https://pic4.zhimg.com/80/v2-f89a142991f0aa025dd567ff840e9f83_720w.jpg)

### Other work:
I was in charge of training recognition part of the pipeline. So here are some tricks/tips that I used:
1. I used warm-up learning rate to train the model.
2. I manually find the hard case or bad case of the results and have clear idea of "What kind of cards/ What scenario we are not doing well". Then we proposed the demond for collecting the specific data to solve the problem.
3. I write a fake photo generate code to generate fake but useful credit card image and make combine training using real image and fake image. Fake image contributed a lot to the final performance.
4. I write profile code to check the efficiency of training and inference to find out what made the process slow.
5. I used 64 NVIDIA 1080Ti GPUs for training.
6. In Objects 365, I built machine prelabeling pipeline to accelerate labeling process.
7. There are many works I was responsible for like modifying network architechture, using fast-pace experiments to check some ideas, writing documents, designing data labeling standards, designing pipeline standards, managing datasets(Objects365, credit card, car plate license).
