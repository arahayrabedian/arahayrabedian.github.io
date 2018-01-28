---
layout: post
title: DIY Quotes Lock Screen (for i3wm)
---

I've always had a txt quotes file of my favourite quotes, and recently I realised that while it grows and grows, I don't really see/remember enough of it. I already had a fancy `i3` lock screen so I decided to up the ante and overlay quotes each time I lock my screen.

This is a run-through of how you can do the same, a lot of it is bespoke so I won't be publishing code straight up, but you should be able to figure out enough to implement it yourself.

# Prior Situation

Earlier I already had i3lock set up somewhat fancily. Every time I lock it would take a screenshot, place it in `/tmp/screenshot.png`, blur it in-place, then lock with i3lock pointing to that screenshot. Now we want to add quotes.

# Enter `quotes.txt`

My quotes file was originally a bit of a mess, special characters from copy/pasting from all over the place along with years of neglect. I wanted to keep it simple and not move to something like `sqlite` because overkill, and whitespace is not a good delimiter for what is more or less freetext. So I decided, arbitrarily, that I would separate quotes in the file with my own delimiter that I never expect to appear in any quotes, this happened to be `-----`.

## Parsing

Using python, we can easily parse for a random quote, I added a constraint where a quote would only be selected if it had less than a configurable amount of characters (a wall of text does not make a pretty lock screen). Very briefly, the core of the parser looks something like this:

```python
def parse_quote_file(filename, quotelimit=None):
    with open(filename, 'r') as quotefile:
        rawtext = quotefile.read()
    dirty_split = rawtext.split("-----")
    if quotelimit:
        clean_split = [quote.strip() for quote in dirty_split if len(quote.strip()) <= quotelimit]
    else:
        clean_split = [quote.strip() for quote in dirty_split]
    return clean_split

```

That way we have all the quotes that meet our criteria nicely in hand, outside this method we can pull a random quote and return that to the script that generates our overlay (coming up).

# Overlay Generation

So we want to take a screenshot and process that image before displaying the lock screen. For the `convert` command you'll need imagemagick, and the capability of using it (which I fumbled somewhat, just be aware that ordering matters a **lot** to the `convert` command)

The snippet below does several things:
1. Take in input to the filename to use for the screenshot
1. pull the text for a random quote from the random quote selector (in my case, 2 arguments: path to quotes file, and maximum number of characters)
1. take a screenshot
1. blur the image
1. add in a text box with a partially see-through background at the bottom of the image, with a white font with black stroke (I'll be honest, some of these other options were just magic), with the last step being to draw our quote text.

```bash
#!/bin/bash

image=$1
quote="`python /path/to/random_quote_parser.py /path/to/quotes.txt 450`"

scrot "$image" && \
convert "$image" -blur 15x8 \
    -background rgba\(100,100,100,0.5\) -fill white \
    -stroke black -strokewidth 1.75 -gravity west \
    -size 1300x -pointsize 40 -font "Open-Sans-ExtraBold" \
    caption:"$quote" -gravity southwest -geometry +33+33 -compose over -composite "$image"
```

*note: the blur command may take a long time if your resolution is high or you have a lot of screen real estate. You may also want to play around with the blur effect if it's not enough for you to actually provide privacy (if you have large fonts).*

And with that, I can finally give you a sample of what locking my current screen would look like!

![lock_screenshot](https://arahayrabedian.github.io/images/2018-01-28/lock_screenshot.png)

# Tips and Tricks

If you're an i3 user, you may want to add in multiple lock shortcuts, I personally have 3 - one for a standard lock screen with blur and quotes, one for a black lock screen (for privacy) and one that actually turns the screens off (both laptop and external) if I know I'll be away from my computer for a while.

I'm also not sure how this quotes lock screen works with multiple displays, I only added it after I stopped using an external display, I do know the screenshot/blurring mechanism works just fine.

# Fin

I don't really have much more to say, I guess good luck putting it all together, it was actually fun and fulfilling building something I wanted for myself!