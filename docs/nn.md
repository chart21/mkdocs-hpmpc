# Neural Networks

`HPMPC` provides a high-level interface for performing secure inference of neural networks.

- `PIGEON` is a templated inference engine for private inference of neural networks and handles the data flow between layers.
- `PyGEON` is a Python library that allows users to export models and datasets from `PyTorch` to `PIGEON`.
- `Programs/NN` orchestrates the execution of `PIGEON` and the MPC backend.

## PIGEON

`PIGEON` is a templated inference engine for private inference of neural networks. Models and datasets can be exported from `PyTorch` to `PIGEON`. `PIGEON` then performs a forward pass on the model and dataset by relying on high-level functions provided by HPMPC. 
`PIGEON` consists of two main components: `Architectures` and `Headers`.

### Architectures

`Architectures` for neural networks are defined in the `architectures` directory. 
Out of the box `PIGEON` supports multiple `ResNet` and Convolutional Neural Network (CNN) architectures such as AlexNet, VGG, and LeNet.

The following is an example of the LeNet architecture as defined in `CNNs.hpp`. As one can see, layers can be defined in a similar manner to `PyTorch`. Architectures such as ResNets can also be defined in a programmatic manner as seen in `ResNet.hpp`.
The example below also shows a ReLU layer with reduced bitlength. 
In the example, only bits 8-16 are used for the sign bit extraction required by ReLU thus reducing communication complexity at the cost of accuracy. Related Work such as `Hummingbird` can be used to identify the optimal bitlength for each layer.

```cpp
template <typename T>       
class LeNet : public SimpleNN<T>
{
    public:
    LeNet(int num_classes)
    {
        this->add(new Conv2d<T>(1,6,5,1,2));
        this->add(new ReLU<T>());
        this->add(new AvgPool2d<T>(2,2));
        this->add(new Conv2d<T>(6,16,5,1,0));
        this->add(new ReLU<T,8,16>()); // ReLU with reduced bitlength
        this->add(new AvgPool2d<T>(2,2));
        this->add(new Flatten<T>());
        this->add(new Linear<T>(400,120));
        this->add(new ReLU<T>());
        this->add(new Linear<T>(120,84));
        this->add(new ReLU<T>());
        this->add(new Linear<T>(84,num_classes));
    }
};
```

### Headers

Layers are implemented in a generic manner in the `headers` directory. 
`PIGEON` itself only performs non-arithmetic operations such as matrix transposition, reshaping, and handling the data flow between layers.
All arithmetic operations are performed by high-level functions provided by `HPMPC`.
This modular design allows for an easy addition of new layers and neural network architectures to `PIGEON` without knowledge of the MPC backbone. 
`PIGEON` could potentially be used with other MPC backends as long as they provide the required high-level functions with the interface required by `PIGEON`.

The following is a list of layers currently implemented in `PIGEON`:

| Layer | Description |
| --- | --- |
| Conv2d | 2D Convolution |
| Linear | Fully Connected Layer |
| ReLU | Rectified Linear Unit |
| Softmax | Softmax (Argmax) Activation |
| AvgPool2d | 2D Average Pooling |
| MaxPool2d | 2D Max Pooling |
| AdaptiveAvgPool2d | 2D Adaptive Average Pooling |
| BatchNorm2d | 2D Batch Normalization |
| BatchNorm1d | 1D Batch Normalization |
| Flatten | Flatten Layer |

## PyGEON

`PyGEON` is a Python library that allows users to export models and datasets from `PyTorch` to `PIGEON`. The library provides the following functionalities.
- Download, transform, and edit datasets in PyTorch and export them as `.bin` files
- Train models in PyTorch and export them as `.bin` files
- Import existing model parameters as `.pth` files and export them as `.bin` files

The generated `.bin` files are compatible with `PIGEON` and can be used to achieve similar accuracy as the original model in PyTorch.


### Training and Exporting a Model

A single line of code suffices to train a model in PyTorch and export it to `PIGEON`. The following command trains an AlexNet model on the CIFAR-10 dataset for 30 epochs and exports the model and datasets as `.bin` files.

```bash
 python main.py --action train --export_model --export_dataset --transform standard --model AlexNet --num_classes 10 --dataset_name CIFAR-10 --modelpath ./models/alexnet_cifar --num_epochs 30 --lr 0.01 --criterion CrossEntropyLoss --optimizer Adam
```

The `main.py` script provides the following functionalities:
| Argument | Description |
| --- | --- |
| `--action` | Action to perform on the model: `train`, `import`, `train_all (for training all predifined model architectures)`, `none` (i.e. for only exporting the dataset) |
| `--export_model` | Export the model as a `.bin` file for `PIGEON` |
| `--export_dataset` | Export the test dataset as a `.bin` file for `PIGEON` |
| `--model` | Model architecture as defined in `cnns.py` |
| `--num_classes` | Number of classes in the dataset |
| `--dataset_name` | Name of the dataset as defined in `data_load.py` |
| `--modelpath` | Path to save/load the model |
| `--num_epochs` | Number of epochs to train the model |
| `--lr` | Learning rate for the optimizer |
| `--criterion` | Loss function to use |
| `--optimizer` | Optimizer to use |
| `--transform` | Type of transformation to apply to the dataset: `custom` or `standard` |

New model architectures can be added to `cnns.py` and new datasets can be added to `data_load.py` to extend the functionality of `PyGEON`.

### Importing pretrained models

We provide a set of pretrained models that can be imported to `PIGEON` using the `download_pretrained.py` script. 

The following command downloads all pretrained models to the `models/pretrained' folder and all datasets to the `data/datasets` folder.
```bash
python download_pretrained.py all
```
Note that downloading all models requires a few GB of disk space. Thus, we also provide the option to download some of the models and datasets with the following options.

| Argument | Description |
| --- | --- |
| `all` | Download all models and datasets |
| `single_model` | Download VGG16, trained on CIFAR-10 (standard transform)|
| `cifar_adam_001_pretrained` | Download several models, trained on CIFAR-10 with Adam optimizer and lr=0.01 |
| `cifar_adam_005_pretrained` | Download several models, trained on CIFAR-10 with Adam optimizer and lr=0.05 |
| `cifar_sgd_001_pretrained` | Download several models, trained on CIFAR-10 with SGD optimizer and lr=0.01 |
| `lenet5_pretrained` | Download LeNet5, trained on MNIST (different transforms) |
| `datasets` | Download all datasets |

The different options can be combined to download multiple models and datasets at once.
```bash
python download_pretrained.py single_model datasets # Downloads VGG16 and all datasets
```

## NN

`Programs/NN` orchestrates the execution of `PIGEON` and the MPC backend. The `NN` program provides the following functionalities:
- Load a model and dataset from `PyGEON` using environment variables.
- Obtain the model parameters and the dataset from the right party and secretly share them.
- Define the model architecture and dataset dimensions for performing a forward pass.
- Perform a forward pass on the model using `PIGEON` and `HPMPC` as a backend.

### Evaluating a Model

To evaluate a model, the program first assigns a FUNCTION_IDENTIFIER to the model and dataset. 
For instance, the following line defines that the VGG model is evaluated when the FUNCTION_IDENTIFIER is set to 74.
```cpp
#if FUNCTION_IDENTIFIER == 74 
    int n_test = NUM_INPUTS*BASE_DIV, ch = 3, h = 32, w = 32, num_classes = 10; // CIFAR-10 input dimensions
    auto model = VGG<modeltype>(num_classes); // CNN architecture as defined in CNNs.hpp of PIGEON
#endif
```

A custom model can be evaluated by defining a new architecture in `CNNs.hpp` and assigning a new FUNCTION_IDENTIFIER to the model in `NN.hpp`. The program ensures that each process handles a separate part of the dataset and prints the accuracy of its classifications in the terminal.

### Sharing Model Parameters and Data

The program then checks which party is responsible for sharing the model parameters and which party is responsible for sharing the dataset. 
The parties can be specified with the `MODELOWNER` and `DATAOWNER` config options. `MODELOWNER=P_0` and `DATAOWNER=P_1` specify that party 0 is responsible for sharing the model and party 1 is responsible for sharing the dataset. Setting `MODELOWNER=-1` and `DATAOWNER=-1` skips secret sharing which is useful for benchmarking.

For the node acting as the model owner, the program loads the model parameters from the `.bin` file as defined by the environment variables `MODEL_DIR` and `MODEL_FILE`. The model at `MODEL_DIR/MODEL_FILE` is loaded and its parameters are secretly shared. Below is an example of how to set the environment variables for the VGG16 model trained on CIFAR-10.
```bash
export MODEL_DIR=nn/Pygeon/models/pretrained
export MODEL_FILE=VGG16_CIFAR-10_standard.bin
```

For the node acting as the data owner, the program loads the dataset from the `.bin` file as defined by the environment variables `DATA_DIR`, `SAMPLES_FILE`, and `LABELS_FILE`. The samples at 
`DATA_DIR/SAMPLES_FILE` are loaded and secretly shared. Below is an example of how to set the environment variables for the CIFAR-10 dataset.
```bash
export DATA_DIR=nn/Pygeon/data/datasets
export SAMPLES_FILE=CIFAR-10_standard_test_images.bin
export LABELS_FILE=CIFAR-10_standard_test_labels.bin
```

Note that each party that requires obtaining the correct accuracy needs the labels of the dataset in plaintext. The environment variables can be adjusted without the need to recompile the program. Also, the program prints in the terminal whether the model and dataset were loaded correctly.



