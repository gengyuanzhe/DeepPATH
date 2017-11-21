
Preliminary comment: On the NYU HPC cluster, 3 modules are needed. The commands mentioned below must be run through qsub scripts and not on the head node! Module mostly needed are:
* module load cuda/8.0
* module load python/3.5.3
* module load bazel/0.4.4

Most of the codes below show example of command line included in phoenix qsub scripts.

For the path, it is advised to always put the full path name and not the relative paths.

For all the steps below, always submit the jobs via a qsub script (if on Pheonix) and always check the output and error log files are fine. 

This procesude is based on the inception v3 architecture from google. See [Inception v3](https://github.com/tensorflow/models/blob/f87a58cd96d45de73c9a8330a06b2ab56749a7fa/research/inception/README.md) for information about it. 

# 0 - Prepare the images.

Code in 00_preprocessing. 

All original images must start with the patient ID.

SVS images can be extremely large (+100,000 pixel wide). Optimal input size for inception is 299x299 pixels but the network has been designed to deal with variable image sizes. 

SVS images are first tiled, then sorted according to chosen labels. There will be one folder per label and all the jpg images in the corresponding folder. Also, tiles will be sorted into a train, test and validation set. All tiles generated from the same patient should be assigned to the same set. 

Finally, the jpg images are converted into TFRecords. For the train and validation set, there will be randomly assigned to 1024 and 128 shards respectively. For the test set, there will be 1 TFRecord per slide.



## 0.1 Tile the svs slide images

This step also required ```module load openjpeg/2.1.1```.

Example of qsub script to submit this script on Phoenix cluster (python 2.7 used):

```shell
#!/bin/tcsh
#$ -pe openmpi 32
#$ -A TensorFlow
#$ -N rqsub_tile
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q
#$ -l excl=true

python /path_to/0b_tileLoop_deepzoom2.py  -s 299 -e 0 -j 32 -B 25 -o <full_path_to_output_folder> "full_path_to_input_slides/*/*svs"  
```

On Prince, you may want to try this header instead (and adjust option ```-j``` to ```28```):

```shell
#!/bin/bash
#SBATCH --job-name=rq_tile
#SBATCH --nodes=1
#SBATCH --cpus-per-task=28
#SBATCH --mem=125GB
#SBATCH --time=47:00:00
#SBATCH --output=rq_00tile_%A_%a.out
#SBATCH --error=rq_00tile_%A_%a.err

module load openslide-python/intel/1.1.1


```


Example of options:
*  `-s` is tile_size: 299 (299x299 pixel tiles)
*  `-e` is overlap, 0 (no overlap between adjacent tiles)
*  `-j` is number of threads: 32 (for a full GPU node on gpu0.q)
*  `-B` is Max Percentage of Background: 25% (tiles removed if background percentage above this value)
*  `-o` is the path were the output images must be saved
*  The final mandatory parameter is the path to all svs images.

Output:
* Each slide will have its own folder and inside, one sub-folder per magnification. Inside each magnification folder, tiles are named according to their position within the slide: ```<x>_<y>.jpeg```.


## 0.2 Sort the tiles into train/valid/test sets according to the classes defined


Then sort according to cancer type (script example for Phoenix):

```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rqsub_sort
#$ -cwd
#$ -S /bin/tcsh
#$ -q all.q

python /full_path_to/0d_SortTiles.py --SourceFolder=<tiled images path> --JsonFile=<JsonFilePath> --Magnification=<Magnification To copy>  --MagDiffAllowed=<Difference Allowed on Magnification> --SortingOption=<Sorting option> --PercentTest=15 --PercentValid=15 --PatientID=12 --nSplit 0
```

For Prince, the header of the script may be:
```shell
#!/bin/bash
#SBATCH --gres=gpu:1
#SBATCH --job-name=sort
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --output=rq_train_%A_%a.out
#SBATCH --error=rq_train_%A_%a.err
#SBATCH --mem=2G

module load numpy/intel/1.13.1
```

*  `--SourceFolder`: output of ``` 00_preprocessing/0b_tileLoop_deepzoom.py```, that is the main folder where the svs images were tiled
*  `--JsonFile`: file uploaded with the svs images and containing all the information regarding each slide (i.e, metadata.cart.2017-03-02T00_36_30.276824.json)
*  `--Magnification`: magnification at which the tiles should be considerted (example: 20)
*  `--MagDiffAllowed`: If the requested magnification does not exist for a given slide, take the nearest existing magnification but only if it is at +/- the amount allowed here(example: 5)
*  `--SortingOption` In the current directory, create one sub-folder per class, and fill each sub-folder with train_, test_ and valid_ test files. Images will be sorted into classes depending on the sorting option:
   - `1` sort according to cancer stage (i, ii, iii or iv) for each cancer separately (classification can be done separately for each cancer)
   - `2` sort according to cancer stage (i, ii, iii or iv) for each cancer  (classification can be done on everything at once)
   - `3` Sort according to type of cancer (LUSC, LUAD, or Nomal Tissue)
   - `4` Sort according to type of cancer (LUSC, LUAD)
   - `5` Sort according to type of cancer / Normal Tissue (2 variables per type)
   - `6` Sort according to cancer / Normal Tissue (2 variables)
   - `7` Random labels (3 labels. Can be used as a false positive control)
   - `8` Sort according to mutational load (High/Low). Must specify --TMB option.
   - `9` Sort according to BRAF mutations for metastatic only. Must specify --TMB option (BRAF mutant for each file).
   - `10` Do not sort. Just create symbolic links to all images in a single label folder and assign images to train/test/valid sets.
   - `11` Sample location (Normal, metastatic, etc...)
   - `12` Osman's melanoma: Response to Treatment (Best Response) (POD vs other)
   - `13` Osman's melanoma: Toxicity observed 
* `--TMB`: addional option for optoin 8: path to json file with mutational loads
*  `--PercentTest`: percentage of images for validation (example: 15); 
*  `--PercentValid` Percentage of images for testing (example: 15). All the other tiles will be used for training by default.
* `PatientID`: Number of digits used to code the patient ID (must be the first digits of the original image names)
* `nSplit`: interger n: Split into train/test in n different ways.  If split is > 0, then the data will be split in train/test only in "# split" non-overlapping ways (each way will have 100/(#split) % of test images). `PercentTest` and `PercentValid` will be ignored. If nSplit=0, then there will be one output split done according to `PercentValid` and `PercentTest`

The output will be generated in the current directory where the program is launched (so start it from a new empty folder). Images will not be copied but a symbolic link will be created toward the <tiled images path>. The links will be renamed ```<type>_<slide_root_name>_<x>_<y>.jpeg``` with <type> being 'train_', 'test_' or 'valid_' followed by the svs name and the tile ID. 


## 0.3a Convert the JPEG tiles into TFRecord format for 2 or 3 classes jobs

Notes:
* This code was adapted from [awslabs' deeplearning-benchmark code](https://github.com/awslabs/deeplearning-benchmark/blob/master/tensorflow/inception/inception/data/build_image_data.py)


Check code in subfolder 00_preprocessing/TFRecord_2or3_Classes/ if it aimed at classifying 2 or 3 different classes.

For the whole training and validation sets, the following code can be to convert JPEG to TFRecord:
```shell
#!/bin/tcsh
#$ -pe openmpi 4
#$ -A TensorFlow
#$ -N rqsub_TFR_trval
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q 

module load cuda/8.0
module load python/3.5.3

python build_image_data.py --directory='jpeg_label_directory' --output_directory='outputfolder' --train_shards=1024  --validation_shards=128 --num_threads=4
```

For Prince, the header of the script may be:

```shell
#!/bin/bash
#SBATCH --gres=gpu:1
#SBATCH --job-name=TFR_Vset
#SBATCH --cpus-per-task=1
#SBATCH --output=rq_TFR_%A_%a.out
#SBATCH --error=rq_TFR_%A_%a.err
#SBATCH --mem=20G

module load numpy/intel/1.13.1
module load cuda/8.0.44    
module load tensorflow/python2.7/1.0.1
module load bazel/gnu/0.4.3 

```


The jpeg must not be directly inside 'jpeg_label_directory' but in subfolders with names corresponding to the labels (for example as `jpeg_label_directory/TCGA-LUAD/...jpeg` and `jpeg_label_directory/TCGA-LUSC/...jpeg`). The name of those tiles are : `<type>_name_x_y.jpeg` with type being "test", "train" or "valid", name the TCGA name of the slide, x and y the tile coordinates.


The same was done for the test set with this slightly modified script:
```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rqsub_TFR_test
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q 

module load cuda/8.0
module load python/3.5.3

python  build_TF_test.py --directory='jpeg_tile_directory'  --output_directory='output_dir' --num_threads=1 --one_FT_per_Tile=False --ImageSet_basename='test'

```

The difference is that for the train and validation sets, the tiles are randomly assigned to the TFRecord files. For the test set, it will created 1 TFRecord file per Slide (solution prefered) - though if `one_FT_per_Tile` is `True`, there will be 1 TFRecord file per Tile created.

An optional parameter ```--ImageSet_basename='test'``` can be used to run it on 'test' (default), 'valid' or 'train' dataset

expected processing time for this step: a few seconds to a few minutes. Once done, check inside the resulting directory that the images have been properly linked.


## 0.3b Convert the JPEG tiles into TFRecord format for a multi-ouput prediction (example mutations)


Check subfolder 00_preprocessing/TFRecord_2or3_Classes/ if it aimed at multi-output classsification with 10 possibly concurent sclasses:

For the training and validation sets:
```shell
#!/bin/tcsh
#$ -pe openmpi 4
#$ -A TensorFlow
#$ -N rqsub_TFR_trval
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q 

python build_image_data_multiClass.py --directory='jpeg_main_directory' --output_directory='outputfolder' --train_shards=1024 --validation_shards=128 --num_threads=4  --labels_names=label_names.txt --labels=labels_files.txt  --PatientID=12
```
* ``` label_names.txt``` is a text file with the 10 possible labels, 1 per line
* ```labels_files.txt``` is a text file listing the mutations present ifor each patient. 1 patient per line, first column is patient ID (TCGA-38-4632 for example), second is mutation (TP53 for example)
* ```--PatientID``` The file names are expected to start with the patient ID. This value represent the number of digits used for the PatientID


For the test set:
```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rqsub_sort
#$ -cwd
#$ -S /bin/tcsh
#$ -q all.q

python  build_TF_test_multiClass.py --directory='jpeg_tile_directory'  --output_directory='output_dir' --num_threads=1 --one_FT_per_Tile=False --ImageSet_basename='test' --labels_names=label_names.txt --labels=labels_files.txt  --PatientID=12
```


expected processing time for this step: a few seconds to a few minutes. Check the output log files and the resulting directory (check that the sizes of the created TFRecord files make sense)


# 1 - Training
## 1.1 - Training from scratch
### 1.1.b Training (new version)

Code in the subfolders of 01_training/xClasses - can be used for any type of training.

Build the model from the proper directory, that means from ```cd 01_training/xClasses```:

```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rqs_build
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q
#$ -l excl=true

module load cuda/8.0
module load python/3.5.3
module load bazel/0.4.4


bazel build inception/imagenet_train
```

Note, on the Prince cluster, the header could be something like:
```shell
#!/bin/bash
#SBATCH --gres=gpu:1
#SBATCH --job-name=train
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --output=rq_train_%A_%a.out
#SBATCH --error=rq_train_%A_%a.err
#SBATCH --mem=20G
#SBATCH --time=168:00:00

module load numpy/intel/1.13.1
module load cuda/8.0.44    
module load tensorflow/python2.7/1.0.1
module load bazel/gnu/0.4.3 
```


Run it for all the training images:
```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rqs_train
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q
#$ -l excl=true

module load cuda/8.0
module load python/3.5.3
module load bazel/0.4.4


bazel-bin/inception/imagenet_train --num_gpus=1 --batch_size=30 --train_dir='output_directory' --data_dir='TFRecord_images_directory' --ClassNumber=3 --mode='0_softmax'
```
 The ```mode``` option must be set to either ```0_softmax``` (original inception - only one ouput label possible) or ```1_sigmoid``` (several output labels possible)


## 1.2 - Transfer learning

See [inception v3 github page](https://github.com/tensorflow/models/tree/f87a58cd96d45de73c9a8330a06b2ab56749a7fa/research/inception#adjusting-memory-demands) for more details.


Bassically:

Build the model (the following two commands must be run from the proper directory, for example ```cd 01_training/xClasses```):

```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rqs_build
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q
#$ -l excl=true

module load cuda/8.0
module load python/3.5.3
module load bazel/0.4.4

bazel build inception/imagenet_train
```

download the checkpoints of the network trained by google on the 
```shell
curl -O http://download.tensorflow.org/models/image/imagenet/inception-v3-2016-03-01.tar.gz
```

This will create a directory called inception-v3 which contains the following files:
```shell
> ls inception-v3
README.txt
checkpoint
model.ckpt-157585
```
```

Run it for all the training images:
```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rqs_train
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q
#$ -l excl=true

module load cuda/8.0
module load python/3.5.3
module load bazel/0.4.4

bazel-bin/inception/imagenet_train --num_gpus=1 --batch_size=30 --train_dir='output_directory' --data_dir='TFRecord_images_directory' --pretrained_model_checkpoint_path="path_to/model.ckpt-157585" --fine_tune=True --initial_learning_rate=0.001  --ClassNumber=3 --mode='0_softmax'
```

Adjust the input parameters as required. For mode, this can also be '1_sigmoid'.




## 1.3 Validation
### 1.3.b New Version
Should be run on the validation test set at the same time as the training but on a different node (memory issues occur otherwise).


Code is in 02_testing/xClasses/. 


run the job:

```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rqs_Valid
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q
#$ -l excl=true

module load cuda/8.0
module load python/3.5.3

python nc_imagenet_eval --checkpoint_dir='full_path_to/0_scratch/' --eval_dir='output_directory' --data_dir="full_path_to/TFRecord_valid/"  --batch_size 30 --ImageSet_basename='valid' --ClassNumber 2 --mode='0_softmax' --run_once
```


Note, on the Prince cluster, the header could be something like:
```shell
#!/bin/bash
#SBATCH --gres=gpu:1
#SBATCH --job-name=valid
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --time=150:00:00
#SBATCH --output=rq_valid_%A_%a.out
#SBATCH --error=rq_valid_%A_%a.err
#SBATCH --mem=10G

module load numpy/intel/1.13.1
module load cuda/8.0.44    
module load tensorflow/python2.7/1.0.1
```

Replace ClassNumber with the number of classes used and mode by "1_sigmoid" if multi-output classification done (ex for mutations). 

You need to either:
* run it manually once in a while and keep track of the evolution of validation score.
* or run the script without the ```--run_once``` option (the program will run in an infinite loop and will need to be killed manually). To set how often the validation script needs to be run, you need to modify the code: in file ```02_testing/2Classes/inception/nc_inception_eval.py```, line 46, the default value of ```eval_interval_secs``` set to 5 minutes by default (for very long jobs, every 1 or 5 hours may be enough. This has to be changed before compilation with bazel).

Note: The current validation code only saves the validation accuracy, not the loss (saved in an output file named `precision_at_1.txt`). The code still needs to be changed for that. 



## 1.4 Comments on the code

This is inception v3 developped by google.  Full documentation on (re)-training can be found here: https://github.com/tensorflow/models/tree/master/inception


Main modifications when adjusting the code:
* in slim/inception_model.py: default ```num_classes``` in ```def inception_v3```
* in inception_train.py: default ```max_steps``` in  ```tf.app.flags.DEFINE_integer``` definition
* in imagenet_data.py: 
    * default number of classes in  ```def num_classes(self):```
    * size of the train and validation subsets in ```def num_examples_per_epoch(self)```
* Other changes for multi-output classification: 
    * - in slim/inception_model.py:
        * line 329 changed from ```end_points['predictions'] = tf.nn.softmax(logits, name='predictions')``` to ```end_points['predictions'] = tf.nn.sigmoid(logits, name='predictions')```
    * in slim/losses.py:
        * in ```def cross_entropy_loss``` (line 142 and next ones): ```tf.contrib.nn.deprecated_flipped_softmax_cross_entropy_with_logits``` replaced by  ```tf.contrib.nn.deprecated_flipped_sigmoid_cross_entropy_with_logits```
    * in ```image_processing.py``` (line 378):
        * ```'image/class/label': tf.FixedLenFeature([1], dtype=tf.int64, default_value=-1)``` changed to ```'image/class/label': tf.FixedLenFeature([FLAGS.nbr_of_classes+1], dtype=tf.int64, default_value=[0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0])```
        * line 513: replaced ```return images, tf.reshape(label_index_batch, [batch_size])``` with ```return images, tf.reshape(label_index_batch, [batch_size, FLAGS.nbr_of_classes+1])```
    * in ```inception/inception_model.py``` ```sparse_labels = tf.reshape(labels, [batch_size, 1])  [....]  dense_labels = tf.sparse_to_dense(concated,
 [batch_size, num_classes], 1.0, 0.0)``` replaced by ```dense_labels = tf.reshape(labels, [batch_size, FLAGS.nbr_of_classes+1])```







# 2 - Run the classification on the test images

Code in 02_testing/xClass:

Code is the same as the one used for the validation, but with different options: 


```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rq_Test
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q
#$ -l excl=true

module load cuda/8.0
module load python/3.5.3

python nc_imagenet_eval --checkpoint_dir='full_path_to/0_scratch/' --eval_dir='output_directory' --data_dir="full_path_to/TFRecord_perSlide_test/"  --batch_size 30 --ImageSet_basename='test_' --run_once --ClassNumber 2 --mode='0_softmax'
```

An optional parameter ```--ImageSet_basename='test'``` can be used to run it on 'test' (default), 'valid' or 'train' dataset

data_dir contains the images in TFRecord format, with 1 TFRecord file per slide.

In the eval_dir, it will generate the following files:
*  out_filename_Stats.txt: a text file with output information: <tilename> <True/False classification> [<output probilities>] <corrected output probability for the true label> labels: <true label number>
*  node2048/: a subfolder where each file correspond to a tile such as the filenames are ```test_<svs name>_<tile ID x>_<tile ID y>.net2048``` and the first line of the file contains: ``` <True / False> \tab [<Background prob> <Prob class 1> <Prob class 2>]  <TP prob>```, and the next 2048 lines correspond to the output of the last-but-one layer


expected processing time for this step: on a gpu, about 1000 tiles per minute.


# 3 - Analyze the outcome

On the Phoenix  cluster, the header for the following commands could be something like:
```shell
#!/bin/tcsh
#$ -pe openmpi 1
#$ -A TensorFlow
#$ -N rq_Analyze
#$ -cwd
#$ -S /bin/tcsh
#$ -q gpu0.q

module load cuda/8.0
module unload python/3.5.3
# Note: the scikit-learn seems to work properly only wiyh Native python, so unload 3.5.3 - heatmaps work with python/3.5.3
```

On the Prince cluster, the header could be something like:
```shell
#!/bin/bash
#SBATCH --job-name=ROC
#SBATCH --nodes=1
#SBATCH --cpus-per-task=1
#SBATCH --output=rq_ROC_%A_%a.out
#SBATCH --error=rq_ROC_%A_%a.err
#SBATCH --mem=2G

module load numpy/intel/1.13.1
module load scikit-learn/intel/0.18.1

```


## Code in 03_postprocessing/2Classes for 2 classes:
Generate heat-maps per slides (all test slides in a given folder; code not optimized and slow):
```shell
python 0f_HeatMap.py  --image_file 'directory_to_jpeg_classes' --tiles_overlap 0 --output_dir 'result_folder' --tiles_stats 'out_filename_Stats.txt' --resample_factor 4 --tiles_filter 'TCGA-05-5425'
```
*  ```image_file``` is the outcome folder of ```00_preprocessing/0d_SortTiles_stage.py``` (it has one sub-folder per class and jpeg tile images in each of them)
*  ```--tiles_stats out_filename_Stats.txt``` is one of the output files generated by ```02_testing/nc_imagenet_eval.py```. 
* the size of the heatmaps will be reduced by ```resample_factor```. For large slides, a high number is advised.
* ```--tiles_filter```: to be used if you want to process only some of the images


## Code in 03_postprocessing/3Classes for 3 classes:
ROC curves (to be run with native python: "module unload python/3.5.3"):
```shell
python 0h_ROC_sklearn.py  --file_stats out_filename_Stats.txt  --output_dir 'output folder'
```

Generate heat-maps per slides (all test slides in a given folder; code not optimized and slow):
```shell
python 0f_HeatMap_3classes.py  --image_file 'directory_to_jpeg_classes' --tiles_overlap 0 --output_dir 'result_folder' --tiles_stats 'out_filename_Stats.txt' --resample_factor 4 --slide_filter 'TCGA-05-5425' --filter_tile '' --map 'CancerType' --tiles_size 512
```
* ```slide_filter```: process only images with this basename.
* ```filter_tile```: if map is a mutation, apply cmap of mutations only if tiles are LUAD (```out_filename_Stats.txt``` of Noemal/LUAD/LUSC classification)
* ```map```: ```CancerType``` for Normal/LUAD/LUSC classification, or mutation name

Generate probability distribution with means for each class for each slide:
```shell
python 0f_ProbHistogram.py --output_dir='result folder' --tiles_stats='out_filename_Statsout_filename_Stats.txt' --ctype='Lung3Classes'
```

## Code in 03_postprocessing/multiClasses for 10-multi-output classification:
```shell
python  0h_ROC_MultiOutput.py  --file_stats 'MultiOuput/out_filename_Stats.txt'  --output_dir 'output folder' --labels_names label_names.txt --ref_stats 'LUAD/out_filename_Stats.txt'
```
* ```--file_stats``` is the output generated by the multi-output classification
* ```--ref_stats``` (optional) is the output generated by the 2 or 3 classes classification and is used to filter and selected only LUAD tiles.

Generate probability distribution with means for each class for each slide:
```shell
python 0f_ProbHistogram.py --output_dir='result folder' --tiles_stats='out_filename_Stats.txt of mutations' --ctype='Mutations' --filter_file='out_filename_Stats.txt of 3-class classification'
```

* `ctype` can be `Lung3Classes` or `Mutations`

