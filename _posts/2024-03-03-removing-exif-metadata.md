---
layout: post
title: Removing Exif metadata from images
tags: git jekyll
---

When uploading anything to the web, you should always be mindful about not sharing more than you intend to.

Jekyll lets you include images in your posts, by convention stored in `assets/images/`, and if these are images you've created yourself, or photos you've taken, it's worth considering the Exif metadata as a potential source of information leakage (for example, geolocation data).

## Using exiftool

Fortunately, by using `exiftool` it's easy to strip this from image files. You can do so selectively, leaving behind colour profiles but removing everything else with the following:

```bash
exiftool -all= --icc_profile:all path/to/image
```

## Setting up a git hook

If you're using a git repository, you could take things a bit further by adding the following to `.git/hooks/pre-commit`. This will be invoked by `git commit`, and prohibit a commit if metadata can't be removed.

```bash
#!/bin/sh

if command -v exiftool > /dev/null; then
    exiftool -all= --icc_profile:all assets/images/*
    if stat -t assets/images/*_original > /dev/null 2>&1; then
        echo "Aborting: Exif metadata was stripped, please review!"
        exit 1
    fi
else
    echo "Aborting: unable to perform image metadata checks!"
    exit 1
fi
```
