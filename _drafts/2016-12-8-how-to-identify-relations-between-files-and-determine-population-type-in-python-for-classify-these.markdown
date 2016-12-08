---
layout: post
title:  "Port knocking best practices"
date:   2016-12-6 12:55:28 +0100
categories: [security, linux, port-knocking]
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
