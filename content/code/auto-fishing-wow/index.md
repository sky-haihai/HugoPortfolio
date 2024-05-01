---
title: "Auto Fishing in World of Warcraft Based on RMS of the Audio Loudness"
slug: "auto-fishing-wow"
date: 2024-05-01T10:45:53-06:00
tags: []
categories: []

draft: false
---

## Morally Speaking

Fishing in most video games are basically free mini Gacha games, players can get literal crap or something extremely rewarding, all in the same pool. Unlike mobile Gacha game that cost players money, fishing typically trade player's wait time with some expectation for a bigger reward.

The only bad part of this no risk high reward process called fishing in game design aspect is that it's designed to be boring and repetitive to keep players from doing it forever.

Unfortunately, I have back problems and fishing addiction. I want utilize every second to make my life looks less empty. It's the same reason why people look at their phones on the dinner table or on the toilet.

So, I have two options:

1. Sacrifice my back health and sleep time.
2. Automate it.

I choose the second option.

## Code

Most game has a sound queue when the fish is hooked, so I can just write a simple program to listen to the sound and press the button when a certain loudness threshold is reached.

### Detect the volume of the sound.

```python
THRESHOLD = 1000  # rms threshold (need twicking)
...
def detect_sound():
    data = stream.read(1024)
    # get rms
    decibel = audioop.rms(data, 2)
    print_decibel_bar(decibel)
    return decibel > THRESHOLD
```

### If the volume is higher than a certain threshold, press the button.

```python
def random_press(key):
    press_time = random.uniform(0.1, 0.2)
    pyautogui.keyDown(key)
    time.sleep(press_time)
    pyautogui.keyUp(key)
```

### Repeat
```python
while time.time() - overall_start_time < random_fish_time*3600:
    fish()
```

## Result

Although I'm not caught by the anti-cheat department, I felt greedy when I cook meals in order to get buffs, when I sell fishs for golds, and when I exchange tokens to the frog dragon.

What I really wanted to enjoy is the moment when I wake up and there is that one rare item sitting in the middle of all the craps that I got over the last night.

So I quit WOW in 2 weeks after making this script, due to the increased amount of work for one of my game projects: [Echoes of the Roots](/project/).

LOL