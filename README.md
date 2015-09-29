# Dokku Buildpack for s3fs-fuse (FUSE on AWS S3)

This is a [buildpack](http://devcenter.heroku.com/articles/buildpacks) for FUSE on AWS S3.

## History and Status
The buildpack is forked from `znetstar/heroku-buildpack-s3fs`, which apparently written for heroku. Znetstar's built both fuse and s3fs-fuse binaries properly but did not mount the filesystem. On heroku, I was hitting permission errors related to `modprobe` trying to mount a filesystem. As of writting, `fuse` capacity is not compiled into heroku host OS, or it was not exposed to its app container. I suspected the problem could not be fixed by heroku user, until Heorku decided to enable it. 

I ported the script to [dokku](https://github.com/progrium/dokku) and was able to get it to work, with [dokku v0.3.26](https://github.com/progrium/dokku/releases/tag/v0.3.26). (see, #catches section below)

## How it Works
This buildpack downloads both fuse and s3fs-fuse projects from its project locations (sourceforge and github respectively), builds the binaries and mount a single s3 bucket with a single location.

The s3 buckets details and location are specified with these environments variables in your app:

```bash
dokku config:set << your appname >> AWS_ACCESS_KEY_ID=...        # or, S3FS_AWS_ACCESS_KEY_ID
dokku config:set << your appname >> AWS_SECRET_ACCESS_KEY=...    # or, S3FS_AWS_SECRET_ACCESS_KEY
dokku config:set << your appname >> AWS_S3_BUCKET_NAME=...       # or, S3FS_AWS_S3_BUCKET_NAME
dokku config:set << your appname >> S3FS_AWS_MOUNT_POINT=...     # must be prefixed with S3FS_
```

Optionally, mounting options can be added with these enviorments variables:

```bash
dokku config:set << your appname >> S3FS_CACHE_DIR               # default is "/tmp/s3fs"
dokku config:set << your appname >> S3FS_MOUNT_OPTIONS           # default is "-o allow_other -o use_cache=${S3FS_CACHE_DIR}"
```

This buildpack assumes [multi buildpack](https://github.com/heroku/heroku-buildpack-multi) environments, and might not work as a single buildpacks. If you have not already installed the multi buildpack, please follow links and add multi buildpack first.

With [multi buildpack] installed, add this line in .buildpacks

```bash
cd << project root >>   # your local git repo
echo https://github.com/beedesk/dokku-buildpack-s3fs.git >> .buildpacks
```

Fuse and s3fs-fuse require additional privileges that need to be specified. Earlier versions of dokku (such as v3.0.11) requires  [docker-options](https://github.com/dyson/dokku-docker-options) plugin. Newer versions (such as v3.0.26) has it builtin. The syntax is slightly different. The text below is for newer version.

```bash
dokku docker-options:add << app name >> "deploy --privileged --cap-add=MKNOD --cap-add=SYS_ADMIN --device=/dev/fuse"
dokku docker-options:add << app name >> "run --privileged --cap-add=MKNOD --cap-add=SYS_ADMIN --device=/dev/fuse"

                         # Note 1:
                         # See, this issue [Docker with FUSE](https://github.com/docker/docker/issues/9448)
                         # if you get need to understand the options
```

Now, follow these steps to deploy the project

```bash
cd << project root >>    # your local git repo
touch CHECKS
git add CHECKS           # See [CHECKS file](https://github.com/broadly/dokku)
git add .buildpacks
git commit -m 'Added dokku-buildpack-s3fs buildpack.'
git push dokku master    # your repo and branch might be different
```

On a non-EC2 machine, mounting might take a good minute, so be patient. On a EC2, mounting takes only a few seconds.

## Catches
Mounting `FUSE` drive requires root access (within the docker container). The `.profile.d/` step of deploy / run command in [buildstep](https://github.com/progrium/buildstep) happened to run as root (within the docker container). It is why this buildstep mounts the filesytem with `.profile.d/s3fs.sh` script generating during `bin/compile` phrase.

I have only tested it on ubuntu 14.0 LTS with Docker version 1.7.1, build 786b29d.

If you see an error building the binaries, you might need to install some of the following libs with apt-get on the host machine.

```bash
cd << vagrant root >>    # 
vagrant ssh              # or, ssh -i ec2@private.pem ec2....

sudo apt-get install daemontools
sudo apt-get install build-essential
sudo apt-get install libtool
sudo apt-get install g++ (does it included in libtool already?)
sudo apt-get install pkg-config
```
