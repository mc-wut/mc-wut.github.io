---
title: "Page Sorting of Public Works Specification PDFs based on text-features"
excerpt: "Text Based Section Classification of Public Works Construction Documents. <br/><img src='/images/classifier-post-image.png'>"
collection: portfolio
---

In Spring 2024 I was working for [LUM AI](http://lum.ai). Initially, I was tasked with building an [ODIN](https://github.com/lum-ai/odinson) grammar that could be applied to Construction Contract Documents and Specifications to extract a few specific pieces of data for a customer. I knew going in that the information we were looking for would be on a small number of pages in large PDFs, but I didn't anticipate the difference. At the end of the project, it was obvious that there was a greater disparity than anticipated. On even the largest documents (around 700 pages), we were only pulling data of 7-10 pages. 

Each page that didn't contain relevant data was still being, segmented, tokenized, parsed, tagged, and run through Named Entity Recognition, which means that even in the best case 90% of our processing resources and time were being wasted. I proposed an extension to the project that would run a simple classifier over the raw text of each page of the PDF, and attempt to sort them into *annotate* and *don't annotate* bins. This ended up being changed somewhat to classifying the PDFs along existing section lines, as that would be relevant to other projects being built for this text as well. 

The texts we were working with were not uniform but followed one of two different section schemes. They contained either a 5 or 6-digit section code, which corresponded to a type of information. From the original dataset, we discluded a document that followed no scheme, at the direction of our customer.
![5-digit Section Number](/images/5-digit-section.png)
![6-digit Section Number](/images/6-digit-section.png)

## Dataset Creation
To train our classifier we had to create a dataset. We had 12 larger specs and 16 addenda to build into our dataset. 5120 pages overall. 

The script in [`build_dataset.py`](https://github.com/mc-wut/internship_files/blob/main/classifiers/build_dataset.py) was used to start building a dataset.

[`Section_Label`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/build_dataset.py#L9) is our workhorse class here, an extension of `StrEnum`, for consistency of labels across the dataset.

[`Section_Label.to_section_name()`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/build_dataset.py#L60-L124) contained a mapping of our different section schemas (5 or 6-digit section numbers depending on which format they followed) to English section names.

The occasional page lacking a section number necessitated the function [`confirm_continuous_section`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/build_dataset.py#L188-L209) as a sanity check. It checked if pages 234 and 236 were both in *ELECTRICAL* it made sure that page 235 wasn't accidentally in *TRANSPORTATION*
```python

```


This got us most of the way there. However, the dataset still needed a good amount of hand correction. Many of the documents didn't have a label on their *BIDDING AND CONTRACT DOCUMENTS* sections, which were the most important sections for our purposes, and some sections had multiple consecutive pages missing section numbers, which is more than `confirm_continuous_section` could account for. 


## Custom Vectorizer“Discluded” rather than “excluded”? I think, for this meaning, “exclude” is more common.
We decided the best way to do this was to create an extension of Sci-Kit Learn's `DictVectorizer` in a pipeline with traditional vectorizers. After experimentation, we decided on 'TfidfVectorizer' for our traditional vectorizers. 

Our custom vectorizer, [`TextBasedFeatureExractor`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/page_classifier.py#L24), followed the intuition that there were a handful of important features we could (largely) base our classification on.
 1. Section Number
 2. Page Length
 3. Presence of "Table of Contents"
 4. Presence of the word "Addenda"
 5. Page number

We modified [`_to_feature_dictionary()`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/page_classifier.py#L24-L168) to return a dictionary with these features.


The key helper functions here are:
    
[`check_page_length()`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/page_classifier.py#L171-L176) This was excellent for helping with occasional blank pages,   "This page intentionally left blank" pages, and cover pages, which tend to have few characters, but many pertinent extractions.

[`check_addendum()`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/page_classifier.py#L191-L197) Which uses a Regex to check for addend* in the header and footer. This one was pretty productive but probably produced the most cloudy signal. A few times in the dataset it produced false positives.

[`check_table_of_contents()`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/page_classifier.py#L226C1-L235C1) Which uses a Regex to check for a "Table of Contents " or "TOC" label.

[`check_page_num()`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/page_classifier.py#L200-L212) checked if pages were in specific ranges between 1 & 30. Including the page number presented a challenge at the PDF reader level, and there was also concern about combined documents throwing off the classifier, for example, a file where Addenda documents are merged onto the end of a larger spec. As a result, we didn't include it in the final version.

[`check_section_num()`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/page_classifier.py#L214-L225) is the star of the custom vectorizer and takes a section number and the page of text and checks if they match.

The compiled dictionary is then transformed. 

## Pipeline
Our pipeline is housed in [`TextBasedPageClassifer`](https://github.com/mc-wut/internship_files/blob/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers/page_classifier.py#L250). It is highly parameterized to allow for testing of combinations of different features.

After experimentation with `GridSearch`, we found that using a character-level `TfidfVectorizer` with the range (5,6), a word-level `TfidfVectorizer` with range (1,2) both with `min_df` set to 4 and `max_features` set to 5000 was the optimal configuration. We wanted to use polynomial features, but even at a depth of two the feature space was too large to compute. Finally, we applied sklearn's `SelectKBest` to limit to the best 10,000 features.

For classification, we used sklearn's LogisticRegression set to One vs Rest. 

## Results
We were able to achieve a 0.94 f1 score in the sections we were interested in.

The notable shortcomings here are *ADDENDA* AND *COVER PAGE*. Cover pages tend to lack consistent features, especially features we could capture with `TextBasedFeatureExtractor`. *ADDENDA* tend to resemble *CONTRACT DOCUMENTS*. I suspect that's where the bulk of the errors are. I was very excited that we did so well on *BIDDING AND CONTRACT DOCUMENTS* as those pages often lacked section numbers or anything of significance in their headers or footers. 

```
{'char_ngram_range': (5, 6),
 'custom_vectorizer': True,
 'k': 5000,
 'min_df': 4,
 'page_number_features': False,
 'polynomial_features': True,
 'tf_idf_character_vectorizer': True,
 'tf_idf_word_vectorizer': True,
 'word_ngram_range': (1, 2)}
                                precision    recall  f1-score   support

                    ELECTRICAL       0.93      0.99      0.96       543
BIDDING AND CONTRACT DOCUMENTS       0.90      0.99      0.94      1156
                       ADDENDA       0.71      0.37      0.49       129
          GENERAL REQUIREMENTS       0.95      0.96      0.96       801
                    COVER PAGE       0.67      0.28      0.39        36
           PROCESS INTEGRATION       0.98      1.00      0.99       358

                     micro avg       0.93      0.95      0.94      3023
                     macro avg       0.86      0.76      0.79      3023
                  weighted avg       0.92      0.95      0.93      3023
```

If you were wondering why I introduced `page_number_features` at all earlier, here's your answer. When we included them they dramatically improved our performance in both *COVER PAGE* and *ADDENDA*.


```
{'char_ngram_range': (5, 6),
 'custom_vectorizer': True,
 'k': 5000,
 'min_df': 4,
 'page_number_features': True,
 'polynomial_features': False,
 'tf_idf_character_vectorizer': True,
 'tf_idf_word_vectorizer': True,
 'word_ngram_range': (1, 2)}
                                precision    recall  f1-score   support 

                    ELECTRICAL       0.98      0.99      0.98       543
BIDDING AND CONTRACT DOCUMENTS       0.92      0.99      0.95      1156
                       ADDENDA       0.88      0.82      0.85       129
          GENERAL REQUIREMENTS       0.95      0.96      0.96       801
                    COVER PAGE       0.88      0.42      0.57        36
           PROCESS INTEGRATION       0.99      1.00      0.99       358

                     micro avg       0.95      0.97      0.96      3023
                     macro avg       0.93      0.86      0.88      3023
                  weighted avg       0.95      0.97      0.96      3023
```

It occurred to me as we struggled to classify those two targets that they were the only targets locked into certain page lengths. The longest Addendum we had was approximately 25 pages. And while we were treating Cover Pages as a section, it was never greater than 5 pages. 

## Issues
The main issue with the approach that was used here is that the Contract Documents section ended up being huge in a number of cases. The Contract Documents section also often held a lot of our extractions. The classifier was still successful and useful. But it wasn't quite as effective at cutting processing speed as we hoped. I do wonder about the efficacy of building out a completely different binary dataset comparing pages that contain target extractions against those that do not.

This approach would create an interesting logical loop. The ODIN system matches certain phrase structures, and then if it worked the classifier would end up searching for those same phrase structures.


[Relevant Code to this project.](https://github.com/mc-wut/internship_files/tree/905323ee86b7c2360188fb03e79316c3882e47a9/classifiers)