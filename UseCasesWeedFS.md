# Moving to github https://github.com/chrislusf/weed-fs/wiki #

Different Use Cases of Seaweed-FS

# Introduction #

This page is for different Use Cases of Seaweed-FS.


# Add image with different sizes #

Each image usually store one file key in database. However, one image can have several versions, e.g., thumbnail, small, medium, large, original. And each version of the same image will have a file key. It's not ideal to store all the keys.

One way to resolve this is here.

  1. Reserve a set of file keys, for example, 5
```
> curl http://<host>:<port>/dir/assign?count=5
{"fid":"3,01637037d6","url":"127.0.0.1:8080","publicUrl":"localhost:8080","count":5}
```
  1. Save the 5 versions of the image to the volume server. The urls for each image can be:

```
 http://<url>:<port>/3,01637037d6
 http://<url>:<port>/3,01637037d6_1
 http://<url>:<port>/3,01637037d6_2
 http://<url>:<port>/3,01637037d6_3
 http://<url>:<port>/3,01637037d6_4
```

# Overwrite mime types #
The correct way to send mime type:
> curl -F "file=@myImage.png;type=image/png" http://127.0.0.1:8081/5,2730a7f18b44

The wrong way to send it:
> curl -H "Content-Type:image/png" -F file=@myImage.png http://127.0.0.1:8080/5,2730a7f18b44

# Securing Seaweed-FS #

The simple way is to front all master and volume servers with firewall.

However, if blocking servicing port is not feasible or trivial, a white list option can be used. Only traffic from the white list IP addresses have write permission.
```
 weed master -whiteList="::1,127.0.0.1"
 weed volume -whiteList="::1,127.0.0.1"
```
"::1" is for IP v6 localhost.