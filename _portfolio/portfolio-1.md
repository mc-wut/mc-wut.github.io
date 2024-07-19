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
It would be tedious and unnecessary to cover the detail of every rule in `variable_grammar.yml`, so instead I will highlight the process for a few rules to give a general sense, and after these explanations any other questions should be able to be answered by examining the code.


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

Engineer's Estimate was one of the cleaner examples of these, I was able to capture all instances of the Estimate with one rule. Most other examples required a few rules to catch all the cases.

### Example 2: Project Owner - Surface Capture
There were three different rules that were used to capture *Project Owner*, but the most successful was a surface level rule that captured a string that occured in the contract documents frequently. 

The first part is a series of base rules for `@Organization`, attempting to capture the Companies, Municipalities, Counties, etc involved in these deals. 

```yaml
  - name: "org-ner-basic"
    label: Organization
    type: token
    priority: 1
    keep: false
    example: "Placer County Water Agency"
    pattern: |
      [entity=B-ORG] [entity=I-ORG]* [tag=NNP]?

  - name: "org-surface-1"
    label: Organization
    type: token
    priority: 1
    keep: false
    example: "Town of Windsor | County of Yolo | University of Arizona"
    pattern: |
      ([lemma=city]|[lemma=town]|[lemma=university]|[lemma=county]) [lemma=of] [tag=NNP]
  
  - name: "org-surface-2"
    label: Organization
    type: token
    priority: 1 
    keep: false
    example: "Tranquility Public Utility Districty | Yolo County"
    pattern: |
      [tag=NNP]{1,3} ([lemma=district]|[lemma=county]|[lemma=university])
```

Named Entity Recognition was only so successful at catching these, especially with strings containing regular English words like "Public Utility" or "Town of." In the end relatively hard coded lemma capture was required to achieve consistent capture. 

It took 7 rules to capture the project Owner across 12 sample documents.

```yaml
  - name: "owner-event-2"
    label: ProjectOwner
    priority: 3
    example: "The Malaga County Water District is soliciting bids for ... "
    pattern: |
      trigger = [lemma=solicit] | [lemma=accept]
      dummy_bid:Bid = >dobj 
      project_owner:Organization = >nsubj

  - name: "owner-surface-3"
    label: ProjectOwner
    type: token
    priority: 2
    example: "Tranquility Public Utility District, hereinafter reffered to as owner"
    pattern: |
      @Organization (?=([]{0,3} ([lemma=refer]|[lemma=call]) []{0,3} @Owner))
```
The use of `[]{0,3}` in `owner-suface-3` is allowing any three tokens to occur in that spot. 

This is an example output of that rule.
![owner-surface-3 capturing an instance of Project Owner](/images/owner-capture.png)

## FILTERING AND SECTION GRAMMAR
Beyond just capturing these fields, we needed to present it to the customer in a digestible way. One of the drawbacks of my approach was that it might capture the *Bid Date* in several different locations in the document, so we needed to resolve that.

Another was that **Project Duration** was a particularly important field to the customer, but they only wanted the duration of the entire project, and there were typically smaller durations included in the text that were in syntactically identical positions. I wrote a few functions for comparing these values in [`utils.py`](https://github.com/mc-wut/internship_files/blob/b4b956a74a035f2c5357a4f483b9a7c2eca65aa6/utils.py#L63).

```python
    @staticmethod
    def _filter_duplicate_extractions(
        unfiltered_extractions: list[schema.Extraction],
    ) -> list[schema.Extraction]:
        """Removes duplicate extractions"""
        # Filter for duplicate extractions
        filtered_extractions = []
        variable_value_pairs = set()
        for ex in unfiltered_extractions:
            if (ex.variable, ex.value) not in variable_value_pairs:
                variable_value_pairs.add((ex.variable, ex.value))
                filtered_extractions.append(ex)
        return filtered_extractions

    @staticmethod
    def _duration_filter(
        extractions: list[schema.Extraction],
    ) -> list[schema.Extraction]:
        """Returns the longest duration extraction"""
        measure_duration = lambda ex: (float("".join(filter(str.isdigit, ex.value))))
        max_duration = (-1, None)
        for ex in extractions:
            n = measure_duration(ex)
            if n > max_duration[0]:
                max_duration = (n, ex)
        longest: schema.Extraction = max_duration[-1]
        return longest
```

Also it was important ot the customer that should certain fields occur in an addendum, that they are flagged differently. This led to the creation of our [`section_grammar.yml`](https://github.com/mc-wut/internship_files/blob/main/section_grammar.yml).

To flag *Addenda* we applied `section_grammar.yml` to the processed text before applying `variable grammar.yml`. Then the api would call `WMLUtils.section_determiner()`, to populate a field called `section:` which the only valid labels for were `Unknown` and `Addenda`. This with the mention label were produce a final, end-user visible label from a schema not included in this documentation.

```python
    def section_determiner(mns: list[Mention]) -> str:
        """section_determiner checks to see what the earliest match is for our
        section_grammar, presuming that a section name is likely to appear in the
        header or title of a doc. If there is no match in the first 200 characters,
        section-determiner returns "Unknown".
        """
        if not mns:
            return schema.SectionLabel.UNKNOWN
        else:
            # Looking at offsets
            offset_dict = {}
            for mn in mns:
                offset_dict[mn.char_start_offset] = mn.label
            key = min(offset_dict.keys())
            if key <= 200:
                return schema.SectionLabel.label_map(offset_dict[key])
            else:
                return schema.SectionLabel.UNKNOWN
```

## Final Product
The end result of the project was an API to which a pdf could be uploaded for processing and it would return a json like the following output for our file [bowman-all-extractions.pdf](/files/bowman-all-extractions.pdf). 

```json
[
  {
    "variable": "Project Owner",
    "value": "Placer County Water Agency",
    "document_name": "bowman-all-extractions.pdf",
    "section": "Unknown",
    "page": 1,
    "html": null
  },
  {
    "variable": "Project Owner",
    "value": "Placer County Water Agency",
    "document_name": "bowman-all-extractions.pdf",
    "section": "Unknown",
    "page": 1,
    "html": null
  },
  {
    "variable": "Bid Time",
    "value": "2:00 p.m.",
    "document_name": "bowman-all-extractions.pdf",
    "section": "Unknown",
    "page": 1,
    "html": null
  },
  {
    "variable": "Bid Date",
    "value": "October 25, 2023",
    "document_name": "bowman-all-extractions.pdf",
    "section": "Unknown",
    "page": 1,
    "html": null
  },
  {
    "variable": "Engineer's Estimate",
    "value": "$1,750,000",
    "document_name": "bowman-all-extractions.pdf",
    "section": "Unknown",
    "page": 1,
    "html": null
  },
  {
    "variable": "Project Duration",
    "value": "450",
    "document_name": "bowman-all-extractions.pdf",
    "section": "Unknown",
    "page": 2,
    "html": null
  },
  {
    "variable": "Project Name",
    "value": "Bowman WTP",
    "document_name": "bowman-all-extractions.pdf",
    "section": "Unknown",
    "page": 5,
    "html": null
  },
  {
    "variable": "Project Location",
    "value": "595 Christian Valley Rd,\nBowman, CA, 95602",
    "document_name": "bowman-all-extractions.pdf",
    "section": "Unknown",
    "page": 6,
    "html": null
  }
]
```


## Room for Improvement
### Efficiency
### 



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
