# ColmapForVisSat Patches
This repository uses `git patch` to implement [ColmapForVisSat](https://github.com/Kai-46/ColmapForVisSat).

Since `ColmapForVisSat` is a full copy of [Colmap](https://github.com/colmap/colmap), it is difficult to maintain. By relying on `git apply <patch>` this repository offers a simpler approach to incorporate the latest changes of the original `Colmap` library.

Different `patched Colmap versions` (mimicking `ColmapForVisSat`) can be found [here](https://github.com/SBCV/colmap) - check the individual branches.

## Compatibility

**In contrast to the original ColmapForVisSat library, this repository supports current CUDA versions (validated with CUDA 13.2 on Ubuntu 24.04; earlier patch set generations were also validated with CUDA 11.8).**

Current patch files are created for the Colmap commit `b0af55be2808366609ee2023c5b9a980fdabd1ea (2026-08-05 16:20:14 +0000)`.
The authoritative value is always the one in [compatible_commit.txt](compatible_commit.txt).

## Apply a set of satellite specific patch files to the original Colmap repository

This repository uses [Repository Patcher](https://github.com/SBCV/RepositoryPatcher) to apply the patches. Follow the install instructions [here](https://github.com/SBCV/RepositoryPatcher?tab=readme-ov-file#installation) and the configuration instructions [here](https://github.com/SBCV/RepositoryPatcher?tab=readme-ov-file#configuration-of-the-target-repository-and-the-patch-directory).

Afterwards apply the patch files in the `patches folder` using the instructions [here](https://github.com/SBCV/RepositoryPatcher?tab=readme-ov-file#apply-a-set-of-patch-files-to-the-target-repository).

## Build the patched Colmap repository
- Run `sudo apt-get install libmetis-dev libopenimageio-dev openimageio-tools libopencv-dev libqt5svg5-dev` (the OpenImageIO/OpenCV/Qt5Svg packages are required since the `b0af55be`-era Colmap replaced FreeImage with OpenImageIO and added a Qt5Svg dependency)
- If anaconda/miniconda is installed, make sure to run `conda deactivate` before running `cmake`.
- Follow the [official install insctructions of Colmap for Linux](https://colmap.github.io/install.html#linux).

## For contributors/developers

### Update patch files for newer Colmap versions
See the instructions [here](https://github.com/SBCV/RepositoryPatcher?tab=readme-ov-file#update-patch-files-for-newer-versions-of-the-target-repository).

### Create a branch for the patched colmap version

```
git clone https://github.com/SBCV/colmap
cd $PathToSBCVColmap
git switch -c vissat_<patches-commit-hash>
```

## Non-trivial conversion notes

### Inconsistencies
- ```src/base/reconstruction.cc```  -->  ```Reconstruction::Normalize()``` -->  ```if (translation_applied && scale_applied) {```
  - add
    - ```const Eigen::Vector3d mean_coord = std::get<2>(bound);```
    - ```const Eigen::Vector3d translation = mean_coord;```

- ```src/controllers/bundle_adjustment.cc``` --> ```void BundleAdjustmentController::Run() {``` -->
  - can not uncomment ```reconstruction_->Normalize();```, since line does not exist.
  - => uncomment ```reconstruction_->Normalize();``` in ```src/ui/bundle_adjustment_widget.cc```
  - keep ```reconstruction_->Normalize();``` in ```src/sfm/incremental_mapper.cc```, since these are not touched in the adapted colmap version

### Obsolete ? (TODO)
```
- src/base/reconstruction.cc  -->  void Reconstruction::WriteCamerasText() -->  file << std::setprecision(PRECISION);
- src/base/reconstruction.cc  -->  void Reconstruction::WriteCamerasText() -->  line << std::setprecision(PRECISION);
- src/base/reconstruction.cc  -->  void Reconstruction::WriteImagesText() -->  file << std::setprecision(PRECISION);
- src/base/reconstruction.cc  -->  void Reconstruction::WriteImagesText() -->  line << std::setprecision(PRECISION);
- src/base/reconstruction.cc  -->  void Reconstruction::WritePoints3DText() -->  file << std::setprecision(PRECISION);
- src/base/reconstruction.cc  -->  void Reconstruction::WritePoints3DText() -->  line << std::setprecision(PRECISION);
- src/mvs/math.h
```

### Questionable Changes (TODO)
- ```src/mvs/patch_match_cuda.cu``` --> Deleted ```return``` in line 181/385
