# TranX

A general-purpose **Tran**sition-based abstract synta**X** parser 
that maps natural language queries into machine executable 
source code (e.g., Python) or logical forms (e.g., lambda calculus). **[Online Demo](http://moto.clab.cs.cmu.edu:8081/)**.

## System Architecture

For technical details please refer to our [ACL '18 paper](https://arxiv.org/abs/1806.07832) and [EMNLP '18 demo paper](https://arxiv.org/abs/1810.02720). 
To cope with different 
domain specific logical formalisms (e.g., SQL, Python, lambda-calculus, 
prolog, etc.), TranX uses abstract syntax trees (ASTs) defined in the 
Abstract Syntax Description Language (ASDL) as intermediate meaning
representation.

![Sysmte Architecture](doc/system.png)

Figure 1 gives a brief overview of the system.

1. TranX first employs a transition system to transform a natural language utterance into a sequence of tree-constructing actions, following the input grammar specification of the target formal language. The grammar specification is provided by users in textual format (e.g., `asdl/lang/py_asdl.txt` for Python grammar).

2. The tree-constructing actions produce an intermediate abstract syntax tree. TranX uses ASTs defined under the ASDL formalism as general-purpose, intermediate meaning representations.

3. The intermediate AST is finally transformed to a domain-specific representation (e.g., Python source code) using customly-defined conversion functions.

**File Structure** tranX is mainly composed of two components: 

1. A general-purpose transition system that defines the generation process of an AST `z`
 using a sequence of tree-constructing actions `a_0, a_1, ..., a_T`.
2. A neural network that computes the probability distribution over action sequences, conditional on the natural language query `x`, `p(a_0, a_1, ..., a_T | x)`.

These two components are implemented in the following two folders, respectively:

* `asdl` defines a general-purpose transition system based on the ASDL formalism, and its instantiations in different programming languages and datasets. The transition system defines how an AST is constructed using a sequence of actions. This package can be used as a standalone library independent of tranX. See Section 2.2 of the technical report for details.

* `model` contains the neural network implementation of the transition system defined in `asdl`, which computes action probabilities using neural networks.See Section 2.3 of the technical report for details.

Here is a detailed map of the file strcuture:
```bash
├── asdl (grammar-based transition system)
├── datasets (dataset specific code like data preprocessing/evaluation/etc.)
├── model (PyTorch implementation of neural nets)
├── server (interactive Web server)
├── components (helper functions and classes like vocabulary)
```

## Supported Language and Datasets

TranX officially supports the following grammatical formalism and datasets.
More languages (C#) are coming! 

Language | Transition System | Grammar Specification | Example Datasets
---------|--------------------| -------- | -------- 
Python 2   | `asdl.PythonTransitionSystem` | `asdl/lang/py/py_asdl.txt` | Django (Oda et al., 2015)
Python 3 | `asdl.Python3TransitionSystem` | `asdl/lang/py3/py3_asdl.simplified.txt` | CoNaLa (Yin et al., 2018) 
Lambda Calculus| `asdl.LambdaCalculusTransitionSystem` | `asdl/lang/lambda_asdl.txt` | ATIS, GeoQuery (Zettlemoyer and Collins, 2005)
Prolog | `asdl.PrologTransitionSystem` | `asdl/lang/prolog_asdl.txt`  | Jobs (Zettlemoyer and Collins, 2005)
SQL | `asdl.SqlTransitionSystem` | `asdl/lang/sql/sql_asdl.txt` | WikiSQL (Zhong et al., 2017)

### Evaluation Results

Here is a list of performance results on six datasets using pretrained models in `data/pretrained_models`

| Dataset | Results      | Metric             |
| ------- | ------------ | ------------------ |
| GEO     | 88.6         | Accuracy           |
| ATIS    | 87.7         | Accuracy           |
| JOBS    | 90.0         | Accuracy           |
| Django  | 77.2         | Accuracy           |
| CoNaLa  | 24.5         | Corpus BLEU        |
| WikiSQL | 79.1         | Execution Accuracy |


## Usage


### TL;DR

```bash
git clone https://github.com/pcyin/tranX
cd tranX

./pull_data.sh  # get datasets, training scripts, and pre-trained models

conda env create -f data/env/py3torch3cuda9.yml  # create conda Python environment

./scripts/atis/train.sh 0  # train on ATIS semantic parsing dataset with random seed 0
./scripts/geo/train.sh 0  # train on GEO dataset
./scripts/django/train.sh 0  # train on django code generation dataset
./scripts/conala/train.sh 0  # train on CoNaLa code generation dataset
./scripts/wikisql/train.sh 0  # train on WikiSQL SQL code generation dataset
```

### Web Server/HTTP API

`tranX` also ships with a web server for demonstraction and interactive debugging perpuse. It also exposes an HTTP API for online semantic parsing/code generation.


To start the web server, simply run:

```
source activate py3torch3cuda9
PYTHONPATH=../ python app.py --config_file data/server/config_py3.json
```

This will start a web server at port 8081 with ATIS/GEO/CoNaLa datasets.



**HTTP API** To programmically query `tranX` to get semantic parsing results, send your HTTP GET request to

```
http://<IP Address>:8081/parse/<dataset_name>/<utterance>

# e.g., http://localhost:8081/parser/atis/show me flight from Pittsburgh to Seattle
```



### Conda Environments

TranX supports both Python 2.7 and 3.5. Please note that 
some datasets only support Python 2.7 (e.g., Django) or Python 3+ (e.g., WikiSQL). We provide example
conda environments (`data/env/(py2torch3cuda9.yml|py3torch3cuda9.yml)`) for both Python versions.
You can export the enviroments using the following command:

```bash
conda env create -f data/env/(py2torch3cuda9.yml|py3torch3cuda9.yml)
```

**Note** The conda enviroments are generated on a Ubuntu 16.04 machine. If you are unable to import the enviroment with error message like `cannot found package numpy=1.14.3=py36h14a74c5_0`, please try removing the sha after the version number (e.g., `=py36h14a74c5_0`), since it might be different on different platforms. We keep the detailed version number with sha to ensure reproducibility.



## Re-ranking Model

This branch contains the code for training a reranker. 

Install additional dependent libraries. 
```shell script
pip install xgboost
``` 

Before training a reranker, we need to first train 
a tranX semantic parser. For instance, the following script trains a parser on the `ATIS` dataset:

```shell script
./scripts/atis/train.sh 0
```

Once we have the trained parsing model, we could then train the reconstruction model and the paraphrase identification model:

### Reconstruction Model

The following example command trains a reconstruction model on `ATIS`:
```shell script
python -u exp.py \
    --cuda \
    --lang lambda_dcs \
    --asdl_file asdl/lang/lambda_dcs/lambda_asdl.txt \
    --mode train_reconstructor \
    --batch_size 10 \
    --train_file data/atis/train.bin \
    --dev_file data/atis/dev.bin \
    --vocab data/atis/vocab.freq2.bin \
    --lstm lstm \
    --hidden_size 256 \
    --embed_size 128 \
    --dropout 0.3 \
    --patience 5 \
    --max_num_trial 5 \
    --lr_decay 0.5 \
    --log_every 50 \
    --save_to saved_models/atis/reranker.reconstructor.bin
``` 
### Paraphrase Model

The following example command trains a paraphrase model on `ATIS`. Note that it needs the decoded hypotheses on train/dev sets.
To generate the decoding files, you may use the pre-trained models under the `saved_models`, or train a new one using the above 
training script.
```shell script
# Perform inference over trained semantic parsing models on training and development set, dump the n-best decoding results``
python exp.py \
    --cuda \
    --mode test \
    --load_model saved_models/atis/path_to_trained_semantic_parsing_model.bin \
    --beam_size 5 \
    --test_file data/atis/(train.bin|dev.bin) \
    --evaluator default_evaluator \
    --save_decode_to decodes/atis/(train.decode|dev.decode) \
    --decode_max_time_step 110

python -u exp.py \
    --cuda \
    --lang lambda_dcs \
    --asdl_file asdl/lang/lambda/lambda_asdl.txt \
    --mode train_paraphrase_identifier \
    --batch_size 10 \
    --train_file data/atis/train.bin \
    --dev_file data/atis/dev.bin \
    --vocab data/atis/vocab.freq2.bin \
    --hidden_size 256 \
    --embed_size 128 \
    --train_decode_file decodes/atis/train.decode \
    --dev_decode_file decodes/atis/dev.decode \
    --dropout 0.2 \
    --patience 5 \
    --max_num_trial 5 \
    --lr_decay 0.5 \
    --log_every 50 \
    --save_to saved_models/atis/reranker.paraphrase_identifier.bin
```

### Tune the reranker

Finally, with the pre-trained reconstruction and paraphrase identification models, 
we could tune the weights of the reranker. First, we generate the input n-best files 
used by Travatar.
 
```python
from exp import *

LinearReranker.prepare_travatar_inputs_for_lambda_dcs_dataset(
    reconstructor_path='saved_models/atis/reranker.reconstructor.bin', 
    paraphrase_identifier_path='saved_models/atis/reranker.paraphrase_identifier.bin',
    dev_set_path='data/atis/dev.bin', 
    test_set_path='data/atis/test.bin',
    dev_decode_results_path='decodes/atis/dev.decode', 
    test_decode_results_path='decodes/atis/test.decode',
    nbest_output_path='travatar_files/atis/', 
    nbest_output_file_suffix = '.seed0' # suffix is used to mark the generated n-best file with extra flags; useful when you have multiple n-best files for the same dataset 
)
```

Next, call the Travatar to perform MERT training. Before running the following command, make sure you have 
Travatar properly installed and change the `TDIR` variable in `travatar_rerank.sh` to the 
correct installation path.

```shell script
./scripts/rerank/travatar_rerank.sh \
  travatar_files/atis/ \
  ".seed0"  # the second argument is the suffix as defined above
```

This would generate two fodlers inside `nbest_output_path`: a `model` folder with trained feature weights and a `log` folder
with tuning and evaluation logs. `ZEROONE` in the log files denote exact match accuracy.

### Trained Reranking Models

You may find a list of trained reraking models used in our paper 
at [this shared folder](https://www.dropbox.com/sh/nj3ec1z5j96gnvn/AADBwTB2uv07Opw260ZYqKC5a?dl=0)

## FAQs

#### How to adapt to a new programming language or logical form?

You need to implement the 
`TransitionSystem` class with a bunch of custom functions which (1) convert between 
domain-specific logical forms and intermediate ASTs used by TranX, (2) predictors which 
check if a hypothesis parse if correct during beam search decoding.
You may take a look at the examples in `asdl/lang/*`.

#### How to generate those pickled datasets (.bin files)?

Please refer to `datasets/<lang>/dataset.py` for code snippets that converts 
a dataset into pickled files. 

## Reference

TranX is described/used in the following two papers:

```
@inproceedings{yin18emnlpdemo,
    title = {{TRANX}: A Transition-based Neural Abstract Syntax Parser for Semantic Parsing and Code Generation},
    author = {Pengcheng Yin and Graham Neubig},
    booktitle = {Conference on Empirical Methods in Natural Language Processing (EMNLP) Demo Track},
    year = {2018}
}

@inproceedings{yin18acl,
    title = {Struct{VAE}: Tree-structured Latent Variable Models for Semi-supervised Semantic Parsing},
    author = {Pengcheng Yin and Chunting Zhou and Junxian He and Graham Neubig},
    booktitle = {The 56th Annual Meeting of the Association for Computational Linguistics (ACL)},
    url = {https://arxiv.org/abs/1806.07832v1},
    year = {2018}
}
```

## Thanks

We are also grateful to the following papers that inspire this work :P
```
Abstract Syntax Networks for Code Generation and Semantic Parsing.
Maxim Rabinovich, Mitchell Stern, Dan Klein.
in Proceedings of the Annual Meeting of the Association for Computational Linguistics, 2017

The Zephyr Abstract Syntax Description Language.
Daniel C. Wang, Andrew W. Appel, Jeff L. Korn, and Christopher S. Serra.
in Proceedings of the Conference on Domain-Specific Languages, 1997
```

We also thank [Li Dong](http://homepages.inf.ed.ac.uk/s1478528/) for all the helpful discussions and sharing the data-preprocessing code for ATIS and GEO used in our Web Demo.