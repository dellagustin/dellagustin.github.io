---
layout: post
title:  "How to create GIFs and MP4 videos from NES emulator captures"
date:   2018-09-03 21:50:38 +0200
categories: videogames 
tags: NES gif mp4
---

## Intro and some background (Jump over for the how to)

I recently started collaborating with a Brazilian website and podcast called
[Fliperama de Boteco][fliperama-de-boteco]{:target="_blank"}. The name refers to 
a video arcade that happened to be in a low level type of bar (common on our late 
80s and through the 90s) and the website is mainly focused on retro gaming.

I wanted to write an analysis on the game _Vice: Project Doom_ for the NES and
put some GIFs to illustrate parts of the gameplay, [here is the result][fdb-vice-project-doom]
(in portuguese, but you can see the GIFs and videos).

Beforehand it sounded like an easy peasy tasks. It is 2018, GIFs are everywhere
(as if it was the early 2000s), no big deal.

Well, It is easy, if you know where to look for, and you know some tips and tricks.

I did not record all my failed attempts, but I tested many software before reaching 
what I consider a working confortable solution.

See below how I did.

## How to

Here is what you will **need**:

* The NES emulator [FCEUX](http://www.fceux.com)
* The [Lagarith](https://lags.leetcode.net/codec.html) lossless video codec
* The [FFmpeg](https://www.ffmpeg.org/) software (In our tutorial, we'll run it on [Docker](https://www.docker.com/))
* Optionally, since we are going to use FFmpeg on the command line, I recommend to use [ConsoleZ](https://github.com/cbucher/console)

### Do your NES captures

Fire up the FCEUX emulator, load your ROM, play until the part you want to record.
Now simply go to the menu an select `File > AVI/Wav > Record AVI...`, select the path
where to save your file and when prompted for a codec select `Lagarith Lossless Codec` in the drop down.

Now play until you are finished (you can you the pause key to pause the recording), and select in the menu
`File > AVI/Wav > Stop AVI`.
That's it, your capture is done.

### Generate a GIF from a capture

Now, based on this nice [blog post][high-quality-vid-to-gif], I adapted the script `convert_to_gif.bat` 
to run FFmpeg from Docker container (because I don't like installing stuff):

```bat
SET palette=tmp/palette.png
SET filters=fps=50

ECHO "Generating palette"
docker run --rm -v %CD%:/data jrottenberg/ffmpeg -v warning -i /data/%1 -vf "%filters%,palettegen" -y /data/%palette%

ECHO "Generating gif"
docker run --rm -v %CD%:/data jrottenberg/ffmpeg -v warning -i /data/%1 -i /data/%palette% -lavfi "%filters% [x]; [x][1:v] paletteuse=diff_mode=rectangle" -y /data/%2
```

You can use it like: `convert_to_gif.bat nes-capture.avi nes-capture.gif`

One important remark, I set the output framerate to 50 FPS, even though I captured at 60fps.  
The reason is that browsers are not very good at rendering GIFs with very small delay between
frames. After some tests I reached the conclusion 50 FPS was a good number as it was well 
rendered in Edge and Chrome.

See the result below:

| 50 FPS - size: 47KB                                                   | 60 FPS - size: 53KB                                                   |
|-----------------------------------------------------------------------|-----------------------------------------------------------------------|
| ![GIF at 50 FPS](/assets/Vice - Project Doom (USA)-Moves-50fps.gif) | ![GIF at 50 FPS](/assets/Vice - Project Doom (USA)-Moves-60fps.gif)     |

### Generate a MP4 video from a capture

Again using Docker, I wrote this one line script `convert_to_mp4.bat`:  
```bat
docker run --rm -v %CD%:/data jrottenberg/ffmpeg -i /data/%1 -map 0:0 -vf format=yuv420p -profile:v baseline -level 3.0 -preset veryslow -r 30 /data/%2`
```

Explaining the parameters a bit:
* `-map 0:0` - Filters only track 0, which means only the video, no audio. Remove if you want audio
* `-vf format=yuv420p` - I don't even know how to explain, but it does not work without it
* `-profile:v baseline -level 3.0` - See https://trac.ffmpeg.org/wiki/Encode/H.264#Compatibility
* `-preset veryslow` - Best compression, takes a bit more time, see https://trac.ffmpeg.org/wiki/Encode/H.264#Preset
* `-r 30` - Output at 30 FPS. It is sad, but certain devices still do not support HTML5 video at 60 FPS :(

You can use it like: `convert_to_mp4.bat nes-capture.avi nes-capture.mp4`

See the result below:

| `-profile:v baseline -level 3.0 -r 30` <br> size: 187KB               | `-profile:v high -level 4.2` <br> unchanged framerate (~60FPS) <br> size: 133KB  |
|-----------------------------------------------------------------------|-----------------------------------------------------------------------|
| <video src="/assets/Vice - Project Doom (USA)-baseline-3.0-30fps.mp4" controls autoplay loop></video> | <video src="/assets/Vice - Project Doom (USA)-high-4.2-60fps.mp4" controls autoplay loop></video> |

## Final Considerations

At the end I used both GIFs and videos in my blog post.  
Both have their pros and cons.

GIFs can achieve a higher framerate (up to 50 FPS) in most devices and can produce smaller
files depending on the type of action you have in the video.

Videos will produce smaller files if you have a lot of action going on, but you will have to lower
the framerate if you want more compatibility with older devices.

My tip is to always test both and see which one fits best to your needs.

[fliperama-de-boteco]: http://fliperamadeboteco.com
[fdb-vice-project-doom]: http://fliperamadeboteco.com/vice-project-doom-descubra-um-classico-obscuro-do-nes/
[high-quality-vid-to-gif]: http://blog.pkh.me/p/21-high-quality-gif-with-ffmpeg.html