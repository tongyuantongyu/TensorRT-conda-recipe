# conda recipe for NVIDIA TensorRT

This conda recipe builds NVIDIA's TensorRT deep neural network inference library.

## Current release info

| Name | Downloads | Version | Platforms |
| --- | --- | --- | --- |
| [![Conda Recipe](https://img.shields.io/badge/recipe-tensorrt-green.svg)](https://anaconda.org/tongyuantongyu/libnvinfer) | [![Conda Downloads](https://img.shields.io/conda/dn/tongyuantongyu/libnvinfer.svg)](https://anaconda.org/tongyuantongyu/libnvinfer) | [![Conda Version](https://img.shields.io/conda/vn/tongyuantongyu/libnvinfer.svg)](https://anaconda.org/tongyuantongyu/libnvinfer) | [![Conda Platforms](https://img.shields.io/conda/pn/tongyuantongyu/libnvinfer.svg)](https://anaconda.org/tongyuantongyu/libnvinfer) |

## Install

The `libnvinfer` package, which contains runtime dynamic libraries, can be installed with `conda`:

```
conda install -c tongyuantongyu libnvinfer
```

Do to license issue, the development package `libnvinfer-dev` is not available for distribution, and have to be built yourself.

## Build

### Setup `conda-build`

To build TensorRT conda package, first set up `conda-build` following the [Getting started](https://docs.conda.io/projects/conda-build/en/stable/user-guide/getting-started.html) guide.

### Clone repository

Clone this repository.

### Download TensorRT archive

TensorRT does not provide public redistribution. You need to download TensorRT from its [website](https://developer.nvidia.com/nvidia-tensorrt-8x-download)
with your developer account. This recipe currently targets version 8.6 GA (`8.6.1.6`). Download the TAR Packages for Linux or ZIP Packages for Windows, and place the downloaded archive files into `src` folder in the repository.

### Run build

Run the following command to start build process.

```
conda build <path-to-cloned-repository>
```

The build process will take a while. After it finished, you will have `libnvinfer` and `libnvinfer-dev` (and `libnvinfer-static` on Linux) in your `local` conda channel. `libnvinfer-dev` and `libnvinfer-static` are not distributable.