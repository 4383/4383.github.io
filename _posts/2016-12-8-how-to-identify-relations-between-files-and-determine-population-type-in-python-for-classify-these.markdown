---
layout: post
title:  "Industrial treatment and classification of huge amount for create generate parser"
date:   2016-12-8 14:33:28 +0100
categories: [html, parsing, industrial, crawling]
---
# How to find relation between files and determine kind of population (kind of files)
In this article I want to describe an method that I have created for classify a large quantity of differents files 
and find similarities in these.

## Problem
I have a huge amount of html files and I know that these files are created with a somes templates. 
I want identify how many templates are used for generate these files for generate somes generic parser on these.
The idea is to classify these html files by population (used templates) in the first time and after I want create somes generic parser
(one by population identified) for make the job and extract data on these.

## Prerequisites
- Python programming
- have somes files html in local with somes similiraties (generated from somes templates)

Also you can directly create an websites crawler if you have identify some target that use templates on they websites.

## Setup your labs
Place all your html files in the same place example `/tmp/labs/`
So you must have a content directory like this:
```shell
$ ls /tmp/labs/
file1.html
file2.html
...
file10000.html
```
## Choose a reference file
For make the job at first you must choose a file (any of them) which will serve as a reference during our classification analyze.
Well now we can implement our algorithme !

## Identify different templates files and classify all your files from these
### Comparison algorithme
For compare files to reference file I have choose to use the python standard library.

Python is battery included so we can use the difflib module for make the job.

Example of comparison function:
```python
from difflib import SequenceMatcher

def compare(file1, file2):
  text1 = open(file1).read()
  text2 = open(file2).read()
  m = SequenceMatcher(None, text1, text2)
  result = m.ratio()
  return result
```

So you can easily have a ratio of similitary between two files.

If I compare `file1.html` to himself I obtain a ratio of `1.0`.

If I compare `file1.html` to `file2.html` I obtain a ratio of `0.35`.

If I compare `file1.html` to `file3.html` I obtain a ratio of `0.12` etc...

I let's you implement the complete script of this comparison this is not the main topic of this post.

### Presupposing a quantity of different kinds of population
Make the previous test like 50 times (easy to write script for automatize the job) and you can now observe a sequence.

Some files ratios are near from `0.35` and other `0.12` you have already two population of files.

### Classify all your files
Now modify your previous script and implement function for call the `compare` function and who copying your file into a dedicated directory (you must have the same number of used directory that the number of population previously observated).

In your script you must implement a rule that copy your compared files in the right directory that corresponding to a population.

Example:
```python
import shutil

def copy(page, directory):
    shutil.copy(page, '/tmp/classified/{0}/'.format(directory))
    print("copied to {0}".format(directory))

def classify(level, page):
  directory = "type1"
  if level > 0.18 and level <= 0.32:
    directory = "type2"
  if level > 0.32:
    directory = "type3"
  copy(page, directory)
```
Let's the job running.

### Generate generic HTML parser for all population found
Now all your files are classified you can easily create an HTML parser based on xpath to extract all your data.

Depends on your HTML source origin and source files.

### Beyond the parsing
All your files are already classified but you can easily improve similarity identification.

If you compare all the files copied into a classified directory (example test1) you can observe that the ratio increase !
So you can add more granularity in your classification and improve your parsing like CSS template detection if a image carousel are present and a lot of good stuffs !

## Conclusion
By splitting your treatment in sub sequence and observations you can easily manage a huge amount of files without similiraty at the first time analyze.

Have Fun !
