# Using with chaiNNer

If you have a new installation then run chaiNNer, let it download all the basic components but don't install any dependencies.

Then ensure your distro packages are installed, that the venv can access global packages, make it install all, then, guess what: pip uninstall torch torchvision

After that run chaiNNer. Remember to keep uninstalling the torch/torchvision wheels that will get downloaded from times to times.

