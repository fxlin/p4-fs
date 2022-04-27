# P4: Filesystem image forensics

We will inspect filesystem images. We will reason about the filesystem structures, extract useful information (programmatically), and verify their integrity. 

**To be completed on your own machine:** WSL, or Linux. 

Mac users: try Digital Ocean https://www.digitalocean.com/ You can have a free account. 

**Contain some exercises requiring the root privilege, which cannot be done on granger1/2. ** 

## Objective

* (primary) experience with filesystem data structures
* (primary) reverse-engineering binary data 

## Overview

This las has two phases. Each phase has its own description and assignments. 

* [Interpretation](interpretation.md): parse given ext2 images and dump their key data structures.

* [Consistency](consistency.md): check given ext2 images and report any inconsistency 

**Credits**: Derived from UCLA CS111



