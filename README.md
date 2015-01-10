# Dokku Buildpack for s3fs-fuse (FUSE on AWS S3)

This is a [buildpack](http://devcenter.heroku.com/articles/buildpacks) for FUSE on AWS S3.

## History and Status
The script forked from `znetstar/heroku-buildpack-s3fs` which seemed to be intent for heroku. The original buildpack builds both binaries properly but did not seem to be a complete solution. I was hitting permission errors on `modprobe` trying to mount a drive on heroku. I thought it could not be fixed.

I ported the script to [dokku](https://github.com/progrium/dokku) and was able to get it to "work" on [v3.0.2](https://github.com/progrium/dokku/releases/tag/v0.3.12). (see, #catches section below)

## How it Works
This buildpack downloads both fuse and s3fs-fuse projects from its project locations (sourceforge and github respectively), builds the binaries and mount a single s3 bucket with a single location.

The s3 buckets details and location are specified with these environments variables in your app:

```bash
dokku config:set << your appname >> AWS_ACCESS_KEY_ID=...        # or, S3FS_AWS_ACCESS_KEY_ID
dokku config:set << your appname >> AWS_SECRET_ACCESS_KEY=...    # or, S3FS_AWS_SECRET_ACCESS_KEY
dokku config:set << your appname >> AWS_S3_BUCKET_NAME=...       # or, S3FS_AWS_S3_BUCKET_NAME
dokku config:set << your appname >> S3FS_AWS_MOUNT_POINT=...     # must be prefixed with S3FS_
```

Then, you would need to install [multi buildpack](https://github.com/heroku/heroku-buildpack-multi). (warning: I have only tested it with and it might not work as a single buildpacks.)

After you install [multi buildpack], add this line in .buildpacks

```bash
cd << project root >>   # your local git repo
echo https://github.com/beedesk/dokku-buildpack-s3fs.git >> .buildpacks
```

Fuse and s3fs-fuse require additional privileges that need to be specified. This [docker-options](https://github.com/dyson/dokku-docker-options) plugin can be used.

```bash
cd << vagrant root >>    # 
vagrant ssh              # or, ssh -i ec2@private.pem ec2....

# once you login 
cd /var/lib/dokku/plugins
sudo git clone https://github.com/dyson/dokku-docker-options

sudo su - dokku
dokku                    # now you should see these new options displayed
                         #     docker-options:add <app> OPTIONS_STRING  add an option string an app
                         #     docker-options <app> display docker options for an app
                         
dokku docker-options:add --cap-add mknod --cap-add=sys_admin --device=/dev/fuse

                         # Note 1:
                         # See, this issue [Docker with FUSE](https://github.com/docker/docker/issues/9448)
                         # if you get need to understand the options
                         #
                         # Note 2:
                         # At this time, I was not able to change the docker-options once I added it. If
                         # you hit the same bug, ssh in and edit /home/dokku/<< app name >>/DOCKER_OPTIONS

exit                     # Exit ssh        
```

Now, push the project to deploy

```bash
cd << project root >>    # your local git repo
touch CHECKS
git add CHECKS           # See [CHECKS file](https://github.com/broadly/dokku)
git add .buildpacks
git commit -m 'Added dokku-buildpack-s3fs buildpack.'
git push dokku master    # your repo and branch might be different
```

Mounting s3fs takes a good minute for me on my local machine, so be patient.


## Catches
Mounting `FUSE` drive seemed to require root access (inside the docker container). So, it must be done before the `setuidgid` call. I was doing it on `.profile.d/sf3s.sh` which is created at the build step. I am not very confident it is a supported usage.

I have only tested it on ubuntu 14.0.? LTS.

If you see erorr building the binaries, you might need to install some of the following libs with apt-get on the host machine.

```bash
cd << vagrant root >>    # 
vagrant ssh              # or, ssh -i ec2@private.pem ec2....

sudo apt-get install daemontools
sudo apt-get install build-essential
sudo apt-get install libtool
sudo apt-get install g++ (does it included in libtool already?)
sudo apt-get install pkg-config
```
