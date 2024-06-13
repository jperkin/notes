## ctfconvert appears to be quadratic with number of objects

I've noticed some packages, for example qemu, spending inordinate amounts of
time during the CTF conversion phase.  The files that appear to be the slowest
to convert were those with a higher number of object files.

Running manually to see how long it takes using default threads:

```shell
$ ptime ctfconvert -m -o qemu-system-sh4.ctf qemu-system-sh4

real     2:57.529658300
user     5:05.438214880
sys        23.082903387
```

Does increasing the number of threads help much?  On an otherwise idle 16-core
VM:

```shell
$ ptime ctfconvert -j16 -m -o qemu-system-sh4.ctf qemu-system-sh4

real     2:26.547793113
user     6:56.944126603
sys      1:14.503069502
```

Not really, with considerably more system time.

As there are a fair number of `qemu-system-*` binaries in the qemu packages (31
as of 9.0.1), and the conversion process does one file at a time, this ends up
considerably increasing the packaging time.

### Analysis

Running under `LIBCTF_DEBUG=1` (with `-j 1` to avoid output interleaving), I
noticed that there were many duplicates of the line:

```
libctf DEBUG: Trying to find match for ...
```

Here are the largest offenders for `qemu-system-sh4`:

```
$ sort libctf.out | uniq -c | sort -n | tail
2856 libctf DEBUG: Trying to find match for block.c/__func__.3/0
2856 libctf DEBUG: Trying to find match for core.c/__func__.1/0
3808 libctf DEBUG: Trying to find match for block.c/__func__.0/0
3808 libctf DEBUG: Trying to find match for block.c/__func__.1/0
3808 libctf DEBUG: Trying to find match for block.c/__func__.2/0
3808 libctf DEBUG: Trying to find match for core.c/__func__.3/0
3808 libctf DEBUG: Trying to find match for core.c/__func__.4/0
3808 libctf DEBUG: Trying to find match for core.c/__func__.5/0
4760 libctf DEBUG: Trying to find match for core.c/__func__.0/0
4760 libctf DEBUG: Trying to find match for core.c/__func__.2/0
```

Rather than looking at these compiler-generated function names, if we instead
look at a real unique function:

```shell
$ grep zrle_encode_tile24ble libctf.out  | uniq -c
 952 libctf DEBUG: Trying to find match for vnc-enc-zrle.c/zrle_encode_tile24ble/0
```

We can see that it maps to very closely to the total number of object files:

```shell
$ nm qemu-system-sh4 | awk -F'|' '$4 ~ /FILE/ {print $NF}' | wc -l
     966
```

To better see the performance difference between the number of object files I
wrote a silly script to generate a one-function-per-file program with a
configurable number of object files, as well as a combined `.c` that did the
same thing but in one file.

```bash
#!/bin/sh

if [ $# -ne 1 ]; then
    echo "usage: $0 <iterations>" >&2
    exit 1
fi
iters=$1; shift
outdir="test.${iters}

rm -rf ${outdir}
mkdir -p ${outdir}

cat >${outdir}/main.c <<EOF
#ifdef SINGLE
#include "f.h"
#endif

int
main()
{
EOF

for i in $(seq 1 $iters); do
    f="func$i"
    echo "  $f();" >>${outdir}/main.c
    echo "int $f(void);" >>${outdir}/f.h
    echo "int $f() { return $i; }" >${outdir}/$f.c
    gcc -gdwarf-2 -c ${outdir}/$f.c -o ${outdir}/$f.o
done

echo "}" >>${outdir}/main.c
cat ${outdir}/func*.c ${outdir}/main.c >${outdir}/one.c

gcc -gdwarf-2 -DSINGLE ${outdir}/main.c -o ${outdir}/main ${outdir}/*.o
gcc -gdwarf-2 ${outdir}/one.c -o ${outdir}/one

echo "Converting with ${iters} object files..."
ptime ctfconvert -m -o ${outdir}/main.ctf ${outdir}/main

echo
echo "Converting combined single object file..."
ptime ctfconvert -m -o ${outdir}/one.ctf ${outdir}/one
```

Start with 10 objects.  There's already a noticeable difference between them.

```shell
$ ./generate-test.sh 10
Converting with 10 object files...

real        0.446682863
user        0.011439896
sys         0.032540722

Converting combined single object file...

real        0.021660145
user        0.003100705
sys         0.007459779
```

100.

```shell
$ ./generate-test.sh 100
Converting with 100 object files...

real        0.712378989
user        0.092084691
sys         0.260975441

Converting combined single object file...

real        0.022756690
user        0.004772264
sys         0.008717603
```

1000.

```shell
$ ./generate-test.sh 1000
Converting with 1000 object files...

real        3.944281077
user        3.010809735
sys         3.113936638

Converting combined single object file...

real        0.117654340
user        0.070433419
sys         0.011847592
```

10000 objects is the largest I'm prepared to run, as it takes long enough to
simply prepare the object files!  You get the idea though...

```shell
$ ./generate-test.sh 10000
Converting with 10000 object files...

real     2:52.963397493
user     4:37.932977432
sys      1:00.700876741

Converting combined single object file...

real        3.570698631
user        3.295235543
sys         0.034113123
```

Almost 3 minutes to convert a 6.2MB file.

### Fix

I'm not familiar with how the conversion process works, but is there a way to
cache entries that we've already looked for, or perform an initial scan first
and then create a hash table for entries per file?
