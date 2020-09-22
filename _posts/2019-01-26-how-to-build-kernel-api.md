---
layout: post
title: How to build the documentation of Linux kernel API
description: How to build the documentation of Linux kernel API
summary: How to build the documentation of Linux kernel API
tags: [doc]
---


Step 0. [download](https://www.kernel.org/) kernel source

Step 1. uncompress kernel source and go to its root

Step 2. sudo apt install gcc make

Step 3. sudo apt install python-sphinx

Step 4. sudo apt install texlive-xetex

Step 5. sudo apt install python-pip

Step 6. pip install sphinx_rtd_theme

Step 7. sudo apt install graphviz

Step 8. sudo apt install imagemagick

Step 9. make htmldocs

Step 10. also, you can try *make pdfdocs*, *make epubdocs*, *make xmldocs* or *make latexdocs* as you want

Step 11. when finished, documentation you built is located at Documentation/output/

Step 12. enjoy it
