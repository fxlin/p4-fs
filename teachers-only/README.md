# HOW TO GRADE

## Generate the test filesystem image. Use: EXT2_test.script  

* Follow step 0 in docs/lite.md to create & mount a blank file system image
* (Modify the above script to customize the test filesystem image)
* Run this script to generate files & dirs
* Unmount the filesystem 

## Generate the reference output 

* Run the reference implementation on the test image. 

## Test harness (TBD)

* Build student's program, run it, collect output, and diff output vs. the reference output
* Depending on the diff outcome, generate statistics (# correctness, # mistakes, # missing)
* The test harness should take into account: the lines in the output are unordered. 