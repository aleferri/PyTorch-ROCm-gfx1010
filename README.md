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

For usage examples (chaiNNer, ComfyUI, etc.) see the \`[usage](usage)\` directory.

## Pre-built rpms

Pre-built rpms of PyTorch (built for Fedora 43) are located in the \`prebuilt\` directory.

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

## Build

edit the spec file to include "export PYTORCH_ROCM_ARCH=gfx1010" (on newline, without quotes) after "export USE_ROCM=ON" (also without quotes)

```
cd python-torch
sudo dnf buildep python-torch.spec
spectool -gR python-torch.spec
cp *.patch ~/buildroot/SOURCES
rpmbuild -ba python-torch.spec
```

### Go on with your life for about a week, then come back and hope to find the compilation finished

if something has failed, well, bad luck i guess, if you did the exact thing for the correct distro version, then check that you can build *standard* torch with the untouched spec file, if so, then the problem lies in inherent incompatiblity of RDNA 1 with your desired version of torch

The resulting RPM will be in the *~/rpmbuild/RPMS*

### Install PyTorch

To install your own version:
1. uninstall with purge any version of python-torch that is not your own
2. install your generated RPM, to do so execute:

```
cd ~/rpmbuild/rpms/<your-torch-version>.rpm
dnf install --norepolist <your-torch-version>.rpm
```

Notes:

1. You will build the version that your distro also provide, but with gfx1010 support. If a file spec for an earlier version of torch, but for a previous distro exists, it is possible that switching to the correct branch and fiddling some deps in the spec file will give you a working build, but don't count on it.
2. At this point is possible that you built a version that is supported by your current python, but only because you built it custom. I.E. Fedora 43 comes with python-torch-2.8.0, that is officially incompatible with python 3.14, but since the spec file was created for the specific combination, the build will work anyway.
3. No, you can't convince pip of this fact. Be prepared at *pip install -r requirements* followed by *pip uninstall torch*, **forever**.
4. In the case you're also going to need TorchVision and TorchAudio or Triton install them also by dnf, they will work with your own custom version of Torch, after all you only changed the target compilation.
5. If you're using a venv (and that's a good idea), after the usual uninstall of torch & torchvision & torchaudio & triton, allow the venv to use globally installed modules by switching *false* -> *true* the flag ``` include-system-site-packages ``` in pyvenv.cfg, just be sure to not pollute the global space with random libraries 

## Troubleshooting

### CMake version errors

*CMAKE_POLICY_VERSION_MINIMUM=3.5* have to be added to the enviromnent.

Compile with this command instead:

CMAKE_BUILD_PARALLEL_LEVEL="$(nproc --all)" MAKEOPTS="-j$(nproc --all)" CMAKE_POLICY_VERSION_MINIMUM=3.5 PYTORCH_ROCM_ARCH=gfx1010 python3 setup.py bdist_wheel

## Triton

See notes

## TorchVision

See notes

## TorchAudio

See notes

## GENERAL ISSUES

This guide suppose that one torch (and relatives!) version for the entire system is sufficient, if having a working torch as a global dependency do not work for your usecase, you can use a virtual machine/docker for each version of torch, not efficient, i know; or you can investigate on converting a rpm to whl, there are tools that do whl to rpm already, perhaps it is possible to do the reverse 

