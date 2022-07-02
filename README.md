## Workshop/tutorial on EFI's cloud-native forecasting workflow


_Why do you want me to replace all my `read_*` functions with something else??_


Cloud native workflows access data directly over the network interfaces, rather than first being downloaded to the local hard-disk and read in from there. 
While this approach can run on any machine with network access, it is particularly efficient in contexts with high network bandwidth, such as cloud computing centers.
Cloud-native approaches allow us to work with big data: specifically, data that is _larger-than-ram_ and even _larger-than-disk_ of the machine performing the computing. 
This also makes th strategy compatible with small, distributed computing tools such as continuous integration.

## Introduction

In fall 2021, a popular [article in Verge](https://www.theverge.com/22684730/students-file-folder-directory-structure-education-gen-z) made the rounds bemoaning the fact that kids these days don't know how to navigate files and folders on their computer. The article chalks this up to advent of search engines and the demise of grey steel file drawers, which I think rather misses the mark. Cloud-native software doesn't use this metaphor either. The idea that data on a computer lives in a set of nested folders is not a fundamental fact of computer architecture but a convenient mental abstraction created in the 1980s to provide a standardized mechanism for software programs to access data: [POSIX filesystem standard](https://www.quobyte.com/storage-explained/posix-filesystem). 


## Range requests

A computer can only operate on data that it has read into working memory (RAM).
When individual data files or objects are too large to read in all at once (e.g. larger then several GB for most machines), software must operate on the data in sequential chunks. 
Software can ask for a particular range of bytes to be read into memory. 
If the data is appropriately structured, software can access only the range of bytes required -- e.g. a particular range of dates or locations from a large table -- without ever having to access data outside of that range. 

## What about databases?

The core capabilities here are generally true of relational databases as well, with two important differences.
While a remote database requires a powerful central server -- i.e. a fully fledged computer instance -- which receives and execute the requests.  In this case the data host is purely _static_, essentially just a storage interface. 
Computational processing and memory are provided by the client machine. 
This can be slower, but it is much more cost-effective and much more scalable -- many clients can access the same static data far more efficiently.  

## Is this just about R?

The techniques discussed here apply equally to any modern computing language. The underlying libraries that provide the functionality we will use: Apache Arrow and GDAL, are written primarily in C++, and clients exist which allow users to access these functions from R, Python, javascript, and most other major languages. While clients can differ in syntax and even certain features, they are broadly compatible. Data designed for these workflows is thus easily compatible with any of these languages.


## Are cloud object stores required?

Technically no. 
Almost all web servers today support range requests, meaning that anywhere you can host a static file (e.g. GitHub, or a scientific data repository such as Zenodo, for instance), you can perform a range request. 
However, not all clients have built in support for HTTP range requests at this time.
Our examples using `arrow` to subset from Parquet files from R will require the data be in an S3-compliant object store.
However, the spatial examples will work with a wide range of network protocols, including HTTPS and even FTP, and include the ability to peek inside and extract content from `.zip` or `tar` archives directly over a network connection.
`duckdb` also has the ability to access and subset parquet content directly over any HTTP connection, though this is not yet easily available in the R client.

In addition to range requests, object stores provide a host of common functions comparable to those provided by a local POSIX filesystem, such as the ability to query certain metadata (e.g. creation and last modified time, ETags, user access/write permissions, copy/rename/delete operations, etc). S3-compliant object stores can optionally support additional features such as data lifecycle and version control, and much else.
In practice, object stores are particularly popular in contexts where secure, authenticated access is required. 
Because most object stores are hosted by large data centers, these sources generally provide very high maximum download speeds, especially when data is accessed from virtual machines running in the same region (or same data center).


### Acknowledgements

This material is based upon work supported by the National Science Foundation under Grant DBI-1942280. Any opinions, findings, and conclusions or recommendations expressed in this material are those of the author(s) and do not necessarily reflect the views of the National Science Foundation.


