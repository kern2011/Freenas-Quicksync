# FreeNAS-Quicksync
##### How to guide for getting Intel Quicksync working on FreeNAS
Plex states that using hardware transcode on FreeBSD requires FreeBSD 12.0 or later. After some testing, it turns out it will work in FreeBSD 11.3 as well, although devfs is not as stable as in FreeBSD 12.x.

Note that for good results, you will need Intel gen 4 (Haswell) or later for 1080p transcode, and Intel gen 7 (Kaby Lake) or later for 4k transcode.

## Test Results
- Using an i7 4790K and using Quicksync for one 1080p to 720p or 1080p to 1080p stream left CPU usage at 1%.
- Testing on a 1225v6 Xeon (7th Gen Kaby Lake) used ~2-3% CPU for a 1080p VC-1 -> 1080p H.264 hardware transcode. The same machine was at 70-75% CPU for a software transcode of VC-1 -> H.264. Some Bluray disks used VC-1, before H.264 became ubiquitous.
- Testing on a 1225v6 Xeon (7th Gen Kaby Lake) used ~4% CPU for a 4k HDR HEVC 10-bit -> 1080p SDR H.264 hardware transcode. The same machine was at 100% CPU for a software transcode of the same file. Running four 4k hardware transcodes at once did not significantly tax the machine, further testing to determine the max number of possible 4k transcodes was not done.
- Testing on a G4560 Pentium (7th Gen Kaby Lake) used ~30% CPU for a 4k HEVC -> 1080p H.264 transcode.
- Testing on a Core i3 6100T (6th Gen SkyLake) shows that acceleration of a 4k HEVC -> 1080p H.264 transcode is minimal on it and takes 95% CPU. This is likely due to limited HEVC features in that generation.
- Core 4th Gen (Haswell) added H.264 and VC1 support in QuickSync. For 1080p transcode, "Haswell or better" is reasonable guidance.
- The Kaby Lake iGPU improved HEVC acceleration and added 10-bit (HDR) HEVC. The HEVC (H.265) codec is used on UHD Blurays. Consider a Kaby Lake or later CPU a requirement for significant acceleration of HEVC (Bluray 4k UHD) content. Note that without HDR10 tone mapping, 4k HDR transcode remains suboptimal, and it is often better to prepare a good SDR 1080p file ahead of time. HDR10 tone mapping is not yet available in a Desktop / Server CPU/iGPU combination.
## Compatibility
- The 11.3-Ux FreeNAS drivers support the iGPU in Intel CPU generation 2-7, Sandy Bridge through Kaby Lake.
- The 11.3-Ux FreeNAS drivers DO NOT support the iGPU in Intel CPU generation 8 and 9. This means Coffee Lake and Coffee Lake Refresh, Intel Xeon E-2xxxG and Intel Core i3/i5/i7/i9 8xxx and 9xxx and related Pentium.
- Support for the iGPU in generation 8 and 9 CPUs can not be added without a kernel upgrade to 12.x. Preliminary testing shows that TrueNAS Core 12.0 / FreeBSD 12.1 supports the iGPUs in 8th generation CPUs (Coffee Lake i3/5/7/9-8xxx and related Pentium/Celeron, Xeon E-21xxG), does not support the iGPUs in 9th generation desktop CPUs (Coffee Lake Refresh i3/5/7/9-9xxx and related Pentium/Celeron), and likely does support the iGPUs in 9th generation entry-level Xeon CPUs (Xeon E-22xxG).
- Full support for the iGPUS in generation 9 and 10 desktop CPUs (Coffee Lake Refresh and Comet Lake) is expected with a kernel upgrade to 13.x, potentially sometime 2021. Gen 10 support will require drm-kmod to be based on [Linux kernel >= 5.4](https://kernelnewbies.org/Linux_5.4#Graphics) by that time.
- Support for the iGPUs in generation 11 and 12 desktop CPUs (Rocket Lake, Alder Lake) is to be determined. Watch FreeBSD 13 and 14 developments for that, as well as [Linux kernel release notes](https://kernelnewbies.org/LinuxChanges).

## Hardware requirements Intel Xeon / Intel server C2xx chipsets
- Your chipset and motherboard/BIOS need to support the iGPU in your E3-xxx5 or E-2xxxG Xeon. Other Xeons do not have an iGPU.
- C206, C216, C226, C236, and C246 chipsets are able to support iGPU; C202/204, C222, C232 and C242 will not.
- Caveat that C246 will require TrueNAS 12.x.
- C226: SuperMicro X10SLH-F and Intel S1200V3RPM confirmed working; X10SLM+-F expected to work
- C236: SuperMicro X11SSH-F and AsRock Rack E3C236D2I confirmed working; X11SSA-F and X11SSi-LN4F expected to work; X11SSM-F will not work, X11SSV-M4F likely won't work. Dell R330/R230/T330/T130 will not work.
- C246: SuperMicro X11SCH-F expected to work; X11SCM-F will not work
- You can use these boards and chipsets with Intel Pentium or Intel i3/i5/i7 processors, verify supported CPUs for your specific board.
- These considerations do not apply to consumer chipsets.
- See https://ahelpme.com/servers/supermicro/enable-internal-graphics-in-supermicro-servers/ for details of enabling iGPU on a Supermicro board.
- Caveat: Turning on hardware acceleration for Plex will disable the KVM inside your IPMI. You will still have it available during initial boot, for switching boot environments or dropping into a debug session, but it will switch off once the iGPU drivers are loaded. The `compat.linuxkpi.modeset` (FreeNAS 11.3) or `compat.linuxkpi.i915_modeset` (TrueNAS 12.0) tunable can re-enable KVM in IPMI, but then hardware acceleration no longer works. Be sure that you are okay with needing to disable the bootup script when you require KVM after boot, for example when fixing a networking setting.
## Prep
-  First you must upgrade to FreeNAS 11.3 or later. This can be done from the UI System>Update and changing the train to 11.3 or later.
-  It may not be possible to upgrade your existing iocagei jails, in testing a jail broke during the upgrade. Keeping plex metadata outside of the jail makes it easy to recover by destroying and recreating the jail. If your metadata is inside the jail, make sure to back it up. Verify Plex is working before moving on.
-  Your mileage will vary: `iocage upgrade -R 11.3-RELEASE plex` worked for one user on a plugin jail (base jail). When in doubt, recreate the jail, and do make sure to use a base jail for ease of maintenance.
-  Note that using hardware transcode is likely easier in a custom Plex jail than in a plugin, because upgrades to plexmediaserver will not remove the necessary group membership.
-  Hardware transcode requires a PlexPass subscription. It does not require using the beta code in the plexmediaserver-plexpass package.
##  Making it work
### Easy Button
Use either the [freenas-iocage-plex](https://github.com/danb35/freenas-iocage-plex) or [jailman](https://github.com/jailmanager/jailman) scripts and tell them to use hardware transcode, they'll attempt to set everything up for you. See the script's documentation for details.
### Manually
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
 
`ls /dev/d*`

If everything worked; you should be able to find “dri” and “drm” folders in the output.

- Now go into your Plex settings, go to the “Transcoder” tab and make sure “Use hardware acceleration when available” is ticked.

- Play back a video at a resolution or bitrate that is different than the source file. 

- From your Plex dashboard you should see transcode with `(hw)` at the end. If you do then congratulations, it worked!

##### Troubleshooting

- `/dev/dri` needs to be visible in FreeBSD itself. If it isn't there, you can't make it visible in the jail. Check that your hardware is supported by your version of FreeNAS/TrueNAS, and that the UEFI/BIOS is set to enable the iGPU. You can use the `lspci` command from FreeNAS console to show the hardware detected by FreeBSD. If this does not show human-readable output, try `lspci -q`. You are looking for a Display Controller entry for Intel, for example for Kaby Lake Xeon: "Display controller: Intel Corporation HD Graphics P630 (rev 04)". If that is missing, either your CPU doesn't have an iGPU, your chipset does not support it, the BIOS doesn't support it, or it's not enabled in BIOS.
- There is a known issue with FreeBSD 11.3 where restarting a jail may cause it to lose access to devfs and /dev/dri may no longer be accessible in the jail. There are two known workarounds. One, create a dummy jail that also uses ruleset 10 and have it autostart. That way, there is always a jail that uses the ruleset, and restarting the Plex jail won't trigger the bug. Two, you could "iocage stop plex; service devfs restart; iocage start plex". In testing, this worked, but eventually crashed FreeNAS. Some care needed. TrueNAS 12.0 Core has been tested to not exhibit the issue, and no workarounds are needed
- The workarounds below for Plex 1.17.0 are *not* needed in Plex 1.18.8 or later.
- As of Plex 1.19.5, there is a [bug in Intel drivers](https://github.com/intel/media-driver/issues/752) that can cause "blocky" video in high-contrast VC1.
- There is a bug that causes "blocky" video on Gemini Lake (J4xx5/J5xx5 Celeron/Pentium) iGPUs.
- There is a bug present in Plex 1.19 and 1.20 that can stop hardware transcode from starting when using PGS/VOBSUB subtitles, TrueNAS Core 12.0, and a Kaby Lake CPU.

##### Links
- Intel QuickSync versions and their capabilities can be found here: https://en.wikipedia.org/wiki/Intel_Quick_Sync_Video
- Intel iGPUs supported by FreeBSD version: https://wiki.freebsd.org/Graphics/Intel-GPU-Matrix 
- Intel iGPUs supported by Linux kernel version: https://github.com/Intel-Media-SDK/MediaSDK/wiki/Intel-Graphics-Support-in-Linux-Kernels

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

- 6/22/20 - Added note about incompatible Dell models; mention scripts that support hardware transcode; added to troubleshooting section.

- 9/03/20 - Additional compatibility notes and links
