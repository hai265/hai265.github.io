---
layout: post
title:  "How to Clip and Subtitle Videos"
date:   2026-07-03 09:42:21 -0400
categories: jekyll update
---

| ![space-1.jpg]({{site.baseurl}}/images/subtitle-example.png) | 
|:--:| 
| *One of my subbed videos* |

# Background
I found it really weird that there's not that many guides on how to create subtitles using video editing software (the ones that do exist are just for ai generating subtitles).

I pretty much had to research this myself, and after a while I've landed on this workflow.


# Guide
## 1. Find a video you like
Usually I just watch a video on youtube normally, and if there's funny or otherwise important moments, I create timestamps notes so I can refer to them later when editing. 

I used to create timestamps manually on notepad or some other note taking app, but I created an app on Android to make this process a lot easier. You can check it out here: [Timestamper]({% post_url 2026-07-07-timestamper-test %})




## 2. Download the video
I used `yt-dlp` to download the video. Here's a sample command below I use.
`-f 140+399` downloads the video with mp3 audio and av1 codec for maximum compatibility. 
```
yt-dlp -f 140+399 (youtube_video_url)
```

## 3. Cut down the video
After downloading the video, open the video in whatever video editing program you like (I use [Davinci Resolve](https://www.blackmagicdesign.com/products/davinciresolve))
I use my timestamped notes I took in step #1 to create a rough cut on the video and identify parts that I want to actually subtitle.

## 4. Create the subtitles

| ![space-1.jpg]({{site.baseurl}}/images/aegisub.png) | 
|:--:| 
| *aegisub in action* |

To create the actual subtitles, I use a program called [Aegisub](https://aegisub.org/). The program includes handy tools like hotkeys, an audio visualizer, and dictionary to make creating subtitles easier. [There's a handy youtube playlist tutorial here](https://youtu.be/4gXF6Y-v6BE?si=sQ7qJTmb0Tssz4eF).

Aegisub is nice because you can configure shortcuts to make adding subtitles fast. It also includes an audio spectrum display which makes distinguishing speakers much easier than normal audio waveforms.

Additionally, you can generate subtitles from your cut down video in the previous step and import them to Aegisub to make timing subtitles easier. I use a [desktop GUI for Whisper](https://github.com/mehtabmahir/easy-whisper-ui) since it can run on my mac.

## 5. Include the subtitles in the video
Once you have your subtitles timed and ready, there are two ways to include those subtitles in the video.

  * **Burn the subtitles directly into the cut video created from step #3.**
  
  Aegisub is nice because it will automatically move lines that appear at the same time up so they won't overlap each other. However, subtitle effects are limited (if you want to shake your text, wobble it, etc, then do #2)

  * **Export the subtitles into Davinci resolve and style them there.**

  Davinci resolve takes longer because you'll have to manually move overlapping lines, but you'll gain the ability to apply various effects to your subtitles.

## 5a. Burn in the subtitles
Once you have your subtitles timed and ready, you can burn the subtitles in the video using `ffmpeg`. Here's a sample command:

```
ffmpeg -i prores.mov -c:v libx265 -crf 22 -vf subtitles="subtitles.ass" finalh265.mkv
```

The output video file will now contain the subtitles you created in step #4.
## 5b. Export the subtitles to Davinci Resolve
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

## 6. Final Edits

| ![space-1.jpg]({{site.baseurl}}/images/davinci.png) | 
|:--:| 
| *the same scene above but subs styled in Davinci Resolve* |

In this step I make my final edits, like adding images and applying transitions and video effects.

If you decided to create `.srt` subtitles instead, then you would need to convert the subtitle files into `Text+` nodes so you can apply styling. I use the [Snap Captions](https://orsonlord.com/snap-caption-help-guides/snap-captions-help-database/snap-captions-install-guide) script to achieve this.

It's a little annoying if you have multiple styles, since you have to add and apply the script to each style individually. I'd like to create my own script for this eventually though that just does this in one step.

Once you're done, you now have a subtitled video!

## Resources
* Timestamper App: [https://github.com/hai265/Android-Youtube-Timestamps](https://github.com/hai265/Android-Youtube-Timestamps)
* Aegisub: [https://aegisub.org/](https://aegisub.org/)
* Snap Captions (Use 1, not 2): [https://orsonlord.com/](https://orsonlord.com/])
* Subtitle Style guide: [https://lyger.github.io/scripts/guides/subtitling.html](https://lyger.github.io/scripts/guides/subtitling.html)
