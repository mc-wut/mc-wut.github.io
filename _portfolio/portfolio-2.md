---
title: "Page Sorting of Public Works Specification PDFs based on text-features"
excerpt: "Text Based Section Classification of Public Works Construction Documents. <br/><img src='/images/classifier-post-image.png'>"
collection: portfolio
---

In Spring 2024 I was working for LUM AI. Initially I was tasked with building an ODIN grammar that could be applied to Construction Contract Documents and Specifications to extract a few specific pieces of data for a customer. I knew going in that the information we were looking for would be on a small number of pages in large PDFs, but I didn't anticipate the difference. At the end of the project it was obvious that there was a greater disparity than anticipated. On even the largest documents (around 700 pages), we were only pulling data of of 7-10 pages. 

Each page that didn't contain relevent data was still being, segmented, tokenized, parsed, tagged, and run through Named Entity Recognition. Which means that even in the best case 90% of our processing resources and time were being wasted. I proposed an extension to the project that would run a simple classifier over the raw text of each page of the PDF, and attempt to sort them into *annotate* and *don't annotate* bins. This ended up being changed somewhat to classifying the pdfs along existing section lines, as that would be relevant to other projects being built for this text as well. 

The texts we were working with were not uniform but followed one of two different section schemes. They contained either a 5 or 6 digit section code, which corresponded to a type of information. 

**ScreenGrabs of different section types.**

As an extension of my work at LUM AI, I built a `Text Based Page Classifier` 