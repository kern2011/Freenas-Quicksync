# Freenas-Quicksync
###### How to guide for getting Intel Quicksync working on Freenas
While researching this, everything I found led me to believe that Intel Quicksync would not work until we got FreeBSD 12. After researching forums online, I have gotten it to work on 11.3. I am not much more than a noob that is only tinkering but this works for me. I am using a i7 4790K and using Quicksync for 1 1080p to 720p or 1080p to 1080p stream leaves my cpu usage at 1%. I have not tested much more yet.
## Prep
-  First you must upgrade to Freenas 11.3 to be able to run a FreeBSD 11.3 jail. This can be done from the UI System>Update and changing the train to 11.3 (the Beta is working just fine for me).
-  Unfortunately I don't know if it's possible to upgrade your existing iocage, I couldn't and broke a jail trying. So I destroyed my Plex jail and remade it with 11.3. This was pretty painless for me because all of my plex data is in a location outside of the jail, but if yours isn't make sure to back it up. Verifi everything is running before moving on.
##  Making It work
###### From the Freenas console:
- Create a script file at `/root/scripts/plex-ruleset.sh`
 
- Add this to the script: 
```
#!/bin/sh

kldload /boot/modules/i915kms.ko

echo '[devfsrules_bpfjail=101]
add path 'bpf*' unhide

[plex_drm=5]
add include $devfsrules_hide_all
add include $devfsrules_unhide_basic
add include $devfsrules_unhide_login
add include $devfsrules_jail
add include $devfsrules_bpfjail
add path 'dri*' unhide
add path 'dri/*' unhide
add path 'drm*' unhide
add path 'drm/*' unhide' >> /etc/devfs.rules
```
- Make the script executable

`chmod +x /root/scripts/plex-ruleset.sh`

- Execute the script

`/root/scripts/plex-ruleset.sh`

###### Now we need to make this happen on boot

- In the UI go to Tasks>Init/Shutdown Scripts>Add
```
Type: Script
Script: /root/scripts/plex-ruleset.sh
When: Post Init
```

###### From the Plex jail console:
- Install the Intel graphics driver:

For older CPUs - Intel GMA 4500 or newer

`pkg install multimedia/libva-intel-driver`

For newer CPUs - Intel HD 5000 (Gen8) or newer

`pkg install multimedia/libva-intel-media-driver`

- add plex to the “video” group

`pw groupmod -n video -m plex`

###### Now we have to edit some jail settings. This can be done in the UI.

- Stop the jail. Use the arrow on the jail to expand the options, then click edit.

- Expand the “Jail Properties” header
Change `devfs_ruleset` to `5`

- Tick `allow_mount`

- Tick `allow_mount_devfs`

- Save your settings

- Start the Jail

## Testing

###### From the Jail console:
 
`ls /dev`

If everything worked; you should be able to find “dri” and “drm” in the output.

- Now go into your Plex settings go to the “Transcoder” tab and make sure “Use hardware acceleration when available” is ticked.

- Play back a video at a resolution or bitrate that is different than the source file. 

- From your Plex dashboard you should see transcode with `(hw)` at the end. If you do then congratulations, it worked!

## Notes
- 11/29/19- This was working for me but I think I broke somthing as it no longer works.

- 11/29/19- Fixed! It turns out my version of Plex was broken (Version 1.17.0.1709) If this wont work for you there is a work around:

In your Plex jail backup:

`/usr/local/share/plexmediaserver-plexpass/lib/libva-drm.so.2`

`/usr/local/share/plexmediaserver-plexpass/lib/libdrm.so.2`

`/usr/local/share/plexmediaserver-plexpass/lib/libva.so.2`

Then replace the files with good versions by running (From your Plex jail):

`cp-a /usr/local/lib/libva-drm.so.2.500.0 /usr/local/share/plexmediaserver-plexpass/lib/libva-drm.so.2`

`cp -a /usr/local/lib/libdrm.so.2.4.0 /usr/local/share/plexmediaserver-plexpass/lib/libdrm.so.2`

`cp -a /usr/local/lib/libva.so.2.500.0 /usr/local/share/plexmediaserver-plexpass/lib/libva.so.2`

Also copy your driver to Plex

`cp -a /usr/local/lib/dri/i965_drv_video.so /usr/local/share/plexmediaserver-plexpass/lib/dri/`

- 11/29/19- There is also 1 more problem I'm having, after rebooting Freenas my Plex Jail can't see “dri” and “drm” but after I resart the jail everything works fine.

- 12/1/19- Fixed a mistake in the "Testing" section.
