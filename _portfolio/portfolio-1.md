---
title: "Information Extraction from long texts using ODIN"
excerpt: "Capturing specific entities, locations and figures in text extracted from PDFs using syntactic rules. ![Internship Post Image](/images/internship-post-image.png)"  
collection: portfolio  
---

The task of my internship with LUM AI was to create the grammar and extraction filters for an ODIN-based Information Extraction system. 

The documents that required extraction were technical and contract documents provided by government agencies for the construction of public projects. They covered the details of projects like rebuilding Sewer Lift Pump Stations, adding wells to landfills, and other public construction projects. The client was a construction company looking to streamline their estimate process. The goal was to have the most critical details at a glance rather than digging through these documents, which could be up to 700 pages long.

This blog post will be in four sections.
	1) A Brief Introduction to ODIN
	2) Documents and Extractions
	3) Extraction Grammar
	4) Section Grammar and Filtering

## A Brief Introduction to ODIN

[Odin (Open Domain INformer)](https://arxiv.org/abs/1509.07513) is an extraction framework that operates over documents that have been tokenized, sentence-segmented, part of speech tagged, lemmatized, and named entity tagged, and dependency parsed using an NLP pipeline. For a complete introduction see the link, I'll only include some basics in this post.

The framework is characterized by its rule-based approach. Each rule can rely on any of the underlying annotation systems or the plain text. 

A simple rule might be a lemma like "bid" or a NER tag of "org".

```yaml
  - name: "bid"
    priority: 1
    label: Bid
    type: token
    keep: false
    example: "Bids"
    pattern: |
      [lemma=bid]|[lemma=proposal]


  - name: "org-ner"
    label: Org
    priority: 1
    type: token
    keep: false
    pattern: |
      [entity=/ORG/]+
```

Complex rules combine several rules or the occurrence of a previous rule in a specific syntactic position.

```yaml
  - name: 'bid-time-event-1'
    priority: 2
    label: BidTime
    pattern: |
      trigger = [lemma=receive]|[lemma=open]
      dummy_bid:Bid = (>dobj|>nsubjpass|>nsubj) (>nmod_for)?
      time:Time = (>nmod_at|>nmod_to)? (>nmod_of)? (>nmod_until|>/^nmod/|>dep)? (>nummod|>compound)?
```
In this example we've got a 'bid' rule and a 'time' rule being utilized as parts of an **Event-Mention**.

![Bid-Time rule diagram](/images/internship-post-image.png)

After applying the grammar (set of all rules) to the annotated document, we are left with our **Mentions**. 

In this project, Odin **Mentions** were then filtered and pruned into **Extractions**, the end product for users. The initial and main mechanism for filtering rules is theh ```keep:``` flag. Which defaults to True. If you are curious about seeing which of our (admittedly many) rules were reaching end users, that is the easiest way to tell.  

## Documents and Extractions	 
The documents we were processing did not follow a universal standard but came in two main types. 

[Type A](https://mc-wut.github.io/files/bowman-specs-contract-docs.pdf) was an expansive tome (200-700) containing many sections on various technical requirements, and a single large section covering the legal agreements between various construction companies and the government entity contracting the work. 

[Type B](https://mc-wut.github.io/files/ceres-addendum.pdf)  was an "Addendum" a 1-15 page document, that indicated that something had changed in the original document. A mistake or typo occured in the original document, or some circumstance had changed. This will be important later.

From these documents our clients wanted ten different fields extracted.
    - Project Name
    - Project Owner
    - Project Duration
    - Project Location
    - Bid Date
    - Bid Time
    - Engineer's Estimate
    - Engineer of Record
    - Qualifications and Concerns
    - Liquidated Damagese

Each of the rules in our [`variable_grammar.yml`](https://github.com/mc-wut/internship_files/blob/main/variable_grammar.yml) is for the capture of these variables as they occur in the texts. 

##  Variable Grammar
It would be tedious and unnecessary to cover the detail of every rule in `variable_grammar.yml`, so instead I will highlight the process for just a few.


### Example 1: Regex for Dollar - Engineer's Estimate
The *Engineer's Estimate* is the Engineer of Record's estimate for how much the project will cost overall. It shows up in a variety of different formats, and needs to be differentiated from *Liquidated Damages* which is also a dollar amount. The two differed mainly by length.

```yaml
  - name: "5-8-digit-dollar"
    label: Dollars
    priority: 1
    type: token
    keep: false
    example: "$1,000,000"
    pattern: |
      [tag=$] /\d{1,3},*\d{3},*(\d{3})*/
```

This rule is then combined with a setting rule that looks for phrasing we found around *Engineer's Estimates*. 

```yaml
  - name: "estimated-cost"
    label: Cost 
    priority: 1
    type: token
    keep: false
    example: "estimated cost"
    pattern: |
      [lemma=estimate] []? [lemma=cost]
  
  - name: "estimate-opinion"
    label: Cost
    priority: 1 
    type: token
    keep: false
    example: "engineers opinion"
    pattern: |
      ([lemma=engineer]|[lemma=budget]) [tag=POS]* []? ([lemma=opinion]|[lemma=estimate])
```

Note that the **label** for both `estimated-cost` and `estimate-opinion` is `Cost`. This allows us to reference either of them in a later rule.

```yaml
  - name: "engineers-estimate"
    label: EngineersEstimate 
    priority: 2
    pattern: |
      trigger = @Cost
      dollars:Dollars = (<nsubj|>nmod_of|>advcl)
```

This *Event-Mention* looks for our `@Dollars` regex that are `nsubj`, `nmod_of`, or `advcl` of any occurences of  `@Cost` 

![Engineer's Estimate dependency graph](/images/engineers-estimate-diagram.png)

### Example 2: Project Owner 

snippet from pdf where it's applied

### Example 3: Project Owner - A Surface Example

'''yaml

'''


snippet from pdf where it's applied




## FILTERING AND SECTION GRAMMAR

** LINK TO WMLUTILS GOES AT THE TOP**

DESCRIBE HOW FILTRATION AFFECTS END-USER EXPERIENCE (DUPLICATE ETC)

DESCRIBE HOW FILTERING ALSO HELPED WITH THE DIFFERENTIATION OF EXTRACTIONS FROM ADDENDA VS GENERAL DOCUMENTS WITHOUT A DIRECT NEED FOR USER INPUT

## FINAL PRODUCT

SHOW EXAMPLE CONTRACTED DOC (HIGHLIGHTED EXTRACTION FIELDS????)

THEN SHOW THE OUTPUT OF THE LOCAL SYSTEM. (TEXT OR IMAGE???)

[Code Relevent to this Project](https://github.com/mc-wut/internship_files/tree/main)

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
