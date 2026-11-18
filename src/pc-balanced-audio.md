Balanced audio direct from the PC
=================================

This started with a simple problem: wanting to connect an active monitor (i.e. powered speaker) to a (personal) computer running a modern Linux-based OS. But how?

This article concerns a minimalist approach: one cable per speaker to your desktop/laptop computer's existing audio output jacks.

### Motivations

Usually a Y-cable (3.5mm TRS to two 6.35mm TS or XLR connectors) is used for this job. While that works, professional equipment like audio mixers usually use a separate connector for each speaker. We can do the same, allowing:

- No need for complex adapter cables: just one cable per speaker
- No need for external DACs or audio interfaces
- We can get proper mono output, instead of missing a channel or using an adapter to bridge the left and right channels (not guaranteed to work correctly)
- Reduced susceptibility to noise from external sources (despite being the raison d'être for [balanced audio] this ranks dead last: it usually won't make a notable difference; further it won't fix the noise from poorly isolated DACs built into some PCs/screens)

### Retasking input jacks

If your computer only has a single output jack you will only be able to connect a single speaker (mono) using this approach; otherwise use a Y-cable or get an external audio interface.

*But* as it turns out, many (desktop) computers have input jacks which can be retasked as outputs, intended to support 4-, 5.1- or 7.1-channel audio output. On Linux, the [`hdajackretask`](https://github.com/alsa-project/alsa-tools/tree/master/hdajackretask) utility can be used to do this. Install via the `alsa-tools` (Fedora) or `alsa-tools-gui` (Debian, Ubununtu) package.

### Cables

Almost every active studio monitor supports [balanced audio] input using a three-pin XLR or ¼ inch (6.35mm) TRS connector. Since PC audio outputs are usually 3.5mm TRS jacks we'll want some type of adapter cable:

- 3.5mm TRS to ¼ inch (6.35mm) TRS: widely available "stereo" adapter cables ([example](https://www.thomann.co.uk/cordial_cfm_15_wv.htm))
- 3.5mm TRS to XLR: not widely available; your best bet might be to buy a quality 3.5mm-to-3.5mm stereo cable, cut it in half and solder on XLR connectors (connections: 1 to shield, 2 to tip, 3 to ring)

Note: without software configuration this cable configuration won't work well.


PipeWire
--------

Many (I think most) modern Linux distributions now use [PipeWire](https://pipewire.org/) as the default audio server. This is all you need. (An alternative [using ALSA](#ALSA) may be possible. I have no clue what is possible in other operating systems.)

First, verify that your system is actually running PipeWire:
```sh
$ systemctl --user status pipewire
● pipewire.service - PipeWire Multimedia Service
     Loaded: loaded (/usr/lib/systemd/user/pipewire.service; disabled; preset: disabled)
    Drop-In: /usr/lib/systemd/user/pipewire.service.d
             └─00-uresourced.conf
             /usr/lib/systemd/user/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Mon 2026-11-16 08:59:45 GMT; 2 days ago
```

### Configuration

Now, prepare a configuration directory:
```sh
mkdir -p $HOME/.config/pipewire/pipewire.conf.d
```
User configuration is placed in this directory, using any number of files each named <code><em>something</em>.conf</code>. You should be able to save the raw files linked below directly to this directory.

If modifying the samples below, ensure that each has a unique node name (`node.name`) to help your system remember which output was last used and its volume levels.

After updating configuration, restart the sound server:
```sh
systemctl --user restart pipewire pipewire-pulse
```
Each new module should be listed as an "output device" in your desktop's sound configuration.

#### Jack/channel names

In each case, we specify an `audio.position` for playback, e.g. `FL FR` for front-left and front-right channels (i.e. the first output jack). Input jacks can sometimes be retasked as outputs ([see above](#retasking-input-jacks)). Jack channel names:

- `FL FR` (front)
- `RL RR` (rear)
- `FC LFE` (centre + sub)
- `SL SR` (side)

### Unbalanced mono

The simplest functional setup would be unbalanced mono: a single capture cannel (`MONO`) routed to a single output pin (`FL` aka front-left, or alternatively `FR`).
```json
context.modules = [
    {
        name = libpipewire-module-loopback
        args = {
            node.name        = "mono"
            node.description = "Mono (unbalanced)"
            capture.props = {
                media.class    = "Audio/Sink"
                audio.position = [ MONO ]
            }
            playback.props = {
                audio.position = [ FL ]
                node.passive   = true
            }
        }
    }
]
```
Raw: [`unbalanced-mono.conf`](raw/unbalanced-mono.conf)

### Balanced mono

Why not use (active) balanced output? This is a bit more complicated than unbalanced but nothing you can't copy+paste:
```json
context.modules = [
    {
        name = libpipewire-module-filter-chain
        args = {
            node.name        = "mono-balanced"
            node.description = "Mono (balanced)"

            capture.props = {
                media.class    = "Audio/Sink"
                audio.position = [ MONO ]
            }

            playback.props = {
                audio.position = [ FL FR ]
            }

            filter.graph = {
                nodes = [
                    {
                        name  = mono
                        type  = builtin
                        label = copy
                    }
                    {
                        name  = normal
                        type  = builtin
                        label = copy
                    }
                    {
                        name  = inversion
                        type  = builtin
                        label = invert
                    }
                ]
                inputs  = [ "mono:In" ]
                links = [
                    { output = "mono:Out" input = "normal:In" }
                    { output = "mono:Out" input = "inversion:In" }
                ]
                outputs = [ "normal:Out", "inversion:Out" ]
            }
        }
    }
]
```
Raw: [`balanced-mono.conf`](raw/balanced-mono.conf)

### Balanced stereo

The above can be adapted for stereo output over two output jacks:
```json
context.modules = [
    {
        name = libpipewire-module-filter-chain
        args = {
            node.name        = "stereo-balanced"
            node.description = "Stereo (separate jacks)"

            capture.props = {
                media.class    = "Audio/Sink"
                audio.position = [ FL FR ]
            }

            playback.props = {
                audio.position   = [ FL FR RL RR ]
            }

            filter.graph = {
                nodes = [
                    {
                        name  = left
                        type  = builtin
                        label = copy
                    }
                    {
                        name  = left1
                        type  = builtin
                        label = copy
                    }
                    {
                        name  = left2
                        type  = builtin
                        label = invert
                    }
                    {
                        name  = right
                        type  = builtin
                        label = copy
                    }
                    {
                        name  = right1
                        type  = builtin
                        label = copy
                    }
                    {
                        name  = right2
                        type  = builtin
                        label = invert
                    }
                ]
                inputs  = [ "left:In" "right:In" ]
                links = [
                    { output = "left:Out" input = "left1:In" }
                    { output = "left:Out" input = "left2:In" }
                    { output = "right:Out" input = "right1:In" }
                    { output = "right:Out" input = "right2:In" }
                ]
                outputs = [ "left1:Out" "left2:Out" "right1:Out" "right2:Out" ]
            }
        }
    }
]
```
Raw: [`balanced-stereo.conf`](raw/balanced-stereo.conf)

### Other configurations

Half-balanced stereo: [`half-balanced.conf`](raw/half-balanced.conf)


ALSA
----

If not using PipeWire, it may be possible to do something similar with ALSA (Advanced Linux Sound Architecture) configuration. Place the following in `/etc/asound.conf` (untested):
```conf
pcm.balanced_mono {
    type route
    slave {
        pcm onboard
        channels 2
    }
    ttable {
        0.0 0.5
        1.0 0.5
        0.1 -0.5
        1.1 -0.5
    }
}

pcm.!default balanced_mono
```


[balanced audio]: https://en.wikipedia.org/wiki/Balanced_audio
