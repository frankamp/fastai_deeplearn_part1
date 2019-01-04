# Lesson 10:  NLP Classification and Translation
### notes are continued from [lesson_10_1](lesson_10_1.md)

(02-Apr-2018, live)  

### Part 2 / Lesson 10
- [Wiki Lesson 10](http://forums.fast.ai/t/part-2-lesson-10-wiki/14364)
- [Video Lesson 10](https://www.youtube.com/watch?v=h5Tz7gZT9Fo&feature=youtu.be) 
  - video length:  2:07:55
- http://course.fast.ai/lessons/lesson10.html
- Notebook:  
   * [imdb.ipynb](https://github.com/fastai/fastai/blob/master/courses/dl2/imdb.ipynb)

---

## Language Model vs Word2Vec `58:20`
- one question that came up during break was: "How does this compare to word2vec?"
- this is actually a great thing for you to spend time thinking about during the week, is how does this compare to word2vec?
- I'll give you the summary now, but it's a very important conceptual difference
- The main conceptual difference is, what is word2vec?
- **word2vec** is a single embedding matrix.  Each word has a vector and that's it.  So, in other words, it's **a single layer** from a pre-trained model and, specifically, that layer is the input layer
- And also, specifically, that pre-trained model is a linear model, that is pre-trained on something that is called a **co-occurrence** matrix
- so we have no particular reason to believe that this model has learned anything much about the English language.  Or that it has any particular capabilities because it just has a single linear layer, and that's it.
- So, what's this wikitext 103 model? It's a language model.  It has a 400 dimensional embedding matrix, 3 hidden layers, with 1150 activations per layer and regularization, and all of that stuff.  
```python
em_sz, nh, nl = 400, 1150, 3
```
- tied in ?, input, output matrices, matrix equations ? , it's basically a state-of-the-art AWD LSTM
- ASGD Weight-Dropped == AWD
- ASGD = Asynchronous Stochastic Gradient Descent 
- so, what's the difference between a single layer of a single linear model vs a 3-layer recurrent neural network?
- `1:00:15` Everything!  You know, they're very different levels of capability.  You will see when you try using a pre-trained language model vs a word2vec layer, you'll get very, very different results, for the vast majority of tasks
- Rachel:  what if the numpy array does not fit in memory?  Is it possible to write a PyTorch data loader directly from a large csv file?
- JH: It almost certainly won't come up, so I won't spend time on it.  These things are tiny, they're just ints, think about how many ints you would need to run out of memory.  It's not going to happen.  They don't have to fit in GPU memory, just in your memory.
- So, I've actually done another Wikipedia model, which I called Giga Wiki which is on *all* of Wikipedia and even that easily fits in there 
- The reason I'm not using it [Giga Wiki] is because it turned out not to really help very much vs wikitext 103, but I've built a bigger model than anyone else I've found in the academic literature and it fits in memory on a single machine.  
- Rachel:  What is the idea behind averaging the weights of the embeddings?
- JH:  They're going to be set to something, you know.  There are words that weren't there, so other options is we could leave them at 0, but that seems like a very extreme thing to do, like zero is a *very extreme number*.  Why would it be 0? We could set it equal to some random numbers, but if so, what would be the mean and standard deviation of those random numbers? Or should they be uniform? If we just average the rest of the embeddings, than we have something that's a reasonable scale.
- Rachel:  Just to clarify, this is how you're initializing words that didn't appear in the training?
- JH:  Yes, that's right
- Rachel: And, then, I think you've pretty much answered this one, but someone had asked if there is a specific advantage to creating our own pre-trained embedding over using glob or word2vec? 
- JH: Yes, I think we have.  We're not creating a pre-trained embedding, we're creating a pre-trained model

## Language Model `1:02:20`
- ok, let's talk a little bit more... there's a ton of stuff you've seen before, but it's changed a little bit, it's actually a lot easier than it was in Part 1, but I want to go a little bit deeper into the language model loader.  
- on slide:  "You can use any iterable that creates a stream of batches as a data loader"
- So, this is the language model loader.  I really hope by now you've learned in your editor or IDE how to jump to symbols.  I don't want it to be a burden for you to find out what the source code of language model loader is, alright? 
- https://github.com/fastai/fastai/blob/7ac2c490c22e2f0c0ffe983e593c4671d6beed2b/fastai/nlp.py
- and, if it's a burden, please go back and try to learn those keyboard shortcuts in VS Code
- you know, if your editor doesn't make it easy, don't use that editor anymore, ok?  There are lots of good free editors that make this easy
- Here's the source code for `LanguageModelLoader`.  It's interesting to note that it is not doing anything particularly tricky.  It's not deriving from anything at all.  What makes it something that is capable of being a data loader is that it's something that you can iterate over.  
```python
class LanguageModelLoader():

    def __init__(self, ds, bs, bptt, backwards=False):
        self.bs,self.bptt,self.backwards = bs,bptt,backwards
        text = sum([o.text for o in ds], [])
        fld = ds.fields['text']
        nums = fld.numericalize([text],device=None if torch.cuda.is_available() else -1)
        self.data = self.batchify(nums)
        self.i,self.iter = 0,0
        self.n = len(self.data)

    def __iter__(self):
        self.i,self.iter = 0,0
        return self

    def __len__(self): return self.n // self.bptt - 1

    def __next__(self):
        if self.i >= self.n-1 or self.iter>=len(self): raise StopIteration
        bptt = self.bptt if np.random.random() < 0.95 else self.bptt / 2.
        seq_len = max(5, int(np.random.normal(bptt, 5)))
        res = self.get_batch(self.i, seq_len)
        self.i += seq_len
        self.iter += 1
        return res

    def batchify(self, data):
        nb = data.size(0) // self.bs
        data = data[:nb*self.bs]
        data = data.view(self.bs, -1).t().contiguous()
        if self.backwards: data=flip_tensor(data, 0)
        return to_gpu(data)

    def get_batch(self, i, seq_len):
        source = self.data
        seq_len = min(seq_len, len(source) - 1 - i)
        return source[i:i+seq_len], source[i+1:i+1+seq_len].view(-1)
```
- `1:03:40` So, specifically, here is the `fit` function inside fastai.model.  This is where everything ends up eventually which goes through each epoch, and then it creates an iterator from the data loader and then just does for-loop through it
- So anything you can do a for-loop through, can be a data loader.  Specifically, it needs to return tuples of mini-batches, independent and dependent variables, for mini-batches.  
- Anything with a ?dunda iter method (`__iter__`), is something that can act as an iterator, and `yield` is a neat little Python keyword you should probably learn about if you don't already know it, but it basically spits out *a thing* and waits for you to ask for another thing, normally in a for-loop or something
- [Python yield keyword explained](https://pythontips.com/2013/09/29/the-python-yield-keyword-explained/)
- So, in this case, we start by initializing the language model, passing it in the numbers.  So, this is the numericalized, big long list of all of our documents concatenated together
- and the first thing we do is to "batchify" it.  And this is the thing **quite a few of you got confused** :heavy_exclamation_mark: about last time, right?  If our batch size is 64 and we have 25 million numbers in our list, we are not creating items of length 64.  We're not doing that.  We are creating 64 items in total.  So, each of them is size t/64 (t divided by 64), which is 390,000 (or 390,590 to be exact)
- That's what we do here when we reshape it (`def batchify`) so this axis here `data = data.reshape(self.bs, -1).T` is of length 64 and this `-1` is everything else, so that's 390,590 blob, and then we transpose it
- That means we now have **64 columns**, **390,590 rows** and then what we do each time we do an iterate is we grab one batch of some sequence length (we'll look at that in a moment), but basically, it's approximately equal to `bptt` which we set to 70.  **`bptt` = back prop through time**
- and we just grab that many rows, so from i to i+70 rows and then we try to predict that plus 1.  Remember, so we're trying to predict 1 past what we're up to.  So, we've got 64 columns and each of those is 1/64th of our 25 million tokens, you know, hundreds of thousands long, and we just grab, you know 70 at a time so each of those columns, each time we grab it, is going to hook out to the previous column.  And that's why we get this consistency.  This language model is **stateful** which is very important.
- Pretty much all the cool stuff in the language model is stolen from Steven Merity's AWD-LSTM, including this little trick here which is, if we always grab 70 at a time, and then we go back and do a new epoch, we're going to grab exactly the same batches every time -- there is no randomness.  
  - Reference: [Regularizing and Optimizing LSTM Language Models](https://arxiv.org/pdf/1708.02182.pdf)
- Now, normally, we shuffle our data, every time we do an epoch or every time we grab some data, we grab it at random.  But, you can't do that with a language model because this set has to join up with the previous set because it's trying to learn the sentence, right? And if you suddently jump somewhere else, then that doesn't make any sense as a sentence
- So, Steven's idea is to say, ok, since we can't shuffle the order, let's instead randomly change the size, **the sequence length**.  So basically he says, 95% of the time we'll use bptt 70, *but* 5% of the time we will use half of that.  And then he says, you know what, I'm not even going to make that the sequence length, I'm going to create a normally distributed random number with that average and a standard deviation of 5, and I'll make that the sequence length, right? So the sequence length is 70'ish, and that means every time we go through, we're getting slightly different batches.  So, we've got that little bit of extra randomness.
- I asked Steven Merity where he came up with this idea, did he think of it? And, he was like, I think I thought of it, but it seemed so obvious that I bet I didn't think of it.  Which is like true of like every time I come up with an idea in deep learning, you know, it always seems so obvious that somebody else has thought of it, but I think he thought of it.
- So, yes, this is a nice thing to look at if you're trying to do something a bit unusual with the data loader, ok, here's a simple kind of raw (?role) model you can use as to creating a data loader from scratch, something that spits out batches of data

## Language Model Loader (back to notebook) `1:09:10`
- So our langauge model just took in all of the documents concatenated together, along with the batch size and bptt
```python
trn_dltrn_dl  ==  LanguageModelLoaderLanguage (np.concatenate(trn_lm), bs, bptt)
val_dl = LanguageModelLoader(np.concatenate(val_lm), bs, bptt)
md = LanguageModelData(PATH, 1, vs, trn_dl, val_dl, bs=bs, bptt=bptt)
```
#### Creating Your Own Learner and ModelData Classes is easy as this!
- Now, generally speaking, we want to create a learner and the way we normally do that is by getting a model data object and by calling some kind of method which have various names but sometimes, often we call that method `md.get_model`.  So the idea is that the model data object has enough information to know what kind of model to give you.  So, we have to create that model data object:  
`md = LanguageModelData(PATH, 1, vs, trn_dl, val_dl, bs=bs, bptt=bptt)` which means we need that class.  And that's very easy to do, right?
```python
learner= md.get_model(opt_fn, em_sz, nh, nl, 
    dropouti=drops[0], dropout=drops[1], wdrop=drops[2], dropoute=drops[3], dropouth=drops[4])

learner.metrics = [accuracy]
learner.freeze_to(-1)
```
- So, here are all of the pieces. We are going to create a custom learner, a custom model data class and a custom model class
- [nlp.py](https://github.com/fastai/fastai/blob/7ac2c490c22e2f0c0ffe983e593c4671d6beed2b/fastai/nlp.py)
- A model data class, again this one doesn't inherit from anything, so you really see... there is almost nothing to do. You need to tell it, most importantly: what's your training set, give it a data loader `trn_dl`, what's the validation set `val_dl`, give it a data loader, and optionally, give it a test set `test_dl1` data loader
- `01:10:22` Plus anything else it needs to know. So, it might need to know the `bptt`, it needs to know the number of tokens(`nt`, that's the vocab size), it needs to know what is the padding (`pad_idx`) index.  And so that it can save temporary files in models, model data's always need to know the path (`path`)
- and so we grab all of that stuff, and we dump it.  and that's it, that's the entire **initializer** (`__init__`), there's no logic there at all 
- so then, all of the work happens inside `get_model`.  And so, `get_model` calls something that we will look at later which just grabs a normal PyTorch nn.module architecture, ok? 
- and chucks it on the GPU:  `model = LanguageModel(to_gpu(m))`.  Note:  with PyTorch, normally we would say `.cuda()`.  With fast.ai, it's better to say `to_gpu`.  And the reason is that if you don't have a GPU, it will leave it on the CPU, and it also provides a global variable you can set to choose whether it goes on the GPU or not.  So, it's a better approach
- So, we wrap the model in a language model and the language model is `class LanguageModel()`
    - a Language Model is a sub-class of BasicModel
    - it almost basically does nothing except it defines layer groups.  And so, remember how when we do discriminative learning rates where different layers have different learning rates or like we freeze different amounts, we don't provide a different learning rate for every layer, because there can be 1000 layers, **we provide a learning rate for EVERY LAYER GROUP**.  So, when you create a custom model, you just have to overwrite this one thing which returns a list of all of your layer groups.
    - So, in this case, my last layer group contains the last part of the model and one bit of dropout:  `(self.model[1], m.dropouti)`
    - And, the rest of it, this * here, means pull this apart, so this going to be 1 layer per RNN layer: `*zip(m.rnns, m.dropouths)`
    - So, that's all that is 
```python
class LanguageModel(BasicModel):
    def get_layer_groups(self):
        m = self.model[0]
        return [*zip(m.rnns, m.dropouths), (self.model[1], m.dropouti)]
        
class LanguageModelData():
    def __init__(self, path, pad_idx, nt, trn_dl, val_dl, test_dl=None, bptt=70, backwards=False, **kwargs):
        self.path, self.pad_idx, self.nt = path, pad_idx, nt
        self.trn_dl, self.val_dl, self.test_dl = trn_dl, val_dl, test_dl
    
    def get_model(self, opt_fn, emb_sz, n_hid, n_layers, **kwargs):
        m = get_language_model(self.nt, emb_sz, n_hid, n_layers, self.pad_idx, **kwargs)
        model = LanguageModel(to_gpu(m))
        return RNN_Learner(self, model, opt_fn=optn_fn)

class RNN_Leaner(Learner):
    def __init__(self, data, models, **kwargs):
        super().__init__(data, models, **kwargs)
        self.crit = F.cross_entropy
```
- And then, finally, turn that into a **learner**, and so a learner, you just pass in the model, and it turns it into a learner! `return RNN_Learner(self, model, opt_fn=optn_fn)`
- In this case, we have overwritten learner; and the only thing we've done is to say, I want the default loss function to be cross-entropy
- So this entire set of custom Model, custom Model Data, custom (RNN) Learner, all fits on a single screen, and they always basically look like this
- So, that's a kind of dig inside this boring part of the code base
- the **interesting** part of this **code base** is **get_language_model**.  Because `get_language_model` is the thing that gives us our **AWD LSTM** and it actually contains the big idea -- the big incredibly simple idea that everybody else here thinks is really obvious.  But everyone in the NLP community I spoke to thought was insane, which is basically, every model can be thought of... pretty much every model can be thought of as a backbone plus a head -- and if you pre-train the backbone and stick on a random head, you can do fine tuning and that's a good idea, right?  
- So, these 2 bits of code right here, literally right next to each other, this is kind of all there is inside this bit of fast.ai, ..? other than LM RNN. Here's get language model, here is get classifier.  `get_language_model` creates an RNN encoder, and then creates a sequential model that sticks on top of that a linear decoder.  Classifier creates an RNN encoder and then a sequential model that sticks on top of that a pooling linear classifier 
- We'll see what these differences are in a moment, but you get the basic idea.  They are basically doing pretty much the same thing.  They've got this head, and they are sticking on a simple linear layer on top. So, that's worth digging in a little deeper and seeing what's going on here
- Rachel:  There was a question earlier about whether any of this translates to other languages
- JH: yes, this whole thing works in any language you like.  
- Rachel: Obviously, would you have to retrain your language model on a corpus from that language?
- JH: absolutely.  So, the wiki text 103 pre-trained language model knows English, right? You could use it maybe as a pre-trained start for a French or German model, start by retraining the embedding layer from scratch... might be helpful.  Chinese, maybe not so much.  But, given that a language model can be trained from any unlabeled documents at all, you'll never have to do that, right? Because, every, almost every language model in the world has, you know, plenty of documents.  You can grab newspapers, webpages, parliamentary records, whatever.  As long as you've got a few thousand documents showing somewhat normal usage of that language, you can create a language model.  So, I know some of our students, one of our students (his name I'll have to look up during the week, very embarrassing) tried this approach for Thai, he said the first model he built easily beat the first previous state-of-the-art Thai classifier.  For those of you that are international fellows, this is an easy way for you to kind of whip out a paper in which you either create the first every classifer in your language or beat everybody else's classifier in your language, and then you can tell them that you've been a student of deep learning for 6 months and piss off all the academics in your country. 

### `01:16:45`
- file:  https://github.com/fastai/fastai/blob/master/fastai/lm_rnn.py
- Here is our RNN encoder.  It's just a standard `nn.Module`.  Most of the text in it is just documentation.  As you can see, it looks like there is more going on in there than there actually is but really all there is we create an embedding layer, we create an LSTM for each layer that has been asked for.  And that's it, everything else in there is **dropout**, right?
- Basically all of the interesting stuff in the **AWD-LSTM paper** is all of the places you can put **dropout**
- And then the `def forward` is basically the same thing.  It's calling the embedding layer, add some dropout, go through each layer, call that RNN layer, append it to our list of outputs, add dropout, that's about it. It's really pretty straightforward

### To remind yourself what all the dropout types, are read...
- and the paper you want to be reading as I have been mentioned is the [Regularizing and Optimizing LSTM Language Models](https://arxiv.org/abs/1708.02182)
- it's well-written and pretty accessible and entirely implemented within fastai as well.  So you can see all of the code for that paper, and a lot of the code actually is, shamelessly plagiarized with Steven's permission from his excellent github repo (https://github.com/salesforce/awd-lstm-lm) in the process of which I fixed up some of his bugs as well, and I even told him about them.
- So, I'm talking increasingly about, **please read the papers**, so here's the paper.  Pleaser read this paper, and it refers to other papers, so for things like "why is it that the encoder weight and the decoder weight are the same?".  It's because there's this thing called **tie weights**.  This is inside that `get_language_model`.  There's a thing called `tie_weights=True` that defaults to True and if it's true, then we actually tie, we literally use the same weight matrix for the encoder and the decoder.
- So, they're literally pointing to the same block of memory if you like, and so why is that, what's the result of it, that's one of the citations in Steven's paper, which is also a well-written paper.  You can go and look it up and learn about weight ? time. There's a lot of cool stuff in it.  
- So we have basically a standard RNN, the only way in which it is not standard is it has lots more types of dropouts in it, and then a sequential model on top of that we stick a linear decoder which is literally half a screen of code.  It's got a single linear layer. We initialize the weights to some range.  We add some dropout, and that's it.  So, it's a linear layer of dropout.  So, we've got:
  - an RNN
  - on top of that stick a linear layer with dropout
  - and we're finished
  - so that's the language model
- So, what dropout you choose matters a lot, and through a lot of experimentation, I found a bunch of dropouts and you can see here we've got... each of these corresponds to a particular argument, a bunch of dropouts that tends to work pretty well for language models
- But, if you have **less data** for your language model, you will need **more dropout**.
- If you have **more data** you can benefit from less dropout.  You don't want to **regularize** more than you have to.  It makes sense, right?  Rather than having to tune everyone of these 5 things, right, my claim is they're already pretty good ratios to each other, so just tune this number:  **0.7**.  I just multiply it all by something
- So, there is really just **one number** you have to **tune**.  If you are **overfitting**, you will need to **increase this number**, if you're **underfitting**, you'll need to **decrease** this number. Cause other than that, these ratios seem pretty good.
```python
drops = np.array([0.25, 0.1, 0.2, 0.02, 0.15])*0.7
```
> in imdb notebook
```python
learnerlearner==  mdmd..get_modelget_mode (opt_fn, em_sz, nh, nl, 
    dropouti=drops[0], dropout=drops[1], wdrop=drops[2], dropoute=drops[3], dropouth=drops[4])

learner.metrics = [accuracy]
learner.freeze_to(-1)
```

### `01:21:45` Measuring Accuracy
- One important idea which may seem pretty minor, but again, it's incredibly controversial, is that we should measure accuracy when we look at a language model `learner.metrics = [accuracy]`
- Normally [in] language models, we look at this loss value, "val_loss", which is just cross entropy loss, but specifically we nearly always take e to the power of that, which the NLP community calls the **perplexity**.  **Perplexity is just e to the power cross-entropy**
- There's a lot of problems with comparing things based on cross-entropy loss. I'm not sure I've got time to go into it in detail now, but the basic problem is that it's kind of like that thing we learned about focal loss... cross-entropy loss, if you're right... it wants you to be really confident that you're right.  It really penalizes a model that doesn't kind of say I'm so sure this is wrong, it's wrong.  Whereas **accuracy** doesn't care at all about how confident you are, it cares about whether you're right.  And this is much more often the thing which you care about in real life. 
- This accuracy is how many... how often do we guess the next word correctly?  I just find that a much more stable number to keep track of.  So, that's a simple little thing that I do

### `01:23:10` 
- so we train for a while and we get down to a 3.9 cross-entropy validation loss, and if you go to e^3.9, `math.exp(3.9)` = 49.402
- and to give you a sense of what's happened with language models, if you look at academic papers from about 18 months ago, you'll see them talking about perplexities, state-of-the-art perplexities of over 100.  The rate at which our ability to kind of understand language and I think measuring language model accuracy, or perplexity is not a terrible proxy for understanding language.  If I can guess what you're going to say next, [then] I pretty much need to understand language pretty well and also the kind of things you might talk about pretty well.  
- These numbers (100 ---> 49.402) have come down so much!  It's been amazing...NLP in the last 12 to 18 months.  And it's going to come down a lot more.  It really feels like 2011-2012 computer vision.  We are just [now] starting to understand transfer learning and fine-tuning and these basic models are getting so much better.  So, everything you thought about what NLP can and can't do, is very rapidly going out of date.  There's still lots of stuff NLP is not good at, to be clear, just like in 2012 there was lots of stuff computer vision wasn't good at.  But, it's changing incredibly rapidly, and now is a very, very good time to be getting very, very good at NLP.  Or starting startups based on NLP because there's a whole bunch of stuff which computers were absolutely shit at 2 years ago.  And now, not quite as good as people, and next year they will be much better than people.
- `01:25:25` Rachel:  2 questions.  1: What is your ratio of paper reading vs coding in a week?
- JH: Gosh, what do you think, Rachel? You see me.  I mean, it's a lot more coding, right? 
- RT:  It's a lot more coding.  I also feel like it really varies from week to week.  I feel like they're...
- JH: With that bounding box stuff, there were all these papers and no map through them.  So, I didn't even know which one to read first, and then I'd read the citations and didn't understand any of them, and so there were a few weeks of just kind of reading papers before I even know what to start coding.  That's unusal though... most of the time I'm... I don't know... anytime I start reading a paper, I'm always convinced I'm not smart enough to understand it, *always*, regardless of the paper, and somehow eventually *I do*, but yeah, I try to spend as much time as I can coding.
- RT: And the second question: is your dropout rate the same through the training or do you adjust it and the weights accordingly? 
- JH: I just [want] to say one more thing about the last bit...which is... very often, the vast majority [of time] nearly always, after I've read a paper, even after I've read the bit that says "this is the problem I'm trying to solve", I'll kind of stop there and try to implement something that I think might solve that problem and then I'll go back and read the paper and I read little bits about like all these bits about how I solved these problem bits, and I'll be like "oh, that's a good idea", and then I'll try to implement those.  And so, that's why, for example, I didn't actually implement **SSD**, you know like my custom head is not the same as their head.  It's because I kind of read the gist of it, and then I try to create something best as I could and then go back to the papers and try to see why.  And so by the time I got to the focal loss paper, Rachel will tell you I was driving myself crazy with like how come I can't find small objects, how come it's always predicting background.  And I read the focal loss paper, and I was like "oh! that's why!"  It's so much better when you deeply understand the problem they're trying to solve and I do find the vast majority of the time by the time I read that bit of the paper, which is like solving a problem, I'm then like, yeah, but these 3 ideas I came up with *they didn't try*.  And you suddenly realize you've got new ideas.  Whereas if you *just implement* the paper, you know mindlessly, you tend not to have these insights about better ways to do it.
- `01:28:10` JH:  **Varying dropout** is interesting and there are some recent papers actually that suggest gradually changing dropout and it was either a good idea to gradually change make it smaller or to gradually make it bigger.  I'm not sure which [laughing].  Let's try... maybe one of us can try and find it during the week.  I haven't seen it widely used.  I tried it a little bit with the most recent paper I wrote... and I had some good results.  I *think I was* gradually making it smaller, but can't remember.
- RT:  And the next question is, am I correct in thinking that this language model was built on word embeddings?  Would it be valuable to try this with phrase or sentence embeddings?  I ask this because I saw from Google the other day **universal sentence encoder**.
- JH:  This is much better than that.  Do you see what I mean?  This is not just an embedding of a sentence.  This is an entire model, right?  An embedding, by definition, is a fixed thing.
- RT: Oh, I think they're asking, they're saying that this language model.. well the first question is, Is this language model built on word embeddings?
- JH: Right, but it's not saying, a sentence or phrase embedding is always a model that creates that, right? And we've got a model that's like trying to understand language, it's not just a phrase, it's not just a sentence, it's a document in the end.  It's not just an embedding, we're training through the whole thing.  This has been a huge problem with NLP for years now, is this attachment they have with embeddings.  And so, even the paper that the community has been most excited about recently from AI2, the Allen Institute, called ELMo.  They found much better results across lots of models.  Again, it was an embedding.  They took a fixed model and created a fixed set of numbers which they then fed into a model.  But, in computer vision, we've known for years that that approach of having a fixed set of features -- they're called **hyper-columns** in computer vision.  People stopped using them 3 or 4 years ago because fine-tuning the entire model **works much better!**  For those of you that have spent quite a lot of time with NLP and not much time with computer vision, you're going to have to start re-learning.  All that stuff you have been told about this idea that there are these things called embeddings and you learn them ahead of time and then you apply these things whether it be word-level or phrase level or whatever level, don't do that, all right?  You want to actually create a pre-trained model and fine-tune it and to it, you'll see some specific results.
- `01:31:15` JH:  more...? [questions]
- RT:  So, more coming up, as you answer the existing ones.  For **using accuracy instead of perplexity** as a metric for the model, could we work that into the loss function, rather than just use it as a metric? 
- JH:  No, you never want to do that whether it be computer vision or NLP or whatever, it's too bumpy, right.  So, cross-entropy is fine as a loss function.  And I'm not saying "instead of", but "in addition to".  You know, I think it's good to look at the accuracy and to look at the cross-entropy, but for your loss function you need something nice and smooth.  Accuracy doesn't work very well.  

#### ELMo (https://allennlp.org/elmo)
ELMo is a deep contextualized word representation that models both (1) complex characteristics of word use (e.g., syntax and semantics), and (2) how these uses vary across linguistic contexts (i.e., to model polysemy). These word vectors are learned functions of the internal states of a deep bidirectional language model (biLM), which is pre-trained on a large text corpus. They can be easily added to existing models and significantly improve the state of the art across a broad range of challenging NLP problems, including question answering, textual entailment and sentiment analysis

### `01:31:55`
- you'll see there's two different versions of save: there's `save` and `save_encoder`.  `save` saves the whole model, as per usual.  `save_encoder` saves just `rnn_enc` part (not the LinearDecoder part, the bit that makes it into a language model, we don't care about in the classifier)
- that's why we save 2 different models here
```python
learner.save('lm1')
learner.save_encoder('lm1_end')
learner.sched.plot_loss()
```

## Classifier Tokens `01:32:30`
- so, let's now create the classifier.  And I'm going to go through this bit pretty quickly, because it's the same.  But, when you go back during the week and *look at the code*, convince yourself that it is the same.
  - we do get all all, pd reads the csv again, chunksize again, get all again, save those tokens again
  - we don't create a new `itos` vocabulary, we obviously want to use the same vocabulary again that we used in the language model, okay?  Because we're about to reload the same encoder
  - same `defaultdict`, same way of creating our numericalized list, which as per before, we can save... so that's all the same
  - later one, we can reload those rather than having to rebuild them
  - all of our hyperparameters are the same; well, not all of them.  The construction of the model hyperparameters are the same.  We can change the dropout
  - optimizer function 
  - **pick a batch size as big as you can that doesn't run out of memory**
- These bits are interesting.  There is some fun stuff going on here.
- The basic idea here is that for the classifier, we do really want to look at one, you know, document.  We need to say, is this document positive or negative? And so, we do want to shuffle the documents, right?  Because we like to shuffle things.  But, **those documents are different lengths**, and so if we stick them all into one batch, and this is a handy thing that fastai does for you, you can stick things of different lengths into a batch, and **it will automatically pad them** so you don't have to worry about that.
- *But*, if they are wildly different lengths, then you're going to be wasting a lot of computation time.  There might be one thing there that is 2000 words and everything else is 50 words long.  And that means you end up with a 2005 wide tensor. --> That's pretty annoying.
- So, James Bradbury, who is actually one of Steven Merity's colleagues, and the guy who came up with TorchText, came up with a neat idea, which was: let's sort the dataset by length'ish, right?  So, kind of make it so the first things in the list on the whole shorter than the things at the end, but a little bit random as well.
```python
trn_ds = TextDataset(trn_clas, trn_labels)
val_ds = TextDataset(val_clas, val_labels)
trn_samp = SortishSampler(trn_clas, key=lambda x: len(trn_clas[x]), bs=bs//2)
val_samp = SortSampler(val_clas, key=lambda x: len(val_clas[x]))
trn_dl = DataLoader(trn_ds, bs//2, transpose=True, num_workers=1, pad_idx=1, sampler=trn_samp)
val_dl = DataLoader(val_ds, bs, transpose=True, num_workers=1, pad_idx=1, sampler=val_samp)
md = ModelData(PATH, trn_dl, val_dl)
```
- And, so I'll show you how I implemented that.  The first thing we need is a dataset.  We have a dataset passing in the documents and their labels. And, so here is a text dataset and it inherits from dataset (`class Dataset(object)`).  Here is dataset from Torch, PyTorch.  And actually, dataset doesn't do anything at all.  It says, you need to `__getitem__`; if you don't have one, you will get an error.  You need a length, `__len__`; if you don't have one, you will get an error.
- So, this is an **abstract class**.  
```python
class Dataset(object):
  def __getitem__(self, index): raise NotImplementedError
  def __len__(self): raise NotImplementedError
  def __add__(self, other): return ConcatDataset([self, other])
```
- So, we're going to pass in our `X`, pass in our `y`, and `__getitem__` is going to grab the `X` and grab the `y` and return them.
- It couldn't be much simpler, right?  Optionally, they could reverse it.  Optionally, it could stick a end of stream at the end.  Optionally, it could stick a stream at the beginning.  We're not doing any of those things, so literally all we're doing is putting in an `X`, putting in a `y`, and then grab an item, we're returning the `X` and `y` as a tuple
- and the length is however long the `X` array is
- that's all the `Dataset` is, something with a length that you can index  
[text.py](https://github.com/fastai/fastai/blob/3f2079f7bc07ef84a750f6417f68b7b9fdc9525a/fastai/text.py)  
```python
class TextDataset(Dataset):
    def __init__(self, x, y, backwards=False, sos=None, eos=None):
        self.x,self.y,self.backwards,self.sos,self.eos = x,y,backwards,sos,eos

    def __getitem__(self, idx):
        x = self.x[idx]
        if self.backwards: x = list(reversed(x))
        if self.eos is not None: x = x + [self.eos]
        if self.sos is not None: x = [self.sos]+x
        return np.array(x),self.y[idx]

    def __len__(self): return len(self.x)
```
- `01:36:27` so to turn it into a data loader, you simple pass the dataset to the DataLoader constructor and it's now going to go ahead and give you a batch of that at a time.  Normally, you can say `shuffle=True` or `shuffle=False`.  It will decide whether to randomize it for you.  In this case, though, we're actually going to pass in a sampler parameter (`sampler=trn_samp`).  Sampler is a class we're going to define that tells the data loader how to shuffle.
- For the validation set, `val_samp`, we're going to define something that actually just sorts it.  It just deterministically sorts it, so all the shortest documents will be at the start, all the longest documents will be at the end, and that's going to minimize the amount of padding.
- For the training sampler, `trn_samp = SortishSampler(trn_clas, key=lambda, x: len(trn_clas[x]), bs=bs//2)`:
  - we're going to create this thing I call a "Sortish Sampler" which also sorts'ish, right?  So, this is where I really like PyTorch is that they came up with this idea for an API for their data loader where we can like hook in new classes to make it behave in different ways.  So here's a sort sampler, `SortSampler` which is simply something, which again it has a length which is the length of the data source (`return len(self.data_source)`).  
  - And it has an iterator which is simply an iterator which goes through the data source, sorted by length.  Well, the key, and I pass in as the key a lambda function which returns the length
- And so, for the `SortishSampler`, I won't go through the details, but basically it does the same thing with a little bit of randomness.  It's a really beautiful, you know, just another of these beautiful little design things in PyTorch that I discovered I could take Jame Bradbury's ideas, which he had written a whole new set of classes around, and I could actually just use the in-built hooks inside PyTorch.
- You will notice, data loader, it's not actually PyTorch's data loader, it's actually fastai's data loader.  But, it's basically almost entirely plagiarized from PyTorch but customized in some ways to make it faster, mainly by using multi-threading, instead of multi-processing.  
- in [text.py](https://github.com/fastai/fastai/blob/3f2079f7bc07ef84a750f6417f68b7b9fdc9525a/fastai/text.py)
```python
class SortSampler(Sampler):
    def __init__(self, data_source, key): self.data_source,self.key = data_source,key
    def __len__(self): return len(self.data_source)
    def __iter__(self):
        return iter(sorted(range(len(self.data_source)), key=self.key, reverse=True))
```
- JH:  Yes, Rachel.
- RT:  Does the pre-trained LSTM depth bptt need to match with the new one we are training?
- JH:  No, the bptt doesn't need to match at all.  That's just, how many things do we look at, at a time? It's got nothing to do with the architecture.
- So, now we can call that function we saw before, `get_rnn_classifer`, it's going to create exactly the same encoder, more or less. And we're going to pass in the same architectural details as before.  But, this time, we can.. the head that we add on, you've got a few more things you can do.  One is you can add more than one hidden layer, so this `layers` here says:
  - this is what the input to my classifier section, my head, is going to be: `em_sz*3`
  - this is the output of the first layer: `50`
  - this is the output of the second layer: `c`
  - and you can add as many as you like, so you can basically create a little multi-layer neural net classifier at the end 
- And, ditto, these are the dropouts: `drops=[dps[4], 0.1]`
  - to go after each of these layers
- And, here are all of the **AWD-LSTM dropouts**
  - `dropouti=dps[0], wdrop=dps[1], dropoute=dps[2], dropouth=dps[3]`
  - which we're going to basically plagiarize that idea for our classifier
```python
mm  ==  get_rnn_classifierget_rnn_ (bptt, 20*70, c, vs, emb_sz=em_sz, n_hid=nh, n_layers=nl, pad_token=1,
          layers=[em_sz*3, 50, c], drops=[dps[4], 0.1],
          dropouti=dps[0], wdrop=dps[1], dropoute=dps[2], dropouth=dps[3])
``` 
- We're going to use the `RNN_learner` just like before 
```python
learnlearn  ==  RNN_LearnerRNN_Lear (md, TextModel(to_gpu(m)), opt_fn=opt_fn)
learn.reg_fn = partial(seq2seq_reg, alpha=2, beta=1)
learn.clip=.25
learn.metrics = [accuracy]
```
- We're going to use **discriminative learning rates**
  - actually the second set below is the one JH used here
```python
lr=3e-3
lrm = 2.6
lrs = np.array([lr/(lrm**4), lr/(lrm**3), lr/(lrm**2), lr/lrm, lr])

lrs=np.array([([1e-4,,1e-4,,1e-4,,1e-3,,1e-2])])
```
- You can try using **weight decay** or not.  I've been trying to fiddle around a bit with that to see what happens 
```python
wd = 1e-7
wd = 0
learn.load_encoder('lm1_enc')
```

```python
learn.freeze_to(-1)
learn.lr_find(lrs/1000)
learn.sched.plot()
```
- And, so we start out just **training the last layer**, and we get **92.87% accuracy**
```python
learn.fit(lrs, 1, wds=wd, cycle_len=1, use_clr=(8,3))
```
```bash
A Jupyter Widget
epoch      trn_loss   val_loss   accuracy                      
    0      0.365457   0.185553   0.928719  

Out[82]:
[0.18555279, 0.9287188090884525]
```
- Then, we **save the model**
```python
learn.save('clas_0')
learn.load('clas_0')
```
- Then, we **unfreeze one more layer**
```python
learn.freeze_to(-2)
```
- And **we train**
```python
learn.fit(lrs, 1, wds=wd, cycle_len=1, use_clr=(8,3))
```
- And we get **93.31% accuracy**
```bash
A Jupyter Widget
epoch      trn_loss   val_loss   accuracy                      
    0      0.340473   0.17319    0.933125  

Out[87]:
[0.17319041, 0.9331253991245995]
```
- And then we **fine tune the whole thing**
```python
learn.fit(lrs, 1, wds=wd, cycle_len=14, use_clr=(32,10))
```
```bash
A Jupyter Widget
epoch      trn_loss   val_loss   accuracy                      
    0      0.337347   0.186812   0.930782  
    1      0.284065   0.318038   0.932062                      
    2      0.246721   0.156018   0.941747                      
    3      0.252745   0.157223   0.944106                      
    4      0.24023    0.159444   0.945393                      
    5      0.210046   0.202856   0.942858                      
    6      0.212139   0.149009   0.943746                      
    7      0.21163    0.186739   0.946553                      
    8      0.186233   0.1508     0.945218                      
    9      0.176225   0.150472   0.947985                      
    10     0.198024   0.146215   0.948345                      
    11     0.20324    0.189206   0.948145                      
    12     0.165159   0.151402   0.947745                      
    13     0.165997   0.146615   0.947905                      

Out[21]:
[0.14661488, 0.9479046703071374]
```
- and **after 3 epochs**, we get **94.4%** accuracy

#### `01:41:00` James Bradbury paper
- [Learned in Translation:  Contextualized Word Vectors](https://arxiv.org/abs/1708.00107)
- Here is the famous James Bradbury we are talking about
- This was kind of the main attempt, before our paper came along, at using a pre-trained model.  What they did was they used a pre-trained translation model, but they didn't fine tune the whole thing.  They just took the activations of the translation model and when they tried IMDB, they got **91.8% accuracy**, which we beat easily after only fine-tuning one layer.  Ours was **92.87% accuracy**. 
- They weren't state-of-the-art (SOTA).  Their **SOTA is 94.1%** (in the paper), which we beat after **fine-tuning the whole thing for 3 epochs, at 94.4% accuracy**.  And so, by the end, we are at **94.8%** which is obviously a huge difference because, in terms of error rate, that has gone down from 5.9.  
- And I'll tell you a simple little trick is... go back to the start of this notebook and **reverse the order of all the documents** and then re-run the whole thing.  When you get to the bit that says `wt103`, replace `fwd_wt103.h5` with **bwd (for backup)** ---> `bwd_wt103.h5`
- That's a **backward English language model** that learns to read English backwards!  So, if you redo this whole thing, put all the documents in reverse, and change `fwd` to `bwd`, you now have a second classifier which classifies things by positive or negative sentiment based on the reverse document.  If you then **take the two predictions and take the average** of them, you then basically have a **bi-directional model** that you've trained each bit separately.  That gets you to **95.4% accuracy**.  So, we've basically lowered it [the error] from 5.9% to 4.6%.  
- So this sort of **20% change in State-of-the-Art**, it's almost unheard of!  You know, it's like you have to go back to Jeffrey Hinton's ImageNet computer vision thing where they drop 30% off the SOTA.  It doesn't happen very often!  And so, you can see this idea of just **use transfer learning**.  It's ridiculously powerful but every new field thinks their new field is too special and you can't do it, right?  So, it's **a big opportunity** for all of us.

## `01:44:00` paper with Sebastian Ruder
- Abstract:  Universal Language Model Fine-tuning for Text Classification
- We turned this into a paper, and when I saw "we", I mean with this guy Sebastian Ruder.  Now, you might remember his name because in Lesson 5 I told you that I had actually shared Lesson 4 with Sebastian because I think he's an awesome researcher who I thought might like it.  I didn't know him personally at all.  And, much to my surprise, he actually watched the damn video.  I thought, what NLP researcher is going to watch some beginner's video?  *But*, he watched the whole video! And he was, like, *that's actually quite fantastic!* 
- JH:  And I said, well, thank you very much, that's awesome coming from you.  
- And he said, *hey, we should turn this into a paper.*  
- JH:  And I said, I don't write papers. I don't care about papers.  I'm not interested in papers.  That sounds really boring.
- And he said, *ok, how about I write the paper for you?* 
- JH:  And I said, You can't really write a paper about this yet, because you would have to do studies to compare it to other things.  They're called ?oblation studies to see.. which if it actually works.  There's no rigor here.  I just put in everything that came in my head and chucked it all together and it happened to work.
- And he said, *ok, what if I write all the paper and do all the oblation studies?  Then can we write the paper?*
- JH: And I said, well, it's like a whole library that I haven't documented and like, you know, I'm not going to yet, and you don't know how it's all works.
- And he said, *ok, if I write the paper and do he oblation studies, and figure out from scratch how the code works without bothering you, THEN can we write the paper?*
- JH: And I said, yeah, if you did all those things, you can then write the paper.  And he said *ok*, and then 2 days later he comes back and he says he has done a draft of the paper.  So, I share this story to say, that if you are some student in Ireland (he's a student in Ireland), and you want to do good work, don't let anybody stop you.  I did not encourage him, to say the least, right?  But in the end, he said, *look, I want to do this work, I think it's going to be good, and I'll figure it out.* And he wrote a fantastic paper, and he did the oblation studies, and he figured out how fastai works.  And, now we're planning to write another paper together.
- You've got to be a bit careful, right? Because sometimes I get messages from random people saying "I've got lots of good ideas, can we have coffee?" I don't want to... you know, I can have coffee in my office anytime, thank you.  But it's very different to say, hey, I took your ideas and I wrote a paper and I did a bunch of experiments and I figured out how your code works.  They added documentation to it. Should we submit this to a conference?  You see what I mean?  There is nothing to stop you doing amazing work and if you do amazing work that helps someone else, like in this case, ok, I'm happy that we have a paper.  I don't particularly care about papers.  But, I think it's cool that these ideas now have this rigorous study.  Let me show you what he did.
- `01:47:20` So, he took all my code. So I had already done the fastai text and stuff like that and as you've seen, it **lets us work with large corpuses.**  So, Sebastian is fantastically well-read and he said here's a paper that Yann LeCun and some guys just came out with where they tried lots of different classification datasets.  So, I'm going to try running your code on all these datasets.  And, so, these are the datsets:
  - Dataset, Type, # classes, # examples
  - TREC-6, Question, 6, 5.5K
  - IMDb, Sentiment, 2, 25K
  - Yelp-bi, Sentiment, 2, 560K
  - Yelp-full, Sentiment, 5, 650K
  - AG, Topic, 4, 120K
  - DBpedia, Topic, 14, 560K
- And so some of them had many, many hundreds of thousands of documents and they were far bigger than anything I had tried, but I thought it should work.  And so he had a few good little ideas as we went along and so you should like totally make sure you read the paper
- He said this thing you called in the lessons **differential learning rates** -- "differential" kind of means something else.  Maybe we should rename it, so we renamed it.  Now it's called **discriminative learning rate**.  So this idea that we had from Part 1 where we used different learning rates for different layers, after doing some literature research, it does seem like that hasn't been done before.  So, now it's officially a thing - discriminative learning rates.  And so all these ideas, this is something we learnt in Lesson 1, but it now has an equation with Greek and everything [audience laughs..]  So when you see an equation with Greek and everything, that doesn't necessarily mean it's more complex than anything we did in Lesson 1, because this one isn't.  Again, that idea of **unfreezing a layer** at a time also seems to never have been done before, so it's now a thing!  And it's got the clever name **gradual unfreezing**.
- So then, long promised what we're going to look at this **slanted triangular learning rates**.  So, this actually was not my idea.  Leslie N. Smith, one of my favorite researchers, who you all now know about, emailed me a while ago, and said: 
> I'm so over learning rates, I don't do that anymore. I now do a slightly different version where I have one cycle which goes up quickly at the start and then slowly down afterwards.  I often find it works better.  I've tried going back over all of my old datasets and it worked better for *all of them, every one* I tried.
- JH:  So, this is what the learning rate looks like [showing graph with upside-down V, skewed, leaning to left].  You can use it in fastai just by adding `use_clr=(32,10)` to your fit.  This first number [32] is the ratio between the highest learning rate and the lowest learning rate.  So, here this is 1/32 of that.  The second number is the ratio between the first peak and the last peak.  And, so the basic idea is if you are doing a cycle length 10, that you want the ~first cycle~, sorry the first epoch, to be the upward bit and the other 9 epochs to be the downward bit, then you would use 10.  I find that works pretty well.  That was also Leslie's suggestion, is about make a tenth of it the upward bit, and 9/10 the downard bit.  
- Since he told me about it, actually it was just maybe 2 days ago, he wrote this amazing paper:  
[A disciplined approach to neural network hyper-parameters: Part 1 -- learning rate, batch size, momentum, and weight decay](https://arxiv.org/abs/1803.09820)
- [In this paper], he describes something very slightly different to this again, but the same basic idea.  This is a **must read paper!** It's got all the kinds of ideas that fastai talks about a lot in great depth and nobody else is talking about this stuff.  It's kind of a slog.  Unfortunately, Leslie had to go away on a trip before he really had time to edit it properly, so it's a little bit slow reading, but don't let that stop you.  It's amazing.
- `01:51:20` [back to upside down V] So this triangle, this is the equation from my paper with Sebastian.  Sebastian was like, Jeremy, can you send me the math equation behind that code you wrote?  I said, no [laughs], I just wrote the code, I could not turn that into math.  So he figured out the math for it.  

## `01:51:35` Concat Pooling
- You might have noticed the first layer of our classifier was equal to embedding size times 3. Why times 3?  Times 3 (x3) because, again this seems to be something which people haven't done before, so new idea.  **Concat pooling** which is that we take the average pooled (meanpool(H)), the average pooling over the sequence of activations, the max pooling (maxpool(H)) of the sequence over the activations, and the **final set of activations** (h_T) and just concatenate them all together.  Again, this is something which we talked about in Part 1, but doesn't seem to be in the literature before.  So, it's not called concat pooling and again it's now got an equation and everything but this is the entirety of the implementation:  
  - Pool with average pool with max   
  - Concatenate those two along with the final sequence 
- so you can go through this paper and see how the fastai code implements each piece
- [Universal Language Model Fine-tuning for Text Classification](https://arxiv.org/pdf/1801.06146.pdf), Jeremy Howard & Sebastian Ruder
```text
H = {h_1, h_2, ... h_T}:
   h_c = [h_T, maxpool(H), meanpool(H)]     (4)

where [ ] is concatenation
```
- Then, to me, kind of one of the interesting pieces is the difference between RNN Encoder, which you've already seen, and **multi-batch RNN encoder**.  So what's the difference there?  The key difference is the normal RNN encoder for the language model, we could just do BPTT chunk at a time, right, no problem.  And predict the next word.  But, for the classifier, we need to do the whole document. We need to do the whole movie review before we decide whether it's positive or negative.  And the whole movie review can easily be 2,000 words long and I can't fit 2,000 words worth of gradients in my GPU memory for every single one of my ~activations~, sorry, for every single one of my *weights*, so what do I do?
- So, the idea was very simple, which is I go through my whole sequence length, one batch of `bptt` at a time:  `for i in range(0, sl, self.bptt):`
- And I call `super().forward`, so in other words, the `RNN_Encoder`, I couldn't call the usual RNN Encoder, so I grab its outputs, and then I've got this maximum sequence length parameter, `if i>(sl-self.max_seq):`, where it says, okay, if you've... as long as you're doing no more than that sequence length, then start appending it, `raw_outputs.append(r)`, to my list of outputs
- So, in other words, the thing that it sends back to this pooling is only as much, as many activations as we've asked it to keep.  And so that way, you can basically figure out how much, what is `max_seq`, do you..., can your particular GPU handle that? So, it's still using the whole document, but let's say `max_seq` is 1000 words, and your longest document length is 2000 words.  Then, it's still going through the RNN creating state for those first 1000 words.  But, it's not actually going to store the activations for the back-prop the first thousand.  It's only going to keep the last thousand, right.  So that it means that it can't back propagate the loss back to any decisions that any state that was created in the first thousand words.  Basically, that's now gone.  
- [lm_rnn.py](https://github.com/fastai/fastai/blob/87d7a489c22fc5daa38fa92683df183f7c9bbe1c/fastai/lm_rnn.py)
```python
class MultiBatchRNN(RNN_Encoder):
    def __init__(self, bptt, max_seq, *args, **kwargs):
        self.max_seq,self.bptt = max_seq,bptt
        super().__init__(*args, **kwargs)

    def concat(self, arrs):
        return [torch.cat([l[si] for l in arrs]) for si in range(len(arrs[0]))]

    def forward(self, input):
        sl,bs = input.size()
        for l in self.hidden:
            for h in l: h.data.zero_()
        raw_outputs, outputs = [],[]
        for i in range(0, sl, self.bptt):
            r, o = super().forward(input[i: min(i+self.bptt, sl)])
            if i>(sl-self.max_seq):
                raw_outputs.append(r)
                outputs.append(o)
        return self.concat(raw_outputs), self.concat(outputs)
```

- So, it's a really simple piece of code.  And honestly, when I wrote it, it was like I didn't spend much time thinking about it.  It seems so obviously the only way that this could possibly work, but again, it seems to be a new thing.  
- So, we now have **back-prop through time for text classification**.
- So you see there are lots of little pieces in this paper.  So, what was the result?  The result was, on every single dataet we tried, we got a better result than any previous academic paper for text classification.
- `01:56:30` IMDB tracked 6, AG News, DBpedia, Yelp... all different types.  And, honestly, IMDB was the only one I spent any time trying to optimize the model, so most of them we just did it like whatever came up first.  So, if we actually spent time with it, I think these [numbers] would be a lot better.
- And the things these are comparing to, most of them are, and you'll see, they're different on each table because they are optimized, you know, these are customized algorithms on the whole.  So, this is saying like one simple fine-tuning algorithm can beat these really customized algorithms.
-  `01:57:00` And so, here's the... one of the really cool things that Sebastian did was his Ablation Studies.  Which is, I was really keen, that if you're going to publish a paper, we had to say, "why does it work?".  So Sebastian went through and tried, you know, removing all of those different contributions I mentioned.  
  - So, what if we don't use **gradual freezing?** 
  - What if we don't use **discriminative learning rates?**
  - What if, instead of discriminative learning rates, we use **cosine annealing?**
  - What if we don't do any **pre-training with Wikipedia?**  
  - What if we don't do any **fine-tuning?**
- And then, the really interesting one to me was, what's the validation error rate on IMDb, *if* we only use 100 training examples? [Referring to graph in paper]: versus 200, versus 500?  and you can see, you know very interestingly, the **full version** of this approach is nearly as accurate on just a hundred training examples as, like it's still very accurate, versus for 20,000 training examples.
- Whereas if you're training from scratch on 100, it's like almost random, all right.  So, kind of like, it's what I expected.  You know, I've kind of said to Sebastian, **I really think this is most beneficial when you don't have much data, and this is like where fastai is most interested in contributing.**
- There's like small data regimes, small compute regimes and so forth.  
- And so he did these studies to check.


### `01:58:30` Trick 1:  Running Firefox on AWS/Paperspace/etc
- I want to show you a couple of tricks as to how you can run these kinds of studies
- The first trick is something which I know you are all going to find really handy.  I know you've all been annoyed when you are running something in Jupyter notebook and you lose your internet connection for long enough that it decides you've gone away and then your session disappears and you have to start it again from scratch.  So, what do you do?  There's a very simple cool thing called **VNC** where basically you can install on your AWS instance or Paperspace or whatever, **X Windows**, a lightweight window manager, a VNC server, Firefox, a terminal, and some fonts:  
```bash
sudo apt-get install xorg lxde-core tightvncserver firefox lxterminal xfonts-100dpi
```
- Chuck these lines at the end of your VNC X start-up configuration file:  
```bash
cat >> ~/.vnc/xstartup
       lxterminal &
       /usr/bin/lxsession -s LXDE &
       <ctrl-d>
```
- and then run this command:  
```bash
tightvncserver :13 -geometry 1200x900
```
- it's now running a server where if you now run for... well, let me explain.  It's now running a server where you can then run the type VNC viewer, or any VNC viewer, on your computer and you point it at your server.  But, specifically, what you do is you use SSH port forwarding to port forward port 5913 to local host 5913.  And so then, you connect to port 5913 on localhost.  It will send it off to port 5913 on your server which is the VNC port, because you said `:13` here, and it will display an X Windows desktop! And then you can click on the Linux start-like button, and you can click on Firefox and you now have Firefox! And, so you can now run, and you can see here on Firefox, it says `localhost`.  Because this Firefox is running on our AWS server!  And so, you now run Firefox, you start your thing running, and then you close your VNC viewer, remembering that Firefox is like displaying on this virtual VNC display, not in the real display.  And so, later on that day, you log back into the VNC viewer, and it pops up again, so it's like a persistent desktop.  And, it's shockingly fast!  It works really well.  So, this trick, number 1.  And, there's lots of different VNC servers and clients and whatever, but this one works fine for me, so there you go.  So you can see here `2:01:20` I connect to localhost 5913 
```bash
jhoward@jhbook-2:/mnt/c/Users/j$ ssh -L5913:localhost:5913 usf2.mine.nu
```
```bash
tightvncserver -kill :13
```

### `02:01:27` Trick 2: Create Python Scripts
- Trick number 2 is to create Python scripts and this is what we ended up doing.  So I ended up creating like a little Python script for Sebastian to kind of say, this is the basic steps you need to do and now you need to create different versions for everything else and I suggested to him that he try using this thing called **Google Fire.**
- What Google Fire does is you create a function, with shitloads of parameters, and these are all the different things that Sebastian wanted to try doing:  
  - different drop-out amounts
  - different learning rates
  - do I use pre-training or not?
  - do I use CLR or not
  - do I use discriminative learning rate or not
  - do I go backwards or not?
  - blah, blah, blah
- So you create a function, and then you add:  `if __name__=='__main__': fire.Fire(train_clas)`
  - and you do nothing else at all
  - you don't have to add any metadata, any docstrings, anything at all
  - and you then call that script and automatically you now have a command line interface 
- and that's it, that's a super fantastic easy way to run lots of different variations in a terminal and this ends up being easier if you want to do lots of variations than using a notebook because you can just like have a bash script that tries all of them and spits them all out
- [train_clas](https://github.com/fastai/fastai/blob/a7b9dc3afd39c31a415d475905e14abd90b5b895/courses/dl2/imdb_scripts/train_clas.py)
```python
def train_clas(dir_path, cuda_id, lm_id='', clas_id=None, bs=64, cl=1, backwards=False, startat=0, unfreeze=True,
               lr=0.01, dropmult=1.0, bpe=False, use_clr=True,
               use_regular_schedule=False, use_discriminative=True, last=False, chain_thaw=False,
               from_scratch=False, train_file_id=''):
```
- `2:02:50` You'll find inside the dl2 course directory there's now something called [imdb_scripts](https://github.com/fastai/fastai/tree/a7b9dc3afd39c31a415d475905e14abd90b5b895/courses/dl2/imdb_scripts) 
- I've put there all of the scripts that Sebastian and I used.
- You'll see because:
  - we needed to tokenize every single dataset... 
  - we had to turn every dataset...to numericalize every dataset
  - we had to train a language model on every dataset 
  - we had to train a classifier on every dataset
  - and we had to do all of those things in a variety of different ways 
- To compare them, we had a script for all those things
- You can check out and see all of the scripts that we used 
- When you're doing a lot of scripts and stuff, they've got different code all over the place.  Eventually, it might get frustrating that you want to, you know you don't want to symlink your fastai library again and again.  But, you probably don't want to pip install it because that version tends to be a little bit old.  We move so fast you want to use the current version in git.
- If you say `pip install -e .` from the [location] `~/git/fastai` , from the fastai repo base, it does something quite neat.  Which is, it basically creates a symlink to the fastai library, in your git installation right here, inside your site packages directory.  Your site packages directory is your main Python library.
- If you do this, you can then access fastai from anywhere, but every time you do `git pull`, you've got the most recent version. 
- One downside of this is that it installs any updated versions of packages from `pip` which can kind of confuse `conda` a little bit.  So, another alternative here is just to symlink the fastai library to your site packages library.  That works just as well.  Then you can use fastai again, from anywhere and it's quite handy when you want to kind of run scripts that use fastai from different directories on your system

### `02:05:05` [sentencepiece](https://pypi.org/project/sentencepiece/)
- [sentencepiece](https://github.com/google/sentencepiece) (github repo)
- One more thing before we go which is something you can try if you like.  You don't have to tokenize words
- Instead of tokenizing words, you can tokenize what are called sub word units.  
  - for example, "unsupervised" could be tokenized as "un supervised" 
  - "tokenizer" could be tokenized as "token izer"
- And then you could do the same thing.  The language model that works on sub-word units, a classifier that works on sub-word units, etc.
```text
SentencePiece is an unsupervised text tokenizer and detokenizer mainly for Neural Network-based text generation systems where the vocabulary size is predetermined prior to the neural model training. SentencePiece implements subword units (e.g., byte-pair-encoding (BPE) [Sennrich et al.]) and unigram language model [Kudo.]) with the extension of direct training from raw sentences. SentencePiece allows us to make a purely end-to-end system that does not depend on language-specific pre/postprocessing.
```
- So, how well does that work?  I started playing with it and with not too much playing, I was getting classification results that were nearly as good as using word-level tokenization.  Not quite as good, but nearly as good. 
- I suspect with more careful thinking and playing around maybe I could have gotten as good or better, but even if I couldn't, if you create a sub-word unit wikitext model, then IMDb model language model, and then classifier, forwards and backwards for sub-word units, and then ensemble it with the forwards and backwards word-level ones, *you should* be able to beat us, right.
- So, here's an approach you may be able to beat our state-of-the-art result.  
- Google has, as Sebastian told me about this particular project, is great.  Google has a project called Sentence Piece, which actually uses a neural net to figure out the optimal splitting up of words.  And so you end up with a vocabulary of sub-word units.  
- In my playing around, I found that creating a vocabulary of about 30K sub-word units seems to be about optimal. 
- So, if you're interested, there is something you can try.  It's a bit of a pain to install.  It's C++.  It doesn't have great error messages but it will work.  There is a Python library for it and if anybody tries this, I am happy to help them get it working.  There's been little, if any experiments with ensembling sub-word and word-level stuff classification, and I do think it should be best approach.
- All right.  Thanks everybody.  Have a great week and see you next Monday.  [applause]








