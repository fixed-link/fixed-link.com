---
layout: post
title: How to build the documentation of Linux kernel API
description: How to build the documentation of Linux kernel API
summary: How to build the documentation of Linux kernel API
tags: [doc]
---


# 如何编译内核API文档

0. [download](https://www.kernel.org/) kernel source

1. uncompress kernel source and go to its root

2. sudo apt install gcc make

3. sudo apt install python-sphinx

4. sudo apt install texlive-xetex

5. sudo apt install python-pip

6. pip install sphinx_rtd_theme

7. sudo apt install graphviz

8. sudo apt install imagemagick

9. make htmldocs

10. also, you can try *make pdfdocs*, *make epubdocs*, *make xmldocs* or *make latexdocs* as you want

11. when finished, documentation you built is located at Documentation/output/

12. enjoy it
