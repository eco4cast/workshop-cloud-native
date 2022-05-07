## Workshop/tutorial on EFI's cloud-native forecasting workflow


_Why do you want me to replace all my `read_*` functions with something else??_

### What is cloud-native?

Cloud native workflows access data directly over the network interfaces, rather than first being downloaded to the local hard-disk and read in from there. While this approach can run on any machine with network access, it is particularly efficient in contexts with high network bandwidth, such as cloud computing centers.


In fall 2021, a popular [article in Verge](https://www.theverge.com/22684730/students-file-folder-directory-structure-education-gen-z) made the rounds bemoaning the fact that kids these days don't know how to navigate files and folders on their computer. The article chalks this up to advent of search engines and the demise of grey steel file drawers, which I think rather misses the mark. Cloud-native software doesn't use this metaphor either. The idea that data on a computer lives in a set of nested folders is not a fundamental fact of computer architecture but a convenient mental abstraction created in the 1980s to provide a standardized mechanism for software programs to access data: [POSIX filesystem standard](https://www.quobyte.com/storage-explained/posix-filesystem). 


## Range requests

A computer can only operate on data that it has read into working memory (RAM).
When individual data files or objects are too large to read in all at once (e.g. larger then several GB for most machines), software must operate on the data in sequential chunks. 
Software can ask for a particular range of bytes to be read into memory. 
If the data is appropriately structured, software can access only the range of bytes required -- e.g. a particular range of dates or locations from a large table -- without ever having to access data outside of that range. 

## What about databases?

