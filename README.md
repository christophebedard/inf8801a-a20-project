# inf8801a-a20-project

INF8801A - applications multimédias (automne 2020)

Project on SLAM.

## Table of contents

* [Goal](#Goal)
* [Prerequisites](#Prerequisites)
* [Setup](#Setup)
* [ORB-SLAM2 modification](#ORB-SLAM2-modification)
* [Run](#Run)
    * [Prepare ground truth data](#Prepare-ground-truth-data)
    * [Original implementation](#Original-implementation)
    * [Modified implementation](#Modified-implementation)
* [References](#References)

## Goal

The goal is to see how removing ORB's [[1]](#References) rotation invariance impacts ORB-SLAM2 [[2]](#References).

We compare using sequence 00 of the KITTI odometry dataset [[3]](#References).

## Prerequisites

1. Use Ubuntu 18.04
    * Ubuntu 20.04 could be made to work, but it's simpler to use 18.04 because of ORB-SLAM2 and its dependencies

## Setup

1. Create a workspace directory in which to clone the various libraries and tools
    ```sh
    $ cd ~
    $ mkdir ws
    $ cd ws
    ```
1. Clone [Pangolin](https://github.com/stevenlovegrove/Pangolin), install its dependencies, build, and install
    * See [*Dependencies*](https://github.com/stevenlovegrove/Pangolin#dependencies) and [*Building*](https://github.com/stevenlovegrove/Pangolin#building)
    ```sh
    $ cd ~/ws
    $ git clone https://github.com/stevenlovegrove/Pangolin.git
    $ sudo apt-get update
    $ sudo apt-get install -y libgl1-mesa-dev libglew-dev cmake ffmpeg libavcodec-dev libavutil-dev libavformat-dev libswscale-dev libavdevice-dev
    $ cd Pangolin
    $ mkdir build
    $ cd build
    $ cmake ..
    $ cmake --build .
    $ sudo make install
    ```
1. Install ORB-SLAM2 dependencies
    * See [*Prerequisites*](https://github.com/christophebedard/ORB_SLAM2#2-prerequisites)
    * Make sure you have >= C++11
    * Tested with Eigen 3.3.4 and OpenCV 3.2.0
    ```sh
    $ sudo apt-get install -y libeigen3-dev libopencv-dev
    ```
1. Clone ORB-SLAM2 (using my fork for a small compilation fix)
    ```sh
    $ cd ~/ws
    $ git clone https://github.com/christophebedard/ORB_SLAM2.git
    ```
1. Clone this repository
    ```sh
    $ cd ~/ws
    $ git clone https://github.com/christophebedard/inf8801a-a20-project.git
    ```
1. Build ORB-SLAM2
    * See [*Building ORB-SLAM2 library and examples*](https://github.com/christophebedard/ORB_SLAM2#3-building-orb-slam2-library-and-examples)
    ```sh
    $ cd ~/ws/ORB_SLAM2
    $ chmod +x build.sh
    $ ./build.sh
    ```
1. Download [KITTI odometry dataset](http://www.cvlibs.net/datasets/kitti/eval_odometry.php)
    * Two separate downloads/archives: "odometry data set (grayscale)" and "ground truth poses"
    * Extract content so that the files structure matches the following
    ```sh
    $ cd ~/ws
    $ mkdir data
    $ ls data/KITTI/data_odometry_gray/dataset/sequences/00/
    $ ls data/KITTI/data_odometry_poses/dataset/poses/00.txt
    ```
1. Clone [evo](https://github.com/MichaelGrupp/evo.git) [[4]](#References), switch to a supported branch, and install
    ```sh
    $ cd ~/ws
    $ git clone https://github.com/MichaelGrupp/evo.git
    $ cd evo
    $ git checkout v1.12.0
    $ pip install .
    ```
    * Add `/home/$USER/.local/bin/` to `PATH` to be able to call the `evo_*` commands, e.g. add this line to your `~/.bashrc` (and source `~/.bashrc` or start another terminal)
    ```
    export PATH=$PATH:/home/$USER/.local/bin/
    ```

## ORB-SLAM2 modification

See block comments (`/** ... */`) in [`ORBextractor.cc`](./ORBextractor.cc) and [`ORBextractor.h`](./ORBextractor.h).
The comments explain the ORB extraction steps as well as the modifications that have been made to remove the orientation invariance.

## Run

### Prepare ground truth data

1. Use evo to combine KITTI00's timestamps file (`data_odometry_gray/dataset/sequences/00/times.txt`) and ground truth poses file (`data_odometry_poses/dataset/poses/00.txt`) into a TUM-format trajectory file (timestamp + pose). This is because [monocular ORB-SLAM2 outputs TUM-format data even for KITTI data](https://github.com/christophebedard/ORB_SLAM2/blob/ec64ccef347a3ae9dd2fd0519ee53d5f300e9de8/Examples/Monocular/mono_kitti.cc#L122). This produces our reference file: `kitti00_groundtruth.traj`.
    ```sh
    $ cd ~/ws
    $ python evo/contrib/kitti_poses_and_timestamps_to_trajectory.py data/KITTI/data_odometry_poses/dataset/poses/00.txt data/KITTI/data_odometry_gray/dataset/sequences/00/times.txt kitti00_groundtruth.traj
    ```

### Original implementation

1. Build (if not already built)
    ```sh
    $ cd ~/ws/ORB_SLAM2
    $ ./build.sh
    ```
1. Run ORB-SLAM2 on KITTI00. This will export a TUM-format file (in traj format, i.e. timestamp + pose). This produces our estimation file `KeyFrameTrajectory.txt`, which we will rename to `kitti00_orb-slam2`.
    ```sh
    $ cd ~/ws
    $ ./ORB_SLAM2/Examples/Monocular/mono_kitti ORB_SLAM2/Vocabulary/ORBvoc.txt ORB_SLAM2/Examples/Monocular/KITTI00-02.yaml data/KITTI/data_odometry_gray/dataset/sequences/00
    $ mv KeyFrameTrajectory.txt kitti00_orb-slam2
    ```
1. (optional) Plot/compare estimated trajectory to ground truth/reference (use matplotlib's save function to export to PNG)
    * without scale correction (but aligned so that it is in the middle), which is the default since monocular SLAM does not solve for scale
        ```sh
        $ cd ~/ws
        $ evo_traj tum kitti00_orb-slam2 --ref=kitti00_groundtruth.traj -p --plot_mode=xz
        ```
    * with scale correction
        ```sh
        $ cd ~/ws
        $ evo_traj tum kitti00_orb-slam2 --ref=kitti00_groundtruth.traj --align --correct_scale -p --plot_mode=xz
        ```
1. Plot relative pose error (RPE) and absolute pose error (APE) [[5]](#References), [[6]](#References) (use matplotlib's save function to export to PNG)
    * RPE
        ```sh
        $ cd ~/ws
        $ evo_rpe tum --align --correct_scale -p --plot_mode=xz kitti00_groundtruth.traj kitti00_orb-slam2
        ```
    * APE
        ```sh
        $ cd ~/ws
        $ evo_ape tum --align --correct_scale -p --plot_mode=xz kitti00_groundtruth.traj kitti00_orb-slam2
        ```

### Modified implementation

1. Replace implementation of `ORBextractor` in ORB-SLAM2 with the modified implementation from this repo
    ```sh
    $ cd ~/ws
    $ cp -f inf8801a-a20-project/ORBextractor.cc ORB_SLAM2/src/
    $ cp -f inf8801a-a20-project/ORBextractor.h ORB_SLAM2/include/
    ```
1. Rebuild
    ```sh
    $ cd ~/ws/ORB_SLAM2
    $ ./build.sh
    ```
1. Re-do [original steps](#Original-implementation), but renaming the estimated trajectory file to `kitti00_orb-slam2_modified`
    1. Run our modified version of ORB-SLAM2 on KITTI00. This produces our estimation file `KeyFrameTrajectory.txt`, which we will rename to `kitti00_orb-slam2_modified`.
        ```sh
        $ cd ~/ws
        $ ./ORB_SLAM2/Examples/Monocular/mono_kitti ORB_SLAM2/Vocabulary/ORBvoc.txt ORB_SLAM2/Examples/Monocular/KITTI00-02.yaml data/KITTI/data_odometry_gray/dataset/sequences/00
        $ mv KeyFrameTrajectory.txt kitti00_orb-slam2_modified
        ```
    1. (optional) Plot/compare estimated trajectory to ground truth/reference
        * without scale correction
            ```sh
            $ cd ~/ws
            $ evo_traj tum kitti00_orb-slam2_modified --ref=kitti00_groundtruth.traj -p --plot_mode=xz
            ```
        * with scale correction
            ```sh
            $ cd ~/ws
            $ evo_traj tum kitti00_orb-slam2_modified --ref=kitti00_groundtruth.traj --align --correct_scale -p --plot_mode=xz
            ```
    1. Plot RPE and APE
        * RPE
            ```sh
            $ cd ~/ws
            $ evo_rpe tum --align --correct_scale -p --plot_mode=xz kitti00_groundtruth.traj kitti00_orb-slam2_modified
            ```
        * APE
            ```sh
            $ cd ~/ws
            $ evo_ape tum --align --correct_scale -p --plot_mode=xz kitti00_groundtruth.traj kitti00_orb-slam2_modified
            ```

## References

[1] E. Rublee, V. Rabaud, K. Konolige and G. Bradski, "ORB: An efficient alternative to SIFT or SURF," *2011 International Conference on Computer Vision*, Barcelona, 2011, pp. 2564-2571, doi: 10.1109/ICCV.2011.6126544.

[2] R. Mur-Artal, J. M. M. Montiel and J. D. Tardós, "ORB-SLAM: A Versatile and Accurate Monocular SLAM System," in *IEEE Transactions on Robotics*, vol. 31, no. 5, pp. 1147-1163, Oct. 2015, doi: 10.1109/TRO.2015.2463671.

[3] A. Geiger, P. Lenz and R. Urtasun, "Are we ready for autonomous driving? The KITTI vision benchmark suite," *2012 IEEE Conference on Computer Vision and Pattern Recognition*, Providence, RI, 2012, pp. 3354-3361, doi: 10.1109/CVPR.2012.6248074.

[4] M. Grupp, "evo: Python package for the evaluation of odometry and SLAM.," 2017. [Online]. Available: https://github.com/MichaelGrupp/evo.

[5] J. Sturm, N. Engelhard, F. Endres, W. Burgard and D. Cremers, "A benchmark for the evaluation of RGB-D SLAM systems," *2012 IEEE/RSJ International Conference on Intelligent Robots and Systems*, Vilamoura, 2012, pp. 573-580, doi: 10.1109/IROS.2012.6385773.

[6] D. Prokhorov, D. Zhukov, O. Barinova, K. Anton and A. Vorontsova, "Measuring robustness of Visual SLAM," *2019 16th International Conference on Machine Vision Applications (MVA)*, Tokyo, Japan, 2019, pp. 1-6, doi: 10.23919/MVA.2019.8758020.
