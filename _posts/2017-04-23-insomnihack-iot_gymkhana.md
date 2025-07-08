---
title: "Insomni'hack 2017 - IoT Gymkhana"
date: 2017-03-25
categories: [CTF, Hardware]
tags: [Insomnihack]
media_subpath: /assets/media/2017-04-23-insomnihack-iot_gymkhana
---

## IoT Gymkhana
The 10th edition of Insomni’hack took place between the 23rd (workshops) and 24th (conferences & contest) of March 2017 at Geneva Palexpo. For this special anniversary edition, the organizers added some new enjoyable content aside of the main contest such as an escape room and a FPS shooter game based on Unity 3D framework, great ideas!

This year the goal was to solve the hardware challenges during both the teaser (write-up here) and the finals, which ended up being a little more challenging than expected. Indeed, the final scoreboards speak for itself, the first part of the challenge was relatively easy compared to the second and third parts.

This year’s IoT Gymkhana hardware challenge included a teaser and final round, with increasing difficulty. You can tell just by watching the number of solves:

<img src="image1.png" alt="number of solves" /> 

## Part1 – Noobs will be publicly ashamed!
```
IoT Gymkhana - Part 1

Hey ! Hope you liked the quals, here is the first step in your 
journey through the wonderful world of IoT.

Rating: Easy

A link and a file (download it here) were also provided.
```

The website was quite simple with only a form requiring a password:

<img src="image2.png" alt="Web interface" /> 

The statement on the web page makes the goal of this challenge obvious: find the correct password in the provided file.

Let’s confirm what architecture is being chosen to host the challenge:

```bash
$ file iot_gymkhana_2d746169901cca78c7a726f4bd9c94036e0bc4516c977b68be372e0ce9f547bb.elf
 …
 ELF 32-bit LSB executable, Tensilica Xtensa, version 1 (SYSV), statically linked, stripped

$ binwalk iot_gymkhana_2d746169901cca78c7a726f4bd9c94036e0bc4516c977b68be372e0ce9f547bb.elf
 DECIMAL  HEXADECIMAL  DESCRIPTION
 ------------------------------------------------------------------
 0        0x0          ELF, 32-bit LSB executable, version 1 (SYSV)
 5204     0x1454       Unix path: /tmp/esp-idf/components/esp32/./heap_alloc_caps.c
```

Perfect it’s the same architecture as the teaser! Let’s fire up Radare2 and search for the web page in the strings:

```
$ radare2 iot_gymkhana_2d746169901cca78c7a726f4bd9c94036e0bc45
16c977b68be372e0ce9f547bb.elf

[0x3ffb2aac]> aaaa
 [Cannot find function 'entry0' at 0x3ffb2aac entry0 (aa)
 [x] Analyze all flags starting with sym. and entry0 (aa)
 [ ]
 [aav: using from to 0x3f3ff000 0x3f467384
 Using vmin 0x3f3ff000 and vmax 0x40112aa5
 aav: Cannot find section at 0x1073442345
 [x] Analyze len bytes of instructions for references (aar)
 [x] Analyze function calls (aac)
 [x] Emulate code to find computed references (aae)
 [x] Analyze consecutive function (aat)
 [x] Type matching analysis for all functions (afta))unc.* 
     functions (aan)
 [x] Type matching analysis for all functions (afta)
 [0x400eecf4]> fs strings
 [0x400eecf4]> f | grep THIS
 0x3f406f88 242 str._html__head__title_IoT_Gymkhana__title___head__body__h1_Welcome__h1__strong_THIS_IS_NOT_A_WEB_CHALLENGE._Noobs_dirbusting_will_be_publicly_ashamed__strong__form_method_GET__input_type_text_name_p_placeholder_password...___form___body___html_
```
The webpage is located at `0x3f406f88`. As seen during the teaser, one needs to find the address where those bytes are stored and search for xrefs to this virtual address.

```
[0x400eecf4]> /x 886f403f
Searching 4 bytes in [0x3f3ff000-0x40112aa5]
hits: 1
0x400d0ae8 hit0_0 886f403f
[0x400eecf4]> /c 400d0ae8
0x40108b58   # 3: l32r a11, 0x400d0ae8
```

To be able to resolve faster the xrefs, it is possible to activate the code emulation feature of Radare2 (`e asm.emu=true`). 

Here is an example of what should be seen when entering the graphical view:

<img src="image3.png" alt="Web interface" /> 

This feature can be toggled on/off by typing a single quote in the graphical view. It is highly recommended to do so to avoid any confusion.

Let’s jump to this location and use the graphical view to better understand what is going on.

```
[0x400eecf4]> s 0x40108b58
[0x40108b58]> VV 
```
<img src="image4.png" alt="Radare2 function at 0x40108b58" /> 

Looking around this location provides information on how the webserver is routing the visitor depending on the requested URI. Indeed, the top function `0x40108b0c` is performing a strncmp() (`0x400d0784->0x4000c5f4`) to check whether the requested URI match GET /.

In this case, the execution continues to `0x40108b1d` to perform some additional operations before responding with either a 200 OK (welcome page) or a 302 Found redirecting the visitor to another page located at `/b87c0f0790488c93284556744a0dc14576d87d95`.

The function `0x40108b1d` has an interesting parameter "%17s" which could be an indication of the length of the required password. The function `0x401123d8` called right before the last instruction gives probably more information about what’s happening to the a10 register.

Before going forward, one has to remember that when call8 instruction is used the caller registers a8..a15 are the same as the callee registers a0..a7 after the callee executes the ENTRY instruction. This means that the a10 register will be referred to as a2 in the following section.

Here is a high-level view of what is going on:

<img src="image5.png" alt="Radare2 function calls" />

As shown below, each of the cascaded blocks are loading a byte from memory with an incremented offset (from 0x0 to 0x10) and performs a byte comparison with a specific value. If the values are the same, next check is performed, in the opposite case a2 is set to 0 and execution resumes to the caller function (FAIL below). In the end, if all checks were good, a2 is set to 1 which means that the password is correct (WIN below).

<img src="image6.png" alt="Radare2 WIN condition" />

The comparison being in order, it is just a matter to write down all the values and resolve them to actual chars.

```python
print "".join([chr(x) for x in [113,117,52,108,115,95,119,51,82,101,95,117,53,101,70,117,49]])
qu4ls_w3Re_u5eFu1
```

Let’s try it:

<img src="image7.png" alt="Flag1" />


Flag: `INS{qu4ls_w3Re_u5eFu1}`


## Part2 – Software Defined Radio
``` 
Got the authentication ? being lucky ? The flag is all around 
us, listen carefully

Rating: Medium
```

When hitting "Send flag", the browser makes a request to `/c3cff53e08485843821fc62a462c01061426d1ea` which redirects back to this page and nothing more happens.

Let’s try to find the related code section to better understand what’s going on, starting from the latest webpage from part1.

```
[0x40108b58]> f | grep Congrats
0x3f406dd0 439 str._html__head__title_IoT_Gymkhana__title___head__body__h1_Menu__h1__p_Congrats___The_first_flag_is_INS_password___p__p_Now_try_to_get_the_second_flag_br__a_href___c3cff53e08485843821fc62a462c01061426d1ea__Send_flag__a___p__p_Here_you_can_send_your_own_message._A_third_flag_will_be_send_beforehand._br__form_action__37624f02f2c66ffe19ddb172a5a081b4002227ad_method_GET__input_type_text_name_m_placeholder_message...___form___p___body___html_
[0x40108b58]> /x d06d403f
Searching 4 bytes in [0x3f3ff000-0x40112aa5]
hits: 1
0x400d0acc hit1_0 d06d403f
[0x40108b58]> /c 400d0acc
0x40108a5c   # 3: l32r a11, 0x400d0acc
[0x40108b58]> s 0x40108a5c
[0x40108a5c]> VV
```

<img src="image8.png" alt="Radare2 function handling requests" />

Looking around the current location provides a better view on how the routing is working within the webserver.

Let’s have a look on the function `0x401089fc` which sends the second flag.

<img src="image9.png" alt="Radare2 redacted flag in function" />

A long and suspicious string is used right before the 302 found is sent. This string is probably the flag obfuscated, but how is it transmitted then? In the middle of the function `0x401087ac` there is some interesting things:

<img src="image10.png" alt="Radare2 interesting hints in the function" />

SPI interaction and nrf transmission…wait could, it be the SDR that @Baldanos promised in his tweet?

<img src="image11.png" alt="Baldanos message" />

Looking at the device itself confirm this is most likely the case 😊

<img src="image12.jpeg" alt="Lunch box" />


The box being transparent, it is possible to identify that the wireless module is a PTR8000. This module is basically an nrRF905, a single chip radio transceiver for the `433/868/915 MHz` ISM band. Because it embeds all high-speed signal processing related to RF protocol on-chip, it is perfect to be used with microcontrollers with a simple SPI interface.

Even if you didn’t own a SDR device, some were provided during the contest. Among them, a Terratec NOXON DAB Stick, that I used to solve the challenge.

<img src="image13.jpeg" alt="Antenna" />

By monitoring the frequencies at 433 MHz and pressing "Send flag" one can find the correct frequency that we found at 433.250MHz.

<img src="image14.png" alt="Central frequency" />

The nrf905 is using a special kind of FSK modulation named GFSK, where a Gaussian filter is applied to smoother the level transition. The full frame being sent is usually encoded in Manchester as it increases the number of transition which is good for transferring data. The nrf905 frame looks like this:

<img src="image15.png" alt="nrf905 frame" />

At first, we attempted to retrieve the flag with URH (Universal Radio Hacker) which propose all features required to perform such demodulation and decoding. However, after spending hours on it, we failed to decode the frame during the time of the CTF. The 3 teams who solved the challenge decoded it by hand in the end.

Being very frustrated by this first experience with SDR, I asked @Baldanos to lend me the device because I really wanted to solve the second part of his challenge as I already had some clues about the third part, he accepted.

Back home, I looked on the following blog post where the author mentioned a decoder for nrf905 frames that can be used together with rtl_fm. During the CTF, the decoder did not spit anything out on the console. We thought that this decoder was not working, but after having a closer look to it, we were all wrong.

Indeed, by reviewing the code I noticed that the payload size was set to 8 bytes:

```
// Line 53-63#
static void nrf905_init( nrf905_handle_t* nrf, size_t pwidth, 
size_t awidth, unsigned int flags)
/* pwidth the payload width, in bytes */
/* awidth the address width, in bytes */
/* size the buffer size */
…

// Line 216
nrf905_init(&nrf, 8, 4, NRF905_FLAG_NONE);
```

The obfuscated string was 29 bytes long, therefore the length of the payload must be adapted in the decoder. I decided to increase it to the maximum payload size (32 bytes).

```shell
$ rtl_fm -f 433250000 -s 1600k -g 0 | ./nrf905_decoder -
 Found 1 device(s):
 0: NOXON, DAB Stick, SN: 0
 Using device 0: Terratec NOXON DAB/DAB+ USB dongle (rev 1)
 …
 f502daa10b494e537b52346431305f4672337175336e63595f41635175317233647d00000023
```

From this frame, it is required to remove the preamble (first byte) + address (following 4 bytes) and CRC (last byte).

```python
print "f502daa10b494e537b52346431305f4672337175336e63595f41635175317233647d00000023"[10:-2].decode("hex")
'INS{R4d10_Fr3qu3ncY_AcQu1r3d}\x00\x00\x00'
```

Flag: `INS{R4d10_Fr3qu3ncY_AcQu1r3d}`
## Part3 - SDR Encryption

```
IoT Gymkhana - Part 3

Let's add some crypto on top of all that, shall we ?

Rating: Medium
```

For the third and last part of the challenge, one can send its own message over the radio.

Upon submitting the form, `/37624f02f2c66ffe19ddb172a5a081b4002227ad?m=` is requested and a `302` redirect answer is sent back to confirm that the message was successfully sent with a specific nonce.

<img src="image16.png" alt="Burp response" />

At the same time on the SDR, two frames are received, the first one being the flag and the second the message, both encrypted.

Let’s find which cryptographic algorithm is used and how we are supposed to recover the flag. Let’s analyze the function which is called when the form is submitted.

<img src="image17.png" alt="Radare2 function logic" />

The aforementionned function is loading an obfuscated value and calls function `0x40108830` two times before going to step 2.

<img src="image18.png" alt="Radare2 0x40108830" />

The function `0x40108830` checks if the value of register a3 (caller register a11) is not equal to 1. When the program enters this function the first time, this value is equal to 2 and will therefore jump to `0x4010887c`.

<img src="image19.png" alt="Radare2 0x4010887f" />

Two different values are loaded in memory in this function:

- `NI@/<Y8^wyX’msO6` which will be stored at `0x178031`
- .. which will be stored at `0x178010`

After this, the execution resumes to the caller function and set caller register a11 to 1 before calling `0x4010887c` a second time. This time the execution will continue to `0x40108836` and after manipulating the value stored in 0x178031 it will go to `0x40108899`.

<img src="image20.png" alt="Radare2 0x40108899" />

Let’s have a look at `0x4010b688` right before the message is sent through the SPI interface.

<img src="image21.png" alt="Radare2 0x40108899" />

To resolve the above function, one has to look in the linker file which contains the ESP32 ROM address table. If you installed the ESP32 ESP-IDF development environment, the file is located in `/components/esp32/ld/esp32.rom.ld` or on GitHub.

The message is encrypted using AES. We know that a nonce is also used from the response we had from the webserver. Let’s have a look on what happens when sending different messages, maybe we will find something interesting.

By increasing the value of the message to 33 characters, the first 2 chars of the nonce will be fixed to 00. If the payload is increased even more, the whole nonce is overwritten with the values provided hex encoded.

<img src="image22.png" alt="Burp query with fixed nonce" />

Let’s analyze the ciphertexts received on the SDR when sending different payloads with the same nonce, keeping in mind that each time a message is sent, two frames will be sent instead of one.

```
??FLAG??     95380ff51f06c35017eda9f5a124ee0e16dabbf704820593
             09ebcfef94ce1aa5
32*A+16*B    3c2be8eb7bc17c31389db6b473f5785b3b36cd03f7943102
             e5ae0cdb66aa34f5

??FLAG??     95380ff51f06c35017eda9f5a124ee0e16dabbf704820593
             09ebcfef94ce1aa5
32*C+16*B    3e29eae979c37e333a9fb4b671f77a593934cf01f5963300
             e7ac0ed964a836f7

??FLAG??     95380ff51f06c35017eda9f5a124ee0e16dabbf704820593
             09ebcfef94ce1aa5
32*D+16*B    392eedee7ec479343d98b3b176f07d5e3e33c806f2913407
             e0ab09de63af31f0
```
As shown above, one nibbles out of two is the same in both encrypted ciphertexts. Usually this behavior is observed because of repeated sequences of bytes in the plaintext whenever AES-CTR or OFB mode is used. These modes turn the AES algorithm into stream cipher mode by concatenating an IV (or nonce) with some other data which will produce a keystream. The keystream is then XORed with the plaintext to produce the ciphertext.


<img src="image23.png" alt="AES CTR mode" />

Also, if AES-CTR is used, it is possible to recover the plaintext without having the key because of the following relationship:

```
plaintext  XOR keystream = ciphertext
ciphertext XOR keystream = plaintext
```

The only thing that is missing to recover the correct keystream, is the nonce used to encrypt the flag… but wait we found a value before that is used when the first frame is sent. (`NI@/<Y8^wyX’msO6`)


Let’s try to see if this works:

Request
```bash
echo "GET /37624f02f2c66ffe19ddb172a5a081b4002227ad?m=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAANI@/<Y8^wyX'msO6" | nc 192.168.1.222 80
```

Answers
```bash
 $ rtl_fm -f 433250000 -s 1600k -g 0 | ./nrf905_decoder -
 Found 1 device(s):
 0: NOXON, DAB Stick, SN: 0
 Using device 0: Terratec NOXON DAB/DAB+ USB dongle (rev 1)
 …
 f503daa10b95380ff51f06c35017eda9f5a124ee0e16dabbf70482059309ebcfef94ce1aa523     << FLAG
 f503daa10b9d371dcf1622ee7d66f38be69935db2008f6a3e92af2208d0ed8bf9dbbeb26e423     << A*33
```

Request
```bash
 echo "GET /37624f02f2c66ffe19ddb172a5a081b4002227ad?m=CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCNI@/<Y8^wyX'msO6" | nc 192.168.1.222 80
```
Answers
```bash
 $ rtl_fm -f 433250000 -s 1600k -g 0 | ./nrf905_decoder -
 Found 1 device(s):
 0: NOXON, DAB Stick, SN: 0
 Using device 0: Terratec NOXON DAB/DAB+ USB dongle (rev 1)
 …
 f503daa10b95380ff51f06c35017eda9f5a124ee0e16dabbf70482059309ebcfef94ce1aa523     << FLAG
 f503daa10b9f351fcd1420ec7f64f189e49b37d9220af4a1eb28f0228f0cdabd9fb9e924e623     << C*33
```

Now recover the keystream and decrypt the flag:

```python
p1 ="AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA".encode("hex")
ct1="f503daa10b9d371dcf1622ee7d66f38be69935db2008f6a3e92af2208d0ed8bf9dbbeb26e423"[10:-2] #Remove preamble, address and CRC

fc="f503daa10b95380ff51f06c35017eda9f5a124ee0e16dabbf70482059309ebcfef94ce1aa523"[10:-2] #Remove preamble, address and CRC

#keystream = payload1 ^ ciphertext1
 ks= '%x' % (int(p1,16)^int(ct1,16))

#flag = keystream ^ ciphertext_flag
print ('%x' % (int(ks,16)^int(fc,16))).decode("hex")
```

Flag: `INS{Hell0_cRyPto_mY_o1d_Fr13nd}`
