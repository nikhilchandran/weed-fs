# Moving to github https://github.com/chrislusf/weed-fs/wiki #

list of actual WeedFS usage

# Introduction #

Please comment on this page, to put your actual use case. I will compile it periodically into the wiki page itself.

# Details #

| User | Content |Replication | Write |Read|
|:-----|:--------|:-----------|:------|:---|
|yiyantang.com |html files (compressed)|  |100+ file/second|depends on the volume server throughout|
| Karol |7000 files (jpg/png) - random access| 2 on same rack |  |7012.78 file/sec, 100.39 MB/sec, Concurrency:282.40|
|ocapic.com| millions of images(each user has 15000 images)| multiple data centers |  |  |
|your company| millions of images| No | xxx |xxx |