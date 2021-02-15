# Fetal brain segmentation with MONAI DynUNet

MONAIfbs (MONAI Fetal Brain Segmentation) is a Pytorch-based toolkit to train and test deep learning models for automated 
fetal brain segmentation of HASTE-like MR images.
The toolkit was developed within the [GIFT-Surg][giftsurg] research project, and takes advantage of [MONAI][monai], 
a freely available, community-supported, PyTorch-based framework for deep learning in healthcare imaging.

A pre-trained dynUNet model is [provided][dynUnetmodel] and can be directly used for inference on new data using 
the script `src/inference/monai_dynunet_inference.py`. Alternatively, the script `fetal_brain_seg.py` provides
the same inference functionality within an appropriate interface to be used within the [NiftyMIC][NiftyMIC] package and by the 
executable command `niftymic_segment_fetal_brains`. See the sections [Inference][inference_section] and 
[Use within NiftyMIC][use_section] below.

More information about MONAI dynUNet can be found [here][dynUnettutorial]. This deep learning pipeline is based on the 
[nnU-Net][nnunet] self-adapting framework for U-Net-based medical image segmentation. 

### Contact information
This package was developed by [Marta B.M. Ranzini][mranzini] at the [Department of Surgical and Interventional Sciences][sie], 
[King's College London (KCL)][kcl] (2020).
If you have any questions or comments, please open an issue on GitHub or contact Prof Tom Vercauteren at 
`tom.vercauteren@kcl.ac.uk`.

## Important note
Please make sure you download the [pre-trained model][dynUnetmodel] for inference and add it in the MONAIfbs folder as follows 
`<path_to_MONAIfbs>/monaifbs/models/checkpoint_dynUnet_DiceXent.pt`. 
You can either download it manually from the webpage or you can use the `zenodo_get` tool from command line:
```
pip install zenodo-get
zenodo_get 10.5281/zenodo.4282679
tar xvd models.tar.gz
mv models <path_to_MONAIfbs>/monaifbs/
``` 

Alternatively, you can store the model at a different location, but update the [inference config file][inference_config] 
with the right path to the model to load. 

## Installation
After installing git lfs, clone the repository locally using  
`git clone https://github.com/gift-surg/MONAIfbs.git`  
 
Change to the downloaded directory and install all Python and Pytorch dependencies by running the following commands sequentially:  
`pip install -r requirements.txt`  
`pip install -e .`

*Note*: MONAI and monaifbs require Python versions >= 3.6.

*Note*: A CUDA compatible GPU is recommended for training. Inference can be run both on GPU or on CPU.


## Training
A python script was developed to train a [dynUNet][dynUnettutorial] model using [MONAI][monai]. 
dynUNet 基于 [nnU-Net][nnunet] 方法，该方法采用一系列启发式规则来从训练集中确定最佳内核大小，步幅和网络深度。

可用脚本从训练集随机抽样二维切片来训练二维dynUNet。 
默认情况下，Dice + Cross 熵用作损失函数（有其他选项可用）。

应用了整体验证策略：
（在每个3D图像中使用2D滑动窗口方法），并且验证集上的平均3D骰子得分用作最佳模型选择的指标。

#### Setting the training to run

To run the training with your own data, the following command can be used:
```
python <path_to_MONAIfbs>/monaifbs/src/train/monai_dynunet_training.py \
--train_files_list <path-to-list-of-training-files.txt> \
--validation_files_list <path-to-list-of-validation-files.txt>\
--out_folder <path-to-output-directory>
```
The files `<path-to-list-of-training-files.txt>` and `<path-to-list-of-validation-files.txt>` should be either .txt or 
.csv files storing pairs of image-segmentation filenames in each line, separated by a comma, as follows:
```
/path/to/file/for/subj1_img.nii.gz,/path/to/file/for/subj1_seg.nii.gz
/path/to/file/for/subj2_img.nii.gz,/path/to/file/for/subj2_seg.nii.gz
/path/to/file/for/subj3_img.nii.gz,/path/to/file/for/subj3_seg.nii.gz
...
```
Examples of the expected file formats are in `config/mock_train_file_list_for_dynUnet_training.txt` and 
`config/mock_valid_file_list_for_dynUnet_training.txt`.  

See `python <path_to_MONAIfbs>/monaifbs/src/train/monai_dynunet_training.py -h` for help on additional input arguments.

#### Changing the network configurations
By default, the network will be trained with the configurations defined in `config/monai_dynUnet_training_config.yml`.
See [the file][training_config] for a description of the user-defined parameters.  
To change the parameter values, create your own yaml config file following the structure [here][training_config]. The
new config file can be input as an argument when running the training as follows:
```
python <path_to_MONAIfbs>/monaifbs/src/train/monai_dynunet_training.py \
--train_files_list <path-to-list-of-training-files.txt> \
--validation_files_list <path-to-list-of-validation-files.txt>\
--out_folder <path-to-output-directory>
--config_file <path-to-customed-config-file.yml>
```
When running inference, make sure the config file for inference is also updated accordingly, otherwise the model might 
not be correctly reloaded (See the section Inference below).

#### Using the GPU
The code is optimised to be used with 1 GPU (multi-GPU computation is not supported at present).
To set the GPU to use, run the command `export CUDA_VISIBLE_DEVICES=<gpu_number>` before running the python commands 
described above.

#### Understanding the output
The script will generate two subfolders in the indicated output directory:
* folder with name formatted as `Year-month-day_hours-minutes-seconds_out_postfix`, which stores the results
of the training. `out_postix` is `monai_dynUnet_2D` by default, but can be changed as input argument when running the training. 
This folder contains:
    * `best_valid_checkpoint_key_metric=####.pt`: saved pytorch model best performing on the validation set 
    (based on Mean 3D Dice Score)
    * `checkpoint_epoch=####.pth`: latest saved pytorch model
    * directories `train` and `valid` storing the tensorboard outputs for the training and the validation respectively  
 
  
    
* folder named `persistent_cache`. To speed up the computation, the script uses MONAI 
[PersistentDataset][persistent_dataset], which pre-computes and stores to disk all the non-random pre-processing steps 
(pre-processing transforms outputs). This folder stores the results of these pre-computations. 


*Notes on the persistent cache*: 
1. 当需要多次运行时，持久性高速缓存数据集有利于预处理数据的可重用性（例如，用于超参数调整）。
To change the location of this persistent cache or to re-use a pre-existing cache, 
the option `--cache_dir <path-to-persistent-cache>` can be used in the command line for setting the training to run. 
2. 持久性缓存可能会占用相当大量的存储空间，具体取决于训练集的大小和选择用于训练的补丁大小
（例如：具有约400个3D卷和默认补丁大小（418、512），大约需要30G）。
3. 有PersistentCache的替代解决方案，可以不使用这么大的存储空间，但是目前还未在训练脚本中实施。 有关更多信息，请参见此[MONAI tutoria] [monai_datasets]。
要将其他 MONAI Datasets 集成到脚本中，请更改在`src/train/monai_dynunet_training.py`中`train_ds`和`val_ds`的定义。

## Inference
**Note** If using the provided pre-trained model, please make sure you have downloaded it and placed it as expected in 
`<path_to_MONAIfbs>/monaifbs/models`. More details in the [important Note][importantnote] above.

Inference can be run with the provided inference script with the following command:  
```
python <path_to_MONAIfbs>/monaifbs/src/inference/monai_dynunet_inference.py \
--in_files <path-to-img1.nii.gz> <path-to-img2.nii.gz> ... <path-to-imgN.nii.gz> \ 
--out_folder <path-to-output-directory> 
```

By default, this will use the provided [pre-trained model][dynUnetmodel] and the network configuration parameters
reported in this [config file][inference_config]. If you want to specify a different (MONAI dynUNet) trained model,
you can create your own config file indicating the model to load and its network configuration parameters
following the provided [template][inference_config]. Then, you can simply run inference as:  
```
python <path_to_MONAIfbs>/monaifbs/src/inference/monai_dynunet_inference.py \
--in_files <path-to-img1.nii.gz> <path-to-img2.nii.gz> ... <path-to-imgN.nii.gz> \ 
--out_folder <path-to-output-directory> \  
--config_file <path-to-custom-config.yml>
```

## Use within NiftyMIC (for inference)
The automated segmentation tool was developed in support to the Super-Resolution Reconstruction package [NiftyMIC][NiftyMIC]. 
By default, NiftyMIC uses monaifbs utilities to automatically generate fetal brain segmentation masks that can be used
for the reconstruction pipeline.
Provided the dependencies for NiftyMIC and monaifbs are installed, create the automatic fetal brain masks of HASTE-like 
images with the command:
```
niftymic_segment_fetal_brains \
--filenames \
nifti/name-of-stack-1.nii.gz \
nifti/name-of-stack-2.nii.gz \
nifti/name-of-stack-N.nii.gz \
--filenames-masks \
seg/name-of-stack-1.nii.gz \
seg/name-of-stack-2.nii.gz \
seg/name-of-stack-N.nii.gz
```

The interface between the niftymic package and monaifbs inference utilities is defined in `fetal_brain_seg.py`.
Its working scheme is essentially the same as the `monai_dynunet_inference.py`, it simply provides a wrapper to feed
the input data to the `run_inference()` function in `monai_dynunet_inference.py`.
It can also be used as a standalone script for inference as follows:  
```
python <path_to_MONAIfbs>/monaifbs/fetal_brain_seg.py \
--input_names <path-to-img1.nii.gz> <path-to-img2.nii.gz> ... <path-to-imgN.nii.gz> \ 
--segment_output_names <path-to-seg1.nii.gz> <path-to-seg2.nii.gz> ... <path-to-segN.nii.gz> 
```

## Troubleshooting
#### Issue with ParallelNative on MacOS

A warning message from ParallelNative is shown and the computation gets stuck. This issue appears to happen only on 
MacOS and is known to be linked to PyTorch DataLoader (as reported in https://github.com/pytorch/pytorch/issues/46409)

Warning message: `[W ParallelNative.cpp:206] Warning: Cannot set number of intraop threads after parallel work has started 
or after set_num_threads call when using native parallel backend (function set_num_threads` 

When observed: MacOS, Python 3.6 and Python 3.7, running on CPU.

Solution: add `OMP_NUM_THREADS=1` before the call of monaifbs scripts.  
Example 1:
```
OMP_NUM_THREADS=1 python <path_to_MONAIfbs>/monaifbs/src/inference/monai_dynunet_inference.py \
--in_files <path-to-img1.nii.gz> <path-to-img2.nii.gz> ... <path-to-imgN.nii.gz> \ 
--out_folder <path-to-output-directory> 
```
Example 2:
```
OMP_NUM_THREADS=1 niftymic_segment_fetal_brains \
--filenames \
nifti/name-of-stack-1.nii.gz \
nifti/name-of-stack-2.nii.gz \
nifti/name-of-stack-N.nii.gz \
--filenames-masks \
seg/name-of-stack-1.nii.gz \
seg/name-of-stack-2.nii.gz \
seg/name-of-stack-N.nii.gz
```

## Disclaimer

Not intended for clinical use.

## Licensing and Copyright
Copyright (c) 2020 Marta Bianca Maria Ranzini and contributors.
Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

`http://www.apache.org/licenses/LICENSE-2.0`

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

Other licenses may apply for dependencies.

## Acknowledgements 
This work is part of the [GIFT-Surg project][giftsurg] and is funded by the Innovative Engineering for Health award ([Wellcome Trust][wellcometrust] [WT101957] and [EPSRC][epsrc] [NS/A000027/1]),
the Wellcome/EPSRC [Centre for Medical Engineering][cme] [WT 203148/Z/16/Z, NS/A000049/1]] and supported by researchers at the NIHR Biomedical Research
Centre based at GSTT NHS Trust and King's College London.

[giftsurg]: http://www.gift-surg.ac.uk
[tomvercauteren]: tom.vercauteren@kcl.ac.uk
[kcl]: https://www.kcl.ac.uk
[sie]: https://www.kcl.ac.uk/bmeis/our-departments/surgical-interventional-engineering
[monai]: https://monai.io/
[installation]: https://github.com/gift-surg/NiftyMIC/wiki/niftymic-installation
[gitlfs]: https://github.com/git-lfs/git-lfs/wiki/Installation
[dynUnetmodel]: https://zenodo.org/record/4282679#.X7fyttvgqL5
[inference_section]: https://github.com/gift-surg/MONAIfbs#inference
[use_section]: https://github.com/gift-surg/MONAIfbs#use-within-niftymic-for-inference
[dynUnettutorial]: https://github.com/Project-MONAI/tutorials/blob/master/modules/dynunet_tutorial.ipynb
[nnunet]: https://arxiv.org/abs/1809.10486
[inference_config]: https://github.com/gift-surg/MONAIfbs/blob/main/monaifbs/config/monai_dynUnet_inference_config.yml
[training_config]: https://github.com/gift-surg/MONAIfbs/blob/main/monaifbs/config/monai_dynUnet_training_config.yml
[mranzini]: https://www.linkedin.com/in/marta-bianca-maria-ranzini
[bsd]: https://opensource.org/licenses/BSD-3-Clause
[persistent_dataset]: https://github.com/Project-MONAI/MONAI/blob/9f51893d162e5650f007dff8e0bcc09f0d9a6680/monai/data/dataset.py#L71
[monai_datasets]: https://github.com/Project-MONAI/tutorials/blob/master/acceleration/dataset_type_performance.ipynb
[wellcometrust]: http://www.wellcome.ac.uk
[epsrc]: http://www.epsrc.ac.uk
[cme]: https://medicalengineering.org.uk/
[NiftyMIC]: https://github.com/gift-surg/NiftyMIC
[importantnote]: https://github.com/gift-surg/MONAIfbs#important-note
