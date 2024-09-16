
### Read this
- main problem is compatibility issues of driver-cuda-pytorch
- Currently direct pytorch installation support 12.1 CUDA, above it you need to install manually
- Thus we will select 12.1 CUDA
- it requires driver >530
### Install Nvidia driver
- https://askubuntu.com/questions/524242/how-to-find-out-which-nvidia-gpu-i-have
	- `sudo update-pciids`
	- `lspci -nn | grep '\[03'`
- ![[Pasted image 20240605144929.png]]
- RTX A6000
- https://ubuntu.com/server/docs/nvidia-drivers-installation
	- `sudo ubuntu-drivers list`
	- `sudo ubuntu-drivers install nvidia:535`
		- select the version
- This gives available versions with ubuntu package manager, but if you want latest drivers you can go here
	- https://www.nvidia.com/download/index.aspx
	- select your gpu download the driver
	- `sudo sh <filename>.run`
- After installation
	- ![[Pasted image 20240605160623.png]]
	- CUDA version written here is maximum version compatible with this driver (its not yet installed)
	- ![[Pasted image 20240605145434.png]]
### Installing CUDA
- https://docs.nvidia.com/cuda/cuda-installation-guide-linux/
- check GCC is installed
	- ![[Pasted image 20240605145610.png]]
- update kernels
	- `sudo apt-get install linux-headers-$(uname -r)`
- It is recommended to use the distribution-specific packages,
- Download and install CUDA (search on google with the exact version you want) - choose upto 12.1 version as for later version you need to install pytorch manually from source which would be difficult
	- https://developer.nvidia.com/cuda-downloads
	- ![[Pasted image 20240605145747.png]]
	- ![[Pasted image 20240605153616.png]]
	- check the version mentioned in commands here it is 12.1
	- sudo apt-get update should show cuda repo
		- ![[Pasted image 20240605154327.png]]
	- **install with following command**
		- `sudo apt install cuda-toolkit-12-1` and not `sudo apt-get -y install cuda` as driver is already installed so it will conflict with the driver
	- reboot after installation
- After reboot do following 
	- https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#post-installation-actions
	- Environment Setup``
		- `echo "export PATH=/usr/local/cuda-12.1/bin${PATH:+:${PATH}}" >> ~/.bashrc`
		- restart the terminal
	- Verify installation
		- `cat /proc/driver/nvidia/version`
		- `nvcc --version`
			- ![[Pasted image 20240605162637.png]]
	- Install Third-party Libraries
		- `sudo apt-get install g++ freeglut3-dev build-essential libx11-dev libxmu-dev libxi-dev libglu1-mesa-dev libfreeimage-dev libglfw3-dev`

### Install Pytorch
- check which version is compatible with the cuda
	- https://pytorch.org/get-started/previous-versions/
- https://pytorch.org/get-started/locally/
- create a conda env and install in it
	- ![[Pasted image 20240605153859.png]]
	- `conda install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia`
