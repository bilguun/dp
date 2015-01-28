# Language Model Tutorial #

In this tutorial, we will explore the implementation of 
[language models](http://en.wikipedia.org/wiki/Language_model) (LM) using dp and nn. 
LMs offer a way to estimate the relative likelihood of different phrases, 
which is useful in many statistical [natural language processing](http://en.wikipedia.org/wiki/Natural_language_processing) 
(NLP) applications. 
Many approaches and variations of the concept exist, but here the basic idea is to maximize the likelihood of the next word
given a context of previous words. 

## One Billion Words Benchmark ##

For our experiments, we use the [one billion words benchmark](http://arxiv.org/abs/1312.3005) used for 
measuring progress in language models.
The dataset contains one billion words. The task consists in using the 
previous `n` words (the context) to predict the next word (the target word).
All sentences are shuffled such that only the words from the target's sentence 
can be used for prediction. The end of the sentence, identified 
by token `"</S>"`, must also be predicted. To predict the first $n$ words of a sentence, 
padding is added. Using sentence `"<S> Alice is writing</S>"`
as an example, each input -> target word would have the following contexts of 3 words:

  * `"</S> </S> <S>" -> "Alice"` ;
  * `"</S> <S> Alice" -> "is"` ;
  * `"<S> Alice is" -> "writing"` ; and
  * `"Alice is writing" -> "</S>"`.

The entire dataset is divided into 100 partitions of equal size,
99 of which are used for training. The remaining partition is further 
divided into 50 partitions, one of which is used for testing,
while the remaining 49 are reserved for cross-validation. 
All words with less then 3 occurrences in the training set are replaced 
with the `"<UNK>"` token. This is the same split used in the [original paper](http://arxiv.org/abs/1312.3005).
The dataset is wrapped by the [BillionWords](data.md#dp.BillionWords) [DataSource](data.md#dp.DataSource).
The downloaded `billionwords.tar.gz` compressed tarball contains the following files:
 
  * `train_data.th7`, `train_small.th7` and `train_tiny.th7` training sets of different size (from fullest to smallest) ;
  * `valid_data.th7` the validation set ;
  * `test_data.th7` the test set ;  
  * `word_freq.th7` the frequencies of words (not really used) ;  
  * `word_tree1.th7`, `word_tree2.th7` and `word_tree3.th7` different hierarchies of words ; and
  * `word_map.th7` maps the word IDs (efficient integers) to the actual words (bulky strings). 
  
The training, validation and test set files contain serialized 2D Tensors.
Each such Tensor has 2 columns. First column is for storing start indices of sentences. 
Second column is for storing the sequence of word IDs of shuffled sentences. 
Sentences are seperated by `sentence_end` word ID (see [SentenceSet](data.md#dp.SentenceSet)).

Using the BillionWords DataSource is very light:
```lua
--[[data]]--
local train_file = 'train_data.th7' 
if opt.small then 
   train_file = 'train_small.th7'
elseif opt.tiny then 
   train_file = 'train_tiny.th7'
end

local datasource = dp.BillionWords{
   context_size = opt.contextSize, train_file = train_file
}
```
If the compressed file isn't found on disk, it will be downloaded from a 
University of Montreal server auto-magically. You can specify the `contextSize` (`n`)
and whether or not you wish to use the memory-efficient tiny and small files instead.

## Neural Network Language Model ##

A neural network language model (NNLM) uses a neural network to model language. 
There are various approaches to building NNLMs.
The first NNLM was presented in [(Bengio et al., 2001)](http://papers.nips.cc/paper/1839-a-neural-probabilistic-language-model.pdf), 
which we used as a baseline to implement a  
[NNLM training script](https://github.com/nicholas-leonard/dp/blob/master/examples/languagemodel.lua) for dp.
In many respects, the script is very similar to the other training scripts includes in the 
[examples directory](https://github.com/nicholas-leonard/dp/tree/master/examples). Since the basics of these scripts 
are explained in the [Neural Network](https://github.com/nicholas-leonard/dp/blob/master/doc/neuralnetworktutorial.md#neural-network-tutorial) and 
[Facial Keypoint Detection](https://github.com/nicholas-leonard/dp/blob/master/doc/facialkeypointstutorial.md#facial-keypoints-tutorial) 
Tutorials, we assume that the reader has consulted these beforehand. 

The actual training is very similar to the first tutorial. An NNLM can be 
formulated as a classification problem where the objective is to correctly 
classify the next word (target class) given the previous words (inputs). 
Lets take a look at the different components used to build a NNLM.

### Input Layer ###

The input is a sequence of `n` word indices. We use these to concatenate `n` 
context word embeddings at the input layer to form an 
input of size `n x m`, where `m` is the number of units per word embedding. 
An embedding is just a vector of weights. It is called an embedding in the sense 
that each word will be embedded into a common representation space of `m` dimensions.

This input layer takes the form of a *lookup table*. We can think of it as a weight matrix `W` 
of size `N x m` where `N` is the number of words in our vocabulary.
In the case of the billion words dataset, we have approximately 800,000 words. 
Each word is assigned a single row of weight matrix `W` which will serve as its embedding. 
These embeddings are parameter vectors that can be learned through backpropagation. 

A [LookupTable](https://github.com/torch/nn/blob/master/doc/convolution.md#nn.LookupTable)
Module is available in [nn](https://github.com/torch/nn/blob/master/README.md).
The [Dictionary](model.md#dp.Dictionary) Model adapts the Module for use within dp.

The (non-batch) input to the LookupTable is a vector `x` of dimension 
`n` where each variable `x[i]` contains the index of the word at position `i` of the context.
These are used to extract all embeddings of the lookup table that correspond to the context words:
```lua
y = W[x[1]] || W[x[2]] || W[x[3]] || ... || W[x[n]]
```
where `||` concatenates its left and right vectors. The gradient can be calculated as follows:
```lua
 dy      1 for j in x  
----- =  
dW[j]    0 for j not-in x
```
which makes this layer efficient for both forward and backward propagation 
since only the `n` context words need to be queried, concatenated and updated.
The code for this looks like :
```lua
inputModel = dp.Dictionary{
   dict_size = datasource:vocabularySize(),
   output_size = opt.inputEmbeddingSize,
   acc_update = opt.accUpdate
}
```
The `acc_update` argument is available for most [Layer](model.md#dp.Layer)
instances. Its faster and more memory efficient to set this to true as 
in many cases the Tensor storing gradients with respect to parameters (weights, biases, etc.)
can be omitted, thereby freeing up some of that much needed GPU memory.
The `dict_size` and `output_size` are the aforementioned `N` and `m`. 

### Hidden Layers ###

The resulting concatenation of embeddings can be forwarded through 
parameterized hidden layers having the following form:
```
y = sigma(Wx + b)
```
where `sigma` is an element-wise [transfer](https://github.com/torch/nn/blob/master/doc/transfer.md#transfer-function-layers) 
function. NNLMs are often shallow networks having no more than 1 or 2 parameterized hidden layers :
[(Schwenk et al., 2005)](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.228.2482&rep=rep1&type=pdf#page=237), 
[(Lee et al., 2011)](http://www.researchgate.net/publication/220733157_Structured_Output_Layer_neural_network_language_model/file/e0b4951cbf8e0d0c6b.pdf).

As seen in the previous tutorials, the [Neural](model.md#dp.Neural) Model can be used to implement hidden layers.
This model adapts a [Sequential](https://github.com/torch/nn/blob/master/doc/containers.md#nn.Sequential) 
[Container](https://github.com/torch/nn/blob/master/doc/containers.md#nn.Container) 
which itself encapsulates a [Linear](https://github.com/torch/nn/blob/master/doc/simple.md#nn.Linear) 
followed by a transfer Module, like [Tanh](https://github.com/torch/nn/blob/master/doc/transfer.md#nn.Tanh).

For our model we use two hidden layers. The first is instantiated as :
```lua
hiddenModel = dp.Neural{
   input_size = opt.contextSize*opt.inputEmbeddingSize,
   output_size = opt.neuralSize, 
   transfer = nn.Tanh(),
   dropout = opt.dropout and nn.Dropout() or nil,
   acc_update = opt.accUpdate
}
```
The training script also provides the option of substituting a 
[TemporalConvolution](https://github.com/torch/nn/blob/master/doc/convolution.md#nn.TemporalConvolution)
for this Neural layer instead. The resulting model would be similar to what was implemented in 
[(Collobert et al., 2011)](http://arxiv.org/pdf/1103.0398). We recommend against it 
as the TemporalConvolution is still very slow on GPU.

And the second hidden layer is instantiated as :
```lua
dp.Neural{
   input_size = inputSize, 
   output_size = opt.outputEmbeddingSize, 
   transfer = nn.Tanh(),
   dropout = opt.dropout and nn.Dropout() or nil,
   acc_update = opt.accUpdate
}
```
The `outputEmbeddingSize` is the size of the embedding space used to 
model words in the output layer.

### Output Layer ###

The Neural Model can also be used to instantiate a NNLM output layer, where the
transfer Module is a [SoftMax](https://github.com/torch/nn/blob/master/doc/transfer.md#nn.SoftMax)
A very popular choice for classification output layers, softmax is a normalizing non-linearity of the form : 
```lua
                     exp(x[i])
y[i] = -------------------------------------
       exp(x[1])+...+exp(x[i])+...+exp(x[N])
```
where `N` is again the size of vectors `x` and `y` (the size of the vocabulary), 
and `exp` is the exponential function.

The softmax's use of the exponential function has a tendency of increasing 
the weight of the highest input values, thus forming a kind of soft version 
of the `max` function. By dividing by the sum of the exponential of each 
variable in the vector `x`, it has a normalizing effect in that `y[1]+y[2]+..+y[N] = 1`, 
thus making it useful for generating multinomial probabilities `P(Y|X)`. The code for 
this particular implementation of the output layer is as follows:
```lua
softmax = dp.Neural{
   input_size = opt.outputEmbeddingSize,
   output_size = table.length(datasource:classes()),
   transfer = nn.LogSoftMax(),
   dropout = opt.dropout and nn.Dropout() or nil,
   acc_update = opt.accUpdate
}
```

The forward and backward propagation of this layer are extremely costly in 
terms of processing time for large vocabularies. This inefficiency is due 
to the normalization which requires calculation of all `x[i]` for `1 < i < N`. 
Various solutions have been proposed to circumvent the issue. 
All of these are variants of the original class decomposition idea 
[(Bengio, 2002)](http://old-site.clsp.jhu.edu/ws2002/seminars/bengio_tr1215.pdf) :
Other similar approaches exist, including importance sampling (bengio2003quick) and 
uniform sampling of ranking criterion (\cite{collobert2011natural}), we chose to focus 
our attention on the following: 

 * hierarchical softmax : \cite{morin2005hierarchical};
 * hierarchical log-bilinear model : \cite{mnih2009scalable};
 * structured output layer : \cite{le2011structured};
 * noise-constrastive estimation : \cite{mnih2012fast};

Faster alternatives to using a pure `softmax` exist, such as the [SoftmaxTree](model.md#dp.SoftmaxTree).
This Model encapsulates the nnx package's SoftMaxTree Module. It is a particular variation of 
the well-known [hierarchical softmax](citation needed). 


#### SoftMaxTree ####

For the Billion Words dataset, we performed a hierarchical clustering of 
the Vocabulary using the sets of context words preceding each 
word in the training set. The resulting tree of words is *approximately* organized as follows :
 * 10 cluster1s containing 10 cluster2s each 
 * 100 cluster2s containing 10 cluster3s each
 * 1000 cluster3s contraining 10 cluster4s each
 * 10000 cluster4s contraining 10 cluster5s each
 * 100000 cluster5s contraining 10 words each
 
The BillionWords DataSource comes with 3 such hierarchies. The second 
word tree is clustered in such a way as to minimize overlap with the first,
the third minimizes overlap with both of these. (They can be combined together 
using the [SoftMaxForest](model.md#dp.SoftmaxForest)).

The

### Criterion ###

The empirical risk function of the model is the ubiquitous mean negative log-likelihood (NLL):
```lua
      -log(y[1,t[1]])-log(y[2,t[2]])-...-log(y[K,t[K]])
NLL = -------------------------------------------------  
                             K
```
where `K` is the total number of examples, and `t[k]` is the target word having context `x[k]`. 
Finally, `y[k,t[k]]` is the likelihood of word `t[k]` for example `k`, 
where `y[k]` is the output of the NNLM given context `x[k]`.

To evaluate our NNLMs, we use perplexity (PPL) as this is the 
standard metric used in the field NLP for language modeling:
```lua
PPL = exp(NLL)
``` 
Note: the above is true as long as the NLL and PPL use the same logarithm basis (`e` in this case).

## Output Layer ##
SoftMaxTree for faster propagations.