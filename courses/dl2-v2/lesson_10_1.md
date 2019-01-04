# Lesson 10:  NLP Classification and Translation
(02-Apr-2018, live)  

### Part 2 / Lesson 10
- [Wiki Lesson 10](http://forums.fast.ai/t/part-2-lesson-10-wiki/14364)
- [Video Lesson 10](https://www.youtube.com/watch?v=h5Tz7gZT9Fo&feature=youtu.be) 
  - video length:  2:07:55
- http://course.fast.ai/lessons/lesson10.html
- Notebook:  
   * [imdb.ipynb](https://github.com/fastai/fastai/blob/master/courses/dl2/imdb.ipynb)

---

## Review of Last Week
- if you are finding the material difficult, that's ok
- there are quite a few in-class fastai students who are doing this full-time and some of them are struggling with this material
- one of the reasons Jeremy had that content early on is we've got something to **cogitate** and think about and gradually work towards so by Lesson 14, you'll get a second crack at it

## Reading Papers (Research Journal Publications)
- not enough students are reading the papers!  The papers are the real ground truth
- a lot of people aren't reading the papers because they think they can't read the papers, but YOU ARE, because you are here

### Debugger
```python
pdb.set_trace()
```
### `00:14:20` NLP
- we've seen the idea of taking a pre-trained model, whip off some stuff from the top, replace it with new stuff, get it to do something similar
- we've dived a little bit deeper with that; with ConvLearner.pretrained, it had a standard way of sticking stuff on the top which does a particular thing, which was classification
- and then we learned, we can stick any PyTorch module at the end and have it do anything we like with a custom head
- suddenly you discover there are some interesting things we can do
- `00:15:35` YangLu said, what if we did a different kind of custom head? 
    - take original picture, rotate it, and then make our dependent variable the opposite of that rotation
    - and see if it can learn to un-rotate it
    - and this is a super useful thing, Google photos has an option to automatically rotate photos for you
    - as Yang Lu showed here, you could build that network right now by doing exactly the same as our previous lesson, but your custom head is one that spits out a single number, which is how much to rotate by and your dataset has a dependent variable which is how much did you rotate by?  And your dataset has a dependent variable, which is how much did you rotate by?
- so, you suddenly realize the idea of a backbone with a custom head, you can do almost anything you can think about
- Next, let's think about, how does that apply to NLP?


### Match the Equations to the Code
- [List of Mathematical Symbols](https://en.wikipedia.org/wiki/List_of_mathematical_symbols)

### Re-create things (graphs) you see in the papers
- "Try to recreate this chart, and make sure you understand what it's saying and why it matters"

### `00:16:40` We've moved from *torchtext* to *fastai.text*
#### NLP
- in the next lesson, we're going to go further and say, if NLP and computer vision lets you do the same basic ideas, then how do we combine the two? and we're going to learn about a model that we'll learn to find word structures from images, OR images from word structures, OR images from images!
- and that will form the basis, if you wanted to go further, of doing things from an image to a sentence, "image captioning", or going from a sentence to an image, which we will start to do "phrased image"
- so, from there, we'll go deeper into computer vision to think about what other kinds of things we can do with this idea of pre-trained network + custom head
- `00:17:45` we'll look at various kinds of image enhancement, like:
  - increasing the resolution of a low-res photo to guess what was missing
  - or adding artistic filters on top of photos
  - or changing photos of horses into photos of zebras and stuff like that 
- and finally, that will bring us all the way back to bounding boxes again
- to get there, we will first learn about segmentation, which is not just figuring out where the bounding box is, but figuring out what every single pixel in an image is part of
  - is this pixel part of a person, a car? 
- and then we will use that idea, idea of **U-Net** and it turns out this idea from unet, we can apply the idea of bounding boxes, which are called **feature pyramids** (everything has to have a different name in every slightly different area)
- and we'll use that to get very good results with bounding boxes
- that's our path from here, it all builds on each other, but take us into lots of different areas

### 4 Reasons Why..  *torchtext* to *fastai.text*
- no parallel processing
- hard to do simple things (like multi-label classification)
- no obvious way to save intermediate calculations
- somewhat convoluted API

#### `00:18:55` Section notes
- For NLP in Part 1, we relied on a pretty great library called torchtext, but as pretty great as it was, JH has 
found problems with it that are limiting, too problematice to keep using it
- as a lot of students complained on the forums, it's very slow because it's not doing parallel processing and doesn't remember what it did previously, so reruns it, does it all over from scratch
- and it's hard to do fairly simple things, a lot of students tried to the Kaggle Toxic Comment Challenge, which was a multi-label problem, and doing that with torchtext, JH eventually got it working, but it took him a week to hack away

### fastai.text
- `00:19:50` to fix all these problems, JH has created a new library called *fastai.text*
- fastai.text is a replacement for the combination of torchtext and fastai.nlp.  **Don't use fastai.nlp anymore** --> that is obsolete
  - it's slower, more confusing, less good in every way, but lot of overlaps, 
  - a lot of the classes & functions have the same names, that is intentional
- this is the non-torchtext version

## IMDb data `00:20:30`
- Notebook:  [imdb.ipynb](https://github.com/fastai/fastai/blob/master/courses/dl2/imdb.ipynb)
- we'll work with the IMDb again, for those of you who have forgotten, go check out lesson 4, IMDb reviews
- this is a dataset of movie reviews, and you remember we used it to find out whether we might enjoy ?somebegeddon
- we are going to use the same dataset
- by default, it calls itself "aclImDB":  `data/aclImdb/')`  --> this is the raw dataset that you can download
- as you can see (there is no torchtext, and I'm not using fastai.nlp):
```python
from fastai.text import *
import html
```
- JH is using `Path` lib, as usual; we'll learn about the tags later
```python
BOS = 'xbos' # beginning-of-sentence tag
FLD = 'xfld' # data field tag

PATH = Path('data/aclImdb/')
```
- you'll remember, the basic path for **NLP** is we have to take sentences and turn them into numbers, and there are a couple of steps to get there
- at the moment, somewhat intentionally, fastai.text doesn't provide that many helper functions, it's really designed more to let you handle things in a fairly flexible way  
- as you can see here, I wrote something called `get_texts` which goes through each thing in "classes".  These are the 3 things they have in classes: negative, positive and unsupervised (stuff they haven't gotten around to labeling yet)
- JH goes thru each one of those classes, find every file in that folder with that name, and I open it up and read it and chuck it into the end of the array `texts`
- as you can see, with path lib, it's easy to grab stuff and read it in
- and then the label is whatever class I'm up to
- JH will go ahead and do it for the **train** bit and the **test** bit
```python
CLASSES = ['neg', 'pos', 'unsup']

def get_texts(path):
    texts,labels = [],[]
    for idx,label in enumerate(CLASSES):
        for fname in (path/label).glob('*.*'):
            texts.append(fname.open('r', encoding='utf-8').read())
            labels.append(idx)
    return np.array(texts),np.array(labels)

trn_texts,trn_labels = get_texts(PATH/'train')
val_texts,val_labels = get_texts(PATH/'test')
```
- so there are 75000 in train, 25000 in test (50,000 of the train are unsupervised); we won't actually be able to use them when we get to the classification piece
- JH actually finds this easier than the torchtext approach of having lots of layers and wrappers and stuff, because at the end, reading text files is not that hard

#### Sorting `00:23:20`
- one thing that is always a good idea is to sort things randomly
- it's useful to know this simple trick for sorting things randomly, particularly when you've got multiple things that you have to sort the same way
- in this case, you've got labels and texts
- `np.random.permutation` if you give it an integer, it gives you a random list from 0 up to the number you give it (not including the number), in some random order, and so you can pass that in as an indexer.
```python
np.random.seed(42)
trn_idx = np.random.permutation(len(trn_texts))
val_idx = np.random.permutation(len(val_texts))
```
- to give you a list that is sorted in that random order
- in this case, it will sort `trn_texts` and `trn_labels` in the same random way
- that's a useful little idiom to use
```python
trn_texts = trn_texts[trn_idx]
val_texts = val_texts[val_idx]

trn_labels = trn_labels[trn_idx]
val_labels = val_labels[val_idx]
```
- now, I've got my texts and labels sorted
- I can go ahead and create a dataframe from them
- Why am I doing this?  Because there is a somewhat standard approach starting to appear for text classication datasets, which is to have your training set as a csv file, 
  - with the labels first, and the text of the NLP document second   `col_names = ['labels', 'text']`
  - with a train.csv and a test.csv
  - and a file called `classes.txt` which just lists the classes
  - it is somewhat standard
  - in a reasonably recent academic paper, Yann LeCun and a team of researchers looked at quite a few datasets, and they used this format for all of them 
  - that's what JH has started using as well for his recent papers
```python
df_trndf_trn  ==  pdpd..DataFrameDataFrame({'text':trn_texts, 'labels':trn_labels}, columns=col_names)
df_val = pd.DataFrame({'text':val_texts, 'labels':val_labels}, columns=col_names)
```
- **if you put your data into this format, you'll find that the whole notebook will work on your dataset, every time!**
- so, rather than having a thousand different classes or formats or readers and writers or whatever; let's pick a standard format and your job is to put your data into that format which is a csv file
- **the csv files have no header, by default**
- you'll notice at the start, there are two different paths:
  - one was the classification path --> contains info we'll use to create the sentiment analysis model
  - other was the language model path (lm = language model) --> info to create language model
- when we create the `CLAS_PATH/'train.csv'`, we remove everything that has a label of 2 (which is "unsupervised")  
```python
CLAS_PATH=Path('data/imdb_clas/')
CLAS_PATH.mkdir(exist_ok=True)

LM_PATH=Path('data/imdb_lm/')
LM_PATH.mkdir(exist_ok=True)
```
- so that means the data we use will have 25K positive and 25K negative 
- and the second difference is the labels we will use for the classification part are the actual labels
- but, for the language model, there are no labels, so we just use a bunch of zero's; that just makes it easier so we can use a consistent dataframe, or csv format
- now, the language model, we can create our own validation set, so you have probably come across by now `sklearn.model_selection.train_test_split(np.concatenate([trn_texts, val_texts]), test_size = 0.1)` which is a simple little function which grabs a dataset and randomly splits it into a training set and validation set, according to whatever proportion is specified by `test_size=0.1` (10%)
- in this case, I concatenate my classification training and validation together, so it is 100K together, split it by 10%, now I've got 90K training, 10K validation for my language model, so go ahead and save that
```python
trn_texts,val_texts = sklearn.model_selection.train_test_split(
    np.concatenate([trn_texts,val_texts]), test_size=0.1)
```
- that's my basic setup, get my data in a standard format for my language model and my classifier

### Language Model Tokens `00:28:00`
- the next thing we do is tokenization
- tokenization means, for a document, we've got a big long string, and we want to turn it into a **list of tokens**, which are *kind of*, a list of words, but not quite.  For example "don't", we want it to be:  "do n't" and a period / "<full stop>" to also be a token, and so forth
- so, tokenization is something we passed off to a terrific library called [spacy](https://spacy.io), partly terrific because an Australian wrote it, [Matthew Honnibal](https://twitter.com/honnibal), and partly terrific because it is good at what it does
- we put some work on top of spacy, but the vast majority of work done has been by spacy
- before we pass it to spacy, JH has written this simple **fixup** function which... each time JH opens a dataset, and has looked at *many*, everyone had different weird things that needed to be replaced.
- Here are all the ones JH has come up with so far:
```python
re1 = re.compile(r'  +')

def fixup(x):
    x = x.replace('#39;', "'").replace('amp;', '&').replace('#146;', "'").replace(
        'nbsp;', ' ').replace('#36;', '$').replace('\\n', "\n").replace('quot;', "'").replace(
        '<br />', "\n").replace('\\"', '"').replace('<unk>','u_n').replace(' @.@ ','.').replace(
        ' @-@ ','-').replace('\\', ' \\ ')
    return re1.sub(' ', html.unescape(x))
```
- hopefully, this will help you out as well
- `html_escape` all the entities
- and then there's a bunch more things that get replaced
- have a look at the result of the text you are using and make sure there are not more weird tokens in there, it's amazing how many weird things people do to text
- `get_texts` function

```python
def get_texts(df, n_lbls=1):
    labels = df.iloc[:,range(n_lbls)].values.astype(np.int64)
    texts = f'\n{BOS} {FLD} 1 ' + df[n_lbls].astype(str)
    for i in range(n_lbls+1, len(df.columns)): texts += f' {FLD} {i-n_lbls} ' + df[i].astype(str)
    texts = list(texts.apply(fixup).values)

    tok = Tokenizer().proc_all_mp(partition_by_cores(texts))
    return tok, list(labels)
```

#### 
- this function called `get_all` which will call `get_text` which calls `fixup`
- let's look thru this because there are some interesting things to point out
```python
def get_all(df, n_lbls):
    tok, labels = [], []
    for i, r in enumerate(df):
        print(i)
        tok_, labels_ = get_texts(r, n_lbls)
        tok += tok_;
        labels += labels_
    return tok, labels
```
- I'm going to use pandas to open our `train.csv` from the language model path (`LM_PATH`)
- I am passing an extra parameter you may not have seen before `chunksize` 
- Python and Pandas can both be pretty inefficient when it comes to storing and using text data
- You will see that very few people working with NLP are using large corpuses, and I think part of the reason is that traditional tools have made it difficult; you run out of memory all the time
- So, this process I am showing you today I have used on corpuses of over a billion words, successfully using this exact code
```python
df_trn = pd.read_csv(LM_PATH/'train.csv', header=None, chunksize=chunksize)
df_val = pd.read_csv(LM_PATH/'test.csv', header=None, chunksize=chunksize)
```
- and so, one of the simple tricks is to use this thing called `chunksize` with pandas
- what this means is pandas does not return a dataframe, but it returns an iterator which we can iterate thru chunks of a dataframe
- that's why JH doesn't say `tok_trn` = get_texts; instead JH calls `get_all`, which loops through the dataframe; but what it is actually doing is **looping through chunks of the dataframe**.
- each of those chunks is a dataframe representing subsets of the data
- Rachel:  when I'm working with NLP data, many times I come across data with foreign text or characters.  Is it better to discard them or keep them?
- JH:  No, definitely keep them.  And this whole process is unicode and JH has used it on Chinese text.  And it is designed to work on pretty much anything;  
- In general, most of the time it's not a good idea to **remove anything**.  Old fashioned NLP approaches tend to do things like lemmatization, and all these normalization steps to get rid of things, like lower case everything, blah, blah, blah etc.  
- But, **that's throwing away information** which you don't know ahead of time whether it's useful or not. 
- So, **don't throw away information**
- `00:32:20` so, we go through each chunck, `get_all`, each of which is a dataframe;  and we call `get_texts`.  `get_texts` is going to grab labels, make them into ints, it's going to then grab texts, and I'll point out a couple of things:
  1.  the first is that before we include the text, we have this `\n{BOS}` beginning-of-stream token, which you might remember we used way back earlier.  There's nothing special about these strings of letter.  I just figure they don't appear in normal text very often.  So, every text will start with `xbos`.  Why is that?  Because often it is really useful for your model to know when a new text is starting.  For example, if it is a language model, we are going to concatenate all the text together and so it would be very helpful for it to know when one article finishes and a new one started, so I should forget some of that context now. 
  - ? Ditto, Devo is quite often text has multiple fields, like a title, abstract and then the main document.  And so, by the same token, I've got this thing here (`get_texts`) which actually lets us have multiple fields in our csv.  
  - So this process is designed to be very flexible, and again, at the start of each one we put a special field starts here token, followed by the number of the field that's starting here, for as many fields as we have
  - then we apply fixup to it, and then most importantly, we tokenize it, and we tokenize it by doing a process or multiprocessor, or multi-processing, I should say.  
  - And, so, tokenizing tends to be pretty slow, but we've all got multiple cores on our machines now, and some of the better machines on AWS and stuff can have dozens of cores.  Here, on a university computer, we have 56 cores.
  - Spacey is not very amenable to multi-processing, but I finally figured out how to get it to work. And the good news is it's all wrapped up in one function now.  `tok = Tokenizer().proc_all_mp(partition_by_cores(texts))`
  - And so all you need to pass to that one function is a list of things to tokenize which each part of that list will be tokenized on a different core
  - and so I've also created this function called `partition_by_cores` which takes a list, and splits it into sub-lists, where the number of sub-lists is the number of cores that you have in your computer
  - so, on my machine, without multi-processing, this takes about **1.5 hours**
  - with multi-processing, it takes about **2 minutes**
  - It's a really handy thing to have.  Now that the code is here, feel free to look inside it and take advantage of it for your own stuff
  - remember, we all have multi-processors, multiple cores even on our laptops and very few things in Python take advantage of it unless you make a bit of an effort to make it work
  
#### `00:35:30` 
- so there a couple of tricks to get things working quickly and reliably
- as it runs, it prints out how it's going
- and so here's the result at the end, beginning of string token "xbos" and "xfld".  Here's the tokenized text.  You'll see that the punctuation on the whole is a separate token 
- you'll see there's a few interesting things.  one is this: "t_up mgm".  "mgm" was obviously originally capitalized.  but, the interesting this is that, normally people often lower case everything, or they leave the case as is.  Now, if you leave the case as is, then, "SCREW YOU" or "screw you" are two totally different sets of tokens that have to be learned from scratch
- or, if you lower case them all, then there's no difference at all
- so, how do you fix this so you both get the semantic impact of "I'M SHOUTING NOW" but not have every single word have to learn the shouted version vs the normal version
- and so, the idea I came up with, and I'm sure other people have done this too, is to come up with a unique token t_up, to mean the next thing is all upper case
- so, then I lower case it and then we can learn the semantic meaning of all upper case
- and so, I've done a similar thing. If you've got like 29 exclamation marks in a row, we don't learn a separate token for 29 !, instead I put in a special token for "the next thing repeats lots of times", and then I put the number 29, and then the "!"
- and so, there are a few little tricks like that. If you're interested in NLP, have a look at the code for tokenizer for these little tricks I've added in, because some of them are kind of fun.

#### `00:37:40` Saving Tokenization
- so, the nice thing with doing things this way, we can just `np.save` that and load it back up later
- we don't have to recalculate all this stuff each time like we tend to have to do with torchtext or a lot of other libraries
```python
np.savesave((LM_PATHLM_PATH//'tmp''tmp'//'tok_trn.npy''tok_trn. , tok_trn)
np.save(LM_PATH/'tmp'/'tok_val.npy', tok_val)
```
- load it later
```python
tok_trn = np.load(LM_PATH/'tmp'/'tok_trn.npy')
tok_val = np.load(LM_PATH/'tmp'/'tok_val.npy')
```

####  `00:38:00` Turning Text into Numbers
- we've not got it tokenized
- the next thing is to turn the text into numbers which we call **numericalizing** it
- and the way we numericalize it is very simple:  we make a list of all the words that appear, in some order.  
- and then we replace every word with its index into that list
- this list of all the words that appear, OR the **tokens** that appear, we call the **vocabulary**
- here's an example of some of that vocabulary
- the `Counter` class in Python is very handy for this.  It basically gives us a list of unique items and their counts
- here is a list of the 25 most common tokens in the vocabulary
```python
freq = Counter(p for o in tok_trn for p in o)
freq.most_common(25)
```
```bash
[('the', 1207984),
 ('.', 991762),
 (',', 985975),
 ('and', 587317),
 ('a', 583569),
 ('of', 524362),
 ('to', 484813),
 ('is', 393574),
 ('it', 341627),
 ('in', 337461),
 ```
 - generally speaking, we don't want every unique token in our vocabulary
 - if it doesn't appear at least 2 times, then it might just be a spelling mistake, or a word, we can't learn anything about it if it doesn't appear that often
 - also the stuff we're going to learn about in this part gets a bit clunky once you've got a vocabulary bigger than 60,000
 - time permitting, we may look at some work I've been doing recently on handling larger vocabularies, otherwise that may come in a future course
 - but, actually for classification I've discovered that doing more than 60,000 words doesn't seem to help anyway, so we're going to limit our vocabulary to 60K words and things that appear at least twice
 ```python
max_vocab = 60000
min_freq = 2
```
- and here's a simple way to do that; use that `.most_common`
- pass in the `max_vocab` size; that will sort it by the frequency, by the way
- and if it appears less often than a specified minimum frequency, don't bother with it
```python
itositos  ==  [[oo  forfor  oo,,cc  inin  freqfreq..most_commo(max_vocab) if c>min_freq]
itos.insert(0, '_pad_')
itos.insert(0, '_unk_')
```
- that gives us "itos", that's the same name that torchtext used
- that means "integer to string"
- so this is just the list of tokens, unique tokens in the vocab
- JH is going to insert 2 more tokens
  - one for unknown
  - one for padding
- then, we can create the dictionary which goes in the opposite direction, so it is `stoi` (string to INT), and that won't cover everything because we intentionally truncated it down to 60K words
- and so if we come across something that is not in the dictionary, we want to replace it with zero for "unknown", so we can use a `defaultdict` for that with a lambda function that always returns 0 
- so you can see all these things we're using that keep coming back up 
```python
stoi = collections.defaultdict(lambda:0, {v:k for k,v in enumerate(itos)})
len(itos)
```
- `40:50` so not that we have our `stoi` dictionary defined, we can just call that for every word for every sentence
```python
trn_lm = np.array([[stoi[o] for o in p] for p in tok_trn])
val_lm = np.array([[stoi[o] for o in p] for p in tok_val])
```
- and now here is our **numericalized version**
```bash
'40 41 42 39 279 320 13
```
- and so, of course the nice again is we can save that step as well
- and each time we get to a step, we can save it. and these are not very big files, compared to what you are used to with images, text is generally pretty small
- very important to also save that vocabulary
- because this list of numbers means nothing, unless you know what each number refers to, and that's what 'itos' tells you
- so you save those 3 things, and later on you can load them
```python
np.save(LM_PATH/'tmp'/'trn_ids.npy', trn_lm)
np.save(LM_PATH/'tmp'/'val_ids.npy', val_lm)
pickle.dump(itos, open(LM_PATH/'tmp'/'itos.pkl', 'wb'))
```
- load back in
```python
trn_lm = np.load(LM_PATH/'tmp'/'trn_ids.npy')
val_lm = np.load(LM_PATH/'tmp'/'val_ids.npy')
itos = pickle.load(open(LM_PATH/'tmp'/'itos.pkl', 'rb'))
```
- so, now our vocab size is 60,002 and our training language model has 90,000 documents in it
- that's the pre-processing you can do.  we can wrap more of that in the utility functions if you want to, but it's all very straightforward 
- basically that exact code will work for any dataset you have once you've got it in that csv format

### `42:15` wikitext103 conversion
- Instead of pretraining on ImageNet, **for NLP** we can pretrain on a large subset of Wikipedia
- here's a kind of a new insight that is now new at all, which is that we'd like to pretrain something
- we know from Lesson 4 that if we pretrain our classifier by first creating a language model and then fine-tuning that as a classifier, that is helpful.  You remember, that actually got us a new state of the art result!  We got the best IMDb classifier result that had ever been published, *by quite a bit*
- Well, we're not going far enough, because IMDb movie reviews are not that different to any other English document
- you know, compared to how different they are to a random string, or even to a Chinese document
- So, just like ImageNet allowed us to train things that recognized stuff that kind of looks like pictures, and we could use it on stuff that was nothing to do with imagenet, like satellite images, why don't we train a **language model** that is good at English, and then fine tune it to be good at movie reviews?
- `43:40` this basic insight led me to try building a language model on wikipedia
- my friend Steven Merity (https://twitter.com/smerity?lang=en) has already processed wikipedia, found a subset of..nearly.. the most overt, but throwing away the stupid little articles, so most of the bigger articles, and he calls it wikitext 103.
- so I grabbed wikitext103 and I trained a language model on it
- and I used exactly the same approach I'm about to show you for training an IMDb model, but instead I've trained a wikitext103 model
- and then I saved it, and I've made it available for anybody who wants to use it at this url: 
```bash
# ! wget -nH -r -np -P {PATH} http://files.fast.ai/models/wt103/# ! wge
```
- this is not a url for wikitext 103 the document, this is the wikitext103 **the language model**
- the idea is, let's train an IMDb language model, which starts with **these weights**
```python
PRE_PATHPRE_PAT  = PATH/'models'/'wt103'
PRE_LM_PATH = PRE_PATH/'fwd_wt103.h5'
```
- `45:00` hopefully to you folks, this is an extremely obvious, extremely non-controversial idea cause it's basically what we've done in nearly every class so far.  **But,** when I first mentioned this to people in the NLP community, I guess, June, July of last year, there couldn't have been less interest.  I asked on twitter, where a lot of the top (twitter) NLP researchers I follow and they follow me back, where I asked "hey, what about if we pre-trained a general language model?" --- no, no language is different... No, you can't do that... I don't know why you would bother anyway... I would talk to people at conferences and they say:  I'm pretty sure people have tried that, and it's stupid... There was just this, I don't know, weird ? past.  I guess because I am arrogant and [obstreperous](https://www.merriam-webster.com/dictionary/obstreperous) [definition: unruly, unmanageable], I ignored them even though they know much more about NLP than I do and just tried it anyway.  And let me show you what happened.

### `46:05` using wikitext103
- Here's how we do it.  
- Grab the wikitext models
- if you use `wget` and `-r`, it will recursively grab the whole directory; it's got a few things in it
```python
! wget -nH -r -np -P {PATH} http://files.fast.ai/models/wt103/
```
- we need to make sure that our language model has exactly the same **embedding size, number of hidden and number of layers** as my wikitext one did, otherwise we can't load the weights in 
- so these are the numbers here
```python
em_sz,nh,nl = 400,1150,3
```
- here is our pre-trained path, and our pre-trained language model path
```python
PRE_PATHPRE_PAT  = PATH/'models'/'wt103'
PRE_LM_PATH = PRE_PATH/'fwd_wt103.h5'
```
- let's go ahead and `torch.load` in those weights from the forward wikitext 103 `fwd_wt103` model 
```python
wgts = torch.load(PRE_LM_PATH, map_location=lambda storage, loc: storage)

enc_wgtsenc_wgts = to_np(wgts['0.encoder.weight'])
row_m = enc_wgts.mean(0)
```
- we don't normally use `torch.load` but that's the PyTorch way of grabbing the file
- and it basically gives you a dictionary containing the name of the layer and a tensor of those weights, or an array of those weights
- now, here's the problem.  That wikitext language model was built with a certain vocabulary which was not the same as this one was built on, so, my number 40 is not the same as wikitext 103 model's number 40.  
- we need to map one to the other.  That's very, very simple, because luckily I saved the `itos` for the wikitext vocab, right?  So, here's the list of what each word is when I train the wikitext 103 model.  And so we can do the same `defaultdict` trick to map it in reverse.  And I am going to use `-1` to mean that it's not in the wikitext dictionary.
```python
itos2 = pickle.load((PRE_PATH/'itos_wt103.pkl').open('rb'))
stoi2 = collections.defaultdict(lambda:-1, {v:k for k,v in enumerate(itos2)})
```
- and now, we can just say, ok, my new set of weights is just a whole bunch of zeroes
- with `vs` (vocab size) by `em_sz` (embedding size); we're going to create an embedding matrix
- and then go through every one of the words in my IMDb vocabulary
- I am going to look it up in `stoi2[w]`, string to int for the wikitext 103 vocabulary
- and see if that word is there, and if the word is there, then I am not going to get this `-1`, so r will be greater than or equal to 0, 
- in that case, I will just that row of the embedding matrix to the weight that I just looked up, which was stored inside this named element `0.encoder.weight`
- these name...you can just look at this dictionary (wgts) and it's pretty obvious what each name corresponds to, because it looks very similar to the names you gave it when you set up your module
- so here are the encoder weights:  `0.encoder.weight` -- so, grab it from the encoder weights `enc_wgts[r]` 
```python
new_w = np.zeros((vs, em_sz), dtype=np.float32)
for i,w in enumerate(itos):
    r = stoi2[w]
    new_w[i] = enc_wgts[r] if r>=0 else row_m
```
- If I don't find it, then I will use the row mean.  In other words, here is the average `row_m = enc_wgts.mean(0)` embedding weight across all of the wikitext 103 things
- so, that's pretty simple.  I'm going to end up with an embedding matrix for every word that's in both my vocabulary for IMDb and the wikitext 103 vocab, I will use the wikitext 103's embedding matrix weights.  For anything else, I will just use whatever was the average weight from the wikitext 103 embedding matrix
- and then I will go ahead and I will replace the encoder weights with that `['0.encoder.weight']  = T(new_w)`, turned into a tensor 
```python
wgts['0.encoder.weight'] = T(new_w)
wgts['0.encoder_with_dropout.embed.weight'] = T(np.copy(new_w))
wgts['1.decoder.weight'] = T(np.copy(new_w))
```
- we haven't talked much about weight tying, we might do so later
- but, basically, the "decoder" `wgts['1.decoder.weight']`, the thing that turns the final prediction back into a word, uses exactly the same weights, so I pop it there as well
- and then there is a bit of a weird thing with how we do embedding dropout, `wgts['1.decoder.weight'] = T(np.copy(new_w))`, that ends up with a whole separate copy of them for a reason that doesn't matter much
- so, we just pop weights back where they need to go
- this is now something, a dictionary, or a..., or a torch state we can load in

## Language Model `50:17`
- let's go ahead and create our language model
- the basic approach we are going to use, and I'm going to look into this in more detail in a moment...
- the basic approach is I am going to concatenate all of the documents together... into a single list of tokens, of length 24,998,320 (24.98 million)
```python
t = len(np.concatenate(trn_lm))
t, t//64
```
```bash
(24998320, 390598)
```
- so, that's going to be what I pass in as my training set.  
- for the language model, we basically just take all our documents, we concatenate them, back to back.  we are going to continuously predict... what's the next word after these words? what's the next word after these words?  
```python
trn_dl = LanguageModelLoader(np.concatenate(trn_lm), bs, bptt)
val_dl = LanguageModelLoader(np.concatenate(val_lm), bs, bptt)
md = LanguageModelData(PATH, 1, vs, trn_dl, val_dl, bs=bs, bptt=bptt)
```
- we'll look at these details in a moment
- I'm going to set up a whole bunch of dropouts
```python
drops = np.array([0.25, 0.1, 0.2, 0.02, 0.15])*0.7
```
- once we've got a `ModelData` object, we can then grab the model from it, so, that's going to give us a **learner**
```python
learner= md.get_model(opt_fn, em_sz, nh, nl, 
    dropouti=drops[0], dropout=drops[1], wdrop=drops[2], dropoute=drops[3], dropouth=drops[4])

learner.metrics = [accuracy]
learner.freeze_to(-1)
```
- `51:20` and then, as per usual, we can call `learner.fit`, so we've first of all, as per usual, just do a single epoch on the last layer, just to get that ok
- and, the way I have it set up is the **last layer** is actually the **embedding words**
- because that's obviously the thing that's going to be the most wrong, cause a lot of those embedding weights, didn't even exist in the vocab, so we're just going to train a single epoch, of just the embedding weights
- and then, we'll start doing a few epochs of the full model
- and, so how is that looking?
- well, here's Lesson 4, which was our academic, world's best ever result, and after 14 epochs, we had a **4.23 loss**
- here, after 1 epoch (in Lesson 10), we have a **4.12 loss**
- so, by pretraining on wikitext 103, in fact, let's go have a look (Lesson 4), we kept training and training at a different rate, eventually we got to 4.16
- so by pre-training on wikitext 103, we have a better loss after 1 epoch, than the **best loss** we got for the language model otherwise
- `52:40` Rachel: what is the wikitext 103 model?  Is it awdlstm again?  
- JH:  yes, and we're about to dig into that again.  The way I trained it, it was literally the same lines of code as you see... here.., but without pre-training it on wikitext 103

## `53:08` fastai doc project 
- welcome back from break
- before we go back into language models and NLP classifiers, a quick discussion about something pretty new at the moment, which is the fastai doc project: https://github.com/fastai/fastai/tree/master/docs
- the goal of the fastai doc project is to create documentation that makes readers say, "wow, that's the most fantastic documentation I've ever read!"
- and so, we have some specific ideas of how to do that, and it's the same kind of idea as top down, thoughtful, take full advantage of the medium approach, you know, interactive experimental code first that we are all familiar with
- if you're interested in getting involved, the basic approach you can see in the docs directory
https://github.com/fastai/fastai/blob/master/docs/transforms-tmpl.adoc
- this is the README in the docs directory:  
  - https://github.com/fastai/fastai/blob/master/docs/README.md
- in there, amongst other things, there is a transforms template, transforms-tmpl.adoc
- what the hell is "adoc"?  It is "ASCII doc"
- how many people here have come across ASCII doc?  .... That's awesome
- ASCII doc is ... people are laughing because there is one hand up... and it was someone in our study group here today who talked to me about ASCII doc.
- [What is AsciiDoc? Why do we need it?](https://asciidoctor.org/docs/what-is-asciidoc/)
- ASCII is the most amazing project, it's like Markdown, but it's what Markdown needs to be to create actual books
- A lot of actual books are written in AsciiDoc
- it's as easy to use as Markdown, but there's way more cool stuff you can do with it.
- In fact, here is an AsciiDoc file here:  https://github.com/fastai/fastai/blob/master/docs/transforms-tmpl.adoc
- `55:00` as you'll see, it looks very normal.  there's a heading, and this is pre-formatted text, and there's lists and whatever else
- it looks pretty standard
- actually, I will show you a more complete AsciiDoc
  - you can say, put a table of contents here:  `:toc:`
  - `NO::`, `::` means put a definition list here
  - `+` means this is a continuation of the previous list item
  - so, there are just little things you can do which are super-handy, or like put this thing, or make it slightly smaller than everything else;  so, it's like turbo-charged Markdown
```text
[[Transform]]
== Class Transform [.small]#[tfm_y=TfmType.NO)#
```
- `55:50` this AsciiDoc creates this html:
- I didn't add any CSS or do anything else.  We literally started this project 4 hours ago, so this is just an example basically
- so, you can see we've got a Table of Contents
  - we can jump straight to a section
  - we've got a cross-reference we can click on
  - each method comes along with its details
  - and so on, and so forth
  - and, to make things even easier, rather than having to know that the argument list is meant to be smaller than the main part, or how do you create a cross reference, or how you meant to format the arguments to the method name, and list out each one of its arguments
- we've created a special template
  - where you can write various stuff in curly brackets
  - please put the arguments here, and here's an example of the argument
  - and here is a cross reference, and here is a method
  - and so forth
- we're in the process of documenting the documentation template
  - there are basically 5 or 6 of these little curly bracket things you need to learn
  - but, for you to create a documentation of a class or method, you can just copy one that is there
- idea is.. it will almost be like a book, there will be tables and pictures and little video segments, and hyper links throughout and all that stuff
- you may be wondering, what about doc strings?  But, I don't know if you've noticed.  If you look at the Python (https://github.com/fastai/fastai/blob/master/docs/README.md) standard library, and look at the docstring for regex compile, it's a single line:  `re.compile()` 
- nearly every docstring in Python is a single line
- and Python then does exactly this:  they then have a website containing the documentation that says, hey this is what regular expressions are, and this is what you need to know about them, and if you want them to go faster, you'll need to use `compile` and here's the information about compile, and here's examples.  It's not in the docstring.
- and that's how we're doing it as well.  our doc strings will be one line, unless like you need 2 sometimes; it's going to be very similar to Python, but even better
- so, everybody is welcome to help contribute to the documentation.  And hopefully, by the time you are watching this on the MOOC, it'll be reasonably flushed out.  And we'll try to keep a list of things to do

### notes are continued in [lesson_10_2](lesson_10_2.md)
