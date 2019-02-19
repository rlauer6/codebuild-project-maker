# README

This is the README for the `codebuild-project-maker` project.

# Background

The purpose of this project is to provide a script (`cbp`) that
creates a CodeBuild project that __only contains a buildspec__.  It
was born of a desire for a relatively simple way to build a version of
the perl interpreter from source.  So, after creating this
`buildspec.yml`...

```
version: 0.2
phases:
  install:
    commands:
      - yum install -y gcc patch wget aws-cli tar gzip make zip
  pre_build:
    commands:
      - wget http://www.cpan.org/src/5.0/perl-$PERL_VERSION.tar.gz -O perl-$PERL_VERSION.tar.gz
  build:
    commands:
      - tar xfvz perl-$PERL_VERSION.tar.gz
      - cd perl-$PERL_VERSION && ./Configure -des -Dprefix=/opt/perl-$PERL_VERSION -Dman1dir=none -Dman3dir=none
      - make && make install DESTDIR=/tmp
artifacts:
  base-directory: '/tmp'
  files:
    - '**/*'
  name: perl-$PERL_VERSION.zip
```

...and seeing it, much to my surprise, actually work, it made sense to
codify the research I did in a bash script.

# Use Case

The purpose of the project is to provide a way to create CodeBuild
projects that build artifacts from a `buildspec.yml` file and make
them available in an S3 bucket. There are times when you have no
source code or the source code is not in a repository that is directly
supported by CodeBuild. CodeBuild supports the
following source repositories:

* CodeCommit
* GitHub
* GitHub Enterprise
* S3
* Bitbucket

Fortunately, there is also a "NO_SOURCE" option that allows you to
simply apply a build a recipe that creates artifacts.  In fact,
CodeBuild could be used as a way to execute some bits of code
triggered by CloudWatch events including scheduled events (think batch
jobs). This script however focuses on simply running a recipe - how
you trigger it is up to you.

The script does not try to provide options for everything you can do
with CodeBuild, but just enough to initiate a generic build that
creates some artifacts.

# Additonal Notes

By default, the container used to execute the recipe is the
__amazonlinux__ Docker image that mimics (for the most part) the image
used by Lambdas.  You can override that by providing an option (-c)
when you create the project.

By default, the project will configure the CodeBuild project to write
your artifacts to the S3 bucket you specify with the bucket prefix as
the project name. You can override the bucket and bucket prefix with
the -b and -k options.  See below for a description of the options.

## cpb Options

* `-c`

  The default Docker image used for the build is the
  __amazonlinux__ image which mimics the same environment used by
  Lambda. You can overrride that with this option.
* `-d`

  Provide a description for your project. If no description is
  provided, then the description will become the project name.
* `-D`

  Use this option (dryrun) to just echo the commands that will be executed.
* `-p`

  Project name.
* `-P`

  If your project needs to run in privileged mode provide this option.
* `-b`

  Name of the S3 bucket that will contain the artifacts of the build.
* `-k`

  Prefix name for storing your artifacts.  If not provided, the project
  name will be used as the bucket prefix.
* `-B`

  By default the buildspec file name is `buildspec.yml` however you
  can override that with this option.
* `-t`

  Compute type (small, medium, large). Default is small.

# Creating a CodeBuild Project

In addition to creating a `buildspec.yml` file, to create a CodeBuild
project you need to follow these steps:

1. create an IAM service role for CodeBuild to assume
1. attach a policy that allows the role to write to CloudWatch logs
and an S3 bucket
1. create a CodeBuild project

_Obviously, in order to create roles and policies you'll need the
ability to make IAM API calls._

You should be prepared to provide:

* an S3 bucket name that you have provisioned
* a `buildspec.yml` file
* a project name

Assuming you have your `buildspec.yml` file ready to roll...to create
the project, follow this recipe.

1. `cbp -p project-name create-role`
1. `cpb -p -b my-bucket-name create-policy`
1. `cpb -p -b my-bucket-name create-project`

To start a build:

```
cbp -p project-name start-build name val ...
```

Where `name` is the name of an environment variable and `val` is the
value of the variable.  You can provide as many environment variable
as necessary.

# Notes

* Originally, the script created the role, policy and CodeBuild project all at the
same time, however IAM policies sometimes take a bit of time to
propagate and the project creation step would fail.

See this link.

https://forums.aws.amazon.com/thread.jspa?threadID=255063

# Copyright

Copyright (c) 2019 Robert C. Lauer.  All rights reserved.

This is free software and may be used without charge for any purpose.
