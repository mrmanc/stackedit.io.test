---
layout: post
title:  "Text mining our customer calls"
author: "Amy O' Leary, Thomas Inman, Tom Collingburn"
---

At Auto Trader, our support staff receive up to 15,000 calls every two months. It's a huge operation to co-ordinate everyone involved and get an objective measurement for how we're doing, and what is making our customers tick this week. Currently, support managers go through a subset of recorded text from the calls we receive, manually classifying the emails in order to understand the volume of calls relating to each topic and identify changes needed to our products or our training. A few months ago, we formed a working group with a few other developers from around Auto Trader to see if we could solve the issue of text mining our calls for an objective and measurable understanding of our customers.

What we work on at Auto Trader from Monday to Thursday varies hugely, but our group spends our Friday afternoons playing with Auto Trader's large stash of data to see what can be learned. Here are a few notes on what we discovered.

A common pipeline
------------

Whatever algorithms you decide to use when topic mapping, the pipeline tends to consist of three basic steps:

1. Cleansing the documents

2. Frequency mapping

3. Clustering


[This paper](http://userpages.umbc.edu/~tri1/docs/unsuperdocumentclass.pdf) describes various methods you can use for those three steps.

Luckily there is a [blog](http://brandonrose.org/clustering) that describes how they used that pipeline to map topics for movies based on reviews.
The blog uses common Python libraries like sklearn and nltk to perform the three steps of the pipeline.

First, it remove stopwords, stems the words, and tokenises them. Then it maps the frequencies of words using [Tf-idf](http://www.tfidf.com/). Then, it clusters on words using the [K-Means](https://sites.google.com/site/dataclusteringalgorithms/k-means-clustering-algorithm) algorithm.
  
Using the code from this blog we decided to see if we could get some results for our support call data.


Our own pass of topic mapping: Frequency Mapping
--------------
First, we ran the code from the [blog](http://brandonrose.org/clustering) with our own set of data. Then we began tweaking parameters on the algorithms to better suit our data.

Frequency mapping using sklearn has quite a lot of parameters that you need to play around with to get the topic mapping just right.

```python
tfidf_vectorizer = TfidfVectorizer(max_df=.30, max_features=200000,
                                 min_df=10, stop_words='english',
                                 use_idf=True, tokenizer=tokenize_and_stem, ngram_range=(1,4))
```

Here we chose an ngram range of 1–4. Ngrams are contiguous sequences of words, so more words likely have more meaning, but are likely to occur less frequently.

It seems sensible to have at most 4 words or at least one word per ngram, as we are unlikely to be able to find 5 contiguous words that occur very frequently in our text.
The blog describes the other parameters to tweak, but we ended up using close to the defaults after a little experimentation. 

Setting our number of clusters to be 20 we managed find topic clusters that allowed us to understand the main topics that were being discussed.

For example, our second cluster contained the topics:

> ('payment', 'payment taken', 'taken', 'make payment', 'make', 'unlock', 'custom', 'dd', 'took payment', 'took')

so we can start to count the amount of calls that are about account payment, and another cluster contained the topics:

> ('atm', 'custom', 'new', 'use', 'new atm', 'vehicl', 'advertis', 'advis', "n't", 'want')

so we can see that a lot of our customers are advertising or want to advertise on ATM, a popular product of ours.


Some more tweaking: Document cleansing
--------

Looking at some of the clusters, we saw some people's names occurring quite frequently. We also saw a lot of frequently used words like 'email' and 'customer' that our topic mapper has attempted to 
cluster on. None of these provide useful information. It seems that a custom list of stopwords is a good idea.
Using nltk, we were able to use the 'names' corpus to remove names as stopwords, and we manually converted any text that looked like a date or time to the word *datetime*. 


An alternative: Clustering
--------
One problem with using K-Means for clustering is that we have to choose the number of clusters we want topics to be a part of.
This means that some outlier calls can end up in a cluster that they don't really belong to, simply because it is unique, and doesn't fit in any one cluster.
If we're trying to quantify the amount of calls that were for a given topic, this can skew results.

We tried out an [alternative topic mapping library](https://amaral.northwestern.edu/resources/software/topic-mapping), which will automatically select the optimal number of clusters to map topics to.

We found that we ended up with many more clusters, as some outlier calls ended up in their own clusters. This can be inconvenient, but we found that we were then able to look at just the larger clusters, and get a more accurate view of 
how many clusters were about this particular topic.


Can we use this?
--------
The classifier that we built runs within a few minutes, meaning that we can easily have a real time system mapping calls to topics as they come in
and reporting on statistics for those topics. With some nice visualisation tools, this could allow us to respond to changes much faster and objectively tell what makes our customers tick!


