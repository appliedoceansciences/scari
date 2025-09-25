# SCARI acoustic data acquisition system

SCARI is a low power acoustic data acquisition system which samples a delta-sigma ADC and records continuously to nonvolatile SDMMC storage with a low steady state power consumption. The data is recorded as s16le PCM .wav files, with measures taken to minimize power consumption and maximize card lifetime. Secondary functionality allows computation of band level statistics and spectrograms in real time, which are emitted in compact representations via UART, and transmission of raw samples via USB to a host processor for soft-realtime processing.

The scope of this document is interface and usage details of interest to an end user or integrator, which are common across different hardware implementations of SCARI. Details specific to the SAMD51 or RP2350 variants of SCARI or of interest only to firmware maintainers shall be documented elsewhere.

## Acoustic data

The resolution of samples emitted by the ADC is 24 bits (although the effective linear SNR of the components upstream implies about 19 effective bits). In order to reduce power consumption and improve lifetime of the nonvolatile storage, these samples are dithered by adding a triangular PDF dither function, prior to truncation to 16 bits of precision. This is a well-established audio technique that whitens the quantization noise, allowing subsequent FFT based processing to recover the extra few bits of resolution, and see tones that would otherwise be below the quantization noise floor.

By default, the most significant 16 of the 24 bits are kept by the dithering and truncation, preserving the full-scale range of the ADC, but a "preshift gain" knob can be used to move the kept 16 bits of dynamic range down by one or several bits, giving 6 dB of additional gain per bit, at the expense of increased risk of clipping.

### Data logged by DAQ microcontroller

The SCARI DAQ performs low-power writes to an attached SD card when commanded. The data is recorded to `.wav` files, with header padding to preserve alignment of the recorded data with the card's erase cycle size, minimizing power consumption and maximizing card lifetime. Secondary functionality allows computation of spectrograms in real time, which are sent in a compact representation via UART, and transmission of raw sample values in near real time via USB to a Linux SBC.

#### Layout

For each logging session, the DAQ firmware will create a numbered directory, and then write timestamped files within it. Each file except the last one within a session will be the maximum size (4 GB, minus one cluster size) allowed by the .wav file format. There will be no gaps between the files as long as the firmware is able to close the last file, create the next file, and write its .wav header within the available time (a function of sample rate, SD card quality, and condition of the filesystem).

#### Fault handling

The file currently being written has a `.tmp` extension, rather than a `.wav` extension, and is always the maximum permissible size (i.e. approximately 4 GB). When closing the file, it is renamed to have a .wav extension, and the most up-to-date notion of its absolute start time is used to generate the filename. If it is the final file in the logging session, it is truncated to its filled amount.

If, upon powerup, the firmware notices that the most recent directory still contains a file with a `.tmp` extension, then its actual filled state is estimated from the `.wav` header, and it is truncated to that size and renamed. Since the `.wav` header is only updated once every few seconds, this will typically result in a loss of the last few seconds of recorded data upon powerup, after an abrupt poweroff.

The set of numbered directories increments from `000000` to `999999` and, unless data has been deleted from the card, it will then fail to continue to log data, although the firmware will continue to perform other processing and passthrough tasks.

#### Notes

Note that as currently implemented, this copy of the data, stored on the acoustic DAQ microSD card, cannot be retrieved by the Linux SBC. Recovery of data recorded to the microSD card on the acoustic DAQ is only possible by opening of the enclosure. If logging of raw data is enabled on the Linux SBC, it and any derived results can be exfiltrated via the SBC's Wi-Fi connection.

## Interface

### LED

An LED will blink once every four seconds as a sign of life. This may be routed to an external higher-power strobe.

### Piezo

On newer SCARI revisions, a piezo buzzer will give audible feedback of successful powerup, commanded state changes, and error conditions.

### Reed switch

Some variants of SCARI have a reed switch populated. If not started by UART command or automatically upon powerup, recording can be started by holding a magnet near the reed switch for more than 0.5 seconds and then removing it. The reed switch is also exposed as a momentary push button on the PCB itself.

### UART Commands

The DAQ firmware will listen for line-oriented commands on the UART, at a nominal initial baud rate of 115200, 8N1, 3.3V signalling level. The firmware shall react identically to lines of input terminated with any combination of `\r` or `\n`, in any order.

- `start`: Commence logging of incoming ADC data to gapless .wav files to nonvolatile storage
    - `start [value]`: If a value is given, it will be interpreted as a number of seconds after which to stop recording. WARNING: as currently implemented, if this feature is abused to generate a large amount of files per unit time, scaling issues in the FAT filesystem will cause the recorder to fail to make hard-rt deadlines after a matter of hours rather than months.

- `stop`: Request that logging of ADC data to .wav files be stopped, at the next cluster boundary, which may be several seconds depending on card cluster size and sample rate.

- `status`: Print whether rtc is set, whether recording is enabled, and whether USB is enabled

- `fs [value]`: Change from default sample rate to the closest allowable value to a desired rate. Has no immediate effect if any task has already requested that the ADC be on.

- `dsp on`, `dsp off`: Enable or disable the generation of NMEA-like `$PGRAM` and/or `$PSPL` messages which will be sent via the UART. These messages encode pixel data which can be rendered into a near-real-time scrolling spectrogram by downstream utilities, or plotted as line plots of band power versus time

- `pgram on`, `pgram off`: Enable or disable `$PGRAM` generation when DSP is on. Defaults to off. Must be performed before `dsp on`.

- `pspl on`, `pgram off`: Enable or disable `$PSPL` generation when DSP is on. Defaults to on. Must be performed before `dsp on`.

- `time`, `gpzda`: Generate and send a human-readable or NMEA `$GPZDA` string, respectively, representing the current RTC clock value.

- `$GPZDA,...`, `$GPRMC,...`: The code will parse a properly checksummed NMEA `$GPZDA` or `$GPRMC` string and use it to set the RTC.

- `flash`: Reset into the UF2 bootloader, and stay there until a firmware update is performed or a physical reset occurs.

- `usb on`, `usb off`: Enable or disable the USB port

- `reset`: Immediately perform a soft reset, without cleanly stopping any existing tasks

## Emitted messages

### `$PGRAM`

A compressed representation of spectrogram data suitable for low-bandwidth exfiltration.

    $PGRAM,[dt],[df],[bands_per_octave],[base64]*XX

These messages encode pixel values, or equivalently, power per band, in a format that can be rendered into a near-real-time scrolling spectrogram by downstream utilities, or plotted as line plots of band power versus time. The exact format of these messages is in flux, but they will be individually small enough to fit in an SBD message. These messages, with different settings for resolution and rate, support many of the desired use cases, ranging from scrolling realtime spectrograms to long-period band power averages. These messages end with a valid NMEA checksum, which need not be forwarded to shore.

The set of frequency bins is logarithmically spaced, switching to linear spacing below a limit determined uniquely by the values of `df` and `bands_per_octave`. Specifically, the first of the log-spaced output bins would be the Nth linearly spaced frequency bin from DC, counting from zero, where `N = ceil(bands_per_octave / log(2))`. By convention, the lowest two linear frequency bins (the DC bin and the one adjacent to it) are omitted, as they are dominanted by whatever residual DC component is present, such that there are `N - 2` linearly-spaced bins in each message, and the first of the log-spaced bins is centered at a frequency of `N * df`. Python functions are available which map back and forth between frequency and bin index, given `df` and `bands_per_octave`.

The `bands_per_octave` parameter is configurable at runtime. The value of `df` depends on the ADC sample rate and the FFT length. As currently implemented, the FFT length is not configurable, as most use cases would prefer a value at least as large as the available SRAM permits, and fixing it at compile time allows for a longer FFT length within the available SRAM.

The log-frequency bins are populated with weighted averages over a sliding window of the original linear Hann-windowed FFT bins, with width and normalization calculated such that properties of interest are continuous across the lin-log boundary. The frequency of a high-SNR tone in either region can be correctly inferred to sub-bin precision, and renormalizing the spectrogram to show noise spectral density in power per Hz works as expected.

The levels within each band are quantized to (by default) 0.75 dB increments and encoded with the half-open interval from 0 to 256 representing the half-open interval from -192 to 0 dB relative to full scale at the ADC. That is, a full-scale sine wave should result in a level of -3.01 dB, and be encoded with a bin value of 252. This range is sufficient to encode the quietest and loudest expected levels assuming the analog stage supports 22 effective bits, followed by a reasonable-length FFT.

As an example, given a sample rate of 31250 sps, an onboard fft length of 2048, and emitting 12 bands per octave: there would be 16 linearly-spaced bins with centers spanning from 30.5 to 259.4 Hz inclusive, followed by 70 log-spaced bins with centers from 274.7 to 14781 Hz inclusive. The lowest frequency captured by the lowest bin would be about 15.3 Hz. The total number of pixel-encoding bytes would be 86.

If the number of bands per octave were changed from 12 to 3, the message would encode 3 linear bins from 30.5 to 61.0 Hz inclusive, followed by 24 log-spaced (third-octave, or decidecade) bins centered from 76.3 to 15502 Hz, for a total of 27 pixel-encoding bytes.

The independent parameter `dt`, assuming it is longer than `0.5 / df`, indicates how many FFT frames were incoherently averaged together to produce each output message. This value can be dialed from milliseconds to hours as desired.

### `$PSPL`

A compressed representation of sound pressure level (SPL) in ANSI decidecade bands.

    $PSPL,[dt],[first band index],[base64]*XX

These messages encode the RMS sound pressure level (SPL) in decidecade bands. During each integration interval of length `dt` (nominally 0.98304 seconds), the mean-squared SPL is calculated for each decidecade band, with a weighting that accounts for alignment with the underlying FFT as in [1]. The set of reported decidecade bands will be those which fully fit within the flat portion of the passband, and have sufficient support within the underlying FFT, as noted in [1].

The rms SPLs are each encoded with 12 bits, representing the half-open interval from -192 dB to 0 dB relative to ADC full scale, in 0.046875 dB increments. After extracting the given unsigned 12-bit integer and applying these offset and scaling values, the final conversion to absolute rms SPL levels in dB re uPa² requires adding the level in dB re uPa² of a full-scale signal at the ADC. This offset, or scaling factor if applied in SI units after converting from dB, will be a simple property of the hydrophone and the analog signal path (primarily preamp gain) between it and the ADC.

Conversion of these values from mean-squared SPL in uPa², to SEL in uPa² seconds, may be performed downstream by multiplying the mean-squared SPL by `dt`. Note that these conversions must be applied in SI units prior to final conversion to dB.

- `$PREC,[flags],[just finished filename],[seconds written],[seconds free]*XX`: These messages will be emitted whenever the recording status changes. The least significant bit of the `flags` will be 0 if recording was not started or has stopped, or 1 if recording has started or is continuing. The name of the just-successfully-closed file, and the approximate number of seconds of data recorded to it, are emitted, followed by the approximate number of seconds (neglecting overhead) of remaining recording capacity, if known.

[1] ISO 7605:2023 section 1.1.4.3, Calculation of band levels in frequency domain using the DFT

## USB connectivity

Whenever commanded using `usb on` at the UART interface, the microcontroller will enable its USB port and present itself to a USB host as a USB CDC serial device. Opening this device with the associated `cobs_to_shm` utility will result in a logging and soft-realtime processing capability as documented in that repository.

## Use

Format an SDMMC card using a tool such as the official tool from sdcard.org, which is careful to align the FAT filesystem clusters to the physical erase cycle size of the card. For cards smaller than 64 GB, the formatter tool will choose FAT32 rather than exFAT, which would result in some operations taking prohivitively long. If this is the case, reformat the card using the mechanism built into your operating system.

Push the momentary push button (or reed switch) for > 0.5 seconds until the light blinks, then release it, to begin recording. Push again for > 0.5 seconds to request that recording stop. The recording will not stop until the next erase cycle boundary, which may be several seconds.

### Cards

Overall power consumption when logging is strongly affected by choice of SDMMC card. As of this writing, the best known card by these metrics is the Samsung Evo Plus 256 GB. The 512 GB card from the same product line is one of the worst, while the 1 TB Samsung Evo Select is closer to the 256 GB card in acceptability, so card manufacturer alone is not necessarily a good indicator of resulting performance.

A Sandisk Extreme Plus 128GB did not work, symptom was solid red activity light and 400-600% card overhead factors.

Other known working cards: Sandisk Ultra 64GB Class 10, Sandisk Extreme 1TB (draws significantly more power than the Samsung 1TB card)

Known NOT working cards: Cheap Amazon Kootion 64 GB. The lower quality cards may or may not work if more strict assumptions are satisfied w/r series resistors on the SPI pins, baud rate, power supply decoupling, and whatnot - this will require more testing.
