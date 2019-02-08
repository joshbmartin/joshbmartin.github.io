---
layout: post
title: RIP G3 Pro
description: "Brand new UVC G3 Pro, DOA with no chance at survival."
author: josh
category: networking
tags: networking video camera ip unifi
finished: true
---

I just received my brand new UniFi UVC G3 Pro, plugged it in, opened up the NVR and found no cameras were available for adoption. Bummer.

# UniFi UVC G3 Pro can't connect, can't adopt

Current network setup:

- UniFi Switch 16 (150W)
- UniFi Security Gateway
- UniFi Cloud Key
- UniFi AC Pro
- UniFi AC Lite
- Windows 10 NVR

You could say I'm no stranger to UniFi gear. The only reason I didn't go with the UniFi branded NVR or even an Ubuntu box of my own was for ease of use and initial setup for testing. My current Windows 10 machine is more than capable of handling one camera to perform my testing. Unfortunately it never made it far enough to test.

## Lets begin our testing

It's roughly 5pm. I've just gotten home from work. I snap a few photos of the G3 Pro still wrapped tight in its beautiful packaging:

![g3]({{ site.url }}/assets/unifi3.jpg)

![g3 side]({{ site.url }}/assets/unifi2.jpg)

![g3 back]({{ site.url }}/assets/unifi1.jpg)

I take my new 50 foot outdoor Cat5e cable hook it into my cable tester. All green lights! So far so good.
I hook the cable into a known working port in the house and connect the other end to my G3 Pro. IR lights come on since the lighting isn't the best in the basement. So far so good! I launch my NVR from my W10 box, but I don't see any cameras to adopt. Weird... I disconnect/reconnect the cam and refresh my browser still no cams showing up to be adopted.

Thinking, I wonder if the switch port is configured to provide POE? Hop into the UniFi controller check the switch port config, sure enough its set correctly. I check the phsyical switch, it's showing POE+ going to the camera.

I swap the cables just in case, no dice. Eventually I started digging around UniFi forums for similar problems. I find a few different articles talking about ports that you need to allow on your firewall to allow UniFi video to work correctly. Specifically 7442 & 7443. I unblock these. Still no luck. I disable my firewall on my W10 box hosting the NVR software, still no luck.

In the UniFi controller I can see the camera would sometimes get an IP and sometimes it wouldn't, rather than letting the camera reach out to DHCP I set a static IP, still no luck. Browsing to the IP of the camera directly in my browser instead of trying to adopt through the NVR also didn't work.

Resetting the camera by pressing the phsyical button on the back of the device for 10 seconds, this has got to be the winner right!? Nope. Still no dice.

Getting to the last of my efforts I decide to use a known working POE port in my house thats currently powering my UniFi AP AC Pro. Hook up the camera using the same cables powering that AP, still no dice. On all the above testing methods. Eventually having exhausted all options I knew about I reached out to UniFi chat support hoping they would have the key.

## UniFi Chat Support

Sean was super helpful in troubleshooting the camera while I was off site. Eventually it came time where I needed to be on site to change out the cables and perform some other hands on testing. A few hours later I made it on site, I reached back out to chat support thinking I'd have to go through the whole process with someone else. Turns out this new agent saw I was working with Sean and got me directly connected to his chat. This was a seamless and great experience. Not having to repeat the entire discussion again. Emailing is not my cup of tea and I'd rather not talk to people on the phone so 24/7 chat support was a perfect.

We go through basically all of the above methods and then we do one thing I hadn't tried. We use a POE Injector in between my Desktop and the camera. We set a static IP on the desktop hooked the lan of the desktop into the lan of the injector, and the POE going into the Camera. Unfortunately we still weren't able to get connected to the cam.

## RMA

Sean advised I fill out an RMA form and they would ship me out a new cam, I assumed I would have to pay shipping to send my broken camera back. But since my cam was DOA they have already approved my claim the same day and said I should expect my new camera within a week! All I need to do is use the label with the new camera & ship the broken camera back in the same box! Sounds easy enough! 

All in all, even though the camera was DOA UniFi customer support has been fantastic. Couldn't recommend them enough. I'll post an update once I get the new cam and we'll see where we go from there!
