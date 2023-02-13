[plot](https://ibb.co/nD93F7L)

# *SPECTROID*: A Centroid-Based Citation-Aware Approach to Paper Classification

> **NOTE**: Similarly to the [original repo](https://github.com/allenai/specter), this repo does not support Windows.

## Introduction and Novelties
This repo presents two extensions to the work presented in [*SPECTER: Document-level Representation Learning using Citation-informed Transformers*](https://arxiv.org/abs/2004.07180). 

The main novelties here presented are:

1. To have reproduced SPECTER's training on a different dataset from the one used in the original work (in particular, we used a subset of Scidocs rather than s2orc).
2. To have reproduced SPECTER's training introducing the concept of "centroid" in the latent space obtained by the encoder through a modification of the way *positive* papers are selected.
3. To have applied SPECTER to classification on the arXiv dataset.
4. To have designed and used MLP Classification Heads on top of SPECTER to influence the geometry of the latent space produced by SPECTER with information related to the loss signal.

Overall, these **4** modifications can be grouped in two main extensions, namely *SPECTROID extension* and *Classification Heads extension*.

For an in-detail explanation of the motivations and experimental settings of our extensions to SPECTER's work, please refer to the [report](linktoreport) enclosed with this repo.

# SPECTROID Extension
## 0- Virtual Env and Downloads
Firstly, one should create a virtual environment supporting the necessary depencencies. These are mainly defined in SPECTER's repo, although some more have been here used.

To do so, please run the following:

```bash
$ conda create --name specter python=3.7 setuptools  
$ conda activate specter

# if you don't have GPUs, remove cudatoolkit argument
$ conda install pytorch cudatoolkit=10.1 -c pytorch   
$ pip install -r requirements.txt  
$ python setup.py install
$ pip install overrides==3.1.0
```

The code here presented requires some additional files in additional folder. To download these please run: 

```bash
$ bash/download.sh
```

In particular, in the `pretrained_models` folder one could find the two models we trained: SPECTER and SPECTROID. 
These models are fully functional and can be used to embed your datasets as per point 4 of the documentation, passing one of the two models' filename as values for the arg `--model` of XXX.py.

To have a full grasp of our work, please stick to the instructions here below.

## 1- Creation of the Scidocs dataset
To train an embedder, we created a new dataset starting from the metadata available on the [Scidocs repository](https://github.com/allenai/scidocs). Specifically, the dataset we created is obtained from:

* `data.json` containing the document ids and their relationship.  
* `metadata.json` containing mapping of document ids to textual fiels (e.g., `title`, `abstract`)
* `train.txt`, `val.txt`, `test.txt` containing document ids corresponding to train/val/test sets (one doc id per line).

The `data.json` file has the following structure (a nested dict):  
```python
{"docid1" : {  "docid11": {"count": 1}, 
               "docid12": {"count": 5},
               "docid13": {"count": 1}, ....
            }
"docid2":   {  "docid21": {"count": 1}, ....
            }
....}
```

Where `docids` are Scidocs paper ids and `count` is a measure of importance of the relationship between two documents. 
Here, we use the same notation used in Specter for `count`. In particular, `count=5` means direct citation while `count=1` refers to a citation of a citation. 

The dataset is already available in the folder `project_data` (obtained after having run `bash/download.sh`).

For full reproducibility, we here report that is also possible to create the same dataset from scratch starting from the Scidocs metadata files. 
Stemming from the files inside the folder `scidocs_metadata`, simply run:

```python
python scripts/create_scidocs.py --data-dir scidocs_metadata --output-dir project_data
```

## 2- Create training triplets
The `create_training_files.py` script processes an input dataset structure with a triplet sampler. 
Triplets can be formed either according to what described in the paper, or according to our improvement. 
Papers with `count=5` are considered positive candidates, papers with `count=1` are considered **hard** negatives. Lastly, other papers that are not cited are considered **easy** negatives. 
The number of hard negatives can be controlled by specifing a `--ratio_hard_negatives` argument for the script's execution. 

Triplets can be formed running the following:
  
```python
python specter/data_utils/create_training_files.py --data-dir project_data --metadata project_data/metadata.json --outdir project_data/preprocessed_improvement/ --max-training-triplets 150000 --add-probabilities True 
```

Please note that `add-probabilities` is the parameter to set to choose between our improvement and Specter-like triplets. 
**If training both versions, it is advisable changing the `outdir` according to this parameter to avoid over-writing a previously trained model file.**
Due to limited computational resources, we also introduced an optional parameter to set the maximum number of triplets (in our work, these are 150K rather than 680+K).

Once the triplets are formed, the embedder can be trained running the following:

```python
./scripts/run-exp-simple.sh -c experiment_configs/simple.jsonnet -s model-output-improvement/ --num-epochs 2 --batch-size 4 --train-path project_data/preprocessed-improvement/data-train.p --dev-path project_data/preprocessed-improvement/data-val.p --num-train-instances 150000 --cuda-device 0

```
In this example the model's checkpoint and logs will be stored in the `model-output-improvement/ ` folder.  
Note that you need to set the correct `--num-train-instances` for your dataset. This number is either the maximum number of training triplets previously specified (if specified), or could be found in the `data-metrics.json` file output from the preprocessing step. 

## 3- Creation of the embedding datasets
Specter requires two main files as input to embed a document: a text file with the `ids` of the documents to embed and a json metadata file consisting of the title and abstract of each `ids`. 

In our work, we embedded two different datasets:
1. All the Scidocs papers for MeSH and MAG paper classification tasks.
2. A subset of 50K papers from the arXiv dataset.

The two datasets are already available in the folder `testing_datasets`. 
This folder contains two subfolders, one for each dataset. Each subfolder contains a text file with the ids of the document, the corresponding json metadata file and the train/val/test split of the ids (in .csv file).

For full reproducibility, one can create the `testing_datasets/arxiv_data` folder from scratch, leveraging the Kaggle interface and running the following: 
```python
python scripts/create_arxiv.py --output-dir 'kaggle_dat/'
```
Please follow this [guide](https://technowhisp.com/kaggle-api-python-documentation/) to correctly set-up the Kaggle Python API first.

## 4- Embedding of the datasets
To use a previously trained model to embed your data, run the following:

```python
python scripts/embed.py
   --ids testing_datasets/arxiv_data/arxiv.ids
   --metadata testing_datasets/arxiv_data/metadata.json
   --model model-output-improvement/model.tar.gz
   --output-file testing_datasets/arxiv_data/output_improvement.jsonl
   --vocab-dir data/vocab/
   --batch-size 64
   --cuda-device 0 
```
The model will run inference on the provided input and writes the output to `--output-file` directory (in the above example `output_improvement.jsonl` ).  
This is a jsonlines file where each line is a key, value pair consisting the id of the embedded document and its specter representation.

To use the Mag\Mesh Scidocs papers, it is enough to change `ids` and `metadata` (and possibly `output-file`) accordingly.

## 5- Classification
The performance of our embedder can be evaluated running the following:

```python
python scidocs/run.py 
--cls testing_datasets/arxiv_data/output_improvement.jsonl 
--data-path testing_datasets/arxiv_data/classification/ 
--val_or_test test 
--n-jobs 12
```