---
layout: post
title:  "How I Clip and Subtitle Videos"
date:   2026-07-15 09:42:21 -0400
categories: jekyll update
---

# Timestamper App (Self-Promotion)
I created this timestamping app to help me with my videos, and I've decided to release it to the public!
In order to publish to the Play Store, I'll need testers to test the app over 14 days. 

[If you're interested, then check out this blog post for information on how to join.]({% post_url 2026-07-07-timestamper-test %})

| ![space-1.jpg]({{site.baseurl}}/images/edit-screenshot.png) | 
|:--:| 
| *disclaimer: This is my app.* |

# Background
![space-1.jpg]({{site.baseurl}}/images/how-to-sub.png) 

# Table of Contents
* TOC
{:toc}

# My Workflow
## 1. Find a video I like
Usually I just watch a video on youtube normally, and if there's funny or otherwise important moments, I create timestamps notes so I can refer to them later when editing. 


## 2. Download the video
I use `yt-dlp` to download the video. Here's a sample command below I use.
`-f 140+399` downloads the video with mp3 audio and av1 codec for maximum compatibility. 
```
yt-dlp -f 140+399 (youtube_video_url)
```

## 3. Cut down the video
After downloading the video, I then import the video into [Davinci Resolve](https://www.blackmagicdesign.com/products/davinciresolve).

I use my timestamped notes I took in step #1 to create a rough cut on the video and identify parts that I want to actually subtitle.

| ![space-1.jpg]({{site.baseurl}}/images/davinci-resolve-notes.png) | 
|:--:| 
| *This is the number of notes / bookmarks I usually take* |

## 4. Create the subtitles

| ![space-1.jpg]({{site.baseurl}}/images/aegisub.png) | 
|:--:| 
| *aegisub in action* |

To create the actual subtitles, I use a program called [Aegisub](https://aegisub.org/). The program includes handy tools like hotkeys, an audio visualizer, and dictionary to make creating subtitles easier. [There's a handy youtube playlist tutorial here](https://youtu.be/4gXF6Y-v6BE?si=sQ7qJTmb0Tssz4eF).

Aegisub is nice because you can configure shortcuts to make adding subtitles fast. It also includes an audio spectrum display which makes distinguishing speakers much easier than normal audio waveforms.

Additionally, I generate Japanese videos from the cut down video in the previous step and import them to Aegisub to make timing subtitles easier. I use a [desktop GUI for Whisper](https://github.com/mehtabmahir/easy-whisper-ui) since it can run on my mac.

## 5. Include the subtitles in the video
Once you have your subtitles timed and ready, there are two different methods I use to put subtitles into the video.

  * **Burn the subtitles directly into the cut video created from step #3.**
  
  Aegisub is nice because it will automatically move lines that appear at the same time up so they won't overlap each other. However, subtitle effects are limited (only supports shaking text, limited text styling). I use the command below:

```
ffmpeg -i prores.mov -c:v libx265 -crf 22 -vf subtitles="subtitles.ass" finalh265.mkv
```
    

  I stopped doing this method in favor of creating subtitles in Davinci Resolve, but I still sometimes do this if I'm feeling lazy.

  * **Export the subtitles into Davinci resolve and style them there.**

  Davinci resolve takes longer because you'll have to manually move overlapping lines, but you'll gain the ability to apply various effects to your subtitles.

## 6. Davinci Resolve
### Converting `srt` to Davinci Resolve Text+
Davinci resolve only supports `.srt` subtitle, while Aegisub files are in the `.ass` format. I've created a handy python script to convert `.ass` subtitles to individual `.srt` files according to their style.
```
#!/usr/bin/env python3

import re
import argparse
from pathlib import Path
from collections import defaultdict


def ass_time_to_srt(t):
    h, m, s = t.split(":")
    s, cs = s.split(".")
    ms = int(cs) * 10
    return f"{int(h):02}:{int(m):02}:{int(s):02},{ms:03}"


def clean_text(text):
    # remove ASS formatting tags
    text = re.sub(r"\{.*?\}", "", text)

    # convert ASS line breaks
    text = text.replace("\\N", "\n")

    return text.strip()


def convert_ass_by_style(ass_path):
    ass_path = Path(ass_path)

    styles = defaultdict(list)

    with open(ass_path, encoding="utf-8") as f:
        for line in f:
            if line.startswith("Dialogue:"):
                parts = line.split(",", 9)

                start = ass_time_to_srt(parts[1])
                end = ass_time_to_srt(parts[2])
                style = parts[3]
                text = clean_text(parts[9])

                styles[style].append((start, end, text))

    # Create output directory
    out_dir = ass_path.parent / "srt"
    out_dir.mkdir(exist_ok=True)

    for style, lines in styles.items():
        out_file = out_dir / f"{style}.srt"

        with open(out_file, "w", encoding="utf-8") as f:
            for i, (start, end, text) in enumerate(lines, 1):
                f.write(f"{i}\n")
                f.write(f"{start} --> {end}\n")
                f.write(f"{text}\n\n")

        print(f"Created: {out_file}")


def main():
    parser = argparse.ArgumentParser(
        description="Convert ASS subtitles into multiple SRT files grouped by style"
    )
    parser.add_argument("file", help="Path to .ass subtitle file")

    args = parser.parse_args()

    convert_ass_by_style(args.file)


if __name__ == "__main__":
    main()
```

For example, if you had two styles, say `Gold Ship` and `Special Week`, they'll be split into `Gold_Ship.srt` and `Special Week.srt`, and then you can individually import them into Davinci Resolve.

### Text Effects
The main advantages of creating subtitles in Davinci Resolve is that you can apply effects to `Text+` as if it was a video or image, which lets you do all kinds of effects.

<video width="100%" height="auto" controls>
  <source src="{{site.baseurl}}/images/resolve-wavy.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>


I use the [Snap Captions](https://orsonlord.com/snap-caption-help-guides/snap-captions-help-database/snap-captions-install-guide) to convert the `.srt` files into Davinci Resolve's `Text+` nodes.

It's a little annoying if you have multiple styles, since you have to add and apply the script to each style individually. I'd like to create my own script for this eventually though that just does this in one step.

| ![space-1.jpg]({{site.baseurl}}/images/davinci.png) | 
|:--:| 
| *the same scene above but subs styled in Davinci Resolve* |

### Text with Icon
The main reason I use Davinci Resolve is that I'm able to create subtitles with an image attached to it to make the speaker easier to identify.

| ![space-1.jpg]({{site.baseurl}}/images/icon-fusion.png) | 
|:--:| 
| *Text+ w/ Icon Fusion Setup* |

The main logic driving this is in the `Merge` node.

```
Point(Text1.Output.DataWindow[1]/Text1.Output.OriginalWidth,
    ((Text1.Output.DataWindow[2] + Text1.Output.DataWindow[4]) / 2) / Text1.Output.OriginalHeight)
```

I'm not exactly sure what this does, since I copied this from some forum somewhere, but this moves the icon according to the length of `Text` to keep the image to the left.

| ![space-1.jpg]({{site.baseurl}}/images/short-text-icon.png) | 
| ![space-1.jpg]({{site.baseurl}}/images/long-text-icon.png) | 
|:--:| 
| *Icon moves left with the text* |


## 7. Areas Of Improvement
The most annoying part about Davinci Resolve that it's not possible to change the style of multiple Text+ nodes, so I have to decide what each text style looks like beforehand, and if I want to change something, then I have to do it for each individual `Text+` node.

I've explored Adobe Premiere and it seems possible to apply a text style across multiple instances, but I haven't found a way to do to achieve the icon w/ text yet. I'm 90% sure it's possible since I'm pretty sure the official Pakatube channel edits with Adobe Premiere and they have Text with icons. If anyone knows how to do this in Adobe then let me know please!

# Misc Tools
For thumbnails, I use [Umaviewer](https://github.com/katboi01/UmaViewer) for custom Uma poses and [Affinity Photo](https://www.affinity.studio/photo-editing-software) for photo editing.

# Resources
* Timestamper App: [https://github.com/hai265/Android-Youtube-Timestamps](https://github.com/hai265/Android-Youtube-Timestamps)
* Aegisub: [https://aegisub.org/](https://aegisub.org/)
* Snap Captions (Use 1, not 2): [https://orsonlord.com/](https://orsonlord.com/])
* Subtitle Style guide: [https://lyger.github.io/scripts/guides/subtitling.html](https://lyger.github.io/scripts/guides/subtitling.html)
