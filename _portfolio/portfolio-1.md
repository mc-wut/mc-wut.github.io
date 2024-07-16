---
title: "Data Extraction from long texts using ODIN"
excerpt: "Capturing specific entities, locations and figures in text extracted from PDFs using syntactic rules. <br/><img src='/images/internship-post-image.png'>"
collection: portfolio
---

**THIS ALL NEEDS TO BE CONVERTED TO MARKDOWN**

The task of my internship with LUM AI was to create the grammar and extraction filters for an ODIN-based Information Extraction system. 

The documents that required extraction were technical and contract documents provided by government agencies for the construction of public projects. They covered the details of projects like rebuilding Sewer Lift Pump Stations, adding wells to landfills, and other public construction projects. The client was a construction company looking to streamline their estimate process. The goal was to have the most critical details at a glance rather than digging through these documents, which could be up to 700 pages long.

This blog post will be in four sections.
	1) A Brief Introduction to ODIN
	2) Documents and Extractions
	3) Extraction Grammar
	4) Section Grammar and Filtering

## A Brief Introduction to ODIN

Odin (Open Domain INformer) **INSERT ARXIV LINK HERE** is an extraction framework that operates over documents that have been tokenized, sentence-segmented, part of speech tagged, lemmatized, and named entity tagged, and dependency parsed using an NLP pipeline.  

The framework is characterized by its rule-based approach. Each rule can rely on any of the underlying annotation systems or the plain text. 

A simple rule could be any noun or a literal string like "BID"

**INSERT BID & POS tag EXAMPLE RULE HERE**
Complex rules combine several rules or the occurrence of a previous rule in a specific syntactic position.

**INSERT BID IN CONTEXT AND OTHER COMPLICATED RULE HERE**

After applying the grammar (set of all rules) to the annotated document, we are left with our **Mentions**. 

**IMAGE OF BID EXTRACTION PULL FROM TUTORIAL PAGE**

In this project, Odin **Mentions** were then filtered, and pruned into **Extractions**. **Extractions** are the end product, filtered and cleaned for users.  

## DOCUMENTS AND EXTRACTIONS	 
The documents we were processing did not follow a universal standard but came in two main types. 

Type A was an expansive tome (200-700) containing many sections on various technical requirements, and a single large section covering the legal agreements between various construction companies and the government entity contracting the work. 

**INSERT LINK TO A FAT TOME OR AN IMAGE OF A MASSIVE PAGE COUNT IN ACROBAT**

Type B  was an "Addendum" a 1-15 page document, that indicated that something had changed in the original document. Someone somewhere made an oopsie, or some circumstance had changed. This will be important later.

**INSERT LINK TO LITTLE PDF OR IMAGE OF AN ADDENDUM COVER PAGE**

**DESCRIBE FIELDS HERE**

## EXTRACTION GRAMMAR (RENAME SECTION??)

**LINK TO CODE GOES AT THE TOP**

CHERRY PICK A COUPLE COOL RULES TO GO OVER HOW SMOART YOU BE

## FILTERING AND SECTION GRAMMAR

** LINK TO WMLUTILS GOES AT THE TOP**

DESCRIBE HOW FILTRATION AFFECTS END-USER EXPERIENCE (DUPLICATE ETC)

DESCRIBE HOW FILTERING ALSO HELPED WITH THE DIFFERENTIATION OF EXTRACTIONS FROM ADDENDA VS GENERAL DOCUMENTS WITHOUT A DIRECT NEED FOR USER INPUT

## FINAL PRODUCT

SHOW EXAMPLE CONTRACTED DOC (HIGHLIGHTED EXTRACTION FIELDS????)

THEN SHOW THE OUTPUT OF THE LOCAL SYSTEM. (TEXT OR IMAGE???)

<!-- This is an item in your portfolio that describes your internship. It can have images or nice text. If you name the file .md, it will be parsed as markdown. If you name the file .html, it will be parsed as HTML.

As you're describing the content of your internship, be sure to describe how you were able to apply the concepts and skills you acquired from HLT courses to your internship. You'll also want to describe the things that you learned from the internship itself that might help you in future work.

## Evaluation criteria
Remember that each of the two projects in your portfolio will be evaluated on these points:

* **Length**: A summary of the project goals, technology used, and outcomes, as appropriate for a general technical audience, between 1000 and 3000 words (not counting code)
* **Content**: student’s experience demonstrates the learning outcomes for the MSHLT program [^note]
* **Code**: Code is contained in the site, or a link to the code (such as in a GitHub repository) exists on the site.
* **Professionalism**: Free of grammatical, mechanical, and stylistic issues
* **Above and beyond**: How well does this component communicate the most relevant features?

[^note]: The learning outcomes of the MSHLT program are:
    
    1. Students will demonstrate programming skills for the workplace.
    2. Students will be able to use fundamental algorithms and concepts in Natural Language Processing.
    3. Students will show knowledge of tools and packages used in Natural Language Processing. -->
