## illumos jdk

Peter maintains the primary patchset in Tribblix.  I merge this, along with a
few other patches, to <https://github.com/illumos/jdk> for easier distribution.

## Setup

The upstream branches are spread across a few different repositories.

```shell
$ git remote -v
jdk	https://github.com/openjdk/jdk.git (fetch)
jdk	https://github.com/openjdk/jdk.git (push)
jdk11u	https://github.com/openjdk/jdk11u.git (fetch)
jdk11u	https://github.com/openjdk/jdk11u.git (push)
jdk17u	https://github.com/openjdk/jdk17u.git (fetch)
jdk17u	https://github.com/openjdk/jdk17u.git (push)
jdk21u	https://github.com/openjdk/jdk21u.git (fetch)
jdk21u	https://github.com/openjdk/jdk21u.git (push)
origin	git@github.com:illumos/jdk.git (fetch)
origin	git@github.com:illumos/jdk.git (push)
```

## Creating new branch / tag

Here we're using `jdk-21.0.3-ga` as an example.

Fetch upstream branches and create new branch based off the `-ga` tag.

```shell
$ git fetch --all
$ git checkout -b il-jdk21u3 jdk-21.0.3-ga
```

Apply Tribblix patches.

```shell
$ TRIB=/work/git/tribblix-build
$ while read args file; do
    git apply ${args/+/ } ${TRIB}/patches/${file}
  done < ${TRIB}/patches/jdk21u-jdk-21.0.3-ga.pls 
$ git add .
$ git commit -m "illumos: Apply Tribblix patchset for 21.0.3-ga.
```

Cherry pick additional patches from previous branch.

```
722de786e84 pkcs11: Disable unsupported digest cloning algorithms.
5576de2510c illumos: Apply pkgsrc patches.
```

```shell
$ git cherry-pick 5576de2510c 722de786e84
```

Tag and push.

```shell
$ git tag il-jdk-21.0.3-ga
$ git push --set-upstream origin il-jdk21u3
$ git push --tags
```
