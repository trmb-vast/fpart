# Wrapper script to do the mounts and create proper fpsync.vast args:

### General Note:  If rsync can do about 200MB/sec on your workload,  fpsync can do up to 1GB/sec, and fpsync.vast can double that (or more).
The idea is to saturate the destination, without reguard to the source, and make it go as f^HVast as possible. 
### Specialty Notes:  a special compiled rsync with the fadvise patch, and the sparse-block-size patch can also help. 
                  Also.. Rolling checksums  with xxhash speed things up for fpsync or rsync "incremental updates" of existing files which havve changed since last time.  That is available (if you enable it) in rsync-3.2.3 and newer.

## fpsync.vast 
The function this adds is to enable multiple writer destinations so we can remove NFSd destination filer bottlenecks 
when using a scale-out NAS such as VAST. Each VIP in vast can be on different servers thus increasing performance.
Originally this patch was written for a single-source like a local NVME flash drive which could not benefit from 
the fpsync -w  scale-out flags to run on multiple hosts.  The acompany wrapper script mk_fpsync_list helps with setup.
This will someday become a pull-request to upstream for martymac. So far it has been tested many times, but in a narrow use-case.


## mk_fpsync_list  wrapper script:

The user should copy this to a filename related to the replication job at hand, and edit it there. It is adviseable to keep this script around so you know who ran what, how, and with what parameters.  make a new copy each time you want to fpsync something new… maybe build a repo of scripts so you have them for future reference. 

anyways…  edit the file and fill in the variables.  (examples for our lab are the defaults)
```
## Change these to match your site
VAST_VIP_PREFIX=172.200.2 # this is the first three octets of your vip-pool
VAST_VIP_PREFIX=172.200.3 # this is the first three octets of your vip-pool
VAST_VIPS=$(seq 1 8)      # this is a sequence of numbers of the last octet of vip-pool

### This script will call sudo to create /mnt/tree# mountpoints, one to each VAST_VIP,
### it will mount those from the following filer:$DST_FILER_EXPORT as described below
### This is the destination path as exported from the filer.  
### For safety it is best to make this an new and empty VAST view with nosquash for the data-movers
DST_FILER_EXPORT_PATH=/scratch1/EXR/tree_mover_dest # this is as exported from filer..

##### This is the source path for stuff you want to replicate
SRC_PATH=/vast/201/scratch1/EXR/tree_mover_src

##### Tuning jobs and fpart to fit your dataset:
##### Your data structure has a big impact on ideal partition size, and resulting performance.
## We typically break it up by qty of files per partition.. (assuming filesize is somewhat uniform)
## or we can break it up by size..  or by a couple other strategies or mix of.
##
FPART_FLAGS="-f 10000 -n 64"    # 10k jobs per partition(or job)  and -n 64 jobs total. -n should be QTY of -w hosts * num -x * CPU cores  or half that.
FPSYNC_W_FLAGS="-w 10.61.10.105 -w 10.61.10.107 -w 10.61.10.109" # If you have multiple data-mover hosts, and all have same paths mounted, then use em!
# these normally dont need tweaking. except know that -S will call sudo
FPSYNC_FLAGS="-vvv -S -o '-lptDogW --inplace --numeric-ids'" # run this as any user, and -S flag will call SUDO to run rsync to get root-nosquash permissions
```
 

comment out the FPSYNC_W_FLAGS  if you want to run on only one host. (good idea to start on one host)

run the script. It will do the vast vip mounts. (mounting them to /mnt/tree##, …) and then it will print out the proper syntax for fpsync (using the other variables you just specified in the script).   watch for errors in the mount process.  double check syntax.. you don’t want to accidentally fill up / partition on the mover, or have the data come or go from the wrong path.  

If you next want to add other data-movers, then after you have finished editing, run this script on those data-movers also. 

Note that fpsync also has a feature to resume a previous fpsync job. read about this more in the docs, add the -r <jobnum> flag to the config file, and a -R replay mode to “do a followup sync”. 
Tuning the -s and -f flags of fpsync/fpart:

This is kind of a black-art.. you have to understand the dataset, the src filer capabilities,  and the data-movers.  You can easily be bottlenecked by:   

    src filer Bandwidth
    src filer read IOPS
    data mover cpu or network
    IOPS per destination mountpoint (smallfiles)

So check those out first before spending time going back and forth on tuning the -s (size) and -f #files  and -n# (number of jobs) flags which create the chunked-sized-partitions of files which are then run by -n jobs at the same time.  remember, there is some ssh setup time for job start, so if you have a very small -f #,  and it finishes in 1 second, then you probably wasted 1 second of setup time.  so make -f bigger. 

You can kind of get a handle on this by watching a single-threaded rsync.. observing how many files/sec are scrolling on the screen.. 2 files per second means largefiles (or narrow network pipes)… so you can make your -f # smaller.. like 1000.     or if you are running a fpsync “redo”  then you have to think a bit about that and possibly make the partitions larger. 

 
