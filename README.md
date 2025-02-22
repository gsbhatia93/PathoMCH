
# Spatial Transcriptomics Inferred from Pathology Whole-Slide Images Links Tumor Heterogeneity to Survival in Breast and Lung Cancer

* This project contains the source code used in: "Spatial Transcriptomics Inferred from Pathology Whole-Slide Images Links Tumor Heterogeneity to Survival in Breast and Lung Cancer" by Levy-Jurgenson et al.

* Modified by Peijin Han. Note that the code is very clean so I didn't change much except for the paths. I just added some explanations. I would also like to thank the author. She is super helpful!

* Please clone the most updated code from Github. The author changed a path in `conf_postprocess.py` after I communicated with her.

### Prerequisites 

* Python 3.6 

* Tensorflow 1.14.0 (pip3 install tensorflow==1.14.0 or on gpu: pip3 install tensorflow-gpu==1.14.0). Make sure to uninstall other Tensorflow versions first (use uninstall instead of install).

* OpenSlide Python:

    Use installations here: https://openslide.org/download/ or use below example for Ubuntu:

    `apt-get install python3-openslide`

    and then:

    `pip3 install openslide-python`

On Google Cloud platform, we specifically used the following settings (adapted to tensorflow 1.14.0):
* For training on GPUs:
    * Operating system: "Deep Learning on Linux" 
    * Version: "Deep Learning Image: TensorFlow 1.15.0 m42" (uninstall 1.15 and install 1.14 instead, see prerequisites)
    * Number of K80 GPUs per vm: 8

* For preprocessing (tiling WSIs and creating tfrecords) we used the same settings above with no GPUs, but with 60 CPUs.
Less CPUs can be used by changing NUM_CPU in conf.py but this will slow down preprocessing significantly.  

PH: In Umich Greatlakes, the maximum task per node is 36. Therefore I used 36 CPUs to run preprocess. Here is my .sh file:

#!/bin/bash
#SBATCH --mail-user=pehan@umich.edu
#SBATCH --mail-type=BEGIN,END
#SBATCH --job-name=preprocess
#SBATCH --cpus-per-task=36
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --mem-per-cpu=5g
#SBATCH --time=48:00:00
#SBATCH --account=ukarvind1
#SBATCH --partition=standard


## Obtaining our pre-trained models / output predictions
We will gladly share pre-trained weights and/or predictions for reasonable requests. Please e-mail us from your institution's e-mail address stating the intended use (levyalona at gmail dot com).   
You can easily load them by using your Conf object's `LOAD_WEIGHTS_PATH` (replace `None` with the directory that contains only the weights you wish to load, e.g.: `../out/<model_name>/auc/`

## The format of clinical data

Please see `/Volumes/umms-ukarvind/PathoMCH_data/data/clinical/CD3D.csv` for the example of clinical data. Here are three important columns:

CD3D_a: the actual CD3D expression, downloaded from cbio portal. 
CD3D_50_pctl: 'lo' or 'hi'. 'lo' is the expression value that lower than the median among samples in this excel spreadsheet.
CD3D_50_pctl_value: the expression percentile value
CD3D: 'lo' or 'high' but only with sample that have extreme expression (less than 20 pct or greater than 80 pct)

## Training and evaluating models
This section covers the code you will likely need to adapt to your needs. We recommend reading through this entire section before starting so that your configuration (which you can control using Conf objects) will be well suited to your needs throughout the entire process. 

Note that the files under src are in a flattened hierarchy to keep things simple when transitioning between a 
local environment and linux servers. 

There are 5 main files required for training and evaluating a model:
1. `conf.py`:
    a useful configuration class where you can create your own Conf object to set parameters for preprocessing and training. We've left our preset Confs for BRCA and LUAD TCGA cohorts as examples.
    Some parameters you will likely have to adapt to your needs are:
    * `CLINICAL_FILEPATH` - path to csv that includes a sample ID column and a label column with the name matching `CLINICAL_LABEL_COLS`
    * `CLINICAL_LABEL_COLS` - name of column used for labeling. All samples that have labels under this column will be included in model development by default (you can create other label columns to store ground truth for all samples). I would use column name 'CD3D'.
    * `CLINICAL_LABELS` - list of labels to be used out of those under `CLINICAL_LABEL_COLS` e.g. `['lo','hi']`
    * `SVS_SLIDES_PATH` - path to slides. Default is data->slides->diagnostic.
    * `NUM_CPU` - number of CPUs available for tiling the slides and generating tfrecords
    * `NAME` - the prefix name of your models. Use a unique name per trait if you don't want results to get run over between traits.
    * `TCGA_COHORT_NAME` - controls your out subfolder name. Useful if you're going to be using more than one dataset so results don't get run over.
    * `LOAD_WEIGHTS_PATH` - should be `None` unless you are: (a) training from a pre-trained model or (b) running inference post-training. In these cases `None` should be replaced with the path to the **directory** in which your weights reside (e.g. `../out/<model_name>/auc/`).  
    
2. `preprocess.py`: tiles slides (if `tile_slides = True`) and generates sharded tfrecords of two types: one type is for training and validation. These end with '.tfrec' and don't have any specific sample ID in their name. The second type ends with '.tfrecords' - there will be one per sample in the data. These are used only during inference (when you set `training=False` in `model.py`).

After generating tfrecords, you will have to move the resulting tfrec and tfrecords to your desired location (we used google cloud storage (GCS)), which means you will need to adapt the following paths in `conf.py`:
`GCS_PATTERN` and `GCS_PATTERN_PER_SAMPLE`. IMPORTANT: Make sure to save the `.pkl` files named `..patient_ids..pkl` and `..img_paths..pkl` as they are used during evaluation.

Note: 

* train/val/test split is based on the image data. This is because the author wants all the traits see the same split. However, when training and validating the model, only samples with extreme expressions are used. This was all taken care in `preprocess.py`. `.pkl` files that contains `*image_path_round_*` are correct images for train/val/test containing extreme samples. 

* Tiling slides is very time consuming and takes much spaces. Please make sure if you want to re-tile the slides before setting `tile_slides = True`. The tiled GBM samples can be found under `/nfs/turbo/umms-ukarvind/peijinhan/PathoMCH/data/images/zoom_20-512`

3. `model.py` 
    * If `training = True` trains Inception-v3 using the training and validation tfrecords under `GCS_PATTERN`. It will use all available GPUs on the machine (or only CPU if no GPUs are detected). 
    
    * If `training = False` it will load the latest checkpoint found under `LOAD_WEIGHTS_PATH` (in conf) and will begin producing predictions for each sample using the per-sample tfrecords files under `GCS_PATTERN_PER_SAMPLE` (note that if `LOAD_WEIGHTS_PATH = None` when `train = False` it will throw an exception).
    
4. `conf_postprocess.py` - settings used for evaluation/cartography/heterogeneity. The current format is derived from settings in your Conf object, so you can tell where the files are expected to be from your conf settings:
    * For pickles generated by preprocessor, the default is: `../res/postprocess/<c.TCGA_COHORT_NAME>/data_splits/<c.NAME>/'` (c is a Conf object from `conf.py`). Under this folder you need three subfolders - 'train','val','test'. Simply place the pickles under these folders according to their name.
    * For predictions, the default is: `../res/postprocess/<c.TCGA_COHORT_NAME>/preds/<c.NAME>/'` (c is a Conf object from `conf.py`). Under this folder you need to place the exact folder produced during inference (i.e. `../out/<c.TCGA_COHORT_NAME>/predict/`). Simply copy these folders from your server to this postprocess path.
If you want to change these formats, simply modify: `Conf_Postprocess().DATA_SPLIT_DIR`) and `Conf_Postprocess().MODEL_PREDS_DIR_WITH_RESAMPLE_ROUND`.

After set up the correct paths in `conf_postprocess.py`, `run predict.py`.

5. `evaluate.py` - evaluates the model predictions and produces log files under '../out/'. Expects to see the aforementioned data_split and predictions pickles under `../res/postprocess/..` (see previous point).

Note: it is easier to run on regular evaluation first. Set `consensus=False` for the first time.

## Training on mock data
We have included under the `res` folder pre-generated train and validation mock tfrec files. You may use these to check 
that the training pipeline works properly by running `model.py` with the default settings (currently set to `training = True`). 
Note that for this data, the model is not expected to converge, nor can this data be used to train a clinically meaningful model. 
For this purpose, original whole-slide images need to be placed under data->slides folder as explained above (see `conf.py`). 
Slides may be obtained from the GDC website, as discussed in the Methods section. The manifest files that list the slides 
used in this research are provided under 'data'. 


## Generating molecular cartographies and deriving heterogeneity
To generate molecular cartogrpahies and heterogeneity simply use `cartography.py` and `heterogeneity.py` with the same configuration settings (Conf and Conf_Postprocess) as used in evaluation.
The output will be generated under `../out/<c.TCGA_COHORT_NAME>/` under `cartography` and `heterogeneity`. HTI appears in the name of each image under `heterogeneity` (e.g. for heterogeneity map ending with: '..._0.64.png', HTI is 0.64).  

