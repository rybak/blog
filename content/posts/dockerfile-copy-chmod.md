---
title: "`COPY --chmod` reduced the size of my container image by 35%"
date: 2022-03-25T00:00:00-07:00
draft: false
tags: ['containers']
---

Earlier this week, I was writing a Dockerfile to download and run a binary when I noticed the image size was way more 
than what I would expect. I'm using ubuntu:21.10 as the base image, which is about 70MB. The binary I'm running
is about 80MB. Other packages I'm installing would add 10-ish MB. But the image size is 267MB? 

Clearly I'm doing something wrong. But what? My Dockerfile is fairly simple and, what I considered, idiomatic:
```Dockerfile
FROM ubuntu:21.10 AS downloader
# Install wget, gnupg; download a zip archive; verify checksum; unzip the binary

FROM ubuntu:21.10
LABEL ...

COPY --from=downloader /bin/<binary> /bin/<binary>

RUN apt-get update && apt-get install -y openssl dumb-init iproute2 ca-certificates  \
    && rm -rf /var/lib/apt/lists/* \
    && chmod +x /bin/<binary>
    && mkdir -p <couple of empty directories> \
    && ...
...
```

I checked the history of the image to see the size of individual layers. The problem became very apparent...
```cmd
$ podman history vamc19/nomad:latest 
ID            CREATED             CREATED BY                                     SIZE        COMMENT
...    
<missing>     36 minutes ago      /bin/sh -c apt-get update && apt-get insta...  94.4 MB     
374515aec770  36 minutes ago      /bin/sh -c # (nop) COPY file:6dbfa42743cc65... 87.7 MB     
22cd380ad224  36 minutes ago      /bin/sh -c # (nop) LABEL maintainer="Vamsi"... 0 B          FROM docker.io/library/ubuntu:21.10
...
```

The layer created by `COPY` is 87.7MB, which is exactly the size of the extracted binary. So, that is normal. Why is the
layer created by `RUN` 94.4MB? What am I doing in it? I'm creating a couple of empty directories, running `chmod` on the
binary and installing 4 packages. Apt says the packages would only consume ~6MB of additional disk space and it is very
unlikely that these packages would do anything crazy post install. So, is `chmod` creating a problem?

To quickly test this, I removed `chmod` from `RUN` and rebuilt the image. And bingo - the image size is down to 174MB. 
And the `RUN` layer's size is down to 6.7MB. So, OverlayFS is copying the binary into `RUN` layer even though `chmod` is
only updating the metadata of the file...?

My understanding of CoW filesystems is very superficial - unless I write to a file, the filesystem would never 
copy the file to upper layer. And since chmod is not writing to the binary (did file's hash change?), it should not be
copied, correct? Obviously not. Honestly, I never thought about it. I looked up [OverlayFS' documentation][overlay-doc].

> When a file in the lower filesystem is accessed in a way the requires write-access, such as opening for write access, 
**_changing some metadata_** etc., the file is first copied from the lower filesystem to the upper filesystem (copy_up).

Well, I have been doing it wrong all these years. I've written a lot of Dockerfiles with shell scripts in `COPY` and 
`chmod` in `RUN`. Maybe I never realized this because these files are usually very small to make a noticeable difference 
in the image size. 

So what's the solution? In my case, since I'm using Podman (which uses Buildah), I can use `--chmod` arg with `COPY` to 
copy a file and set proper permissions in the same layer. If you are using Docker, it is [available][gh-comment] in 
BuildKit.

Note that any metadata update will lead to the same result - not just `chmod`. Both Docker and Podman already support 
`--chown` for both `COPY` and `ADD`. Maybe this should be added to the [Dockerfile Best Practices][docker-doc] page.

&nbsp;

PS: If you are wondering why a metadata update would make OverlayFS duplicate the entire file, it is for security 
reasons. You can enable ["metadata only copy up"][metadata-only-copy] feature which will only copy the metadata instead 
of the whole file.

> Do not use metacopy=on with untrusted upper/lower directories. Otherwise it is possible that an attacker can create a 
handcrafted file with appropriate REDIRECT and METACOPY xattrs, and gain access to file on lower pointed by REDIRECT. 
This should not be possible on local system as setting “trusted.” xattrs will require CAP_SYS_ADMIN. But it should be 
possible for untrusted layers like from a pen drive.


[overlay-doc]: https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html#non-directories
[metadata-only-copy]: https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html#metadata-only-copy-up
[gh-comment]: https://github.com/moby/moby/issues/34819#issuecomment-697130379
[docker-doc]: https://docs.docker.com/develop/develop-images/dockerfile_best-practices

