# distributed_shuffle
[![pypi](https://img.shields.io/pypi/v/distributed_shuffle.svg)](https://pypi.python.org/pypi/distributed_shuffle)
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/rom1504/distributed_shuffle/blob/master/notebook/distributed_shuffle_getting_started.ipynb)
[![Try it on gitpod](https://img.shields.io/badge/try-on%20gitpod-brightgreen.svg)](https://gitpod.io/#https://github.com/rom1504/distributed_shuffle)

**Not ready for usage yet**

A simple implementation of distributed shuffle, intended for learning.

Distributed shuffle is the core of algorithms such as distributed sort, group by, deduplicate, join of distributed data processing. It is employed by many frameworks including for example pyspark and ray.

Conceptually distributed shuffle is quite simple, but surprisingly known by few people. This repository is meant to teach what it is, how to implement it, and how it may be used.

## What is distributed shuffle?

The core idea of distributed shuffle is to partition the data into N groups. That's it. Partition functions include

* hash partitioning: all item of hash modulo 1 go to group 1, the ones of hash modulo 2 go to group 2, etc.
* range partitioning: all items of range 1 go to group 1, all items of range 2 go to group 2, etc.

Distributed shuffle can then be used for:
* group by, deduplicate: those consists of first using distributed shuffle with hash or range partitioning (assuring that each group is completely different from others), then applying local dedup or grouping, then each group can simply be written to the output independently
* joining: items of 2 datasets are first partitioning with hash or range partitioning, then locally joined, then each group can be written independently
* sorting: items can be partitioned using range partitioning, then locally sorting, then each group can be written independently

Note using range partitioning may need an efficient distributed sampler in order to establish ranges.

The pattern of these usage is similar:

1. distributed shuffle using the appropriate partitioning function
2. local combine
3. independent writing

Step 2 and 3 can be done fully independently from other workers which is ideal.

So as you can see, the difficult part is the distributed shuffle.

## How can we implement distributed shuffle?

The key idea of the implementation of distributed shuffle is to use N x N communication:

* the data is first split arbitrarily in N groups
* each worker will process one group, and send the data of partitioned group `i` to worker `i` which will accumulate it, and progressively write it to disk (or keep it fully in ram if it fits)

That means that each worker needs to talk to all other workers, in an efficient way. What are options here:

1. 1 to 1 tcp communication with a custom protocol
2. going through a shared file system
3. using an established reduce communication standard like MPI

Let's explore in this repo what is fast.

## Install

pip install distributed_shuffle

## Python examples

Checkout these examples to call this as a lib:
* [example.py](examples/example.py)

## API

This module exposes a single function `distribute_shuffle` which takes the same arguments as the command line tool:

* **input_file_paths** a collection of parquet files on any storage supported by fsspec. (*required*)
* **partitioner** range or hash (*required*)
* **output_dir** an output dir on any storage supported by fsspec. (*required*)

## For development

Either locally, or in [gitpod](https://gitpod.io/#https://github.com/rom1504/distributed_shuffle) (do `export PIP_USER=false` there)

Setup a virtualenv:

```
python3 -m venv .env
source .env/bin/activate
pip install -e .
```

to run tests:
```
pip install -r requirements-test.txt
```
then 
```
make lint
make test
```

You can use `make black` to reformat the code

`python -m pytest -x -s -v tests -k "dummy"` to run a specific test
