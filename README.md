# FilterExpo: CNN Model Compression via Filter Exploration

This repository is the official implementation of the paper "FilterExpo: CNN Model Compression via Filter Exploration".

## Highlights
Filter-wise network pruning has been broadly recognizedas an effective technique to reduce the computation and mem-ory cost of deep convolutional neural networks.  However,conventional pruning methods require a fully pre-trainedlarge model, and remove filters in one-shot or iterative man-ners unidirectionally, which result to sub-optimal model qual-ity, large memory footprint and expensive training cost. Inthis paper, we propose a novel Filter Exploration methodol-ogy, dubbed as FilterExpo. It repeatedly prunes and regrowsthe filters throughout the training process, which reducesthe risk of pruning important filters prematurely. It also bal-ances the number of filters across all layers with a  sparsity1constraint. In addition, we convert the filter pruning problemto the well known column subset selection (CSS) problem,which produces better results than  previous heuristic prun-ing methods. All the exploration process is done in a singletraining from scratch without the need of a pre-trained largemodel.  Experimental results demonstrate that our methodcan effectively reduce the FLOPs of diverse CNN architec-tures on a variety of  computer vision tasks, including imageclassification, object detection, instance segmentation, and3D vision. For example, our compressed ResNet-50 modelon ImageNet dataset achieves 76% Top-1 accuracy with only25% FLOPs of the original ResNet-50 model, improvingprevious state-of-the-art filter pruning method by 0.7%.

<div align="center">
  <img width="90%" src="figs/overview.png">
</div>

## Dependency

This repository contains Dockerfile which extends the PyTorch NGC container and encapsulates some dependencies. In addition, ensure you have the following components:

* [NVIDIA Docker](https://github.com/NVIDIA/nvidia-docker)
* [PyTorch 20.12-py3 NGC container](https://ngc.nvidia.com/registry/nvidia-pytorch) or newer
* Supported GPUs:
    * [NVIDIA Volta architecture](https://www.nvidia.com/en-us/data-center/volta-gpu-architecture/)

## ImageNet Classification

### Download the dataset

* The script operates on ImageNet-1K, a widely popular image classification dataset from the ILSVRC challenge. 
* [Download the images](http://image-net.org/download-images).
* Extract the training data
```Shell
mkdir train && mv ILSVRC2012_img_train.tar train/ && cd train
tar -xvf ILSVRC2012_img_train.tar && rm -f ILSVRC2012_img_train.tar
find . -name "*.tar" | while read NAME ; do mkdir -p "${NAME%.tar}"; tar -xvf "${NAME}" -C "${NAME%.tar}"; rm -f "${NAME}"; done
cd ..
```
* Extract the validation data and move the images to subfolders
```Shell
mkdir val && mv ILSVRC2012_img_val.tar val/ && cd val && tar -xvf ILSVRC2012_img_val.tar
wget -qO- https://raw.githubusercontent.com/soumith/imagenetloader.torch/master/valprep.sh | bash
```

* Docker related setup
```Shell
docker build . -t nvidia_resnet50
nvidia-docker run --rm -it -v <path to imagenet>:/imagenet --ipc=host nvidia_resnet50
```

### Evaluation
* We released the pruned model at ```./RN50/logs/resnet50_2g_0.774.pth.tar``` (ResNet50 with 2GFLOPs and 77.4% Top-1) for direct evaluation.
* Start inference
```Shell
python ./main.py --data-backend pytorch --arch resnet50 --evaluate --pruned_model ./logs/resnet50_2g_0.774.pth.tar -b 128 /data/imagenet
```
* FLOPs checking
```Shell
python check_flops.py --checkpoint_path ./logs/resnet50_2g_0.774.pth.tar
```

### Training from scratch
```Shell
python ./multiproc.py --nproc_per_node 8 ./main.py /data/imagenet --data-backend pytorch --raport-file raport.json -j8 -p 100 --lr 1.024 --optimizer-batch-size 1024 --warmup 8 --arch resnet50 -c fanin --label-smoothing 0.1 --lr-schedule cosine --mom 0.875 --wd 3.0517578125e-05 -b 128 --amp --static-loss-scale 128 --mixup 0. --grow_prune --delta_T 2 --T_max 0.72 --init_channel_ratio 0.2 --channel_sparsity 0.5 --sampling
```

