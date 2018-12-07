# PyTorch Unsupervised Sentiment Discovery
This codebase contains code to reproduce results from our series of large scale pretraining + transfer papers: _[Large Scale Language Modeling: Converging on 40GB of Text in Four Hours](https://nv-adlr.github.io/publication/2018-large-batch-lm)_ and _[Practical Text Classification With Large Pre-Trained Language Models](https://arxiv.org/abs/1812.01207)_. This effort was born out of a desire to reproduce, analyze, and scale the [Generating Reviews and Discovering Sentiment](https://github.com/openai/generating-reviews-discovering-sentiment) paper from OpenAI.

This codebase supports mixed precision training as well as distributed, multi-gpu, multi-node training for language models (support is provided based on the NVIDIA [APEx](https://github.com/NVIDIA/apex) project). In addition to training language models, this codebase can be used to easily transfer and finetune trained models on custom text classification datasets.

For example, a [Transformer](https://arxiv.org/abs/1706.03762) language model for unsupervised modeling of large text datasets, such as the [amazon-review dataset](http://jmcauley.ucsd.edu/data/amazon/), is implemented in PyTorch. We also support other tokenization methods, such as character or sentencepiec tokenization, and language models using various recurrent architectures.

The learned language model can be transferred to other natural language processing (NLP) tasks where it is used to featurize text samples. The featurizations provide a strong initialization point for discriminative language tasks, and allow for competitive task performance given only a few labeled samples. We illustrate this below and show the model transferred to the [Binary Stanford Sentiment Treebank task](https://nlp.stanford.edu/sentiment/treebank.html).

![main performance fig](./figures/reproduction_graph.png "Reconstruction and Transfer performance")

The model's performance as a whole will increase as it processes more data.

## ReadMe Contents
 * [Setup](#setup)
   * [Install](#install)
   * [Pretrained Models](#pretrained-models)
   * [Data Downloads](#data-downloads)
 * [Usage](#usage)
   * [Classifying Text](#classifying-text)
     * [Classification Documentation](./script_docs/classifier.md)
   * [Training Language Models (+ Distributed/FP16 Training)](#training-language-models-distributed-fp16-training)
     * [Modeling Documentation](./script_docs/modeling.md)
     * [Training HyperParameter Documentation](./analysis/reproduction.md#training-set-up)
     * [FP16 Training Information](./analysis/reproduction.md#fp16-training)
   * [Sentiment Transfer](#sentiment-transfer)
     * [Transfer Documentation](./script_docs/transfer.md)
   * [Classifier Finetuning](#classifier-finetuning)
     * [Finetuning Documentation](./script_docs/finetune.md)
 * [Analysis](#analysis)
    * [Why Unsupervised Language Modeling?](./analysis/unsupervised.md)
     * [Difficulties of Supervised Natural Language](./analysis/unsupervised.md#difficulties-of-supervised-natural-language)
     * [Data Robustness](./analysis/unsupervised.md#data-robustness)
     * [Model/Optimization Robustness](./analysis/unsupervised.md#modeloptimization-robustness)
  * [Reproducing Results](./analysis/reproduction.md)
     * [Training](./analysis/reproduction.md#training)
        * [Training Setup](./analysis/reproduction.md#training-set-up)
     * [FP16 Training](./analysis/reproduction.md#fp16-training)
     * [8K Model Training](./analysis/reproduction.md#going-bigger-with-large-models)
     * [Transfer](./analysis/reproduction.md#transfer)
  * [Data Parallel Scalability](./analysis/scale.md)
     * [PyTorch + GIL](./analysis/scale.md#pytorch-gil)
  * [Open Questions](./analysis/questions.md)
 * [Acknowledgement](#acknowledgement)
 * [Thanks](#thanks)

## Setup
### Install
Install the sentiment_discovery package with `python3 setup.py install` in order to run the modules/scripts within this repo.

### Python Requirements
At this time we only support python3.
 * numpy
 * pytorch (>= 0.4.1)
 * pandas
 * scikit-learn
 * matplotlib
 * unidecode
 * sentencepiece
 * seaborn
 * emoji

### Pretrained models
We've included our sentencepiece tokenizer model and vocab as a zip file:
 * [sentencepiece tokenizer](https://drive.google.com/file/d/1fxdSrcnpB4OtzmE3PT7E6CtY6seoSSth/view?usp=sharing) [1MB]

These models are temporarily deprecated and do not work with current versions of the repo
We've included our trained 4096-d mlstm models in both fp16 and fp32:
 * [Binary SST]() [329MB]
 * [Binary SST (FP16)]() [163MB]
 * [IMDB]() [329MB]
 * [IMDB (FP16)]() [163MB]

We've also included our trained 8192-d mlstm models in fp16:
 * [Binary SST (FP16)]() [649 MB]
 * [IMDB (FP16)]() [649 MB]
 
Each file contains a PyTorch `state_dict` consisting of a language model (encoder+decoder keys) trained on Amazon reviews and a binary sentiment classifier (classifier key) trained with transfer learning on Binary SST/IMDB.

### Data Downloads
We've provided the Binary Stanford Sentiment Treebank (Binary SST) and IMDB Movie Review datasets as part of this repository. In order to train on the amazon dataset please download the "aggressively deduplicated data" version from Julian McAuley's original [site](http://jmcauley.ucsd.edu/data/amazon/). Access requests to the dataset should be approved instantly. While using the dataset make sure to load it with the `-loose_json` flag.

## Usage
In addition to providing easily reusable code of the core functionalities (models, distributed, fp16, etc.) of this work, we also provide scripts to perform the high-level functionalities of the original paper:
 * sentiment classification of input text
 * unsupervised reconstruction/language modeling of a corpus of text (+ script for launching distributed workers)
 * transfer of learned language model to perform sentiment analysis on a specified corpus
 * sampling from language model to generate text (possibly of fixed sentiment) + heatmap visualization of sentiment in text

<!--Script results will be saved/logged to the `<experiment_dir>/<experiment_name>/*` directory hierarchy.-->

### Classifying text
Classify an input csv/json using one of our pretrained models or your own.
Performs classification on Binary SST by default.
Output classification probabilities are saved to a `.npy` file

```
python3 run_classifier.py --load_model ama_sst.pt                               # classify Binary SST
python3 run_classifier.py --load_model ama_sst_16.pt --fp16                     # run classification in fp16
python3 run_classifier.py --load_model ama_sst.pt --text-key <text-column> --data <path.csv>     # classify your own dataset
```

See [here](./script_docs/classifier.md) for more documentation.

### Training Language Models (+ Distributed/FP16 Training)
Train a recurrent language model on a csv/json corpus. By default we train a weight-normalized, 4096-d mLSTM, with a 64-d character embedding.
This is the first step of a 2-step process to training your own sentiment classifier.
Saves model to `lang_model.pt` by default.

```
python3 pretrain.py                                                               #train a large model on imdb
python3 pretrain.py --model LSTM --nhid 512                                       #train a small LSTM instead
python3 pretrain.py --fp16 --dynamic_loss_scale                                   #train a model with fp16
python3 -m multiproc pretrain.py                                                  #distributed model training
python3 pretrain.py --data ./data/amazon/reviews.json --lazy --loose_json \       #train a model on amazon data
  --text_key reviewText --label_key overall --num_shards 1002 \
  --optim Adam --split 1000,1,1 
python3 pretrain.py --tokenizer-type SentencePieceTokenizer --vocab-size 32000 \  #train a model with our sentencepiece tokenization
  --tokenizer-type bpe --tokenizer-path tokenizer.model 
python3 pretrain.py --tokenizer-type SentencePieceTokenizer --vocab-size 32000 \  #train a transformer model with our sentencepiece tokenization
  --tokenizer-type bpe --tokenizer-path tokenizer.model --model transformer \
  --decoder-layers 12 --decoder-embed-dim 768 --decoder-ffn-embed-dim 3072 \
  --decoder-learned-pos --decoder-attention-heads 8
bash ./experiments/train_mlstm_singlenode.sh                                      #run our mLSTM training script on 1 DGX-1V
bash ./experiments/train_transformer_singlenode.sh                                #run our transformer training script on 1 DGX-1V 
```

For more documentation of our language modeling functionality look [here](./script_docs/modeling.md)

In order to appropriately set the learning rate for a given batch size see the [training reproduction](./analysis/reproduction.md#training-set-up) section in analysis.

For information about how we achieve numerical stability with FP16 training see our [fp16 training](./analysis/reproduction.md#fp16-training) analysis.

### Sentiment Transfer
Given a trained language model, this script will featurize text from train, val, and test csv/json's.
It then uses sklearn logistic regression to fit a classifier to predict sentiment from these features.
Lastly it performs feature selection to try and fit a regression model to the top n most relevant neurons (features).
By default only one neuron is used for this second regression.

```
python3 transfer.py --load <model>.pt                               #performs transfer to SST, saves results to `<model>_transfer/` directory
python3 transfer.py --load <model>.pt --neurons 5                   #use 5 neurons for the second regression
python3 transfer.py --load <model>.pt --fp16                        #run model in fp16 for featurization step
```

Expected test accuracy for transfering fully trained mLSTM models to sentiment classification for a given mLSTM hidden size:

![Sentiment Transfer Performance](./figures/sentiment_performance.png)

Additional documentation of the command line arguments available for transfer can be found [here](./script_docs/transfer.md)
 
### Classifier Finetuning
Given a trained language model and classification dataset, this script will build a classifier that leverages the trained language model as a text feature encoder.
The difference between this script and `transfer.py` is that the model training is performed end to end: the loss from the classifier is backpropagated into the language model encoder as well.
This script allows one to build more complex classification models, metrics, and loss functions than `transfer.py`.
This script supports building arbitrary multilable, multilayer, and multihead perceptron classifiers. Additionally it allows using language modeling as an auxiliary task loss during training and multihead variance as an auxiliary loss during training.
Lastly this script supports automatically selecting classification thresholds from validation performance. To measure validation performance this script includes more complex metrics including: f1-score, mathew correlation coefficient, jaccard index, recall, precision, and accuracy.

```
python3 finetune_classifier.py --load <model>.pt --lr 2e-5 --aux-lm-loss --aux-lm-loss-weight .02 #finetune mLSTM model on sst (default dataset) with auxiliary loss
python3 finetune_classifier.py --load <model>.pt --automatic-thresholding --threshold-metric f1   #finetune mLSTM model on sst and automatically select classification thresholds based on the validation f1 score
python3 finetune_classifier.py --tokenizer-type SentencePieceTokenizer --vocab-size 32000 \       #finetune transformer with sentencepiece on SST
  --tokenizer-type bpe tokenizer-path tokenizer.model --model transformer --lr 2e-5 \
  --decoder-layers 12 --decoder-embed-dim 768 --decoder-ffn-embed-dim 3072 \
  --decoder-learned-pos --decoder-attention-heads 8 --load <model>.pt --use-final-embed
python3 finetune_classifier.py --automatic-thresholding --non-binary-cols l1 l2 l3 --lr 2e-5\     #finetune multilayer classifier with 3 classes and 4 heads per class on some custom dataset and automatically select classfication thresholds
  --classifier-hidden-layers 2048 1024 3 --heads-per-class 4 --aux-head-variance-loss-weight 1.   #`aux-head-variance-loss-weight` is an auxiliary loss to increase the variance between each of the 4 head's weights
  --data <custom_train>.csv --val <custom_val>.csv --test <custom_test>.csv --load <model>.pt                                                                
```

Additional documentation of the command line arguments available for `finetune_classifier.py` can be found [here](./script_docs/finetune.md)

## [Analysis](./analysis/)
 * [Why Unsupervised Language Modeling?](./analysis/unsupervised.md)
   * [Difficulties of Supervised Natural Language](./analysis/unsupervised.md#difficulties-of-supervised-natural-language)
   * [Data Robustness](./analysis/unsupervised.md#data-robustness)
   * [Model/Optimization Robustness](./analysis/unsupervised.md#modeloptimization-robustness)
 * [Reproducing Results](./analysis/reproduction.md)
   * [Training](./analysis/reproduction.md#training)
     * [Training Setup](./analysis/reproduction.md#training-set-up)
   * [FP16 Training](./analysis/reproduction.md#fp16-training) 
   * [8K Model Training](./analysis/reproduction.md#going-bigger-with-large-models)
   * [Transfer](./analysis/reproduction.md#transfer)
 * [Data Parallel Scalability](./analysis/scale.md)
   * [PyTorch + GIL](./analysis/scale.md#pytorch-gil)
 * [Open Questions](./analysis/questions.md)

## Acknowledgement
A special thanks to our amazing summer intern [Neel Kant](https://github.com/kantneel) for all the work he did with transformers, tokenization, and pretraining+finetuning classification models.

A special thanks to [@csarofeen](https://github.com/csarofeen) and [@Michael Carilli](https://github.com/mcarilli) for their help developing and documenting our RNN interface, Distributed Data Parallel model, and fp16 optimizer. The latest versions of these utilities can be found at the [APEx github page](https://github.com/NVIDIA/apex).

Thanks to [@guillitte](https://github.com/guillitte) for providing a lightweight pytorch [port](https://github.com/guillitte/pytorch-sentiment-neuron) of openai's sentiment-neuron repo.

This project uses the [amazon review dataset](http://jmcauley.ucsd.edu/data/amazon/) collected by J. McAuley


## Thanks
Want to help out? Open up an issue with questions/suggestions or pull requests ranging from minor fixes to new functionality.

**May your learning be Deep and Unsupervised.**
