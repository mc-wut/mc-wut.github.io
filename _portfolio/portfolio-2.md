---
title: "Page Sorting of Public Works Specification PDFs based on text-features"
excerpt: "Text Based Section Classification of Public Works Construction Documents. <br/><img src='/images/classifier-post-image.png'>"
collection: portfolio
---

In Spring 2024 I was working for LUM AI. Initially I was tasked with building an ODIN grammar that could be applied to Construction Contract Documents and Specifications to extract a few specific pieces of data for a customer. I knew going in that the information we were looking for would be on a small number of pages in large PDFs, but I didn't anticipate the difference. At the end of the project it was obvious that there was a greater disparity than anticipated. On even the largest documents (around 700 pages), we were only pulling data of of 7-10 pages. 

Each page that didn't contain relevent data was still being, segmented, tokenized, parsed, tagged, and run through Named Entity Recognition. Which means that even in the best case 90% of our processing resources and time were being wasted. I proposed an extension to the project that would run a simple classifier over the raw text of each page of the PDF, and attempt to sort them into *annotate* and *don't annotate* bins. This ended up being changed somewhat to classifying the pdfs along existing section lines, as that would be relevant to other projects being built for this text as well. 

The texts we were working with were not uniform but followed one of two different section schemes. They contained either a 5 or 6 digit section code, which corresponded to a type of information. From the original dataset we discluded a document that followed no scheme, at the direction of our customer.
**ScreenGrabs of different section label types.**

## Dataset Creation
In order to train our classifier we obviously had to create a dataset. We had 12 of the larger specs and 16 addenda to build into our dataset. 5120 pages overall. 

The script in `build_dataset.py` **SHOW ME A LINK HERE** was used to start the process of building a dataset.

`Section_Label` is our workhorse class here, an extention of `StrEnum`, for consistency of labels across the dataset.

`Section_Label.to_section_name()` contained a mapping of our different section schemas (5 or 6 digit sectino numbers depending on which format they followed) to English section names.

We had to account for the occasional missing section number, and so included the functino `confirm_continuous_section` **SHOW ME A LINK HERE** as a sort of sanity check. It basically checked if pages 234 and 236 were both in **ELECTRICAL** it made sure that page 235 wasn't accidentally in **TRANSPORTATION**

This got us most of the way there. But the dataset still needed a good amount of hand correction. Many of the documents didn't have a label on their **BIDDING AND CONTRACT DOCUMENTS** sections, which were the most important section for our purposes, and some sections had multiple pages in the middle of sections without any section number. 

## Custom Vectorizer
We decided the best way to do this was to create an extension of Sci-Kit Learn's `DictVectorizer` in a pipeline with traditional vectorizers. After some experimentation we decided on 'TfidfVectorizer' for our traditional vectorizers. 

Our custom vectorizer, `TextBasedFeatureExractor`, followed the intuition that there were a handful of important features we could (largely) base our classification on.
    1. Section Number
    2. Page Length
    3. Page Number
    4. Presence of "Table of Contents"
    5. Presence of word "Addenda"

We modified `_to_feature_dictionary()` to check to return a dictionary with these features.
**shortened code snippet from `_to_feature_dictionary()`**

The key helper functions here are:
    
`check_page_length()` This was really good for helping with occassional blank pages,   "This page intentionaly left blank" pages, and cover pages, which tend to have few characters, but a lot of pertinent extractions.

`check_addendum()` Which uses a Regex to check for addend* in the header and footer. This one was pretty productive, but probably produced the most cloudy signal. A few times in the dataset it produced false positives.

`check_table_of_contents()` Which uses a Regex to check for a "Table of Contents " or "TOC" label.

`check_section_num()` is the star here and takes a section number and the page of text and checks if they match.

The compiled dictionary is then transformed. 

## Pipeline
Our pipeline is housed in `TextBasedPageClassifer`. It is highly parameterized as that allowed us more ease of testing.

## Results
Dig into TextBasedPageClassifier_log.txt for this section. Highlight difference between page number results and non-page number results.

## Issues
The main issue with the approach that was used here is that the Contract Documents section ended up being huge in a number of cases. The Contract Documents section also often held a lot of our extractions. The classifier was still successful and useful. But I do wonder about the efficacy of building out a completely different dataset where we only highlighted pages that contained our target extractions.

This approach would create a kind of interesting logical loop. The ODIN system matches on certain phrase structure, and then if it worked the classifier would end up searching for those same phrase structures.