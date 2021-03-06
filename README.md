# Single Shot Shape-from-Shading and Superresolution

**Contact:** [Florian Windolf](mailto:florian.windolf@tum.de)

## Installation

### Dependencies

Tested with:
- llvm 5.0.2
- clang 5.0.2
- cuda 9.2.148

For Pangolin, see [dependencies section](https://github.com/stevenlovegrove/Pangolin#required-dependencies):
- OpenGL
- Glew (libglew-dev)

For Cuda-image, see [dependencies section](https://github.com/fwindolf/cuda-image#install)
- Pangolin

OpenCV3.4 required

### Installation

Either use the `setup.sh` script by running `source setup.sh`, that automatically gathers dependencies, or manually install everything

#### Manual Install
Every step is meant to be executed from source directory.

0. Set clang as the default C++ compiler (Opt doesnt work without it...)

1. Init submodules (Opt, Pangolin, cuda-image)
```
git submodules update --init
```

2. Build and install Pangolin
```
cd third_party/Pangolin
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=.. -DCMAKE_BUILD_TYPE=Release
make -j8 install
```

3. Build and install cuda-image
```
cd third_party/cuda-image
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=.. -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED=ON -DPANGOLIN_DIR=../../Pangolin -DEIGEN3_INCLUDE_DIR=$SOURCE_DIR/third_party/Eigen/include
make -j8 install
```

4. Download terra
```
cd third_party

wget https://github.com/zdevito/terra/releases/download/release-1.0.0-beta1/terra-Linux-x86_64-2e2032e.zip
unzip -q -o terra-Linux-x86_64-2e2032e.zip 
ln -s terra-Linux-x86_64-2e2032e terra
rm terra-Linux-x86_64-2e2032e.zip
```

5. Build and install Opt
```
cd third_party/Opt/API 
make
cd SOURCE_DIR
```

7. Finally build this repo (set clang as compiler for C++ in order for Opt to build properly)
```
mkdir build && cd build
export CXX=/usr/bin/clang++
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j8
```

You then can run an example appliation by `./bin/AppSfs`.

If Opt fails, try building terra for your distribution, following the [provided instructions](https://github.com/zdevito/terra#installing-terra)
For me, it took several tries to get terra/Opt to work together. The whole setup is super fragile, so if you get it to run, do not change anything :D

## AppSfs

Runs Shape from shading on a single input (image, mask, depth).

```
SFS Application
Usage: ./bin/AppSfs [OPTIONS]

Options:
  -h,--help                   Print this help message and exit
  -a,--optim_alpha FLOAT=-1   Albedo Update : Smoothness of albedo, -1 (=infinity) for piecewise constant
  -l,--optim_lambda FLOAT=0.5 Albedo Update : Tradeoff between smoothness and number of jumps
  -i,--optim_iter FLOAT=200   Albedo Update : Maximum number of iterations per step
  -g,--optim_gamma FLOAT=1    Theta Update  : Influence of the SfS term
  -n,--optim_nu FLOAT=0.0001  Theta Update  : Minimal surface weight for the output depth
  -m,--optim_mu FLOAT=0.001   Depth Update  : Weight controlling the influence of the original depth
  -k,--admm_kappa FLOAT=0.0001
                              ADMM Parameter: Initial Step size for the dual update
  -t,--admm_tau FLOAT=2       ADMM Parameter: Penalty parameter, by which kappa is increased per iteration
  -o,--admm_tolerance FLOAT=1e-06
                              ADMM Parameter: Tolerance of the relative error between theta and thetaZ
  -e,--admm_tolerance_EL FLOAT=1e-06
                              ADMM Parameter: Tolerance of the residual of the primal dual update
  -d,--dataset_path TEXT=/home/flo/projects/sfs-sr-single/data/android/
                              Data Parameter: Path to the root directory of data
  -f,--dataset_frame_num INT=0
                              Data Parameter: Number/Frame of in a set of data
  -r,--dataset_resolution INT=[640,480] ... REQUIRED
                              Data Parameter: Width of the upsampled data
  -b,--dataset_depth_sigma FLOAT
                              Data Parameter: Blur the initial depth
  --dataset_gt_depth          Data Parameter: Use optimal depth for LR depth (will be bilaterally filtered before usage)
  --dataset_gt_albedo         Data Parameter: Use optimal albedo as input
  --dataset_gt_light          Data Parameter: Use optimal light as input
  --dataset_smooth_depth      Data Parameter: Smooth depth initialization
  --dataset_prefer_image      Data Parameter: Use the loaded image over the generated
  -s,--output_results_folder TEXT
                              Out Parameter : Path to save output images, results.
  -p,--output_run_folder TEXT Out Parameter : Run folder to save output to.
  --iter_theta_outer INT      Iterations Parameter: Number of outer theta iterations
  --iter_theta_inner INT      Iterations Parameter: Number of inner theta iterations
  --iter_depth_outer INT      Iterations Parameter: Number of outer depth iterations
  --iter_depth_inner INT      Iterations Parameter: Number of inner depth iterations
```


## Parameterization

If the output is still very close to the original depth, try:
- Increasing the smoothing (`--optim_nu`). This will most likely lead to oversmooth depth maps, that are however visually pleasing.
- Decrease the fidelty (`--optim_mu`). This will allow the depth to change more and not be as close to the original depth.
- Increasing the number of iterations (`--iter_theta_inner`/`--iter_theta_outer`) for the Theta update. This will however increase the runtime a bit.
- Decreasing the initial ADMM step size (`--admm_kappa`) or penalty (`--admm_tau`). I have not seen improvements for kappa < 1e-6. Reducing tau to <1.5 will lead to very long runtimes.
- If the optimization stops after one iteration, disable the tolerance stopping criterion (`--admm_tolerance` to 0) or decrease the tolerances.

For original images with motion blur, it might be neccessary to increase the influence of the SfS data term (`--optim_gamma`).

If the albedo is under segmented, lower the jump penalty parameter `--optim_lambda`. If oversegmented, increase the parameter. Generally, low values will need more iterations to converge (as the areas with constant colors are bigger).

## Dataset

The dataset implementation will expect to find 
- an image named `color.png`
- a low resolution depth map `depth.png`
- a (optional) mask `mask.png`
- two (optional) files containing the camera parameters for depth and color images.

If there is a `depth.exr`, `albedo.png` or `light.txt` present, you can also use the `--dataset_gt_...` parameters to use and not optimize the albedos.


# Thanks to Björn Häfner
whose original code I ported to some extent. 
You can find it at his [github](https://github.com/BjoernHaefner/DepthSRfromShading) as well as the original [publication](https://vision.in.tum.de/_media/spezial/bib/haefner2018cvpr.pdf)

# License
This work is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License with an additional request:

If you make use of the library or this code in any scientific publication, please refer to this repository and cite the original paper
```
@inproceedings{Haefner2018CVPR,
 title = {Fight ill-posedness with ill-posedness: Single-shot variational depth super-resolution from shading},
 author =  {Haefner, B. and Quéau, Y. and Möllenhoff, T. and Cremers, D.},
 booktitle = {IEEE Conference on Computer Vision and Pattern Recognition (CVPR)},
 year = {2018},
 titleurl = {},
}
```

