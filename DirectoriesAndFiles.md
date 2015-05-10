# Moving to github https://github.com/chrislusf/weed-fs/wiki #

#summary more traditional Directories and Files support

# Introduction #

When talking about file systems, many people would assume directories, list files under a directory, etc. These are expected if we want to hook up Seaweed File System with linux by FUSE, or with Hadoop, etc.

## Sample usage ##
Two ways to start a weed filer
```
// assuming you already started weed master and weed volume
> weed filer
// Or assuming you have nothing started yet, this command starts master server, volume server, and filer in one shot. 
// It's strictly the same as starting them separately.
> weed server -filer=true
```

Now you can add/delete files, and even browse the sub directories and files
```
//POST a file and read it back
> curl -F "filename=@README.md" "http://localhost:8888/path/to/sources/"
> curl "http://localhost:8888/path/to/sources/README.md"
//POST a file with a new name and read it back
> curl -F "filename=@Makefile" "http://localhost:8888/path/to/sources/new_name"
> curl "http://localhost:8888/path/to/sources/new_name"
//list sub folders and files
> curl "http://localhost:8888/path/to/sources/?pretty=y"
// if lots of files under this folder, here is a way to efficiently paginate through all of them
> curl "http://localhost:8888/path/to/sources/?lastFileName=abc.txt&limit=50&pretty=y"
```

# Design #
A common file system would use inode to store meta data for each folder and file. The folder tree structure are usually linked. And sub folders and files are usually organized as an on-disk b+tree or similar variations. This scales well in terms of storage, but not well for fast file retrieval due to multiple disk access just for the file meta data, before even trying to get the file content.

Seaweed-FS wants to make as small number of disk access as possible, yet still be able to store a lot of file metadata. So we need to think very differently.


From a full file path to get to the file content, there are several steps:
  1. file\_parent\_directory => directory\_id
  1. directory\_id+fileName => file\_id
  1. file\_id => data\_block

Because default Seaweed-FS only provides file\_id=>data\_block mapping, the first 2 steps need to be implemented.

There are several data features I noticed:
  1. the number of directories usually is small, or very small
  1. the number of files can be small, medium, large, or very large
This leads to a novel (as far as I know now) approach to organize the meta data for the directories and files separately.

A "weed filer" server is to provide these two missing parent\_directory=>directory\_id, and directory\_id+filename=>file\_id mappings, completing the "common" file storage interface.

## Assumptions ##
I believe these are reasonable assumptions:
  1. The number of directories are smaller than the number of files by one or more magnitudes.
  1. Very likely for big systems, the number of files under one particular directory can be very high, ideally unlimited, far exceeding the number of directories.
  1. Directory meta data is accessed very often.

## Data Structure ##
This difference lead to the design that the metadata for directories and files should have different data structure.
  1. Store directories in memory
    1. all of directories hopefully all be in memory
    1. efficient to move/rename/list\_directories
  1. Store files in a sorted string table in <dir\_id/filename, file\_id> format
    1. efficient to list\_files, just simple iterator
    1. efficient to locate files, binary search

## Complexity ##
For one file retrieval, if the parent directory includes n folders, then it will take n steps to navigate from root to the file folder. However, this O(n) step is all in memory. So in practice, it will be very fast.

For one file retrieval, the dir\_id+filename=>file\_id lookup will be O(logN) using LevelDB, a log-structured-merge (LSM) tree implementation. The complexity is the same as B-Tree.

For file listing under a particular directory, the listing in LevelDB is just a simple scan, since the record in LevelDB is already sorted. For B-Tree, this may involves multiple disk seeks to jump through.

For directory renaming, it's just trivially change the name or parent of the directory. Since the directory\_id stays the same, there are no change to files metadata.

For file renaming, it's just trivially delete and then add a row in leveldb.

# Details #

In the current first version, the path\_to\_file=>file\_id mapping is stored with an efficient embedded leveldb. Being embedded, it runs on single machine. So it's not linearly scalable yet. However, it can handle LOTS AND LOTS of files on Seaweed-FS on other servers. Using an external distributed database is possible. Your contribution is welcome!

The in-memory directory structure can improve on memory efficiency. Current simple map in memory works when the number of directories is less than 1 million, which will use about 500MB memory. But I would highly doubt any common use case would have more than 100 directories.

# Use Cases #

Clients can assess one "weed filer" via HTTP, list files under a directory, create files via HTTP POST, read files via HTTP POST directly.

Although one "weed filer" can only sits in one machine, you can start multiple "weed filer" on several machines, each "weed filer" instance running in its own collection, having its own namespace, but sharing the same Seaweed-FS.

# Future #

In future version, the parent\_directory=>directory\_id, and directory\_id+filename=>file\_id mappings will be refactored to support different storage system.

The directory meta data may be switched to some other in-memory database.

The LevelDB implementation may be switched underneath to external data storage, e.g. MySQL, TokyoCabinet, etc. Preferably some pure-go implementation.

Also, a HA feature will be added, so that multiple "weed filer" instance can share the same set of view of files.

Later, FUSE or HCFS plugins will be created, to really integrate Seaweed-FS to existing systems.

# Helps Wanted #

This is a big step towards more interesting Seaweed-FS usage and integration with existing systems.

If you can help to refactor and implement other directory meta data, or file meta data storage, please do so.