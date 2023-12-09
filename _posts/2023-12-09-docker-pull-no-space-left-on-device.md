---
layout:    post
title:    "Fixing 'no space left on device' error while docker pull"
date:     2023-12-09
comments: true
tags: programming tools 
---

## No Space Left On device

I use `docker compose` a lot.  We're setting up development environments using
docker compose files -- this makes sharing work way easier.

Lastly, I got spurious errors while performing a `docker pull` like this:

```shell
$ docker compose pull
...


write /var/lib/docker/tmp/GetImageBlob1094511197: no space left on device
```

As my system disk is always nearly full -- I must admit that I'm a bit-hoarder -- I
started to remove some older files, some VMs etc.  The error went away.  I was happy.

Today, I got the same error again.  I have cleaned out my disk -- so what the hell?

## The fix

[Googling](https://stackoverflow.com/questions/41813774/no-space-left-on-device-when-pulling-an-image#51799695)
reveals a whole set of subcommands commands of `docker` which I up until now I
did not know:

```shell
$ docker system

Usage:  docker system COMMAND

Manage Docker

Commands:
  df          Show docker disk usage
  events      Get real time events from the server
  info        Display system-wide information
  prune       Remove unused data

Run 'docker system COMMAND --help' for more information on a command.
```

Of course the `docker system df` is helpful:

```shell
$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          117       2         47.15GB   46.77GB (99%)
Containers      4         0         3.241MB   3.241MB (100%)
Local Volumes   16        0         1.363GB   1.363GB (100%)
Build Cache     61        0         6.875GB   6.875GB
```

Woot!  Docker uses 46GB for images on my system!  It even tells me that almost all of it is
"reclaimable".  Let's prune them:

```shell
$ docker image prune -a
WARNING! This will remove all images without at least one container associated to them.
Are you sure you want to continue? [y/N] y
Deleted Images:
untagged: ghcr.io/windmill-labs/windmill:main
untagged: ghcr.io/windmill-labs/windmill@sha256:9669fae32f09cddcc9013ec01a89de8773ead19d45f8c3d82a6879a9822752eb
deleted: sha256:141e2d1453f496e2159bbf00946fba03a5bd36165e0c4225ee06ddb4e65db1d8
...
deleted: sha256:d11502b17102889fedd3a37847666b7db3a49ef3f66388d4741ae8f0dcde1637
deleted: sha256:4dc9f4d5bf891a07fdd4fe4e470b3f62408ab2f73196b1848b06be0d0ec47ce8
deleted: sha256:c8716a1abe24b946178fbf63c8c805816bf16283f4eb43677edb699cdd819a50
deleted: sha256:39a8ca38b9480bd5c653ae3fdc048df1907924cefcbadce26132b00e78580327
deleted: sha256:79adfbf18ada127431a5ba9edcee430148ff9741183bbc61c0589054f7ccfd7d
deleted: sha256:7cd52847ad775a5ddc4b58326cf884beee34544296402c6292ed76474c686d39

Total reclaimed space: 33.72GB
```

Aha!

```
$ docker system df

TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          0         0         0B        0B
Containers      0         0         0B        0B
Local Volumes   16        0         1.363GB   1.363GB (100%)
Build Cache     61        0         6.976GB   6.976GB
```

This -- of course -- is not magic.  The `docker image prune -a` will remove all
*unused* images.  As I've no container running, this means *all* images.  This in
turn means, that a `docker pull` will need to pull all layers again.

The [manual](https://docs.docker.com/engine/reference/commandline/image_prune/)
has some additional expanation and some tips on how to exactly know what a
`docker prune` will be removing.

## Conclusion

> Know thy tools!
{: .prompt-tip }

This is what I keep telling people.  It seems I'd better follow my own tips and 
read up on `docker` a bit more ...


## Links

- [Stackoverflow topic](https://stackoverflow.com/questions/41813774/no-space-left-on-device-when-pulling-an-image#51799695) 
- [docker image prune](https://docs.docker.com/engine/reference/commandline/image_prune/)
