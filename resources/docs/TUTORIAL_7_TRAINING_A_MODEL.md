# Tutorial 7: Training a Model

This part of the tutorial shows how you can train your own sequence labelling and text classification models using state-of-the-art word embeddings.

For this tutorial, we assume that you're familiar with the [base types](/resources/docs/TUTORIAL_1_BASICS.md) of this
library and how [word embeddings](/resources/docs/TUTORIAL_3_WORD_EMBEDDING.md) work (ideally, you also know how [flair embeddings](/resources/docs/TUTORIAL_4_ELMO_BERT_FLAIR_EMBEDDING.md) work). 
You should also know how to [load
a corpus](/resources/docs/TUTORIAL_6_CORPUS.md).


## Training a Sequence Labeling Model

Here is example code for a small NER model trained over WikiNER data, using simple GloVe embeddings. 
To run this code, you first need to obtain the CoNLL-03 English dataset (alternatively, use `NLPTaskDataFetcher.load_corpus(NLPTask.WNUT_17)` instead for a task with freely available data).

In this example, we downsample the data to 10% of the original data because the WikiNER dataset is huge:

```python
from flair.data import Corpus
from flair.datasets import WIKINER_ENGLISH
from flair.embeddings import TokenEmbeddings, WordEmbeddings, StackedEmbeddings
from typing import List

# 1. get the corpus
corpus: Corpus = WIKINER_ENGLISH().downsample(0.1)
print(corpus)

# 2. what tag do we want to predict?
tag_type = 'ner'

# 3. make the tag dictionary from the corpus
tag_dictionary = corpus.make_tag_dictionary(tag_type=tag_type)
print(tag_dictionary.idx2item)

# 4. initialize embeddings
embedding_types: List[TokenEmbeddings] = [

    WordEmbeddings('glove'),

    # comment in this line to use character embeddings
    # CharacterEmbeddings(),

    # comment in these lines to use flair embeddings
    # FlairEmbeddings('news-forward'),
    # FlairEmbeddings('news-backward'),
]

embeddings: StackedEmbeddings = StackedEmbeddings(embeddings=embedding_types)

# 5. initialize sequence tagger
from flair.models import SequenceTagger

tagger: SequenceTagger = SequenceTagger(hidden_size=256,
                                        embeddings=embeddings,
                                        tag_dictionary=tag_dictionary,
                                        tag_type=tag_type,
                                        use_crf=True)

# 6. initialize trainer
from flair.trainers import ModelTrainer

trainer: ModelTrainer = ModelTrainer(tagger, corpus)

# 7. start training
trainer.train('resources/taggers/example-ner',
              learning_rate=0.1,
              mini_batch_size=32,
              max_epochs=150)

# 8. plot weight traces (optional)
from flair.visual.training_curves import Plotter
plotter = Plotter()
plotter.plot_weights('resources/taggers/example-ner/weights.txt')

```

Alternatively, try using a stacked embedding with FlairEmbeddings and GloVe, over the full data, for 150 epochs.
This will give you the state-of-the-art accuracy we report in the paper. To see the full code to reproduce experiments, check [here](/resources/docs/EXPERIMENTS.md).

Once the model is trained you can use it to predict tags for new sentences. Just call the `predict` method of the model.

```python
# load the model you trained
model = SequenceTagger.load('resources/taggers/example-ner/final-model.pt')

# create example sentence
sentence = Sentence('I love Berlin')

# predict tags and print
model.predict(sentence)

print(sentence.to_tagged_string())
```

If the model works well, it will correctly tag 'Berlin' as a location in this example.


## Training a Text Classification Model

Here is example code for training a text classifier over the TREC-6 corpus, using  a combination of simple GloVe embeddings and Flair embeddings. 

```python
from flair.data import Corpus
from flair.datasets import TREC_6
from flair.embeddings import WordEmbeddings, FlairEmbeddings, DocumentRNNEmbeddings
from flair.models import TextClassifier
from flair.trainers import ModelTrainer


# 1. get the corpus
corpus: Corpus = TREC_6()

# 2. create the label dictionary
label_dict = corpus.make_label_dictionary()

# 3. make a list of word embeddings
word_embeddings = [WordEmbeddings('glove'),

                   # comment in flair embeddings for state-of-the-art results
                   # FlairEmbeddings('news-forward'),
                   # FlairEmbeddings('news-backward'),
                   ]

# 4. initialize document embedding by passing list of word embeddings
# Can choose between many RNN types (GRU by default, to change use rnn_type parameter)
document_embeddings: DocumentRNNEmbeddings = DocumentRNNEmbeddings(word_embeddings,
                                                                     hidden_size=512,
                                                                     reproject_words=True,
                                                                     reproject_words_dimension=256,
                                                                     )

# 5. create the text classifier
classifier = TextClassifier(document_embeddings, label_dictionary=label_dict)

# 6. initialize the text classifier trainer
trainer = ModelTrainer(classifier, corpus)

# 7. start the training
trainer.train('resources/taggers/ag_news',
              learning_rate=0.1,
              mini_batch_size=32,
              anneal_factor=0.5,
              patience=5,
              max_epochs=150)

# 8. plot weight traces (optional)
from flair.visual.training_curves import Plotter
plotter = Plotter()
plotter.plot_weights('resources/taggers/ag_news/weights.txt')
```

Once the model is trained you can load it to predict the class of new sentences. Just call the `predict` method of the model.

```python
classifier = TextClassifier.load('resources/taggers/ag_news/final-model.pt')

# create example sentence
sentence = Sentence('France is the current world cup winner.')

# predict class and print
classifier.predict(sentence)

print(sentence.labels)
```


## Multi-Dataset Training

Now, let us train a single model that can PoS tag text in both English and German. 
To do this, we load both the English and German UD corpora and create a MultiCorpus object. 
We also use the new multilingual Flair embeddings for this task.

All the rest is same as before, e.g.:

```python
from typing import List
from flair.data import MultiCorpus
from flair.datasets import UD_ENGLISH, UD_GERMAN
from flair.embeddings import FlairEmbeddings, TokenEmbeddings, StackedEmbeddings
from flair.training_utils import EvaluationMetric


# 1. get the corpora - English and German UD
corpus: MultiCorpus = MultiCorpus([UD_ENGLISH(), UD_GERMAN()]).downsample(0.1)

# 2. what tag do we want to predict?
tag_type = 'upos'

# 3. make the tag dictionary from the corpus
tag_dictionary = corpus.make_tag_dictionary(tag_type=tag_type)
print(tag_dictionary.idx2item)

# 4. initialize embeddings
embedding_types: List[TokenEmbeddings] = [

    # we use multilingual Flair embeddings in this task
    FlairEmbeddings('multi-forward'),
    FlairEmbeddings('multi-backward'),
]

embeddings: StackedEmbeddings = StackedEmbeddings(embeddings=embedding_types)

# 5. initialize sequence tagger
from flair.models import SequenceTagger

tagger: SequenceTagger = SequenceTagger(hidden_size=256,
                                        embeddings=embeddings,
                                        tag_dictionary=tag_dictionary,
                                        tag_type=tag_type,
                                        use_crf=True)

# 6. initialize trainer
from flair.trainers import ModelTrainer

trainer: ModelTrainer = ModelTrainer(tagger, corpus)

# 7. start training
trainer.train('resources/taggers/example-universal-pos',
              learning_rate=0.1,
              mini_batch_size=32,
              max_epochs=150,
              )
```

Note that here we use the MICRO_ACCURACY evaluation metric instead of the default MICRO_F1_SCORE. 
This gives you a multilingual model. Try experimenting with more languages!



## Plotting Training Curves and Weights

Flair includes a helper method to plot training curves and weights in the neural network.
The `ModelTrainer` automatically generates a `loss.tsv` and a `weights.txt` file in the result folder.

After training, simple point the plotter to these files:

```python
from flair.visual.training_curves import Plotter
plotter = Plotter()
plotter.plot_training_curves('loss.tsv')
plotter.plot_weights('weights.txt')
```

This generates PNG plots in the result folder.


## Resuming Training

If you want to stop the training at some point and resume it at a later point, you should train with the parameter
`checkpoint` set to `True`.
This will save the model plus training parameters after every epoch.
Thus, you can load the model plus trainer at any later point and continue the training exactly there where you have left.

The example code below shows how to train, stop, and continue training of a `SequenceTagger`.
Same can be done for `TextClassifier`.

```python
from flair.data import Corpus
from flair.datasets import WNUT_17
from flair.embeddings import TokenEmbeddings, WordEmbeddings, StackedEmbeddings
from typing import List

# 1. get the corpus
corpus: Corpus = WNUT_17().downsample(0.1)

# 2. what tag do we want to predict?
tag_type = 'ner'

# 3. make the tag dictionary from the corpus
tag_dictionary = corpus.make_tag_dictionary(tag_type=tag_type)

# 4. initialize embeddings
embedding_types: List[TokenEmbeddings] = [
    WordEmbeddings('glove')
]

embeddings: StackedEmbeddings = StackedEmbeddings(embeddings=embedding_types)

# 5. initialize sequence tagger
from flair.models import SequenceTagger

tagger: SequenceTagger = SequenceTagger(hidden_size=256,
                                        embeddings=embeddings,
                                        tag_dictionary=tag_dictionary,
                                        tag_type=tag_type,
                                        use_crf=True)

# 6. initialize trainer
from flair.trainers import ModelTrainer
from flair.training_utils import EvaluationMetric

trainer: ModelTrainer = ModelTrainer(tagger, corpus)

# 7. start training
trainer.train('resources/taggers/example-ner',
              learning_rate=0.1,
              mini_batch_size=32,
              max_epochs=150,
              checkpoint=True)

# 8. stop training at any point

# 9. continue trainer at later point
from pathlib import Path

checkpoint = tagger.load_checkpoint(Path('resources/taggers/example-ner/checkpoint.pt'))
trainer = ModelTrainer.load_from_checkpoint(checkpoint, corpus)
trainer.train('resources/taggers/example-ner',
              learning_rate=0.1,
              mini_batch_size=32,
              max_epochs=150,
              checkpoint=True)
```


## Scalability: Training with Large Datasets

Many embeddings in Flair are somewhat costly to produce in terms of runtime and may have large vectors. 
Examples of this are `FlairEmbeddings`, `BertEmbeddings` and the other transformer-based embeddings. 
Depending on your setup, you can set options to optimize training time. 

The main parameter you need to set is the `embeddings_storage_mode` in the `train()` method of the `ModelTrainer`. 
It can have one of three values:

1. **'none'**: If you set `embeddings_storage_mode='none'`, embeddings do not get stored in memory. 
Instead they are generated on-the-fly in each training mini-batch (during *training*). 
The main advantage is that this keeps your memory requirements low.

2. **'cpu'**: If you set `embeddings_storage_mode='cpu'`, embeddings will get stored in regular memory. 

* during *training*: this in many cases speeds things up significantly since embeddings only need to be computed in the first epoch, after which they are just retrieved from memory. 
A disadvantage is that this increases memory requirements. 
Depending on the size of your dataset and your memory setup, this option may not be possible.
* during *inference*: this slow down your inference when used with a GPU as embeddings need to be moved from GPU memory to regular memory. 
The only reason to use this option during inference would be to not only use the predictions but also the embeddings after prediction.

3. **'gpu'**: If you set `embeddings_storage_mode='gpu'`, embeddings will get stored in CUDA memory. 
This will often be the fastest one since this eliminates the need to shuffle tensors from CPU to CUDA over and over again. 
Of course, CUDA memory is often limited so large datasets will not fit into CUDA memory. 
However, if the dataset fits into CUDA memory, this option is the fastest one.  


## Next

You can now either look into [optimizing your model](/resources/docs/TUTORIAL_8_MODEL_OPTIMIZATION.md) or
[training your own embeddings](/resources/docs/TUTORIAL_9_TRAINING_LM_EMBEDDINGS.md).
