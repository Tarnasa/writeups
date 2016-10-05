Suspicious AVI - Azerbaijan
===========================

We are given an AVI file `avi.avi` which is a short meme-quality clip of an animated
animal turning a computer on and off.

Analyzing the file with ffprobe (from the ffmpeg package)
```
$ ffprobe avi.avi
[avi @ 0x5626d13b92c0] non-interleaved AVI
Input #0, avi, from 'avi.avi':
  Metadata:
    copyright       : http://countersite.org/images/hehehe.avi
    encoder         : Lavf56.36.100
  Duration: 00:00:10.12, start: 0.000000, bitrate: 37113 kb/s
    Stream #0:0: Video: png (MPNG / 0x474E504D), rgb24(pc), 640x360 [SAR 1:1 DAR 16:9], 37083 kb/s, 25 fps, 25 tbr, 25 tbn, 25 tbc
    Stream #0:1: Audio: aac (LC) ([255][0][0][0] / 0x00FF), 44100 Hz, stereo, fltp, 11 kb/s
```

One interesting thing about this file is the copyright, it is a link to another avi file!
Lets download this file.
```
$ wget http://countersite.org/images/hehehe.avi
```
It appears to be the exact same video, and since it is hosted by a non-competitor website,
is probably the original copy of the video.  Great!  So we can compare the two to find
where they hid the flag.

```
$ ffprobe hehehe.avi
Input #0, avi, from 'hehehe.avi':
  Metadata:
    encoder         : Lavf56.36.100
  Duration: 00:00:10.12, start: 0.000000, bitrate: 37098 kb/s
    Stream #0:0: Video: png (MPNG / 0x474E504D), rgb24(pc), 640x360 [SAR 1:1 DAR 16:9], 37083 kb/s, 25 fps, 25 tbr, 25 tbn, 25 tbc
    Stream #0:1: Audio: aac (LC) ([255][0][0][0] / 0x00FF), 44100 Hz, stereo, fltp, 11 kb/s
```
Almost exactly the same, except for the bitrate between the two streams
(37113 vs. 37098 kb/s) perhaps they are adding extra information to each frame?

So now to actually compare the files,
lets start by extracting the video and audio components of the two streams.
We can see from ffmpeg that the video for streams is actually a sequence of PNG images,
so lets get those images.
```
$ pngcheck -x avi.avi
```
This creates 253 avi-{number}.png files
```
$ pngcheck -x hehehe.avi
```
This creates 253 hehehe-{number}.png files
So apparently no frames were added.

Lets diff the frames between the images.
```
$ mkdir avi-png; mv avi-*.avi avi-png
$ mkdir hehehe-png; mv hehehe-*.avi hehehe-png
$ diff avi-png hehehe-png
Binary files avi-png/avi-173.png and hehehe-avi/hehehe-173.png differ
```

So what is the difference?
Lets use python because I like python
```python
#!/usr/bin/env python2

from PIL import Image

i1 = Image.open('avi-png/avi-173.png')
i2 = Image.open('hehehe-png/hehehe-173.png')

for y in range(i1.height):
    for x in range(i1.width):
        p1 = i1.getpixel((x, y))
        p2 = i2.getpixel((x, y))
        if p1 != p2:
            print(p1, p2, x, y)
```

Looking at the output, it looks like the r,g,b values differ only by at most
1, so they are probably doing LSB stego.

So lets grab the LSB of each component in r,g,b order.
```python
s = ''
for y in range(i1.height):
    for x in range(i1.width):
        p1 = i1.getpixel((x, y))
        p2 = i2.getpixel((x, y))
        for component in p1
            s += '1' if component & 1 else '0'

def unbin(s):
    return [int(s[i:i+8], 2) for i in range(0, len(s), 8)]

print(''.join(chr(c) for c in unbin(s)))
```

Unfortunately this gives garbage.
(I tried many other ways of ordering the bits here before figuring out the following).
Perhaps they want us to use the differences between the two images?
```python
for y in range(i1.height):
    for x in range(i1.width):
        p1 = i1.getpixel((x, y))
        p2 = i2.getpixel((x, y))
        for c1, c2 in zip(p1, p2):
            if c1 != c2:
                s += '1'
            else:
                s += '0'

print(''.join(chr(c) for c in unbin(s)))
```

This prints out a brainfuck program followed by garbage.
```
$ python2 extract.py | tee out.bf
+[----->+++<]>+.[-->+<]>.--[->++<]>-.++++++++.-[-->+<]>----.[--->++<]>--.+++++++
.-[->+++++<]>.[--->+<]>----.[-->+<]>-----.---.-[----->+<]>--.---------------.+++
+++++++++++.+++.[-->+<]>----.----[->++<]>-.+++++++++.[-->+<]>.--[->++<]>-.++++++
++.-[-->+<]>--.-[->++<]>.>--[-->+++<]>
```
So we grab the brainfuck program and compile it.
```
$ bfc out.bf
$ ./a.out
h4ck1t{br41n_mp4_h4ck3d
```
Woo we got the flag.

(Note that I originally spent at least 3 hours in a hex differ trying to figure out
the differences before simply splitting out the images)

