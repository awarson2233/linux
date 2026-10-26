Gaokun3 downstream kernel
========================

This tree carries device changes for Huawei MateBook E Go 2023 (SC8280XP).
The first migration baseline is gregkh/linux linux-rolling-stable commit
d396b05e7e39b0ed6f6d5553fbaf174228e18bdf (Merge v7.2.6).

Status: migration candidate, not a hardware-validated release. Building a DTB
or kernel does not verify suspend, audio, Type-C, display, or hardware decode.

Build
-----

On an arm64 Linux host with kernel build dependencies installed::

    make O=../out ARCH=arm64 gaokun3_defconfig
    make O=../out ARCH=arm64 -j$(nproc) Image modules dtbs

On an x86 host add CROSS_COMPILE=aarch64-linux-gnu- to both commands.
Packaging and image creation belong in gaokun3/buildbot. The defconfig, DTS
and drivers in this tree are the maintained source; builds must not overlay
copies from the image repository.

Migration decisions
-------------------

* Source: KawaiiHachimi/linux-gaokun-buildbot at
  315528c028843794ccd0f3d9dab373b03033150f.
* Preserve the original authors of imported patches.
* Omit the old PDC mapping patch: both added ranges already exist in the
  selected baseline. Rebase the separate touchscreen IRQ workaround.
* Merge the EC DTS change onto the upstream GPIO 103 correction; PDC 215
  maps to that GPIO. Retain the other upstream changes while applying the
  board enablement diff.
* gaokun3-next drops the six downstream Venus patches and their board
  enablement commit. Backport upstream Iris node commit
  3a52eef16b979617156fa6e2ead24ca4d1336f0b and enable the board firmware.
  The stable Iris driver uses the sm8250 fallback compatible.
  Memory sorting from 53275adfb07d416a32004612227265939d0df00d is already
  present in the baseline; do not backport it again. Hardware codec
  operation remains unverified.
* Import right-0903 force-GSI patch fb123793bfdc5a58f94aaf54ae06b47c9c797b7e
  so the existing qcom,force-gsi-mode property is actually consumed.
* Touch defaults to I2C-HID through the mainline hid-over-i2c node; the
  controller computes the touches. The in-kernel Himax/hx-algo SPI driver
  is removed. sc8280xp-huawei-gaokun3-touch-spi.dtb switches both
  controllers to SPI for the himax-thp transport (CONFIG_HIMAX_THP), which
  hands raw frames to a userspace daemon (egotouchd from EGoTouchRev).
* CONFIG_INPUT_UINPUT=m is already present in the imported configuration.
  The user-space parts of PR #2 belong in the image repository.

Before the first release
------------------------

* Move camera support to a separately tested feature series; the four
  inherited HI846 commits and camera DTS are still in this candidate.
* Review panel DSC/backlight workarounds against the upstream panel driver
  before dropping them.
* Review the UCSI and q6apm changes removed by buildbot commit 9420138.
  Their older patches no longer apply to this baseline. Do not classify
  them as upstreamed merely because they conflict.
* Verify distribution-specific LSM selection. The inherited defconfig
  lists AppArmor in CONFIG_LSM; Fedora images must select SELinux via lsm=.
  Verify /sys/kernel/security/lsm and the distro policy on the device.
* Validate boot, touch, 60/120 Hz display, audio, Wi-Fi, Bluetooth, charging,
  USB-C, suspend/resume, video decode, and kernel upgrade/rollback on hardware.
* EL2 is separate work: the newer remoteproc asynchronous attach and q6v5
  running-state handling need review before carrying the older EL2 series.

Updates
-------

The gaokun3-next branch is the Iris/GSI review candidate. The previous
gaokun3 branch is retained for comparison, not a hardware validation claim.

The gaokun3 branch is a linear device patch series and may be rebased by its
maintainer. Record the upstream base SHA with each immutable release tag.
Work on a temporary branch, then compare the old and new series::

    git rebase -i --onto NEW_BASE OLD_BASE WORK_BRANCH
    git range-diff OLD_BASE..OLD_HEAD NEW_BASE..WORK_BRANCH

Drop a patch only after confirming that the chosen new base provides the
required behavior. Keep release tags unchanged and use a lease when updating
the public rebased branch. Image builds consume a reviewed commit SHA.
