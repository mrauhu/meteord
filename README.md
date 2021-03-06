[![Circle CI](https://circleci.com/gh/mrauhu/meteord/tree/master.svg?style=svg)](https://circleci.com/gh/mrauhu/meteord/tree/master)

# MeteorD - Docker Runtime for Meteor Apps

> Docker image for painless deployment from Windows host to Linux server via [Meteor Up](https://github.com/zodern/meteor-up).

## How to use with Meteor Up

1. Change value of the field `docker.image` in file `mup.js` to `mrauhu/meteord:base`.
2. Run `mup deploy`.

## Alternative docker images

* `mrauhu/meteord:node-4.8.4-base` — for Meteor 1.5
  * Node.js 4.8.4
  * [`@mrauhu/npm@4.6.1`](https://github.com/mrauhu/npm/tree/v4-scoped-package-for-meteord)
* `mrauhu/meteord:node-8.4.1-base` — for Meteor 1.6
  * Node.js 8.1.4
  *  [`@mrauhu/npm@5.3.0`](https://github.com/mrauhu/npm/tree/scoped-package-for-meteord)

## Info 

This Dockerfile have scoped NPM package `@mrauhu/npm` for resolve nasty bug with windows-like paths and `npm rebuild`. Without this fix Windows user cannot use native Node.js modules, like `bcrypt`, in Meteor application and making deployments to to Linux server. See details here:

* Fix: normalize windows-like slashes inside `npm rebuild` and magical cure for Meteor Up deployment with `bcrypt` etc. 
https://github.com/npm/npm/pull/17802

Dockerfile based on: https://github.com/abernix/meteord

---


**MeteorD supports these two ways. Let's see how to use MeteorD**

### 1. Build a Docker image for your app

With this method, your app will be converted into a Docker image. Then you can simply run that image.  

For that, you can use `meteorhacks/meteord:onbuild` as your base image. Magically, that's only thing you have to do. Here's how to do it:

Add following `Dockerfile` into the root of your app:

~~~shell
FROM meteorhacks/meteord:onbuild
~~~

Then you can build the docker image with:

~~~shell
docker build -t yourname/app .
~~~

Then you can run your meteor image with

~~~shell
docker run -d \
    -e ROOT_URL=http://yourapp.com \
    -e MONGO_URL=mongodb://url \
    -e MONGO_OPLOG_URL=mongodb://oplog_url \
    -p 8080:80 \
    yourname/app
~~~
Then you can access your app from the port 8080 of the host system.

#### Stop downloading Meteor each and every time (mostly in development)

So, with the above method, MeteorD will download and install Meteor each and every time. That's bad especially in development. So, we've a solution for that. Simply use `meteorhacks/meteord:devbuild` as your base image.

> WARNING: Don't use `meteorhacks/meteord:devbuild` for your final build. If you used it, your image will carry the Meteor distribution as well. As a result of that, you'll end up with an image with ~700 MB.

### 2. Running a Meteor bundle with Docker

For this you can directly use the MeteorD to run your meteor bundle. MeteorD can accept your bundle either from a local mount or from the web. Let's see:

#### 2.1 From a Local Mount

~~~shell
docker run -d \
    -e ROOT_URL=http://yourapp.com \
    -e MONGO_URL=mongodb://url \
    -e MONGO_OPLOG_URL=mongodb://oplog_url \
    -v /mybundle_dir:/bundle \
    -p 8080:80 \
    meteorhacks/meteord:base
~~~

With this method, MeteorD looks for the tarball version of the meteor bundle. So, you should build the meteor bundle for `os.linux.x86_64` and put it inside the `/bundle` volume. This is how you can build a meteor bundle.

~~~shell
meteor build --architecture=os.linux.x86_64 ./
~~~

#### 2.1 From the Web

You can also simply give URL of the tarball with `BUNDLE_URL` environment variable. Then MeteorD will fetch the bundle and run it. This is how to do it:

~~~shell
docker run -d \
    -e ROOT_URL=http://yourapp.com \
    -e MONGO_URL=mongodb://url \
    -e MONGO_OPLOG_URL=mongodb://oplog_url \
    -e BUNDLE_URL=http://mybundle_url_at_s3.tar.gz \
    -p 8080:80 \
    meteorhacks/meteord:base
~~~

#### 2.2 With Docker Compose

docker-compose.yml
~~~shell
dashboard:
  image: yourrepo/yourapp
  ports:
   - "80:80"
  links:
   - mongo
  environment:
   - MONGO_URL=mongodb://mongo/yourapp
   - ROOT_URL=http://yourapp.com
   - MAIL_URL=smtp://some.mailserver.com:25

mongo:
  image: mongo:latest
~~~

When using Docker Compose to start a Meteor container with a Mongo container as well, we need to wait for the database to start up before we try to start the Meteor app, else the container will fail to start.

This sample docker-compose.yml file starts up a container that has used meteorhacks/meterod as its base and a mongo container. It also passes along several variables to Meteor needed to start up, specifies the port number the container will listen on, and waits 30 seconds for the mongodb container to start up before starting up the Meteor container.

#### Rebuilding Binary Modules

Sometimes, you need to rebuild binary npm modules. If so, expose `REBUILD_NPM_MODULES` environment variable. It will take couple of seconds to complete the rebuilding process.

~~~shell
docker run -d \
    -e ROOT_URL=http://yourapp.com \
    -e MONGO_URL=mongodb://url \
    -e MONGO_OPLOG_URL=mongodb://oplog_url \
    -e BUNDLE_URL=http://mybundle_url_at_s3.tar.gz \
    -e REBUILD_NPM_MODULES=1 \
    -p 8080:80 \
    meteorhacks/meteord:binbuild
~~~

## Known Issues

#### Spiderable Not Working (But, have a fix)

There are some issues when running spiderable inside a Docker container. For that, check this issue: https://github.com/meteor/meteor/issues/2429

Fortunately, there is a fix. Simply use [`ongoworks:spiderable`](https://github.com/ongoworks/spiderable) instead the official package.

#### Container won't start on Joyent's Triton infrastructure

There's currently (2015-07-18) an issue relating to how the command or entry point is parsed, so containers won't boot using the 'docker run' commands as above.

Instead, Joyent support has suggested the following workaround until their fix can be rolled out.

~~~shell
docker run -d \
    -e ROOT_URL=http://yourapp.com \
    -e MONGO_URL=mongodb://url \
    -e MONGO_OPLOG_URL=mongodb://oplog_url \
    -p 80:80 \
    --entrypoint=bash \
    yourname/app \
    /opt/meteord/run_app.sh
~~~
