.. SPDX-License-Identifier: GPL-2.0-only

Waveshare 5inch DSI LCD on Arduino UNO Q
========================================

Hardware and reference
----------------------

This configuration targets the Waveshare 5inch DSI LCD Rev2.2 (SKU 18396),
an 800x480 display connected to the UNO Media Carrier ASX00083 DSI connector.
Inspection of the panel identified a separate ICN6211 DSI-to-RGB bridge;
WSDIA1 handles control functions. The board device tree is named ``imola``
and uses the Agatti (QRB2210) display controller. The existing
``panel-5in_touch_a-dsi`` overlay describes a different 720x1280 display.

The panel assembly is represented as a directly attached DSI panel in the
Linux device graph. The physical video path remains DSI to ICN6211 to LCD.
Its video settings come from the Raspberry Pi
``vc4-kms-dsi-waveshare-800x480`` overlay.
Product information is available at:

https://www.waveshare.com/wiki/5inch_DSI_LCD

The port uses these video and control settings:

* DSI0, virtual channel 0, one data lane, RGB888.
* DSI mode flags: ``MIPI_DSI_MODE_VIDEO`` only, without the sync-pulse flag.
* 27.777 MHz pixel clock, 800x480 active pixels, approximately 60 Hz.
* Horizontal front porch/sync/back porch: 59/2/45 pixels.
* Vertical front porch/sync/back porch: 7/2/22 lines.
* CCI I2C0 at 100 kHz, with the display power/backlight controller at
  address 0x45 and the FT5x06-compatible touchscreen at address 0x38.
* The Pi-compatible control driver supplies panel power and backlight
  control. GPIO 0 releases LCD/bridge reset through an always-on fixed
  regulator wrapper; GPIO 1 is used for touch reset. The touchscreen has no
  interrupt on the DSI cable, so the input driver polls it while it is open.

``waveshare,5inch-dsi-lcd`` binds to ``panel-simple-dsi``. This driver
provides the fixed mode and power/backlight integration for the complete
assembly. It sends no Toshiba DSI register commands and does not separately
program the ICN6211. Compatibility with the Pi's I2C power-controller
interface does not imply compatibility with the TC358762 DSI protocol.
The reset wrapper is still needed with this graph: power enable alone
does not release LCD/bridge reset. The GPIO 0 operation also triggers the
control driver's post-reset I2C writes. This use of the Pi-compatible
interface does not establish WSDIA1's underlying silicon or full protocol.

Kernel configuration and device tree
------------------------------------

The arm64 ``defconfig`` enables the required drivers, including::

    CONFIG_DRM_MSM=m
    CONFIG_DRM_MIPI_DSI=y
    CONFIG_DRM_PANEL_SIMPLE=m
    CONFIG_REGULATOR_RASPBERRYPI_TOUCHSCREEN_ATTINY=m
    CONFIG_TOUCHSCREEN_EDT_FT5X06=m
    CONFIG_I2C_QCOM_CCI=m

For an existing board-specific configuration, enable these options and
their dependencies as well. Build the kernel/modules and this composed
device tree using a Linux arm64 build environment, or add the appropriate
``CROSS_COMPILE`` setting on another architecture::

    make O=out ARCH=arm64 defconfig
    make O=out ARCH=arm64 -j"$(nproc)" Image modules
    make O=out ARCH=arm64 \
      qcom/qrb2210-arduino-imola-carrier-media-waveshare-5inch.dtb

The composed DTB contains, in order:

* ``qrb2210-arduino-imola-base.dtb``
* ``qrb2210-arduino-imola-carrier-media.dtbo``
* ``qrb2210-arduino-imola-carrier-media-panel-waveshare-5inch-dsi.dtbo``

Select that DTB using the board's boot configuration and install the
matching kernel and modules. Alternatively, apply those two overlays to
the base DTB in the listed order. The panel overlay also enables CCI so
it can be applied directly to the base for display-only use.

Use only one DSI display overlay. In particular, do not combine this
overlay with ``video_sound-usbc`` or another panel overlay: they share
the same DSI0 output. The default ``qrb2210-arduino-imola.dtb`` already
contains ``video_sound-usbc``; use the base DTB when applying overlays.
Connect and power the panel according to the media carrier and display
connector pinouts.

Hardware status
---------------

No working image or illuminated backlight has been demonstrated on this
unit. The reported measurements and observations are:

* 3.3 V reaches the panel.
* Accepted I2C commands have not produced illumination.
* The traced WSDIA1 output feeding the transistor chain remains near 0 V.
  Its identification as ``DC_EN`` is supported by a related schematic,
  rather than an exact WSDIA1 datasheet.
* Whether active video is required before that output rises remains
  unproven.

Build and device-tree validation establish software integration only.
Actual display output, brightness, touch orientation and suspend/resume
remain to be demonstrated on the connected hardware. The direct DSI model
and timing corrections are not evidence that the illumination issue is
resolved.

During direct-I2C experiments, the panel-controller driver was deliberately
blocked in ``/etc/modprobe.d/mkboot-display-debug.conf`` with::

    blacklist rpi_panel_attiny_regulator
    install rpi_panel_attiny_regulator /bin/false

Touch was also reported disabled or unbound. Check for these blocks before
testing the kernel drivers. An unbound controller prevents dependent
devices from probing, but does not explain the lack of illumination after
acknowledged direct-I2C writes.

The existing control driver does not propagate all power/reset I2C write
errors. Successful binding or cached state therefore cannot establish
that its full initialization sequence reached the hardware. No complete
successful sequence, including the post-reset writes, was confirmed in
the recorded experiments. Additional reset/retry workarounds have not
been validated on this unit.

Software and board verification
-------------------------------

After booting the matching kernel and DTB:

* Confirm that ``panel-simple-dsi``, ``rpi_touchscreen_attiny``
  and ``edt_ft5x06`` bind without probe or I2C errors.
* Check ``/sys/class/drm/card*-DSI-*/modes`` for ``800x480`` and use
  ``modetest -M msm -c`` to inspect the connector and mode. Confirm the
  27777 kHz clock and 59/2/45 horizontal porches. An advertised mode or
  successful modeset alone does not prove physical video output.
* Check the display controller's entry under ``/sys/class/backlight``.
  Its brightness range is 0 to 255; test zero, intermediate and full
  brightness after enabling the display. Record command acceptance and
  observed illumination separately.
* Compare the traced control output before and during active video to
  test whether video affects its level. Keep the ``DC_EN`` identification
  and any video dependency provisional until supported by measurements.
* Use ``evtest`` on the EDT input device. Test all corners, multiple
  fingers and releasing every finger. Both touch axes are inverted to
  match the Raspberry Pi overlay; adjust the touchscreen properties if
  the physical orientation requires it.
* Repeat display blank/unblank and system suspend/resume, checking for
  lost touch input, stuck contacts or regulator errors.
