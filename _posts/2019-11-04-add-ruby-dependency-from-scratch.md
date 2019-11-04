---
layout:     post
title:      "Customize Docker file with Rails Dependency"
subtitle:   "Install "
date:       2019-11-04
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Docker
    - Rails
---

> "Building a Docker image with Ruby and Rails"

Recently I was working on a Rails project, the project uses Docker to build an image and deployed on somewhere. For some reason I can only use a private docker image instead of a Ruby docker image(although there are official image for Ruby), and the private image was based on CentOS docker image.
So here's my solution to install Ruby and Rails on this private image. I'll use CentOS image in the following Docker file.

- Install pre requirement dependencies
We'll compile and install Ruby inside the container and since Ruby source code is writing in C and we need to install C compiler like GCC, I installed a group of development tools with command
```bash
$ yum -y groupinstall "Development Tools"
```
for now the Dockerfile is
```docker
FROM centos:latest

# install development tools
RUN yum -y update 
RUN yum -y groupinstall "Development Tools" 
RUN yum -y install openssl-devel
```
- Install Ruby and Gem
We need to download Ruby source code from official site, unzip, compile and install. I use ruby 2.6.0 on my project, this part of Dockerfile is
```docker
RUN version=2.6.0 
RUN cd /usr/local/src 
RUN wget https://cache.ruby-lang.org/pub/ruby/2.6/ruby-$version.tar.gz
RUN tar zxvf ruby-$version.tar.gz
RUN cd ruby-$version
RUN ./configure
RUN make
RUN make install
```
Gems are installed by a setup script under it's source code, so after Ruby is successfully installed, we can download the gems source code and install
```docker
RUN version=3.0.1
RUN cd /usr/local/src
RUN https://rubygems.org/rubygems/rubygems-$version.tgz
RUN tar zxvf rubygems-$version.tgz
RUN cd rubygems-$version
RUN /usr/local/bin/ruby setup.rb
```
I plan to install gems version 3.0.0. but got "permission denied" error, this is a bug and fixed in gems 3.0.1, this is the related [github issue](https://github.com/rubygems/rubygems/issues/2535)

- Install other dependencies
    - Node.js and Yarn
    ```docker
    RUN yum -y install nodejs

    RUN curl -o- -L https://yarnpkg.com/install.sh | bash
    ```

    - postgres
    The project use postgres database and depend on pg library, but it's not work for the first time and has an error message
    > An error occurred while installing pg (0.18.1), and Bundler cannot continue. Make sure that "gem install pg -v '0.18.1" succeeds before bundling.

    This problem is fixed by install postgresql-devel package
    ```docker
    RUN yum -y install postgresql-devel
    ```
 - Build and config project
 The rest part is to build the project.


### Conclusion
In this article we introduced how to add Ruby dependencies from scratch, the full version of Dockerfile is
```docker
FROM centos:latest

RUN yum -y update && yum -y groupinstall "Development Tools" && yum -y install openssl-devel

RUN version=2.6.0 && cd /usr/local/src && wget https://cache.ruby-lang.org/pub/ruby/2.6/ruby-$version.tar.gz \
  && tar zxvf ruby-$version.tar.gz \
  && cd ruby-$version \
  && ./configure \
  && make \
  && make install

# ruby-gems
RUN version=3.0.1 && cd /usr/local/src && wget https://rubygems.org/rubygems/rubygems-$version.tgz \
  && tar zxvf rubygems-$version.tgz \
  && cd rubygems-$version \
  && /usr/local/bin/ruby setup.rb

RUN yum -y install nodejs

RUN curl -o- -L https://yarnpkg.com/install.sh | bash

RUN yum -y install postgresql-devel

WORKDIR /app

ADD Gemfile Gemfile.lock ./

RUN bundle config mirror.https://rubygems.org \
  && bundle config --global frozen 1 \
  && bundle install --without development test -j4 --retry 3

COPY . .

RUN echo 'randome code' > config/master.key

RUN bin/rake assets:precompile

RUN groupadd -g 100001 mygroup && \
    useradd -r -m -g mygroup -u 100001 projuser
    
EXPOSE 3000

RUN chown -R projuser:mygroup /app

USER mygroup
```

