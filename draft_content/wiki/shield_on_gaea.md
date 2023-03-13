---
title: SHiELD on Gaea
author: Tristan Abbott
categories:
- models
---

# Running SHiELD on Gaea

Running SHiELD is *much* easier than running AM4---maybe unsurprisingly, since the "physics" are much simpler.

## Compiling

I store SHiELD source code and compiled binaries on `/lustre/f2/dev/${USER}`. This directory is not scrubbed (unlike `/lustre/f2/scratch/$USER`) and is not subject to the same 5GB quota as `home` directories. So, to compile the default executable, `cd` to `/lustre/f2/dev/${USER}` and type

```console
$ git clone https://github.com/NOAA-GFDL/SHiELD_build
$ cd SHiELD_build/
$ git submodule update --init --recursive # mkmf submodule
$ ./CHECKOUT_code
$ cd Build/
$ ./COMPILE
```

Compiling on other platforms may require editing a platform-specific configuration file.

## Running

SHiELD is run from a shell script. Some paths have to be set within the script to point to output directories and input data, but not special tools are required. I copied a shell script for a control run at 13km resolution from `/ncrc/home1/Linjiong.Zhou/GFDL_MP_v3_Runscripts` to `/ncrc/home2/Tristan.Abbott/SHiELD`, and got additional input files and scripts used in the runscript from Linjiong's SHiELD directory (available as of 5 December 2022). I had to delete one namelist entry (`memuse_verbose` in `coupler_nml`), since that option isn't present in the version of the model I'm using, and modify the `NAME` environment variable exported by the runscript to specify a date for which initial condition files are available.
