# FreeNAS-Quicksync
##### How to guide for getting Intel Quicksync working on FreeNAS
Plex states that using hardware transcode on FreeBSD requires FreeBSD 12.0 or later. After some testing, it turns out it will work in FreeBSD 11.3 as well, although devfs is not as stable as in FreeBSD 12.x.
## Test Results
- Using an i7 4790K and using Quicksync for one 1080p to 720p or 1080p to 1080p stream left CPU usage at 1%.
- Testing on a 1225v6 Xeon (7th Gen Kaby Lake) used ~2-3% CPU for a 1080p VC-1 -> 1080p H.264 hardware transcode. The same machine was at 70-75% CPU for a software transcode of VC-1 -> H.264. Some Bluray disks used VC-1, before H.264 became ubiquitous.
- Testing on a 1225v6 Xeon (7th Gen Kaby Lake) used ~4% CPU for a 4k HDR HEVC 10-bit -> 1080p SDR H.264 hardware transcode. The same machine was at 100% CPU for a software transcode of the same file. Running four 4k hardware transcodes at once did not significantly tax the machine, further testing to determine the max number of possible 4k transcodes was not done.
- Testing on a G4560 Pentium (7th Gen Kaby Lake) used ~30% CPU for a 4k HEVC -> 1080p H.264 transcode.
- Testing on a Core i3 6100T (6th Gen SkyLake) shows that acceleration of a 4k HEVC -> 1080p H.264 transcode is minimal on it and takes 95% CPU. This is likely due to limited HEVC features in that generation.
- Core 4th Gen (Haswell) added H.265 and VC1 support in QuickSync. For 1080p transcode, "Haswell or better" is reasonable guidance.
- The Kaby Lake iGPU improved HEVC acceleration and added 10-bit (HDR) HEVC. The HEVC (H.265) codec is used on UHD Blurays. Consider a Kaby Lake or later CPU a requirement for significant acceleration of HEVC (Bluray 4k UHD) content. Note that without HDR10 tone mapping, 4k HDR transcode remains suboptimal, and it is often better to prepare a good SDR 1080p file ahead of time. HDR10 tone mapping is not yet available in a Desktop / Server CPU/iGPU combination.
## Compatibility
- The current (11.3-Ux) Freenas Drivers support Intel CPU Generation 2-7, Sandy Bridge through Kaby Lake
- The current (11.3-Ux) Freenas Drivers DO NOT support Intel CPU Generation 8 and 9. This means Coffee Lake and later, Intel Xeon E-xxxxG and Intel Core i3/i5/i7 8xxx and later.
- Support for Generation 8 and 9 can not be added without a kernel upgrade to 12.x. Don't bother trying to compile it from source. TrueNAS 12.x Core will gain driver support for the iGPUs in 8th and 9th gen CPUs.
## Hardware requirements Intel Xeon / Intel server C2xx chipsets
- Your chipset and motherboard/BIOS need to support the iGPU in your E3-xxx5 or E-2xxxG Xeon. Other Xeons do not have an iGPU.
- C206, C216, C226, C236, and C246 chipsets are able to support iGPU; C202/204, C222, C232 and C242 will not.
- Caveat that C246 will require TrueNAS 12.x.
- C226: SuperMicro X10SLH-F and Intel S1200V3RPM confirmed working; X10SLM+-F expected to work
- C236: SuperMicro X11SSH-F and AsRock Rack E3C236D2I confirmed working; X11SSM-F and X11SSV-M4F expected to work
- C246: SuperMicro X11SCH-F expected to work; X11SCM-F will not work
- You can use these boards and chipsets with Intel Pentium or Intel i3/i5/i7 processors, verify supported CPUs for your specific board
- These considerations do not apply to consumer chipsets
- See https://ahelpme.com/servers/supermicro/enable-internal-graphics-in-supermicro-servers/ for details of enabling iGPU on a Supermicro board
- Caveat: Turning on hardware acceleration for Plex will disable the KVM inside your IPMI. You will still have it available during initial boot, for switching boot environments or dropping into a debug session, but it will switch off once the iGPU drivers are loaded. The compat.linuxkpi.modeset tunable can re-enable KVM in IPMI, but then hardware acceleration no longer works. Be sure that you are okay with needing to disable the bootup script when you require KVM after boot, for example when fixing a networking setting.
## Prep
-  First you must upgrade to FreeNAS 11.3 to be able to run a FreeBSD 11.3 jail. This can be done from the UI System>Update and changing the train to 11.3.
-  It may not be possible to upgrade your existing iocage, in testing a jail broke during the upgrade. Keeping plex metadata outside of the jail makes it easy to recover by destroying and recreating the jail. If your metadata is inside the jail, make sure to back it up. Verify Plex is working before moving on.
-  Your mileage will vary: `iocage upgrade -R 11.3-RELEASE plex` worked for one user on a plugin jail (base jail). When in doubt, recreate the jail, and do make sure to use a base jail for ease of maintenance.
-  Note that using hardware transcode is likely easier in a custom Plex jail than in a plugin, because upgrades to plexmediaserver will not remove the necessary group membership.
-  Hardware transcode requires a PlexPass subscription. It does not require using the beta code in the plexmediaserver-plexpass package.
##  Making It work
##### From the Freenas console:
- N.B.: You will have an easier time using the console from SSH. Enable the SSH service in the GUI, then connect to your FreeNAS server using an SSH client like PuTTY, which you can get at https://putty.org. 
- Create a script file at `/root/scripts/plex-ruleset.sh`, for example by `mkdir /root/scripts` followed by `ee /root/scripts/plex-ruleset.sh`.
 
- Add this to the script, most easily done with copy / paste: 
```
#!/bin/sh

echo '[devfsrules_bpfjail=101]
add path 'bpf*' unhide

[plex_drm=10]
add include $devfsrules_hide_all
add include $devfsrules_unhide_basic
add include $devfsrules_unhide_login
add include $devfsrules_jail
add include $devfsrules_bpfjail
add path 'dri*' unhide
add path 'dri/*' unhide
add path 'drm*' unhide
add path 'drm/*' unhide' >> /etc/devfs.rules

service devfs restart

kldload /boot/modules/i915kms.ko
```
- Make the script executable

`chmod +x /root/scripts/plex-ruleset.sh`

- Execute the script

`/root/scripts/plex-ruleset.sh`

##### Now we need to make this happen on boot

- In the UI go to Tasks>Init/Shutdown Scripts>Add
```
Type: Script
Script: /root/scripts/plex-ruleset.sh
When: Post Init
```

##### From the Plex jail console:

- add plex to the “video” group

`pw groupmod -n video -m plex`

##### Now we have to edit some jail settings. This can be done in the UI.

- Stop the jail. Use the arrow on the jail to expand the options, then click edit.

- Expand the “Jail Properties” header
Change `devfs_ruleset` to `10`

- Save your settings

- Start the Jail

## Testing

##### From the Jail console:
 
`ls /dev`

If everything worked; you should be able to find “dri” and “drm” in the output.

- Now go into your Plex settings go to the “Transcoder” tab and make sure “Use hardware acceleration when available” is ticked.

- Play back a video at a resolution or bitrate that is different than the source file. 

- From your Plex dashboard you should see transcode with `(hw)` at the end. If you do then congratulations, it worked!

_Note: Plex 1.18.3 did not use hw transcode with a Xeon 1225v6 during testing. Plex 1.18.8 resolved this, and it worked._

##### Troubleshooting

- /dev/dri needs to be visible in FreeBSD itself. If it isn't there, you can't make it visible in the jail. Check that your hardware is supported, and that the BIOS is set to enable the iGPU.
- There is a known issue with FreeBSD 11.3 where restarting a jail may cause it to lose access to devfs and /dev/dri may no longer be accessible in the jail. There are two known workarounds. One, create a dummy jail that also uses ruleset 10 and have it autostart. That way, there is always a jail that uses the ruleset, and restarting the Plex jail won't trigger the bug. Two, you could "iocage stop plex; service devfs restart; iocage start plex". In testing, this worked, but eventually crashed FreeNAS. Some care needed. TrueNAS 12.0 Core has been tested to not exhibit the issue, and no workarounds are needed
- The workarounds below for Plex 1.17.0 are *not* needed in Plex 1.18.8 or later.

##### Links
- Intel QuickSync versions and their capabilities can be found here: https://en.wikipedia.org/wiki/Intel_Quick_Sync_Video

## Notes
- 11/29/19- Plex Version 1.17.0.1709 was broken and disabled hardware transcode. If that version won't work for you there is a work around. Note this is *not* required for Plex 1.18.8 or later.

In your Plex jail backup:

`/usr/local/share/plexmediaserver-plexpass/lib/libva-drm.so.2`

`/usr/local/share/plexmediaserver-plexpass/lib/libdrm.so.2`

`/usr/local/share/plexmediaserver-plexpass/lib/libva.so.2`

Then replace the files with good versions by running (From your Plex jail):

`cp -a /usr/local/lib/libva-drm.so.2.500.0 /usr/local/share/plexmediaserver-plexpass/lib/libva-drm.so.2`

`cp -a /usr/local/lib/libdrm.so.2.4.0 /usr/local/share/plexmediaserver-plexpass/lib/libdrm.so.2`

`cp -a /usr/local/lib/libva.so.2.500.0 /usr/local/share/plexmediaserver-plexpass/lib/libva.so.2`

Also copy your driver to Plex

`cp -a /usr/local/lib/dri/i965_drv_video.so /usr/local/share/plexmediaserver-plexpass/lib/dri/`

- 11/29/19- In testing, after rebooting Freenas the Plex Jail could not see “dri” and “drm”, but after restarting the jail everything worked fine. This may have been related to the order of operations in the script, or to devfs in FreeBSD 11.3. This behavior has not been observed again after the 3/16/2020 changes.

- 12/1/19- Fixed a mistake in the "Testing" section.

- 26/2/19- Added compatibility (or lack thereof) notice about Gen 8 and Gen 9 support

- 3/16/20 - Added devfs restart and a few more notes about GPU and CPU generations. Added a note about Plex 1.18.8. Moved kldload to bottom of script because of issues encountered (no FreeNAS UI) when it was at the top.

- 3/29/20 - Added a troubleshooting section, clarified CPU generations, added compatible SM motherboards, made headings that explain which shell to be in more visible, changed ruleset to 10 to help work around a FreeBSD 11.3 issue.

- 4/8/20 - Added a note about HEVC and iGPU generations; added some more Xeon chipsets

- 4/15/20 - Removed Intel media driver and allow_mount_devfs notes, as it turns out neither of those is required for hw transcode to work

- 4/26/20 - Calling out codec explicitly for test results; some language cleanup; added a note about hardware acceleration disabling KVM in IPMI. Added a few more explicit instructions for people who are not at home on CLI.
