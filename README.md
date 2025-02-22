[![Python Version](https://img.shields.io/pypi/pyversions/cellfinder-core.svg)](https://pypi.org/project/cellfinder-core)
[![PyPI](https://img.shields.io/pypi/v/cellfinder-core.svg)](https://pypi.org/project/cellfinder-core)
[![Downloads](https://pepy.tech/badge/cellfinder-core)](https://pepy.tech/project/cellfinder-core)
[![Wheel](https://img.shields.io/pypi/wheel/cellfinder-core.svg)](https://pypi.org/project/cellfinder-core)
[![Development Status](https://img.shields.io/pypi/status/cellfinder-core.svg)](https://github.com/brainglobe/cellfinder-core)
[![Tests](https://img.shields.io/github/workflow/status/brainglobe/cellfinder-core/tests)](
    https://github.com/brainglobe/cellfinder-core/actions)
[![Coverage Status](https://coveralls.io/repos/github/brainglobe/cellfinder-core/badge.svg?branch=main)](https://coveralls.io/github/brainglobe/cellfinder-core?branch=main)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/python/black)
[![Contributions](https://img.shields.io/badge/Contributions-Welcome-brightgreen.svg)](https://docs.brainglobe.info/cellfinder/contributing)
[![Website](https://img.shields.io/website?up_message=online&url=https%3A%2F%2Fcellfinder.info)](https://cellfinder.info)
[![Twitter](https://img.shields.io/twitter/follow/findingcells?style=social)](https://twitter.com/findingcells)
# cellfinder-core
Standalone cellfinder cell detection algorithm 

This package implements the cell detection algorithm from 
[Tyson, Rousseau & Niedworok et al. (2021)](https://www.biorxiv.org/content/10.1101/2020.10.21.348771v2) 
without any dependency on data type (i.e. it can be used outside of 
whole-brain microscopy). 

`cellfinder-core` supports the 
[cellfinder](https://github.com/brainglobe/cellfinder) software for 
whole-brain microscopy analysis, and the algorithm can also be implemented in 
[napari](https://napari.org/index.html) using the 
[cellfinder napari plugin](https://github.com/brainglobe/cellfinder-napari).

---

## Instructions

### Installation
`cellfinder-core` supports Python >=3.7, 
and works across Linux, Windows, and should work on most versions of macOS 
(although this is not tested).

Assuming you have a Python environment set up 
(e.g. [using conda](https://docs.brainglobe.info/cellfinder/using-conda)), 
you can install `cellfinder-core` with:
```bash
pip install cellfinder-core
```

Once you have [installed napari](https://napari.org/index.html#installation). 
You can install napari either through the napari plugin installation tool, or 
directly from PyPI with:
```bash
pip install cellfinder-napari
```

N.B. To speed up cellfinder, you need CUDA & cuDNN installed. Instructions 
[here](https://docs.brainglobe.info/cellfinder/installation/using-gpu).

### Usage
Before using cellfinder-core, it may be useful to take a look at the 
[preprint](https://www.biorxiv.org/content/10.1101/2020.10.21.348771v2) which
outlines the algorithm.

The API is not yet fully documented. For an idea of what the parameters do, 
see the documentation for the cellfinder whole-brain microscopy image analysis 
command-line tool ([cell candidate detection](https://docs.brainglobe.info/cellfinder/user-guide/command-line/candidate-detection),
[cell candidate classification](https://docs.brainglobe.info/cellfinder/user-guide/command-line/classification)).
It may also be useful to try the
[cellfinder napari plugin](https://github.com/brainglobe/cellfinder-napari)
so you can adjust the parameters in a GUI. 

#### To run the full pipeline (cell candidate detection and classification)
```python
from cellfinder_core.main import main as cellfinder_run
import tifffile

signal_array = tifffile.imread("/path/to/signal_image.tif")
background_array = tifffile.imread("/path/to/background_image.tif")

voxel_sizes = [5, 2, 2] # in microns
detected_cells = cellfinder_run(signal_array,background_array,voxel_sizes)
```

The output is a list of
[imlib Cell objects](https://github.com/adamltyson/imlib/blob/51ec5a8053e738776ceaa8d44e531b3c4b0e29d8/imlib/cells/cells.py#L15).
Each `Cell` has a centroid coordinate, and a type:

```python
print(detected_cells[0])
# Cell: x: 132, y: 308, z: 10, type: 2
```

Cell type 2 is a "real" cell, and Cell type 1 is a "rejected" object (i.e. 
not classified as a cell):

```python
from imlib.cells.cells import Cell
print(Cell.CELL)
# 2

print(Cell.NO_CELL)
# 1
```

#### Saving the results
If you want to save the detected cells for use in other BrainGlobe software (e.g. the 
[cellfinder napari plugin](https://docs.brainglobe.info/cellfinder-napari/introduction)),
you can save in the cellfinder XML standard:
```python
from imlib.IO.cells import save_cells
save_cells(detected_cells, "/path/to/cells.xml")
```
You can load these back with:
```python
from imlib.IO.cells import get_cells
cells = get_cells("/path/to/cells.xml")
```


#### Using dask for lazy loading
`cellfinder-core` supports most array-like objects. Using 
[Dask arrays](https://docs.dask.org/en/latest/array.html) allows for lazy 
loading of data, allowing large (e.g. TB) datasets to be processed. 
`cellfinder-core` comes with a function 
(based on [napari-ndtiffs](https://github.com/tlambert03/napari-ndtiffs)) to 
load a series of image files (e.g. a directory of 2D tiff files) as a Dask 
array. `cellfinder-core` can then be used in the same way as with a numpy array.

```python
from cellfinder_core.main import main as cellfinder_run
from cellfinder_core.tools.IO import read_with_dask

signal_array = read_with_dask("/path/to/signal_image_directory")
background_array = read_with_dask("/path/to/background_image_directory")

voxel_sizes = [5, 2, 2] # in microns
detected_cells = cellfinder_run(signal_array,background_array,voxel_sizes)

```

#### Running the cell candidate detection and classification separately.
```python
import tifffile
from pathlib import Path

from cellfinder_core.detect import detect
from cellfinder_core.classify import classify

signal_array = tifffile.imread("/path/to/signal_image.tif")
background_array = tifffile.imread("/path/to/background_image.tif")
voxel_sizes = [5, 2, 2] # in microns

home = Path.home()
install_path = home / ".cellfinder" # default

start_plane=0
end_plane=-1
trained_model=None
model_weights=None
model="resnet50_tv"
batch_size=32
n_free_cpus=2
network_voxel_sizes=[5, 1, 1]
soma_diameter=16
ball_xy_size=6
ball_z_size=15
ball_overlap_fraction=0.6
log_sigma_size=0.2
n_sds_above_mean_thresh=10
soma_spread_factor=1.4
max_cluster_size=100000
cube_width=50
cube_height=50
cube_depth=20
network_depth="50"

cell_candidates = detect.main(
    signal_array,
    start_plane,
    end_plane,
    voxel_sizes,
    soma_diameter,
    max_cluster_size,
    ball_xy_size,
    ball_z_size,
    ball_overlap_fraction,
    soma_spread_factor,
    n_free_cpus,
    log_sigma_size,
    n_sds_above_mean_thresh,
)

if len(cell_candidates) > 0: # Don't run if there's nothing to classify
    classified_cells = classify.main(
        cell_candidates,
        signal_array,
        background_array,
        n_free_cpus,
        voxel_sizes,
        network_voxel_sizes,
        batch_size,
        cube_height,
        cube_width,
        cube_depth,
        trained_model,
        model_weights,
        network_depth,
    )
```
#### Training the network
The training data needed are matched pairs (signal & background) of small 
(usually 50 x 50 x 100um) images centered on the coordinate of candidate cells.
These can be generated however you like, but I recommend using the 
[Napari plugin](https://docs.brainglobe.info/cellfinder-napari/user-guide/training-data-generation).

`cellfinder-core` comes with a 50-layer ResNet trained on ~100,000 data points
from serial two-photon microscopy images of mouse brains 
(available [here](https://gin.g-node.org/cellfinder/training_data)).

Training the network is likely simpler using the 
[command-line interface](https://docs.brainglobe.info/cellfinder/user-guide/training#start-training) 
or the [Napari plugin](https://docs.brainglobe.info/cellfinder-napari/user-guide/training-the-network),
but it is possible through the Python API.

```python
from pathlib import Path
from cellfinder_core.train.train_yml import run as run_training

# list of training yml files
yaml_files = [Path("/path/to/training_yml.yml)]

# where to save the output
output_directory = Path("/path/to/saved_training_data")

home = Path.home()
install_path = home / ".cellfinder"  # default

run_training(
    output_directory,
    yaml_files,
    install_path=install_path,
    learning_rate=0.0001,
    continue_training=True, # by default use supplied model
    test_fraction=0.1,
    batch_size=32,
    save_progress=True,
    epochs=10,
)
```

---
### More info

More documentation about cellfinder and other BrainGlobe tools can be 
found [here](https://docs.brainglobe.info). 
 
This software is at a very early stage, and was written with our data in mind. 
Over time we hope to support other data types/formats. If you have any 
questions or issues, please get in touch by 
[email](mailto:code@adamltyson.com?subject=cellfinder-core), 
[on the forum](https://forum.image.sc/tag/brainglobe) or by 
[raising an issue](https://github.com/brainglobe/cellfinder-core/issues).

---
## Illustration

### Introduction
cellfinder takes a stitched, but otherwise raw dataset with at least 
two channels:
 * Background channel (i.e. autofluorescence)
 * Signal channel, the one with the cells to be detected:
 
![raw](https://raw.githubusercontent.com/brainglobe/cellfinder/master/resources/raw.png)
**Raw coronal serial two-photon mouse brain image showing labelled cells**


### Cell candidate detection
Classical image analysis (e.g. filters, thresholding) is used to find 
cell-like objects (with false positives):

![raw](https://raw.githubusercontent.com/brainglobe/cellfinder/master/resources/detect.png)
**Candidate cells (including many artefacts)**


### Cell candidate classification
A deep-learning network (ResNet) is used to classify cell candidates as true 
cells or artefacts:

![raw](https://raw.githubusercontent.com/brainglobe/cellfinder/master/resources/classify.png)
**Cassified cell candidates. Yellow - cells, Blue - artefacts**

---
## Citing cellfinder
If you find this plugin useful, and use it in your research, please cite the preprint outlining the cell detection algorithm:
> Tyson, A. L., Rousseau, C. V., Niedworok, C. J., Keshavarzi, S., Tsitoura, C., Cossell, L., Strom, M. and Margrie, T. W. (2021) “A deep learning algorithm for 3D cell detection in whole mouse brain image datasets’ PLOS Computational Biology, 17(5), e1009074
[https://doi.org/10.1371/journal.pcbi.1009074](https://doi.org/10.1371/journal.pcbi.1009074)

**If you use this, or any other tools in the brainglobe suite, please
 [let us know](mailto:code@adamltyson.com?subject=cellfinder-core), and 
 we'd be happy to promote your paper/talk etc.**
