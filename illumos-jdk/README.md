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

Here we're using `jdk-21.0.5-ga` as an example.

Set required variables.

```shell
$ TRIB=/work/git/tribblix-build
$ JDK_MAJOR=21
$ JDK_BRANCH=21u5
$ JDK_TAG=21.0.5
```

Fetch upstream branches and create new branch based off the `-ga` tag.

```shell
$ git fetch --all
$ git checkout -b il-jdk${JDK_BRANCH} jdk-${JDK_TAG}-ga
```

Apply Tribblix patches.

```shell
$ while read args file; do
    git apply ${args/+/ } ${TRIB}/patches/${file}
  done < ${TRIB}/patches/jdk${JDK_MAJOR}u-jdk-${JDK_TAG}-ga.pls
$ git add .
$ git commit -m "illumos: Apply Tribblix patchset for ${JDK_TAG}-ga."
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
$ git tag il-jdk-${JDK_TAG}-ga
$ git push --set-upstream origin il-jdk${JDK_BRANCH}
$ git push --tags
```
