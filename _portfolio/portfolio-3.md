---
title: "The Dugmore Detector"
slug: "/mclaughlinm/course-project"
date: 2023-11-12
author: Matt McLaughlin
description: "My LING-582 course project"
tags:
  - course project
  - authorship verification
  - Dugmore Boetie
---

In fall of 2023 I worked an independent study with Dr. Gus Hahn-Powell. We were looking at an authorship verification task, specifically one novel with disputed origins.

## Task
*Familiarity is the Kingdom of the Lost* or *Tshotsholoza* is a South African novel published in 1969. It is ostensibly authored by Douglas Buti under the the penname Dugmore Boetie. However, Douglas died in 1966, and the book continued to be worked on by his editor Barney Simon for several years after his death. There is some dispute over how much Simon changed, and how much of the book Buti even authored. 

This authorship attribution issue would be a big deal with any novel, but this novel heavily deals with the rise of Apartheid in South Africa and is considered semi-autobiographical (the main character is named "Duggie"). Whether or not the white editor completely rewrote the black author's book is of special significance here. 

You can read more in Dugmore Boetie's [wikipedia page](https://en.wikipedia.org/wiki/Dugmore_Boetie).

The goal of this project is to determine the authorship of the disputed text. One aspect of the task that makes this inherently challenging is the nature of the relationship between a writer and an editor. The changes that are made by an editor can be very small or very large and Simon's fingerprints might be all over the text. 

## Approach

A collection of texts were assembled from Boetie, Simon, and a number of their contemporaries. From those texts a dataset was built. Testing showed that LogisticRegression was most accurate on snippets of 75 words. So each text was carved up into 75 word chunks, and reassembled into a csv with an id, a chunk and a label, "Boetie", "Simon", and "Other".

The core of the project was an extension of sklearn's DictVectorizer called FeatureEncoder. It creates a large feature dictionary out of the database. The features it uses are word ngrams and character ngrams, as well as custom stylometric features, like average sentence length, and punctuation frequency.

```Python
class FeatureEncoder(DictVectorizer):
    """
    This class extends DictVectorizer specifically for NLP/Authorship use:

    It takes a set of strings and generates word and character level n_grams, 
    polynomials of those n_grams, and a feature of how many pieces of punctuation
    are used in each snippet
    """ 
    def __init__(self, **kwargs):
      super().__init__(sparse=kwargs.get("sparse", True))
      # feature_fns is a dictionary that contains "feature functions"
      # feature functions are the custom 
      self.feature_fns: Dict[FeatureFn] = {
         "word_bigrams": lambda text: generate_ngrams(text=text, ngram_range=(1,2), 
                                                      level='word'),
         "char_fourgrams": lambda text: generate_ngrams(text=text, ngram_range=(2,4), 
                                                        level='char'),
         "word_polynomials": lambda text: create_polynomial_ngrams(
            text=text, ngram_range=(1,2), level='word',degree=2),
        #  "char_polynomials": lambda text: create_polynomial_ngrams(
            # text=text, ngram_range=(2,4), level='char', degree=2),
         "punctuation per snippet": lambda text: punct_count(text)
      }

    def create_feature_dict(self, datum) -> FeatureDict:
        """
        """
        feature_dict = {}
        for ffn in list(self.feature_fns.values()):
          for k, v in ffn(datum).items():
            feature_dict[k] = v
        return feature_dict

    def filter_by(self, min_df: int, dicts: Iterable[FeatureDict]):
       # count occurences of each key across datapoints
       seen = dict()
       for d in dicts:
          for k in d.keys():
            count = seen.get(k,0)
            seen[k] = count+1
       for d in dicts:
          new_d = {}
          for k, v in d.items():
             if seen[k] >= min_df:
                new_d[k]=v
          # hand back one dict at a time
          yield d 

    def fit(self, X, y=None):
        # TODO: add min_df to __init__
        dicts = [self.create_feature_dict(datum = datum) for datum in X]
        new_dicts = self.filter_by(min_df=2, dicts=dicts) 
        super().fit(new_dicts)

    def transform(self, X, y = None):
        return super().transform([self.create_feature_dict(datum) for datum in X])

    # FIXME: might not need to implement this at all
    def fit_transform(self, X, y = None):
        self.fit(X)
        return self.transform(X)

```

After fitting Logistic Regression on our dataset, we classified *Familiarity is the Kingdom of the Lost* in similar 75 word chunks. 

A state of the art approach would be something that resembles [AD-HOMINEM](https://doi.org/10.48550/arxiv.1910.08144), a Siamese NN, which is two identical recurrent neural networks that share the exact same set of parameters. The delta of the output of those two RNN's is used to determine authorship. AD-HOMINEM also uses attention layers making it possible to examine what exactly the classifier is making decisions based on, and possibly generate an entire copy of the book overlaid with heat maps of what was important to classifying the different blocks of text.

## Error Analysis
I tested the system on n=5 kfold cross validation and got the following vital statistics:                  

| | max | min | mean |  
|-----------|-----|-----|------|
| precision |  93.32 | 56.81 | 85.81 | 
| recall    | 70.49 | 60.22 | 65.53  | 
| f1        |72.20  |  58.18 |  64.71 |  

Interestingly when looking at True Positives and False Positives, (positive being Boetie) the average confidence score of a false positive in the testing data was .604, where the average confidence score of a True Positive was .918.  

So another possible next step is raising the threshold for positive predictions. 

## Summary of Contributions
As the only member of Team McWut, all work on this project was completed by me, with the help and oversight of Gus Hahn-Powell.

## Results
The system predicts that 83.8 % of the disputed text is authored by Boetie, while 13.8% is authored by Simon. You can see those results in this [google doc](https://docs.google.com/spreadsheets/d/1KwrEOTVs8zW7kR6xKT_xqWUTv4hGwWdP/edit#gid=1028922336).

As you can see the classifier's confidence score is included in the results spreadsheet, and chunks of the text that were classified as not Boetie are highlighted. 

## Steps for Reproduction
To reproduce results, use the dockerfile provided and run main.py. It will generate a file called '{currentdatetime}_dugmore.csv' to the outputs directory. That will contain predictions and confidence scores for each chunk of the novel. 

## Proposal for Future Improvments
The current interation of the Dugmore Detector has a number of limitations. Firstly it's quite opaque, the only insight we get into the classification is confidence scores. Secondly the text is chunked by word rather than sentence or paragraph. This is probably not ideal for a text that may have multiple authors.

One improvement would be breaking the training data into sentences or groups of sentences. I might use ntlk's sent tokenizer to do that in future iterations of the project. It could also be very interesting to approach the sentences like windows in an n_gram, and see how different the results for overlapping sentences is. 

Another possible avenue for improvment (as mentioned above) would be raising the threshold for positive identification. Our mean confidence score for false postivie was 0.604 so that would be a good place to start. 

A massive improvement would then be to attempt an implementation of something like AD-HOMINEM like discussed above. Generating a heatmap of the entire text that displays which words, sentences, and phrases are given the most weight in classification and sharing that with academics well versed in the topic would be interesting in the least and hopefully helpful.

## Codes

You can find my code here: `https://github.com/clu-ling/dugmore-detector` , please refer to the readme for how things are organized.