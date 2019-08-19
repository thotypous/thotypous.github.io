---
layout: post
title: Running a portable amateur radio station by the Xingu River
categories: hamradio
tags: ham 40m
image: /postdata/2019-08-11/05.jpg
---

Camping at river islets in the Amazon basin is a lot of fun, but what if the agreed time comes and the boatman does not arrive to take you back? This concern has made the perfect excuse for me to finally take the exams to get an amateur radio operator license and to build a portable 40m transceiver. This article describes the adventures of camping in a Xingu River islet with my wife's branch of the family – taking the radio together, of course. Although I still could not make a QSO, it was a joy to listen to other people doing CW and to get our WSPR beacon heard by 45 stations spread through several continents.

<!--more-->

# Materials

 * [QRP Labs QCX kit](https://www.qrp-labs.com/qcx.html)

 * [QRP Labs QLG1 GPS Receiver kit](https://www.qrp-labs.com/qlg1.html)

 * [QRPGuys Multi Z Tuner](https://qrpguys.com/multi-tuner)

 * [American Morse Equipment Dirt Cheap Paddle](https://www.americanmorse.com/dcp.htm)

 * 12V 6Ah VRLA battery (a motorcycle battery)

 * Prysmian Superastic Flex 1mm² electric cable

 * Cotton twine

 * Cutting pliers


# Setting the station up

Everything starts by mounting the antenna. Based on my first [successful](https://twitter.com/thotypous/status/1155287585121624064) [WSPR transmissions](https://twitter.com/thotypous/status/1155519484586799106) when I was at São Paulo, I decided to go with a vertical antenna. I did not have much luck with dipoles – maybe they were not high enough.

First, we tied cotton twine to a twig and launched it, hanging it to a high branch of a tree near the river. This was actually done by my brother-in-law, who was able to hit the perfect spot on the first try.

<a href="https://photos.app.goo.gl/H2ZULvSs7CYVBv2Y8"><img src="/postdata/2019-08-11/01.jpg" /></a>

Then, we constructed sort of a small table using some dry twigs, the purpose of which was to avoid our equipment from touching the soil.

After that, we used the twine to bring some electric cable up. We did not really measure its length. After bringing one end of the wire near the top of the tree, we just cut the other end at the height of the table. This way, we got something between 8 and 9 meters of vertical wire.

To make the antenna radials, we tied 5 pieces of wire (around 3 meters each) to twine, then stretched each of them, fastening the twine to nearby trees. We connected the vertical wire to the *Wire* screw of the tuner and the radials to the *Gnd* screw, and adjusted the tuner switches to the *Balanced* and *High Z* positions.

Finally, we were ready to put the transceiver and the GPS receiver in place.

<a href="https://photos.app.goo.gl/JHeJZkHeWrJU7e8x6"><img src="/postdata/2019-08-11/02.jpg" /></a>

<a href="https://photos.app.goo.gl/Q3HSRcEH3U4HT3fNA"><img src="/postdata/2019-08-11/03.jpg" /></a>


# Transmitting a WSPR beacon

According to predictions I had previously done using [VOACAP](http://www.voacap.com/hf/), we were in the perfect time of day for long range communications in the 40m band. Thus, I decided to enable the transceiver's WSPR beacon mode for a couple of hours.

<a href="https://photos.app.goo.gl/cCCH7cq61gwXGLNz8"><img src="/postdata/2019-08-11/04.jpg" /></a>

WSPR uses a reliable digital modulation scheme called [MEPT-JT](https://www.qsl.net/zl1bpu/MFSK/MEPT-JT.htm) to transmit a message containing our call sign (PU2UID), location ([GI36](https://k7fry.com/grid/?qth=GI36)) and transmit power (37dBm max). When activated, the beacon mode of our transceiver transmits this message each 10 minutes.

When we came back to the city, we checked out [WSPRnet](http://wsprnet.org) to see how far our signal had gone.

<img src="/postdata/2019-08-11/wsprnet_map.png" />

There were 45 unique spots hearing us!

<img src="/postdata/2019-08-11/wsprnet_pu2uid.png" />

For comparison, let's see how many spots hear PY2GN, the only other station in Brazil which is currently transmitting a WSPR beacon in the 40m band.

<img src="/postdata/2019-08-11/wsprnet_py2gn.png" />


# Listening to CW

Back at São Paulo, all I could hear from the radio was noise, to the point I worried I had made some mistake when building the kit.

At the islet, however, I was really amazed by the clean CW signals I was listening to. I could set the gain to the maximum and hear almost no noise at all.

At night time, I could not find any unused frequency to try and call CQ. At every frequency, you could hear someone else doing a QSO.

<iframe width="853" height="480" src="https://www.youtube.com/embed/EaiY9jTIl2c" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

(Sorry for the absence of transceiver audio. I did not realize I placed the earphone next to the incorrect mobile phone's microphone until the next day.)

At day time, I was able to hear a QSO between two Brazilian hams.

<iframe width="853" height="480" src="https://www.youtube.com/embed/qrtHoAD5RQg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

After they finished their QSO, I tried calling CQ several times at different frequencies, without luck.

During the rest of the day, I just heard many local stations doing SSB phone in the 7000 to 7047 kHz range, which is forbidden by the Brazilian regulating agency's (ANATEL) [frequency plan](https://www.anatel.gov.br/legislacao/atos-de-requisitos-tecnicos-de-gestao-do-espectro/2018/1236-ato-9106).


# Conclusions

The portable station exceeded my expectations. I will definitely bring it together again next time.

It is unfortunate that few local hams are doing CW. If we needed to ask for help, probably our odds would be better at night.
