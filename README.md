# ColmapForVisSat Patches
This repository uses `git patch` to implement [ColmapForVisSat](https://github.com/Kai-46/ColmapForVisSat).

Since `ColmapForVisSat` is a full copy of [Colmap](https://github.com/colmap/colmap), it is difficult to maintain. By relying on `git apply <patch>` this repository offers a simpler approach to incorporate the latest changes of the original `Colmap` library.

Different `patched Colmap versions` (mimicking `ColmapForVisSat`) can be found [here](https://github.com/SBCV/colmap) - check the individual branches.

## Compatibility

**In contrast to the original ColmapForVisSat library, this repository supports current CUDA versions (validated with CUDA 13.2 on Ubuntu 24.04; earlier patch set generations were also validated with CUDA 11.8).**

Current patch files are created for the Colmap commit `7d0c470c7de051c8d28fbeb7142278ac56722aa2 (2026-10-07 14:35:31 +0200)` (35 patch files; the `mvs/texture_mapping_test.cc` patch only adapts an upstream unit test to the satellite `mvs::Image` and does not affect the binary).
The authoritative value is always the one in [compatible_commit.txt](compatible_commit.txt).

## Camera model

Since Colmap commit `a938058e` ([PR #4815](https://github.com/colmap/colmap/pull/4815))
the skewed pinhole camera the satellite pipeline needs (`fx, fy, cx, cy, s`)
is an upstream model named **`SKEWED_PINHOLE`** (model id 18). Earlier patch
set generations defined the same model themselves under the name
`PERSPECTIVE`, appended to the end of the model enum, which made its numeric
id drift with upstream's model list (11 → 12 → 14 → 18). That problem is
gone: the patches no longer define a camera model. The only camera-model
hunk left is in `src/colmap/sensor/models/pinhole.h`: `SkewedPinholeCameraModel`
gets a `HasBogusParams` override that returns `false`, because the cameras of
the RPC perspective approximation fail Colmap's focal-length-ratio and
principal-point heuristics by construction (focal length ratios of 250-7000
against a default maximum of 10) and would otherwise be dropped from
triangulation and filtering.

Consequences for **data compatibility**:

- Text models (`cameras.txt`) written by earlier generations name the model
  `PERSPECTIVE`; rename it to `SKEWED_PINHOLE` (same parameters, same order)
  to load them with a current build.
- `database.db` and binary models (`cameras.bin`) persist the numeric id.
  Artifacts of the `b0af55be` and `dbb41680` generations (id 18) load
  unchanged; artifacts of older generations (ids 11, 12, 14) must go through
  the text format.

## Known failing upstream tests

Building with `-DTESTS_ENABLED=ON` compiles every upstream test; five test
binaries fail by satellite design and are left as they are:

- `mvs/fusion_test` and `mvs/workspace_test` abort in the satellite
  `mvs::Model::ReadFromCOLMAP`, whose CHECK admits only `PINHOLE` and
  `SKEWED_PINHOLE` cameras (the vanilla tests synthesize another model).
- `mvs/depth_map_test` (`DepthMap.Empty`): the satellite depth sentinel is
  `-1e20`, the vanilla one `-1`.
- `scene/reconstruction_test` (`Reconstruction.ReadAutoDetectFormat`): the
  satellite `Reconstruction::Read` is text-only.
- `sensor/models_test` (`SkewedPinhole.Nominal`,
  `SkewedPinhole.BoundsSkewRelativeToFocalLength`): they test the
  bogus-parameter check that the bypass above disables for this model.

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
