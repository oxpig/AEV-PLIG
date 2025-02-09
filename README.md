# AEV-PLIG

AEV-PLIG is a GNN-based scoring function that predicts the binding affinity of a bound protein-ligand complex given its 3D structure. The paper is at Nature's Communications Chemistry, at [https://doi.org/10.1038/s42004-025-01428-y](Narrowing the gap between machine learning scoring functions and free energy perturbation using augmented data).

AEV-PLIG was first published in [How to make machine learning scoring functions competitive with FEP](https://chemrxiv.org/engage/chemrxiv/article-details/6675a38d5101a2ffa8274f62), and received the [people's poster prize at the 7th AI in Chemistry Symposium](https://www.stats.ox.ac.uk/news/isak-valsson-wins-poster-prize). In the paper we benchmark AEV-PLIG on a wide range of benchmarks, including CASF-2016, our new out-of-distribution benchmark OOD Test, and a test set used for free energy perturbation (FEP) calculations, and highlight competitive performance accross the board. Moreover, we demonstrate how leveraging augmented data (generated using template-based modelling or molecular docking) can significantly improve binding affinity prediction correlation and ranking on the FEP benchmark (PCC and Kendall’s increases from 0.41 and 0.26, to 0.59 and 0.42), closing the performance gap with FEP calculations while being 400,000 times faster.

In this repo we demonstrate how to use AEV-PLIG for predictions and how to train your own AEV-PLIG model

- [Installation guide](#installation-guide)
- [Demo](#demo)

## Installation guide
AEV-PLIG has been tested on the following systems:
+ macOS: Monterey (12.5.1)

### Create conda environment
Installation times may vary, but took around 30 seconds on Mac M1.
For *macOS*:
```
conda env create --file aev-plig-mac.yml
```
For *Linux*:
```
conda env create --file aev-plig-linux.yml
```
Install packages manually:
```
conda create --name aev-plig python=3.8
conda activate aev-plig
pip install torch torchvision torchaudio
pip install torch-scatter
pip install torch_geometric
pip install rdkit
pip install torchani
pip install qcelemental
pip install pandas
```

## Demo
This section demonstrates how to train your own AEV-PLIG model, and how to use AEV-PLIG to make predictions.

The computational requirements for each script are included, and unless otherwise specified, the hardware used is a Mac M1 CPU.

### Training

#### Download training data
Download the training datasets PDBbind, BindingNet and BindingDB-DCS
```
wget http://pdbbind.org.cn/download/PDBbind_v2020_other_PL.tar.gz
wget http://pdbbind.org.cn/download/PDBbind_v2020_refined.tar.gz
wget http://bindingnet.huanglab.org.cn/api/api/download/binding_database
wget https://www.bindingdb.org/bind/chemsearch/marvin/SDFdownload.jsp?download_file=/rwd/data/surflex/surflex.tar
```
Put PDBbind data into *data/pdbbind/refined-set* and *data/pdbbind/general-set*

Put BindingNet data into *data/bindingnet/from_chembl_client*

Put BindingDB-DCS data into *data/bindingdb/surflex*

#### Generate pickled graphs
The following scripts will generate graphs into *pdbbind.pickle*, *bindingnet.pickle*, and *bindingdb.pickle*. Takes around 40 minutes to run.
```
python generate_pdbbind_graphs.py
python generate_bindingnet_graphs.py
python generate_bindingdb_graphs.py
```

#### Generate data for pytorch
Running this script takes around 2 minutes.
```
python create_pytorch_data.py
```
The script outputs the following files in *data/processed/*:

*pdbbind_U_bindingnet_U_bindingdb_ligsim90_fep_benchmark_train.pt*, *pdbbind_U_bindingnet_U_bindingdb_ligsim90_fep_benchmark_valid.pt*, and *pdbbind_U_bindingnet_U_bindingdb_ligsim90_fep_benchmark_test.pt*

#### Run training
Running the following script takes 28 hours using a NVIDIA GeForce GTX 1080 Ti
GPU. Once a model has been trained, the next section describes how to use it for predictions.
```
python training.py --activation_function=leaky_relu --batch_size=128 --dataset=pdbbind_U_bindingnet_U_bindingdb_ligsim90 --epochs=200 --head=3 --hidden_dim=256 --lr=0.00012291937615434127 --model=GATv2Net
```
The trained models are saved in *output/trained_models*


### Predictions
In order to make predictions, the model requires a *.csv* file with the following columns:
- *unique_id*, unique identifier for the datapoint
- *sdf_file*, relative path to the ligand *.sdf* file
- *pdb_file*, relative path to the protein *.pdb* file

An example dataset is included in *data/example_dataset.csv* for this demo.

```
python process_and_predict.py --dataset_csv=data/example_dataset.csv --data_name=example --trained_model_name=model_GATv2Net_ligsim90_fep_benchmark
```
The script processes data in *dataset_csv*, and removes datapoints if:
1. .sdf file cannot be read by RDkit
2. Molecule contains rare element
3. Molecule has undefined bond type

The script then creates graphs and pytorch data to run the AEV-PLIG model specified with *trained_model_name*.

The predictions are saved under *output/predictions/data_name_predictions.csv*

For the example dataset, the script takes around 20 seconds to run
