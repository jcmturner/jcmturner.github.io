---
layout: post
title:  "Adopting Go Modules and integrating with Github Actions"
date:   2020-02-01 20:00:00 +0000
categories: go
tags: modules github
published: true
---

## TL;DR

- When converting an existing Go library project to use Go modules, release the change as a new major version.
- Versions greater than v1.x.y need a sub-directory (vX) in your source repo.
- Remember your import path now has a /vN at the end.
- GitHub actions can be configured to test just the sub-directory in your repo that has been changed.

---

This blog entry captures some lessons learnt when migrating the [gokrb5 library](https://github.com/jcmturner/gokrb5) to
use modules for dependency management.

## Technically how to adopt modules
It's technically easy to implement Go modules with the following steps:

1. Clone the repo: ``git clone https://github.com/jcmturner/gokrb5``
2. Change into the repo's directory
3. Initialise modules: ``go mod init github.com/jcmturner/gokrb5``
4. Run tests to scan for dependencies: ``go test ./...``
5. Commit the ``go.mod`` and ``go.sum`` to source control

**But wait, if that was all I wouldn't bother writing this blog!**

## Lesson 1: Use a new major version
The gokrb5 library has some other dependencies that I also wrote so I first began by adopting modules on these.
One such "sub-library" implemented [NDR data encoding](https://github.com/jcmturner/rpc). I took version 1.1.0 ran the 
commands to initialise modules, ran the tests, committed the files, and tagged it as v1.1.1. _Sorted, right?_
**No!** It seems that this caused a [bit of a mess](https://github.com/jcmturner/rpc/issues/1) for others that were 
using this library. The error was rather obscure and I prioritised getting this fixed, given others were impacted, than 
figuring out exactly what this all meant. Therefore I decided to rollback by removing the v1.1.1 tag. _Phew! recovered, 
right?_ **No!** It seems that this caused an [issue](https://github.com/golang/go/issues/34033) where the Go modules 
proxy was caching the existence of v1.1.1.

I should have enabled modules without impacting those already using the library. In semantic versioning this is when 
you should iterate the major version number.

Lesson learnt: **Use a new major version when enabling Go modules.**

## Lesson 2: Major versions need sub-directories
My next step was to just apply a v2.0.0 tag to my sub-library. Therefore I added the tag and proceeded to look to enable
Go modules on the parent gokrb5 library. However every time I did this my go.mod file kept referring to v1.1.1 of the 
sub-library rather than v2.0.0. I tried hacking the go.mod to force it to reference v2.0.0 but no luck this resulted in 
an error message that I didn't really understand. I checked the module proxy for versions available via this link:
https://proxy.golang.org/github.com/jcmturner/rpc/@v/list At the time no v2.0.0 was listed! I scratched my head about 
what might be going on. Was this some other caching issue? I decided to wait for a while to see if it would be picked 
up. _Did this work?_ **No!** 

It turns out that I should read the documentation more closely, specifically this document: 
https://blog.golang.org/v2-go-modules. Ultimately for versions >=2 there needs to be a versioned sub-directory (eg "v2") 
in the source repo. The reasons for this is that packages with the same import path should be backwards compatible. 
However if you are changing the major version backwards compatibility is, by definition, not assured. Therefore the 
best way to change the import path is to add this version sub-directory. 

_Why only for versions >=2?_ Version zero, by definition, makes no promises of stability and compatibility, therefore 
only once there is a change from v1 to v2 is there a break in this promise so v2 is the first major version that needs 
the sub-directory.

Lesson learnt: **Create a version sub-directory for versions >=2**

## Lesson 3: Incompatible Red Herrings
On of my aims for enabling modules was to learn more about them. I kept seeing ``+incompatible`` appended to version 
numbers. Thereofre I wanted to know what was this all about. Reading the 
[documentation](https://github.com/golang/go/wiki/Modules#can-a-module-consume-a-v2-package-that-has-not-opted-into-modules-what-does-incompatible-mean) 
it seems this indicates that for a repo tagged with a semantic version one of the following may be the case:
- The version is >= 2 but there is no /vN sub-directory in the import path
- There is a /vN directory but it does not contain a go.mod file

So why, when I had sorted both these things, did the modules proxy show a ``+incompatible`` for my library? See:
https://proxy.golang.org/github.com/jcmturner/rpc/@v/list

After a while i realised that this was a red herring. The library's import path is _not_ github.com/jcmturner/rpc 
any more but github.com/jcmturner/rpc/**v2** so if I look this up on the proxy I see the v2.0.2 without 
``+incompatible``: https://proxy.golang.org/github.com/jcmturner/rpc/v2/@v/list

This occurs because a tag applies to the repo not just the v2 sub-directory.

Lesson learnt: **Remember that the import path includes the version directory**

## Lesson 4: How to make this work with Github actions
My next challenge was to integrate Go modules into my continuous integration. I have recently adopted Github Actions to 
run tests as updates are pushed to the repo. However I wanted to limit the tests to only the version that had been 
changed. Testing every version on every push, even if it was not updated, would add significantly to the duration of 
testing.

For the [gokrb5](https://github.com/jcmturner/gokrb5) library I was creating a new version 8 for the adoption of Go 
modules. This would reside in a /v8 sub-directory with version 7 continuing to residing in the root of the repo. I 
therefore implemented two workflows, one for version 7 and one for version 8:

- v7 (in the root of the repo) : https://github.com/jcmturner/gokrb5/blob/master/.github/workflows/testing.yml
- v8 (in a v8 sub-directory) : https://github.com/jcmturner/gokrb5/blob/master/.github/workflows/testingv8.yml

The key to triggering test on only the version that has been updated in a push was the ``paths`` and ``paths-ignore`` 
[filters](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions#onpushpull_requestpaths) 
that can be associated with the triggering events. For the version in the root of the repo (version 7 in my case) the 
github workflow was configured to ignore changes to files in any /vN sub-directory:
```yaml
on:
  push:
    paths-ignore:
    - 'v[0-9]+/**'
  pull_request:
    paths-ignore:
    - 'v[0-9]+/**'
```
For the new version 8 that resides in the /v8 sub-directory I only want the v8 workflow to run if a file in the /v8
directory has been changed:
```yaml
on:
  push:
    paths:
      - 'v8/**'
  pull_request:
    paths:
      - 'v8/**'
```

With a workflow per major version, I also adopted a convention of naming the Github workflow the same as the name of the 
major version. This means I can use the ``${GITHUB_WORKFLOW}`` environment variable where I need the name of the 
sub-directory in any of the flow's steps. For example, I change to the sub-directory before executing the tests.
```yaml
      - name: Tests including integration tests
        run: |
          cd ${GITHUB_WORKFLOW}
          go test -race ./...
        env:
          INTEGRATION: 1
          TESTPRIVILEGED: 1
        id: intgTests
```

When version 9 comes along I can simply copy the version 8 workflow and replace "v8" with "v9" in a small number of 
places.

---

I hope this blog helps others with adopting Go modules...