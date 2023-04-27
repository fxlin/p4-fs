# P4: Filesystem image forensics

The code location is at: https://github.com/fxlin/p4-fs

We will inspect filesystem images. We will reason about the filesystem structures, extract useful information (both using existing tools and programmatically), and verify their integrity. 

## Environment 

**Contain some exercises requiring the root privilege, which cannot be done on granger1/2.** 

**To be completed on your own machine:** WSL, or Linux. 

Mac users: use free VM service. Try one of the following solutions: 

* AWS free tier https://aws.amazon.com/free
* Digital Ocean https://www.digitalocean.com/ You can have a free account. 
* A Ubuntu VM on VMWare fusion, free to download as a student

## Objective

* (primary) experience with filesystem data structures
* (primary) reverse-engineering binary data 

## Overview

This project has two experiments. Each has its own description and assignments. 

* [exp1: Interpretation](interpretation.md): parse given ext2 images and dump their key data structures.
* [exp2: Consistency](consistency.md): check given ext2 images and report any inconsistency 

**NOTE: depending on semester, students may do exp1 only, or exp1+exp2. Always refer to the syllabus.**

## Files

ext2_fs.h: the header file that contains many ext2 defs

trivial.img, trivial.csv: a 64KB ext2 image and its sample dump 

EXT2_test.img, EXT2_test.csv: a 1MB ext2 image and its sample dump 


## Credits
Derived from UCLA CS111



