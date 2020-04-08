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

As is customary my first thought was to test for a buffer overflow vulnerability by filling the input with a large amount of letters, but alas this did not cause any problems as I had hoped. (Little did I know I made a small mistake here that I wouldn't notice until after the competition had concluded)

![HackTrinity_Locked_Out2.JPG]({{site.baseurl}}/img/HackTrinity_Locked_Out2.JPG)

I knew the flag was likely in the binary and so my next idea was to simply try the "strings" command and grep for "HackTrinity" but once again this produced nothing. It seemed the flag was encrypted somehow within the binary. Finally I decided it was time to open up the binary in a disassembler for which I chose gdb as it was already installed on my ubuntu vm. After disassembling main I immediately noticed some functions which had proper names retrieved from the binary, most notable "enc_password", "decrypt", "print_flag" and "strcmp". Taking a look at the instruction calls it became clear that the binary encrypts the password taken in and then compares it against a decrypted password using the string compare function in C. 

![HackTrinity_Locked_Out3.JPG]({{site.baseurl}}/img/HackTrinity_Locked_Out3.JPG)
  
My first thought was to somehow just modify the jump instruction at "0x0000000040001475" to always jump no matter the result of the string compare but after a bit of research this turned out to not be as easy as I thought it would be. I discovered that a jump instruction in x86 is encoded based on the offset it wishes to jump to and this made creating a new jump instruction a bit more difficult then I intended, however I would have likely tried had my next idea not worked out.

Instead I chose to look into how the strcmp function worked in C by visiting its man page. As can be seen below it returns zero when the strings match. 

![HackTrinity_Locked_Out4.JPG]({{site.baseurl}}/img/HackTrinity_Locked_Out4.JPG)

After a little bit more research I discovered that the "test eax,eax" instruction after the strcmp function in the disassembled main is in fact where the program checks the result of strcmp. Thus I simply had to set a breakpoint at that instruction, modify eax to 0 and then continue. (Note that rax is eax on 64bit systems)

![HackTrinity_Locked_Out5.JPG]({{site.baseurl}}/img/HackTrinity_Locked_Out5.JPG)

Bam! We have the flag:

{: .box-note}
**Flag:** HackTrinity{h0w_cou1d_y0u_th3y_trusted_y0u}





