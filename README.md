# DeepSDF

This is an implementation of the CVPR '19 paper "DeepSDF: Learning Continuous Signed Distance Functions for Shape Representation" by Park et al.

## File Organization

The various Python scripts assume a shared organizational structure such that the output from one script can easily be used as input to another. This is true for both preprocessed data as well as experiments which make use of the datasets.

##### Data Layout

The DeepSDF code allows for pre-processing of meshes from multiple datasets and stores them in a unified data source. It also allows for separation of meshes according to class at the dataset level. The structure is as follows:

Subsets of the unified data source can be reference using split files, which are stored in a simple JSON format. For examples, see `examples/splits/`. 

The file `datasources.json` stores a mapping from named datasets to paths indicating where the data came from. This file is referenced again during evaluation to compare against ground truth meshes (see below), so if this data is moved this file will need to be updated accordingly.

##### Experiment Layout

Each DeepSDF experiment is organized in an "experiment directory", which collects all of the data relevant to a particular experiment. The structure is as follows:

```
<experiment_name>/
    specs.json
    Logs.pth
    LatentCodes/
        <Epoch>.pth
    ModelParameters/
        <Epoch>.pth
    OptimizerParameters/
        <Epoch>.pth
    Reconstructions/
        <Epoch>/
            Codes/
                <MeshId>.pth
            Meshes/
                <MeshId>.pth
    Evaluations/
        Chamfer/
            <Epoch>.json
        EarthMoversDistance/
            <Epoch>.json
```

The only file that is required to begin an experiment is 'specs.json', which sets the parameters, network architecture, and data to be used for the experiment.

## How to Use DeepSDF

### Preparation
Anaconda environment yml file uploaded for your convenience.<br>
To use conda env, run conda env create -f environment.yml.

[8]: https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-vscode


### Attention!

If you train the model using colab, please use gdown to download your data from google drive. Reading files directly from google drives slow down your training process.<br>

Original Data stored in google drive (Shared drives/Dataset_ShapeNetCore/data/ShapeNetCore.v2)

### Pre-processing the Data （Original method / Preferred)

```
In order to use mesh data for training a DeepSDF model, the mesh will need to be pre-processed. This can be done with the `preprocess_data.py` executable. The preprocessing code is in C++ and has the following requirements:

- [CLI11]
- [Pangolin]
- [nanoflann]
- [Eigen3]

Preprocess build steps:

apt install git cmake build-essential libglfw3-dev libgles2-mesa-dev libgtest-dev libeigen3-dev

# Install CLI11
git clone https://github.com/CLIUtils/CLI11.git
cd CLI11
mkdir build
cd build
git submodule update --init
cmake ..
cmake --build .
sudo cmake --install .
cd ../..

# Install Pangolin
git clone --recursive https://github.com/stevenlovegrove/Pangolin.git
cd Pangolin
./scripts/install_prerequisites.sh all
git checkout v0.6
mkdir build && cd build
cmake ..
cmake --build .
sudo cmake --install .
cd ../..

# Install nanoflann
git clone https://github.com/jlblancoc/nanoflann.git
cd nanoflann
mkdir build && cd build
cmake ..
make
sudo make install
mkdir /usr/local/include/nanoflann
cp /usr/local/include/nanoflann.hpp /usr/local/include/nanoflann
cd ../..

# DeepSDF
git clone https://github.com/facebookresearch/DeepSDF.git
cd DeepSDF

###### Comment out line 97 of src/ShaderProgram.cpp
line 97: in int gl_PrimitiveID ;
sed -i "97 s/^/\/\//" src/ShaderProgram.cpp

git submodule update --init
mkdir build && cd build
cmake .. -DCMAKE_CXX_STANDARD=17
make

Possible Error:
If you see error related to Thread. Add find_package(Threads) to CMakeLists.txt

With these dependencies, the build process follows the standard CMake procedure:

mkdir build
cd build
cmake ..
make -j

Once this is done there should be two executables in the `DeepSDF/bin` directory, one for surface sampling and one for SDF sampling. With the binaries, the dataset can be preprocessed using `preprocess_data.py`.


#### Preprocessing with Headless Rendering (Original method)

The preprocessing script requires an OpenGL context, and to acquire one it will open a (small) window for each shape using Pangolin. If Pangolin has been compiled with EGL support, you can use the "headless" rendering mode to avoid the windows stealing focus. Pangolin's headless mode can be enabled by setting the `PANGOLIN_WINDOW_URI` environment variable as follows:

export MESA_GL_VERSION_OVERRIDE=3.3 (required)
export PANGOLIN_WINDOW_URI=headless://
```

### Pre-processing the Data （Not Preferred)
Run ShapeNetData.ipynb<br>
If using wsl, install google drive and change source_dir to '/mnt/g/Shared drives/Dataset_ShapeNetCore/data/ShapeNetCore.v2'<br>
This method utilizes the package

- [mesh_to_sdf][5]

[5]: https://github.com/marian42/mesh_to_sdf


### Training a Model

Once data has been preprocessed, models can be trained using:

```
python train_deep_sdf.py -e <experiment_directory> ('examples/sofas')
```

Models can also be trained using:

```
colab_train_deep_sdf.ipynb
```
Check the main_function() and change parameters for different training <br>
Change datasource in specs.json to google drive data folder like "/mnt/g/Shared drives/Github/DeepSDF/data" for wsl <br>
If you see errors like killed (in Linux) or cuda out of memory (colab), try to increase batch_split <br>

Parameters of training are stored in a "specification file" in the experiment directory, which (1) avoids proliferation of command line arguments and (2) allows for easy reproducibility. This specification file includes a reference to the data directory and a split file specifying which subset of the data to use for training.

##### Visualizing Training Progress

All intermediate results from training are stored in the experiment directory. To visualize the progress of a model during training, run:

```
python plot_log.py -e <experiment_directory>
```
Or colab_plot_log.ipynb <br>

By default, this will plot the loss but other values can be shown using the `--type` flag.<br>
"epoch", "loss", "learning_rate", "timing", "latent_magnitude", "param_magnitude"

##### Continuing from a Saved Optimization State

If training is interrupted, pass the `--continue` flag along with a epoch index to `train_deep_sdf.py` to continue from the saved state at that epoch. Note that the saved state needs to be present --- to check which checkpoints are available for a given experiment, check the `ModelParameters', 'OptimizerParameters', and 'LatentCodes' directories (all three are needed).

### Reconstructing Meshes

To use a trained model to reconstruct explicit mesh representations of shapes from the test set, run:

```
python reconstruct.py -e <experiment_directory>
```
Or colab_reconstruct.ipynb

This will use the latest model parameters to reconstruct all the meshes in the split. To specify a particular checkpoint to use for reconstruction, use the ```--checkpoint``` flag followed by the epoch number. Generally, test SDF sampling strategy and regularization could affect the quality of the test reconstructions. For example, sampling aggressively near the surface could provide accurate surface details but might leave under-sampled space unconstrained, and using high L2 regularization coefficient could result in perceptually better but quantitatively worse test reconstructions.

### Visualizing Meshes

Run visualize.ipynb

## Examples

Here's a list of commands for a typical use case of training and evaluating a DeepSDF model using the "sofa" class of the ShapeNet version 2 dataset. 

```
# navigate to the DeepSdf root directory
cd [...]/DeepSdf

# create a home for the data
mkdir data

# pre-process the sofas training set (SDF samples)
python preprocess_data.py --data_dir data --source [...]/ShapeNetCore.v2/ --name ShapeNetV2 --split examples/splits/sv2_sofas_train.json --skip

# train the model
python train_deep_sdf.py -e examples/sofas

# pre-process the sofa test set (SDF samples)
python preprocess_data.py --data_dir data --source [...]/ShapeNetCore.v2/ --name ShapeNetV2 --split examples/splits/sv2_sofas_test.json --test --skip

# pre-process the sofa test set (surface samples)
python preprocess_data.py --data_dir data --source [...]/ShapeNetCore.v2/ --name ShapeNetV2 --split examples/splits/sv2_sofas_test.json --surface --skip

# reconstruct meshes from the sofa test split (after 2000 epochs)
python reconstruct.py -e examples/sofas -c 2000 --split examples/splits/sv2_sofas_test.json -d data --skip

# evaluate the reconstructions
python evaluate.py -e examples/sofas -c 2000 -d data -s examples/splits/sv2_sofas_test.json 


## License

DeepSDF is relased under the MIT License.

