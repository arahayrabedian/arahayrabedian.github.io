---
layout: post
title: Moving A Directory From Dropbox To Google Drive (The Poor Man's Way)
---

I recently needed to move a directory of files from Dropbox to Google drive, and apparently nobody wants to make this easy for you. So I rube-goldberg'd it for a one-off transfer.

# Preliminaries

Since my whole reason for doing this is to avoid another sync from my home machine, I decided to work with a temporary VM. I fired up a temporary box from [dply](https://dply.co). Note that this process will require downloading the entirety of your directory on to a machine then uploading the entirety to Google Drive.

# Moving out of Dropbox

There are several ways to download files off of Dropbox, since this was a temporary machine, I did a quick search for a Dropbox CLI client and found [dbxcli](https://github.com/dropbox/dbxcli). It's a little clunky but works with a little snake charming.

Connecting dbxcli to your Dropbox account is a simple process of being provided a link and getting a code to paste back in.

There's currently a [bug](https://github.com/dropbox/dbxcli/issues/60) which prevents you from `get`-ing an entire directory. What I did instead was `ls` the directory on Dropbox that I wanted and massaged the output in to a `.sh` file that looks like this:

```
./dbxcli get "/dir of interest/DSC_2199.JPG" local_dir
./dbxcli get "/dir of interest/DSC_2200.JPG" local_dir
./dbxcli get "/dir of interest/DSC_2201.JPG" local_dir
./dbxcli get "/dir of interest/DSC_2202.JPG" local_dir
./dbxcli get "/dir of interest/DSC_2203.JPG" local_dir
```

# Moving in to Google Drive

Using what appeared to be the most mature CLI for Google Drive, aptly named [drive](https://github.com/odeke-em/drive), the process of uploading the directory was super simple. Note that there are [packages](https://github.com/odeke-em/drive/blob/master/platform_packages.md) that make installation much simpler.

After linking to your Google account (in a similar manner to how Dropbox worked out), you can simply issue a `drive push` command (and specify where you want to push) and it'll do a quick check and ask you if you're sure. Once you say yes it'll get uploading and you're done.
