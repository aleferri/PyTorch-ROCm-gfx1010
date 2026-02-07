# PyTorch 2.8 for Radeon RX 5600/5700 (gfx1010) on Fedora

**Contents:**

1. [Pre-built wheels](#pre-built-wheels)
2. [Introduction](#introduction)
3. [Requirements](#requirements)
4. [Prepare](#prepare)
5. [Build](#build)
6. [Troubleshooting](#troubleshooting)
7. [Triton](#triton)
8. [TorchVision](#torchvision)
9. [TorchAudio](#torchaudio)
10. [ONNX](#onnx)
11. [Extras](#extras)

For usage examples (chaiNNer, ComfyUI, etc.) see the \`[usage](usage)\` directory.

## Pre-built wheels DO NOT USE, IT WAS BUILD FOR DEBIAN

Pre-built wheels of PyTorch, TorchVision and TorchAudio (built for ROCm 6.4.3 and Python 3.11) are located in the \`prebuilt\` directory.

There is no guarantee that those will work with your particular configuration of video card, ROCm version, Python version, PyTorch version, kernel version, etc.

For best results use the following instructions to build those wheels yourself.

## Introduction

These instructions are the result of the efforts to make PyTorch work with unofficially-supported AMD gfx1010 GPUs (Radeon RX 5000 series) on Fedora 43.

Unfortunately **none** of the pre-compiled binaries work for gfx1010 GPUs. Anecdotally they did work in the past but no more, even with not-so-fresh versions of PyTorch.

Using various tricks, such as setting the `HSA_OVERRIDE_GFX_VERSION=10.3.0` environment variable or using older versions of ROCm, PyTorch or kernel do not work as well causing either:

1. \`Compile with 'TORCH_USE_HIP_DSA' to enable device-side assertions\` message
2. Crashing X11 every time calculations begin
3. Calculations going on forever with 100% GPU load and no characteristic noises coming out of the GPU, which means no actual work is being done

The only working solution seems to be is to build PyTorch and TorchVision from the source code with the support for gfx1010 architecture explicitly on.

The resulting Python wheels were tested with [chaiNNer](https://github.com/chaiNNer-org/chaiNNer) and [ComfyUI](https://github.com/comfyanonymous/ComfyUI) and both seem to work fine.

## Requirements

    rpmdevtools
    rpmbuild
    dnf
    git
    plus whatever dnf builddep install (find some GBs of free space, possibly in SSD memory)

To begin with you have to clone the official build for fedora of python-torch, no matter how many options you try, you cannot build torch on fedora, as-is. Since a wonderful team of packagers already created an official package, just edit their spec file, on the good news, this is less error prone. Go to the build task for your distro (mine is f43), so i've gone [here](https://koji.fedoraproject.org/koji/taskinfo?taskID=138383103) to recover the official repository [here](https://src.fedoraproject.org/rpms/python-torch/tree/f43), congrats, now you've found your spec file. It's time to clone! Remember to switch to your branch after cloning

```
git clone https://src.fedoraproject.org/rpms/python-torch.git
git checkout f43
```
Go to Iceland for a week, if you are already in Iceland, come in Italy instead



Download and unpack the missing RocBLAS libraries into */opt/rocm/lib/rocblas/library*:

```
wget https://github.com/Efenstor/PyTorch-ROCm-gfx1010/raw/refs/heads/main/prebuilt/rocblas_library_gfx1010.tar.gz
tar xv -f rocblas_library_gfx1010.tar.gz -C /opt/rocm/lib/rocblas/library
```

## Build

edit the spec file to include "export PYTORCH_ROCM_ARCH=gfx1010" (on newline, without quotes) after "export USE_ROCM=ON" (also without quotes)

```
sudo dnf buildep python-torch.spec
spectool -gR python-torch.spec
cp *.patch ~/buildroot/SOURCES
rpmbuild -ba python-torch.spec
```

### Go on with your life for about a week, then come back and hope to find the compilation finished

if something has failed, well, bad luck i guess, i'll add an official troubleshooting session someday

The resulting RPM will be in the *~/rpmbuild/RPM*

### Install PyTorch

To install the built RPM execute:

    TODO, i still finishing the compilation

Notes:

1. You will build the version that your distro also provide, but with gfx1010 support. If a file spec for an earlier version of torch, but for a previous distro exists, it is possible that switching to the correct branch and fiddling some deps in the spec file will give you a working build, but i don't know.
2. TODO
3. This command should be executed inside a venv in which you have your program (e.g. ComfyUI) installed, not the venv in which you had your PyTorch built.
4. In the case you're also going to build TorchVision and TorchAudio (which are highly recommended) install it also inside the PyTorch build venv.
5. If you're installing TorchVision and/or TorchAudio be sure to install PyTorch together with them in a single `pip install` command because they are dependent of the particular PyTorch version.

## Troubleshooting

### CMake version errors

*CMAKE_POLICY_VERSION_MINIMUM=3.5* have to be added to the enviromnent.

Compile with this command instead:

CMAKE_BUILD_PARALLEL_LEVEL="$(nproc --all)" MAKEOPTS="-j$(nproc --all)" CMAKE_POLICY_VERSION_MINIMUM=3.5 PYTORCH_ROCM_ARCH=gfx1010 python3 setup.py bdist_wheel

## Triton

[Triton 3.4.0](https://download.pytorch.org/whl/pytorch-triton-rocm/) seems to be working but I didn't test it too much:

    wget https://download.pytorch.org/whl/pytorch_triton_rocm-3.4.0-cp311-cp311-linux_x86_64.whl
    pip install pytorch_triton_rocm-3.4.0-cp311-cp311-linux_x86_64.whl

## TorchVision

This build should be done as you did with Torch package, find your packaged fedora version, go to the build task on koji/copr/whatever, find the repository of the spec, clone the spec and so on.

The resulting rpm will be in the *RPM* directory.

## TorchAudio

This built should be done in a venv with the previously built pytorch wheel installed.

    git clone https://github.com/pytorch/audio.git --branch=release/2.8 --recurse-submodules audio-release-2.8-git
    cd audio-release-2.8-git
    CMAKE_BUILD_PARALLEL_LEVEL="$(nproc --all)" MAKEOPTS="-j$(nproc --all)" python3 setup.py bdist_wheel

The resulting wheel will be in the *~/rpmbuild/RPM* directory.

## GENERAL ISSUES

This guide suppose that one torch (and relatives!) build for the entire system is sufficient, if having a working torch as a global dependency do not work for your usecase, you can use a virtual machine/docker for each version of torch, not efficient, i know; or you can investigate on converting a rpm to whl, there are tools that do whl to rpm already, perhaps it is possible to do the reverse 

