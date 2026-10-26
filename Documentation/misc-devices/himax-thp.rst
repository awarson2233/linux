.. SPDX-License-Identifier: GPL-2.0

========================================
Himax HX83121A SPI transport (himax-thp)
========================================

In SPI mode the Himax HX83121A touch controller does not compute touches; it
streams raw capacitance frames and leaves touch processing to the host. The
himax-thp driver handles the hardware side of this and nothing else: the SPI
buses, the frame interrupts, the reset line and the panel power state. Chip
initialization, controller commands, the frame reading policy and touch
processing belong to a userspace daemon (egotouchd from EGoTouchRev), which
reports touches through uinput or uhid.

The Huawei MateBook E Go (Gaokun3) has two cascaded controllers. The
default device tree describes them through hid-over-i2c, where the
controller computes the touches itself;
``sc8280xp-huawei-gaokun3-touch-spi.dtb`` switches them to SPI for this
driver. See
Documentation/devicetree/bindings/input/touchscreen/himax,hx83121a-ts.yaml.

Device and channels
===================

The driver creates one character device, ``/dev/himax_thp`` (mode 0600 by
default). It has two channels, ``HIMAX_THP_CH_MASTER`` and
``HIMAX_THP_CH_SLAVE``, one per controller. A file binds a channel with
``HIMAX_THP_IOC_SET_CHANNEL`` before it uses it; a channel is bound to at
most one file at a time, a second file gets ``-EBUSY``. Without a cascaded
slave in the device tree, binding the slave channel fails with ``-ENODEV``.
Closing a file releases its channel and stops frame reading on it.

The bus commands give raw register access to the controllers, including
their flash, so the device node should stay root only.

Bus transfers
=============

``HIMAX_THP_IOC_BUS_READ`` and ``HIMAX_THP_IOC_BUS_WRITE`` take a
``struct himax_thp_bus`` and run one SPI transfer on the bound channel. The
driver adds the bus opcode of the channel (F3/F2 on the master, F5/F4 on the
slave) and the bus header; userspace passes the command byte and up to
``HIMAX_THP_BUS_MAX`` bytes of data. Transfers of both channels and frame
reads are serialized.

Frames
======

A controller pulls its interrupt line low when a frame is ready and keeps it
low until the frame has been read, so the frame interrupt is falling edge.
``HIMAX_THP_IOC_INT_ENABLE`` with 1 starts reading frames on the bound
channel from its interrupt and reads one frame right away: a controller that
already holds a frame raises no further edge until that frame is collected.
A channel whose read failed therefore stalls until userspace enables it
again. ``HIMAX_THP_IOC_INT_ENABLE`` with 0 stops reading and drops the queued
frames.

Each channel queues up to ``HIMAX_THP_FRAME_QUEUE`` frames; when the queue is
full a newly read frame is dropped. ``read()`` returns the oldest frame as a
``struct himax_thp_frame_hdr`` (interrupt time on ``CLOCK_MONOTONIC``,
sequence number, length, frames dropped so far) followed by the frame data
without the bus header. A buffer too small for the whole frame gets
``-EINVAL``. ``poll()`` reports ``EPOLLIN`` for a queued frame, ``EPOLLPRI``
for a panel state change and ``EPOLLERR`` for an unbound file. The file
supports ``O_NONBLOCK``.

Reset
=====

``HIMAX_THP_IOC_RESET`` holds the reset line, which both controllers share,
asserted for the given number of milliseconds (``HIMAX_THP_RESET_MIN_MS`` to
``HIMAX_THP_RESET_MAX_MS``) and releases it; no transfer runs during the
pulse. The controllers select their host interface from the mode line when
their firmware reloads after the reset; the driver drives it to SPI at probe.
Waiting for the firmware is up to the caller.

Panel power
===========

The touch part of the TDDI is powered with the display panel. The driver
follows the panel through ``drm_panel_follower``; ``HIMAX_THP_IOC_PANEL_STATE``
returns ``HIMAX_THP_PANEL_ON`` (panel powered) and
``HIMAX_THP_PANEL_ENABLED`` (scanning out), and clears ``EPOLLPRI`` for the
file. When the panel powers off the driver stops frame reading on both
channels; the controllers lose their state and userspace initializes them
again after power up.

After the driver is unbound, open files get ``-ENODEV`` and ``poll()``
reports ``EPOLLHUP``.
