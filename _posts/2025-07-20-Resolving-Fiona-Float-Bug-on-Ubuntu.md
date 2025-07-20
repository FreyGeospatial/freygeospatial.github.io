---
layout: post
title:  "Resolving Float Read Issue when Using Fiona to Load File Geodatabases"
categories: [Python, Shell Scripting, Linux, GIS]
---

Recently there was an issue at work where the most recent version of the Fiona Python package failed to properly read in float columns in file geodatabases. I will demonstrate here how to resolve this issue on Ubuntu, as that is my team's primary development environment.


### Install Python

After booting from your Ubuntu instance, Install Python if you have not yet done so. For the purposes of this tutorial, I will use version 3.11.7.

```shell
sudo apt update
sudo apt upgrade

sudo apt install build-essential zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev libsqlite3-dev wget libbz2-dev

wget https://www.python.org/ftp/python/3.11.0/Python-3.11.7.tgz

tar -xf Python-3.11.7.tgz

cd Python-3.11.7

./configure --enable-optimizations

make -j$(nproc)

sudo make altinstall

sudo make clean

python3.11 --version
```

Optionally, you may also decide to alias the `python3.11` command with `python` or `py`. To do this, you must edit your `.bashrc` file. This file sets your default settings and environment variables when opening up a new bash session. Run `nano ~/.bashrc` and append `alias python='/usr/local/bin/python3.11'` to the end of the file. verify that this works by running `python --version`.

### Install UV

If you've used Python for long, you know that the language suffers from dependency management hell. There are many tools to alleviate these pains (including `venv`, `conda`, and `Poetry`) but the best one I have used is UV, which is developed by Astral, the same company that makes the `ruff` linter. UV is written in Rust, which makes it quite performant. It also instantiates venv's automatically in the current directory and will install packages there regardless of whether they have been manually activated. It also helps manage python language versions. It's CLI format is similar to Poetry's, which makes it relatively easy to port over.

There are several ways to install UV, including with `snap`, but this is what I have used, straight from Astral's documentation:

```shell
curl -LsSf https://astral.sh/uv/install.sh | sh

# restarts the shell
source $HOME/.local/bin/env bash
```

### Install GDAL

You will likely need to install GDAL, or a newer version than what you already have. Run `gdalinfo --version` to find out your current version. I recommend a GDAL version of at least 3.8.4.

For this, you will need a newer version of GDAL than what is provided by the default `apt` repository. 


```shell
sudo add-apt-repository ppa:ubuntugis/ppa
sudo apt update
sudo apt install gdal-bin libgdal-dev # python3-gdal

gdalinfo --version # 3.8.4 as of this blog post
```

This installs the newest stable version of GDAL.

### Setup your project directory

First, `cd` into the directory of your choosing. Then, let's initialize a UV-managed python project:

```shell
uv init
```

Next, let's install `Fiona`. If you install the newest version as of writing this post, there will be a bug that prevents you from reading float-type columns file geodatabases. Instead, we will download a forked version which contains a fix:

```shell
uv add "fiona @ git+https://github.com/geoscape/fiona@2a28037"
```

Like `poetry add`, this will add a specific version of fiona from the specified repository (instead of the pypi default). Notice that we specified a specific commit version.

**Note**: If you receive the GDAL error: `no such file or directory: gdal-config`. We will need to install GDAL in this case. Please go back to the previous section on installing GDAL and follow those steps.