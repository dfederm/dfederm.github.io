---
layout: post
title: Backing up TrueNAS Scale to OneDrive
categories: [Home Networking]
tags: [Home Networking, NAS, TrueNAS, OneDrive, rclone]
comments: true
toc: true
---

Recently OneDrive was [removed](https://github.com/truenas/middleware/pull/11143){:target="_blank"} as a CloudSync provider for [TrueNAS Scale](https://www.truenas.com/truenas-scale/){:target="_blank"}. As I [built my first NAS]({% post_url 2024-01-06-building-a-nas-and-media-server-for-under-500 %}) and use OneDrive for cloud storage, I was looking for alternate means to back up my NAS to OneDrive. I found individual pieces of possible solutions on the [TrueNAS forums](https://www.truenas.com/community/){:target="_blank"}, but nothing approaching an end-to-end solution, so decided to do a write-up of what I ended up doing in hopes others may find it helpful as well.

TrueNAS Scale allows [custom docker containers](https://www.truenas.com/docs/scale/scaletutorials/apps/usingcustomapp/){:target="_blank"} which they call "custom apps", so the overall idea is just to use [rclone](https://rclone.org/){:target="_blank"} in a Docker container. I like the solution because it's decoupled from anything specific to TrueNAS, so very generic, easy to support, and there's no "magic" involved. It's very straightforward and understandable.

The first step is to create a new dataset which will contain your [rclone configuration file](https://rclone.org/docs/#config-config-file){:target="_blank"}. I named mine "rclone" in my root "Default" dataset. I used the SMB share type, since that's what I plan on using, but left the rest of the settings as default.

Next you'll need to configure the SMB share for the dataset so that you can manage the config file from other machines. For mine I just added the SMB share to the `/mnt/Default/rclone` path and used the default settings. When creating a new share it'll ask to restart the SMB service.

Connect to the new SMB share and create a single file inside called `rclone.conf`. This file should be in the [INI](https://en.wikipedia.org/wiki/INI_file#Format){:target="_blank"} and look like this:

```ini
[onedrivedavid]
type = onedrive
drive_type = personal
drive_id = <your-drive-id>
token = <your-token>
```

All configuration can be found in the [rclone docs for OneDrive](https://rclone.org/onedrive/){:target="_blank"}, but the boilerplate should be enough for most, so you just will need to fill in the two placeholders.

The section header is the name of the remote, so I used "onedrivedavid" since I plan to back up my wife's data on the NAS to her OneDrive separately and wanted to disambiguate.

For `drive_id`, I found the easiest way is to use the [Microsoft Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer). There you'll log in (by default you'll see mock data), and execute the query `https://graph.microsoft.com/v1.0/me/drive`. The first time you do this you'll see an error that says `Unauthorized - 401`. You can easily grant access to Graph Explorer by clicking the "Modify permissions" tab and consenting to `Files.Read`.

![Consenting to Graph Explorer permissions](/assets/images/truenas-onedrive/graph-explorer-permissions.png){:.center}

Run the query again and you should see the JSON response in the bottom pane. Use the `id` field of the response as your `drive_id`. You can also confirm that your `drive_type` is "personal" from the same response.

![Graph Explorer response](/assets/images/truenas-onedrive/graph-explorer-response.png){:.center}

For the `token`, you can follow the rclone [instructions](https://rclone.org/remote_setup/){:target="_blank"} but basically you just download the rclone executable from the website and run `rclone authorize onedrive`. This will pop up a browser window for you to authenticate in and once completed spit out JSON content which you will copy and paste in entirety into the `rclone.conf`. The value should be of the form: `{"access_token":"...}`.

Save your `rclone.conf` file and it's time to create the docker container or "custom app". Go to the Apps tab, click "Discover Apps" and then "Custom App". I named mine "rclone-david" since again I wanted to disambiguate with another user's rclone backups.

I found [robinostlund/docker-rclone-sync](https://github.com/robinostlund/docker-rclone-sync){:target="_blank"} on GitHub, which performs an `rclone sync` command on a schedule, which is exactly the scenario I'm targetting, so for the Image repository use `ghcr.io/robinostlund/docker-rclone-sync`.

As per the docs for that image, a few environment variables need to be set to configure it. Under the "Container Environment Variables" section, add the following environment variables:

* `SYNC_SRC=/rclone-data` - This can be any path, as long as it matched what you use below in the Storage section.
* `SYNC_DEST=onedrivedavid:/nas-backup` - The left hand side of the value needs to match the section header in the ini file, while the right hand side is a path within OneDrive you'd like to back up to.
* `CRON=0 0 * * *` - To schedule the sync daily at midnight.
* `CRON_ABORT=0 6 * * *` - Schedules an abort in case the sync is taking too long.
* `FORCE_SYNC=1` - This syncs on container startup, which makes for easier testing.
* `SYNC_OPTS=-v --create-empty-src-dirs --metadata` - Additional options to pass to `rclone sync`. These are the options I prefer, but all options can be found in the [rclone docs](https://rclone.org/commands/rclone_sync/){:target="_blank"}.

Under the "Networking" section, add an interface so it can reach out to OneDrive properly.

Under the "Storage" section, add:
1. Config
  * Host path: `/mnt/Default/rclone`, or whatever yours is configured to be.
  * Mount path: `/config`, which is what the image expects.
  * Read Only: *unchecked*. rclone will write to the file, in particular to update the access token as it refreshes it.
1. Data
  * Host path: Whatever path on your NAS you'd like to back up
  * Mount path: `/rclone-data`, or whatever you chose for `SYNC_SRC` above.
  * Read Only: *checked*. rclone will only need to sync from the NAS, so only need read permission to the data.

Leave everything else as the defaults and click Install. Now you'll need to wait for the container to deploy, which may take a few moments.

![Docker Container Deployment](/assets/images/truenas-onedrive/container-deploying.png){:.center}

Once the container is deployed, you can click on it and under "Workloads" there should be an icon to click on to show the logs for the container. You can use this to ensure the sync is happening properly.

![Docker Container Logs](/assets/images/truenas-onedrive/container-logs.png){:.center}

And that's all there is to it! You can now have the benefits of storing your data locally in your NAS, while having the peice of mind of having a remote backup.