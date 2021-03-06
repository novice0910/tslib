# tslib

Touchscreen access library

[![Coverity Scan Build Status](https://scan.coverity.com/projects/11027/badge.svg)](https://scan.coverity.com/projects/tslib)

## Getting tslib

Apart from directly [building](#building-tslib) it, tslib is currently
maintained by the following distributors:

* [Buildroot](https://buildroot.org/)
* [Arch Linux](https://www.archlinux.org/)

## Building tslib

#### GNU/Linux
For tarballs `./configure && make && make install` applies, see the
[INSTALL](https://github.com/kergoth/tslib/blob/master/INSTALL)
file for more details. For the sources run `./autogen.sh` first.

#### Android/Linux
Extract tslib's tarball into `<base>/external/` of your
Android sources and build the components you need
like `make libts`, `make ts/plugins/input`, `make ts_uinput`, ...,
see LOCAL_MODULE in Android.mk.
Refer to [Android's documentation](https://developer.android.com/ndk/guides/build.html) for the details.

## What is tslib?

The idea of tslib is to have a core library that provides standardised
services, and a set of plugins to manage the conversion and filtering
of touchscreen input data as needed.

The plugins for a particular touchscreen are loaded automatically by the
library under the control of a static configuration file, ts.conf.
ts.conf gives the library basic configuration information.  Each line
specifies one module, and the parameters for that module.  The modules
are loaded in order, with the first one processing the touchscreen data
first.  For example:

```
  module_raw input
  module median depth=3
  module dejitter delta=100
  module linear
```

These parameters are described below.

With this configuration file, we end up with the following data flow
through the library:

```
  raw read --> median  --> dejitter --> linear --> application
  module       module      module       module
```

You can re-order these modules as you wish, add more modules, or remove them
all together.  When you call `ts_read()` or run `ts_uinput -d` and read from
the new input device, see [below](#use-tslib-via-a-normal-input-event-device),
the values you read are values that
have passed through the chain of filters and scaling conversions.  Another
call is provided, `ts_read_raw()` which bypasses all the modules and reads the
raw data directly from the device.

There are a couple of programs in the `tslib/tests` directory which give example
usages.  They are by no means exhaustive, nor probably even good examples.
They are basically the programs used to test this library.

#### Example use cases

Years ago (or in part still for resistive touch screens) a use case for tslib to *enable*
using the device would look like
* use a hardware specific `module_raw` plugin (single touch only)
* mainly use the `linear` filter plugin (and `ts_calibrate`)
* use `ts_read()` (in the form of a plugin for Qt, X11, ...)

While being fully backwards compatible, nowadays, for capacitive touch screens
a use case to *optimize* the touch experience or work around hardware or driver
bugs would be
* use the `module_raw input` plugin for Linux drivers (and have multi touch)
* use a combination of other [filter plugins](#module-parameters) to optimize the touch experience
* have the `ts_uinput -d` daemon running and use it's
[input device](#use-tslib-via-a-normal-input-event-device) in your environment

## Use tslib via a normal input event device

Instead of using tslib's API calls, you can use `tslib/tools/ts_uinput` which
creates (via uinput) a new standard input event device you can use in your
environment. The new device provides the filtered and calibrated values and
should work with single- and multitouch devices. `ts_uinput_start.sh` starts
`ts_uinput` as a daemon and creates a link named `/dev/input/ts_uinput` for
convenience.

## Multitouch

Similar to the mentioned `ts_read()`, `ts_read_mt()` reads one `struct ts_sample_mt`
per slot (number of possible contacts) and desired number of samples. You have
to provide slots*nr of them to hold the resulting values, see the multitouch
programs in `tslib/tests` for examples; there is, of course, `ts_read_raw_mt()` too.

`ts_read_mt()` aims to be a drop-in replacement for `ts_read()`, so you can use
it for any single touch device too, providing space for one slot.

## Environment Variables

```
TSLIB_TSDEVICE			TS device file name.
				Default (inputapi): 	/dev/input/ts
							/dev/input/touchscreen
							/dev/input/event0
				Default (non inputapi):	/dev/touchscreen/ucb1x00
TSLIB_CALIBFILE			Calibration file.
				Default: ${sysconfdir}/pointercal
TSLIB_CONFFILE			Config file.
				Default: ${sysconfdir}/ts.conf
TSLIB_PLUGINDIR			Plugin directory.
				Default: ${datadir}/plugins
TSLIB_CONSOLEDEVICE		Console device.
				Default: /dev/tty
TSLIB_FBDEVICE			Framebuffer device.
				Default: /dev/fb0
```

## Module Parameters

### module:	variance

#### Description:
  Variance filter. Tries to do it's best in order to filter out random noise
  coming from touchscreen ADC's. This is achieved by limiting the sample
  movement speed to some value (e.g. the pen is not supposed to move quicker
  than some threshold).

  This is a 'greedy' filter, e.g. it gives less samples on output than
  receives on input. It can cause problems on capacitive touchscreens that
  already apply such a filter.

  There is **no** multitouch support for this filter (yet). `ts_read_mt()` will
  only read one slot, when this filter is used. You can try using the median
  filter instead.

#### Parameters:
* `delta`

	Set the squared distance in touchscreen units between previous and
	current pen position (e.g. (X2-X1)^2 + (Y2-Y1)^2). This defines the
	criteria for determining whenever two samples are 'near' or 'far'
	to each other.

	Now if the distance between previous and current sample is 'far',
	the sample is marked as 'potential noise'. This doesn't mean yet
	that it will be discarded; if the next reading will be close to it,
	this will be considered just a regular 'quick motion' event, and it
	will sneak to the next layer. Also, if the sample after the
	'potential noise' is 'far' from both previously discussed samples,
	this is also considered a 'quick motion' event and the sample sneaks
	into the output stream.


### module: dejitter

#### Description:
  Removes jitter on the X and Y co-ordinates. This is achieved by applying a
  weighted smoothing filter. The latest samples have most weight; earlier
  samples have less weight. This allows to achieve 1:1 input->output rate. See
  [Wikipedia](https://en.wikipedia.org/wiki/Jitter#Mitigation) for some general
  theory.

#### Parameters:
* `delta`

	Squared distance between two samples ((X2-X1)^2 + (Y2-Y1)^2) that
	defines the 'quick motion' threshold. If the pen moves quick, it
	is not feasible to smooth pen motion, besides quick motion is not
	precise anyway; so if quick motion is detected the module just
	discards the backlog and simply copies input to output.


### module: linear

#### Description:
  Linear scaling module, primerily used for conversion of touch screen
  co-ordinates to screen co-ordinates. It applies the corrections as recorded
  and saved by the `ts_calibrate` tool.

#### Parameters:
* `xyswap`

	interchange the X and Y co-ordinates -- no longer used or needed
	if the linear calibration utility `ts_calibrate` is used.

* `pressure_offset`

	offset applied to the pressure value
* `pressure_mul`

	factor to multiply the pressure value with
* `pressure_div`

	value to divide the pressure value by


### module: pthres

#### Description:
  Pressure threshold filter. Given a release is always pressure 0 and a
  press is always >= 1, this discards samples below / above the specified
  pressure threshold.

#### Parameters:
* `pmin`

	Minimum pressure value for a sample to be valid.
* `pmax`

	Maximum pressure value for a sample to be valid.


### module: debounce

#### Description:
  Simple debounce mechanism that drops input events for the specified time
  after a touch gesture stopped. [Wikipedia](https://en.wikipedia.org/wiki/Switch#Contact_bounce)
  has more theory.

#### Parameters:
* `drop_threshold`

	drop events up to this number of milliseconds after the last
	release event.


### module: skip

#### Description:
  Skip nhead samples after press and ntail samples before release. This
  should help if for the device the first or last samples are unreliable.

Parameters:
* `nhead`

	Number of events to drop after pressure
* `ntail`

	Number of events to drop before release


### module: median

#### Description:
  Similar to what the variance filter does, the median filter suppresses
  spikes in the gesture. For some theory, see [Wikipedia](https://en.wikipedia.org/wiki/Median_filter)

Parameters:
* `depth`

	Number of samples to apply the median filter to


## Module Creation Notes

For those creating tslib modules, it is important to note a couple things with
regard to handling of the ability for a user to request more than one ts event
at a time.  The first thing to note is that the lower layers may send up less
events than the user requested, because some events may be filtered out by
intermediate layers. Next, your module should send up just as many events
as the user requested in nr. If your module is one that consumes events,
such as variance, then you loop on the read from the lower layers, and only
send the events up when
1. you have the number of events requested by the user, or
2. one of the events from the lower layers was a pen release.
