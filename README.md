bit-torrent-sync-article
========================

#### Intro

Ever since BitTorrent Sync came out I've started tinkering with it, putting it on a raspberry pi, on a laptop but nothing ever serious until a few months ago.

Then the NSA happened and suddenly I became less trusting in placing my files on someone elses servers. Heck to be honest I was never particularly happy with the idea, I mean it sounds so great to sync 2-40GB's of PSD's or mocks onto Google Drive or Dropbox just incase your machine goes belly up but have you ever tried re-downloading 40GB's back, it takes ages plus it's on someone elses hardware which you ultimately don't control which never felt particularly great.

So being a bit of a nerd I wanted to build my own storage network, similar to dropbox or google drive and started looking at BitTorrent Sync more seriously.

The first thing which got me hooked was the speed, I mean once setup it really does fly and even if your macbook goes into powernap it still sync's the files across which came across as a major win. With two machines, one at work and one at home and having them both mirrored effortlessly without any file collision that was a major win for me.

Plus if I could keep the base sync box at home and have another on a vps off-site that made more sense, faster syncing and removes the danger of a fire taking out your one source of backup.

So, to work.

#### Home Unit

First I needed the home unit to be low-powered, noiseless and set-it-and-forget-it. I didn't want to pay massive bills to maintain it and it had to work regardless, even if I moved.

So I opted for a BeagleBone Black and for storage one 128GB USB Stick. That gave me more than enough space and mounted in a black case meant I could leave it running at night without some crazy light-show annoying my wife or cat.

For power I loathed the idea of plugging in another wall plug so used my Airport's USB port to power it which worked like a charm, the Apple Airport supplied enough voltage to keep it running and kept the amount of cables down to a minimum.

For O/S I installed Debian and flashed it to the eMMC [http://elinux.org/BeagleBoardDebian](http://elinux.org/BeagleBoardDebian) which was one of the reason's I like the Bone, that and the added horsepower.

Once the Debian was installed and running I hooked up a custom script to notify me via email when the machine re-booted and what IP it was on, this was a major help as I wanted to avoid setting a static IP in case we moved plus it helped keep the machine's future maintenance low.

Here's a link to the script on Github [https://gist.github.com/johnantoni/8199088](https://gist.github.com/johnantoni/8199088)

Once the script was running I needed to setup the USB stick as storage for the synced files [notes here]( https://github.com/johnantoni/beaglebone-black/blob/master/setup/format-and-mount-usb.md).

With the 128GB stick mounted as local storage and using the nofail mount option to ensure the box will reboot even if the stick dies...

    /dev/sda1 /media/storage ext4 rw,async,user,nofail 0 0

...I was ready to setup BitTorrent Sync.

This was actually more simple than the rest of the setup because BitTorrent Sync doesn't need a static IP or special configuration, it just needs a location for the files and an internet connection.

Setup was fairly simple, grab the ARM build from [here](http://www.bittorrent.com/sync/downloads/complete/os/arm), create relevant directories on the USB Stick (Mocks, Documents, Notes) and create a btsync config file to sync them.

Here's something similar to what I've got:

    {
      "device_name": "box",
      "listening_port" : 12345,
      "storage_path" : "/media/stick/.sync",
      "check_for_updates" : false,
      "use_upnp" : false,
      "download_limit" : 0,
      "upload_limit" : 0,
      "webui" :
      {
      },
      "shared_folders" :
      [
        {
          "secret" : "something",
          "dir" : "/media/stick/mocks",
          "use_relay_server" : true,
          "use_tracker" : true,
          "use_dht" : false,
          "search_lan" : true,
          "use_sync_trash" : true
        }
      ],
      "lan_encrypt_data": true
    }

The first thing you'll notice is that I turn off the web-ui, once setup I'll never need to access it again and it opens up a security hole.

Note: Before this I've already got BitTorrent Sync running on my Mac with a shared folder and have a Read-Write secret to give it.

In the config I define a shared folder and as this will be the primary node I set it to use the R/W secret from my Mac's shared folder. 

Once setup I test it:

    btsync --config ~/sync.conf

See whether the Mac's sync client communicates with the beaglebone box:

![sync](https://github.com/johnantoni/bit-torrent-sync-article/blob/master/sync.png)

...and then add it to the cronjob so btsync will start on boot-up.

#### Remote Unit

After testing it out for several weeks I then put about acquiring a VPS for remote backups, in case the bone box dies, power failure, corrupts or the place burns down.

So I hit the lowendbox forums to purchase one at a good price, all I needed was something with a fast connection and 75-150GB of storage.

[http://lowendbox.com/](http://lowendbox.com/) Is an excellent place to get great deals on VPS's at a good price. What happens is that when the providers build a new data center or have spare capacity they place deals on this site along with a testfile so you can check the transfer speed. I usually hunt for providers like CatalystHost and HostHatch who've over the years given great service and tend to go out of their way to make sure your needs are met.

With the VPS purchased I then set about setting up Debian, installing BTSync in much the same way as before. This time though I opt for using the Read-Only Secret on the shared folder.

The reason being I found having multiple boxes running as R/W creates a good chance for dead-locking the BtSync client and stopping the actual sync process from occuring, much like when Dropbox or Google Drive refuses to sync certain files.

#### Archiving

With the home box running and the VPS syncing as a secondary mirror I then thought about archiving my more important files in case I were to do something stupid like wipe out my shared folder directory. Thankfully BitTorrent Sync keeps files for 30 days after deletion inside the shared folder's .sync directory but I needed some kind of fail-safe.

So I looked at archiving the Documents folder to S3 Storage, I've used meskyanichi [Backup gem](http://meskyanichi.github.io/backup/v4/) before so this was the first one I tried.

Another more secure means is via [Duplicity](http://duplicity.nongnu.org/) which handles encryption well. Here's a good [article](http://blog.phusion.nl/2013/11/11/duplicity-s3-easy-cheap-encrypted-automated-full-disk-backups-for-your-servers/) on setting this up.

At present I'm testing out the Backup Gem and offshooting backups to S3, though due to the low-end nature of the BeagleBone it will lockup if you try to TAR anything like 1-2GB's of files, so at the moment it's on a per-file basis.

To close this part is still a work in progress, but I may opt for Duplicity as it will be ultimately more secure in the long run or put together a script to tar the files I want and rsync them elsewhere's; TBD.

Price wise you needn't worry about the cost of using Amazon S3 storage for encrypted backups, it's pretty minimal, e.g. $0.10 per month for 5-10GB's.

#### Close

After setting this up I am pretty happy with the results.

Firstly the BtSync client runs fast and without a hitch, actually noticably faster than Google Drive, and the crew already know how to stream files across the network at the fastest possible speeds plus being able to run when the machine is on power-nap mode is another win.

Resource-wise the Desktop Client takes up very little processing power & memory and having something that you have total control of does mean you never worry about price hikes or changes to terms-of-conditions. Its yours, you can do with as you will and if it does die you have the added benefit of being able to remote in, plug the box into a monitor and view the logs. The software handles itself well and works great.

Also if you work somewhere where Torrent traffic is blocked you can solve that with a VPN.

Another bonus we found over Winter was when we had a long 2-day powercut. We were forced to work in coffee shops or at friends and like many had no way of knowing when the power was back at our place. However because I added that script to the home unit to notify me what IP it was using I had a reliable means of knowing when the power was back, I got the email and we went home, another win.

Today I've got it running on two machines, I've used it so my Wife can get wedding photos of her sister from Greece and used it to share work with clients.

In all I'm happy with the software and will probably use it for years to come.

Great work team ;-)
