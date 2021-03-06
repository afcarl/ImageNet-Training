Deep Learning on ImageNet using Torch
=====================================
This is a complete training example for Deep Convolutional Networks on the ILSVRC classification task.

Data is preprocessed and cached as a LMDB data-base for fast reading. A separate thread buffers images from the LMDB record in the background.

Multiple GPUs are also supported by using nn.DataParallelTable (https://github.com/torch/cunn/blob/master/docs/cunnmodules.md).

This code allows training at 4ms/sample with the AlexNet model and 2ms for testing on a single GPU (using Titan Z with 1 active gpu)

## Dependencies
* Torch (http://torch.ch)
* "eladtools" (https://github.com/eladhoffer/eladtools) for optimizer.
* "lmdb.torch" (http://github.com/eladhoffer/lmdb.torch) for LMDB usage.
* "DataProvider.torch" (https://github.com/eladhoffer/DataProvider.torch) for DataProvider class.
* "cudnn.torch" (https://github.com/soumith/cudnn.torch) for faster training. Can be avoided by changing "cudnn" to "nn" in models.

To install all dependencies (assuming torch is installed) use:
```bash
luarocks install https://raw.githubusercontent.com/eladhoffer/eladtools/master/eladtools-scm-1.rockspec
luarocks install https://raw.githubusercontent.com/eladhoffer/lmdb.torch/master/lmdb.torch-scm-1.rockspec
luarocks install https://raw.githubusercontent.com/eladhoffer/DataProvider.torch/master/dataprovider-scm-1.rockspec
```

## Data
* To get the ILSVRC data, you should register on their site for access: http://www.image-net.org/
* Extract all archives and configure the data location and save dir in **Config.lua**. You can also change the saved image size by editing the default value `ImageMinSide=256`.
* LMDB records for fast read access are created by running **CreateLMDBs.lua**.
It defaults to saving the compressed jpgs (about ~24GB for training data, ~1GB for validation data when smallest image dimension is 256).
* To validate the LMDB configuration and test its loading speed, you can run **TestLMDBs.lua**.
* All data related functions used for training are available at **Data.lua**.

## Model configuration
Network model is defined by writing a <ModelName>.lua file in `Models` folder, and selecting it using the `network` flag.
The model file must return a trainable network. It can also specify additional training options such optimization regime, input size modifications.

e.g for a model file:
```lua
local model = nn.Sequential():add(...)
return  --optional: you can also simply return model
{
  model = model,
  regime = {
    epoch        = {1,    19,   30,   44,   53  },
    learningRate = {1e-2, 5e-3, 1e-3, 5e-4, 1e-4},
    weightDecay  = {5e-4, 5e-4, 0,    0,    0   }
  }
}
```
Currently available in `Models` folder are: `AlexNet`, `MattNet`, `OverFeat`, `GoogLeNet`, `CaffeRef`, `NiN`. Some are available with a batch normalized version (denoted with `_BN`)


## Training
You can start training using **Main.lua** by typing:
```lua
th Main.lua -network AlexNet -LR 0.01
```
or if you have 2 gpus availiable,
```lua
th Main.lua -network AlexNet -LR 0.01 -nGPU 2 -batchSize 256
```
A more elaborate example continuing a pretrained network and saving intermediate results
```lua
th Main.lua -network GoogLeNet_BN -batchSize 64 -nGPU 2 -save GoogLeNet_BN -bufferSize 9600 -LR 0.01 -checkpoint 320000 -weightDecay 1e-4 -load ./pretrainedNet.t7
```
Buffer size should be adjusted to suit the used hardware and configuration. Default value is 5120 (40 batches of 128) which works well when using a non SSD drive and 16GB ram. Bigger buffer size allows better sample shuffling.

## Output
Training output will be saved to folder defined with `save` flag.

The complete netowork will be saved on each epoch as **Net_<#epoch>.t7** along with
* A complete log **Log.txt**
* Error rate summary **ErrorRate.log** and accompanying               **ErrorRate.log.eps** graph

## Additional flags
|Flag             | Default Value        |Description
|:----------------|:--------------------:|:----------------------------------------------
|modelsFolder     |./Models/             | Models Folder
|network          |AlexNet               | Model file - must return valid network.
|LR               |0.01                  | learning rate
|LRDecay          |0                     | learning rate decay (in # samples)
|weightDecay      |5e-4                  | L2 penalty on the weights
|momentum         |0.9                   | momentum
|batchSize        |128,                  | batch size
|optimization     |'sgd'                 | optimization method
|seed             |123                   | torch manual random number generator seed
|epoch            |-1                    | number of epochs to train, -1 for unbounded
|testonly         |false                 | Just test loaded net on validation set
|threads          |8                     | number of threads
|type             |'cuda'                | float or cuda
|bufferSize       |5120                  | buffer size
|devid            |1                     | device ID (if using CUDA)
|nGPU             |1                     | num of gpu devices used
|constBatchSize   |false                 | do not allow varying batch sizes - e.g for ccn2 kernel
|load             |''                    | load existing net weights
|save             |time-identifier       | save directory
|optState         |false                 | Save optimization state every epoch
|checkpoint       |0                     | Save a weight check point every n samples. 0 for off
|augment          |1                     | data augmentation level - {1 - simple mirror and crops, 2 +scales, 3 +rotations}
|estMeanStd       |preDef                | estimate mean and std. Options: {preDef, simple, channel, image}
|shuffle          |true                  | shuffle training samples
