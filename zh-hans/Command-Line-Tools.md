# Command Line Tools

We have a couple of command line tools for you to experience our algorithms. With these convenient tools you don't need to write any code.

[Terark Wiki Full Documentation](https://github.com/Terark/terark-wiki-en/blob/master/tools/tools.md)

## Downloads
You can download the package from [Terark Downloads](http://www.terark.com/download/tools/latest):
- terark-fsa_all-Linux-x86_64-g++-**VERSION**-bmi2-0.tgz
  - works on older machine
- terark-fsa_all-Linux-x86_64-g++-**VERSION**-bmi2-1.tgz
  - only works on intel-haswell or better CPUs

After you download the package and unpack it, you will find these directories:

- root/bin
  - Command line tools
- root/lib
  - Libraries
- root/samples/bin
  - Examples and benchmark tools
root/samples/src
  - Code examples

## Usage
*NOTE : We've named all the tools with a suffix of `.exe`, but it's executable on Linux and MacOS.*


### Library Dependencies
Please add all the libraries from `lib` dir into your library load path.

```
# For Linux
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:LIB_DIR

# For MacOS
export DYLD_LIBRARY_PATH=$LD_LIBRARY_PATH:LIB_DIR
```

If you have multiple gcc installed, please make sure the correct version's libraries are also in the library path.



- nlt_build.exe
  - Build a Terark `Nested Succinct Trie`, TerarkDB's index is compressed by the same data structure.
  - Example:
    - ./nlt_build.exe -o outputfile inputfile.txt
    - Each line in the input file is a single key
- zbs_build.exe
  - Terark global compression, this algorithm is used for value compression.
  - Example:
    - ./zbs_build.exe -o outputfile inputfile.txt
    - Each line in the input file is a single value
- zbs_unzip.exe
  - De-compress all data or retrieve a single record from `zbs_build.exe`, can be used for benchmark
- fplcat.exe
  - Pack multiple files together then use `zbs_build.exe` to compress them. When passing the packed file to `zbs_build.exe`, you should add a `-B` parameter.
- adfa_build.exe
  - Create a `ADFA (Acyclic DFA)` from the input file, each line of the file is a `KEY`. The generated result could be used for key matching (http://nark.cc/p/?p=172).
- ac_build.exe
  - Create an AC automata from input patterns, the created AC automata can be loaded from Terark's core API
- regex_build.exe
  - Create multiple regex matching automata from the input regex collection
- pinyin_build.exe
  - Create a DFA for PinYin correction.