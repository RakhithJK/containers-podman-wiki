[ 2022-04-20 this is still under development ]

This document describes the **Buildah Vendor Treadmill**, a two-part system used for easing the burden of vendoring buildah into podman. The two parts are:

1. **sync** - run daily-ish by a human (as of this writing, Ed). In theory this is very simple:
   1. Preserve current set of treadmill patches (i.e. results of last sync + fixes)
   1. `git pull` to bring in latest podman @ main
   1. `git rebase` the treadmill PR onto podman @ main
   1. `make vendor` to pull in current buildah @ main
   1. run a few tests; fix any problems
   1. (It's a little trickier than that, and for the most part Ed worries about it so you don't have to)
2. **cherry-pick** - run ad hoc by a developer needing to vendor in a new buildah into podman. This
is the part that you, dear reader, probably want to know about.

We will cover those in reverse order, because chances are you're here for vendoring buildah.

***

## For Vendoring Buildah (Start Here)

Target Audience: **developer vendoring buildah into podman**

Most of the time the vendoring will go smoothly. If you're reading this it's because things haven't gone smoothly. Most likely the **bud** tests are failing: either the patching step, or the `bud` tests themselves. You should now run, from your buildah-vendor branch:
```console
$ hack/buildah-vendor-treadmill --pick
```
This will identify the treadmill PR (currently [#13808](https://github.com/containers/podman/pull/13808) as of 2022-05-02) and cherry-pick its patches onto your branch. If Ed hasn't been slacking, this will resolve your vendoring problems and you can `git commit --amend` (to update the commit message) then `git push --force` and make it past CI.

As of this writing (2022-05-02) this process only works when vendoring into `main`. If you need to vendor a new buildah into a maintenance branch, I'm sorry, you're on your own. (But chances are good that someone has already vendored that buildah into main, and you can cherry-pick the required changes).

As we gain experience with this process, please update this document to reflect best practices.

***

## For Daily Integration (What Ed Does)

Target audience: **someone willing to run the sync step daily, and fix problems**. This is what Ed does daily. Steps involved:
- keep a `vendor_buildah_latest` branch active
- run the `sync` step
- fix problems if they arise

As of the past three weeks in which I've been playing with this, it has been super easy: the process works cleanly:

```console
$ hack/buildah-vendor-treadmill --update
-> buildah old = v1.25.2-0.20220412203738-d41a4fd27c19
|
+---> HEAD is buildah vendor (as expected); dropping it...
|
+---> Pulling podman main...
remote: Enumerating objects: 15, done.
[...]
 * branch                main       -> FETCH_HEAD
   d6f47e692..712c3bb22  main       -> upstream/main
|
+---> Rebasing on podman main...
Successfully rebased and updated refs/heads/vendor_buildah_latest.
|
+---> Vendoring in buildah...
[...]
all modules verified
-> buildah new = v1.25.2-0.20220418231153-74cd96acf358
|
+---> Running 'make' to confirm that podman builds cleanly...
[...]
|
+---> Cross-checking man pages...
|
+---> Confirming that buildah-bud-tests patches still apply...
+ git clone -q https://github.com/containers/buildah test-buildah-v1.25.2-0.20220418231153-74cd96acf358
+ git checkout -q 74cd96acf358
+ git tag buildah-bud-in-podman
+ make bin/buildah
[...]
+ git am --reject
Applying: tweaks for running buildah tests under podman
Checking patch tests/helpers.bash...
Applied patch tests/helpers.bash cleanly.
+ /home/esm/src/atomic/2018-02.podman/libpod/test/buildah-bud/apply-podman-deltas
|
+---> All OK. It's now up to you to 'git push --force'
|
+--->  --- Reminder: New buildah, new podman. Good candidate for pushing.
```

On several occasions it hasn't worked cleanly, and those are the ugly ones, and that's why Ed gets paid the Big Red Hat Bucks. In those cases I've basically just followed the procedures for when this happens on a real vendor PR, then `git commit --amend` my changes on the treadmill PR, then force-push. The cases I've seen so far:
* **man page checker** - new option gets added to `buildah bud`. Solution: look at the buildah PR, find the new docs, copy/paste them into `podman-build` man page. However, when this happens, you also need to check for:
* **new options** - is parsed by podman, but you probably need to muck with ``buildahDefine.BuildOptions`` in [build.go](https://github.com/containers/podman/blob/main/cmd/podman/images/build.go). Possibly also do something so `podman-remote` will work.
  * if the new option makes no sense in podman-remote, you will want to add a `skip_if_remote` to [apply-podman-deltas](https://github.com/containers/podman/blob/main/test/buildah-bud/apply-podman-deltas)
* **other test failures** - maybe the new `buildah bud` tests don't pass under `podman build` and you just don't want to deal with fixing them now. You can add `skip`s to `apply-podman-deltas`
* **conflicts** - when applying the buildah-bud-under-podman patches. Usually happens when someone edits buildah's `helpers.bash`. Solution: see the [buildah bud tests README](https://github.com/containers/podman/blob/main/test/buildah-bud/README.md)

I haven't much tested the case where `podman main` brings in a newly-vendored buildah with the treadmill changes incorporated. Again, this is a work in progress.