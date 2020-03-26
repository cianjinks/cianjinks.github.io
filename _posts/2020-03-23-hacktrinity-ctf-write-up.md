---
layout: post
title: HackTrinity CTF Write Up
subtitle: My first CTF
tags:
  - ctf
comments: true
published: true
---

This last week I took part in my first ever CTF and therefore felt it appropriate to follow that with a write up of my own. The CTF was a collaboration between various societies at my university and was known as HackTrinity. This is my first ever write up and details solutions to the various challenges I was able to solve throughout the competition.

## Challenges

### Welcome

This was the first challenge in the competition and one of a few very simple ones. It acted as a simple tutorial to teach you the flag format and the solution was given.

{: .box-note}
**Flag:** HackTrinity{well_that_was_easy}

### CHAD

This challenge was another of the so-called simple challenges and simply acted as a tutorial for setting up the instances service used for some of the challenges. Once setup the flag is found by visiting the given URL.

{: .box-note}
**Flag:** HackTrinity{rutabaga_span_id_tortellini}

### Trivia - Unknown

This was the first of the trivia challenges. Once again it was another easy one as you were given a quote and upon googling that quote you would quickly come to the conclusion that it was said by Zero Cool from the movie Hackers. This flag was non standard format.

{: .box-note}
**Flag:** ZeroCool

### Trivia - NotBotsAgain

To some this challenge is commong knowledge but a quick google search will also tell you the answer. This flag was non standard format.

{: .box-note}
**Flag:** crossdomain.xml

### Reversing - Locked Out

This was the first of the so-called _real_ challenges. You were presented with a simple binary that when ran produced the following output:

![HackTrinity_Locked_Out1.JPG]({{site.baseurl}}/img/HackTrinity_Locked_Out1.JPG)









