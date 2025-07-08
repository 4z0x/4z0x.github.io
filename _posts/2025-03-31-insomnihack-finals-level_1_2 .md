---
title: "Insomni'hack 2025 - Passvault (level 1 & 2)"
date: 2025-03-31
categories: [CTF, Hardware]
tags: [Insomnihack]
media_subpath: /assets/media/2025-03-31-insomnihack-finals/
---

## Introduction
Insomni’hack 2025 took place from March 10 to 15, 2025, at the SwissTech Convention Center in Lausanne, Switzerland. Organized (now) by Orange Cyberdefense Switzerland, this event has grown from a modest hacking contest in 2008 to one of Europe's largest cybersecurity gatherings.

For this edition, I prepared a 3-stage hardware challenge `Passvault`, which only one team managed to solve even few other teams where not far away from the final solution.

![Final Scoreboard](ins25_scoreboard.jpeg)


## Hardware

Before diving into the technical details, let me introduce the device:

|----|----|
|<img src="ins25_passvault_full.jpeg" alt="Device" /> | <img src="ins25_passvault_teardown.jpeg" alt="Device teardown"/> |



As shown above, the device consists of a Nordic nRF5340 Development Kit, equipped with a custom-designed adapter that enables interfacing with a MikroTik module containing NXP’s SE050C secure element.

All ressources, 3d print and adapter can be found on the [repository](https://github.com/4z0x/INS25_Passvault)

## Passvault 1/3
### Description

```
Introducing Passvault !

Like any password vault, simply remember your master password to access it.

Even if you forget it, Passvault's got your back with its built-in feature to jog your memory !
```

### Solution
As mentionned in the description, the system relies on a user-defined master password for access. This could be verified by the participant on the built-in shell exposed on the USB interface which only provided an `unlock` command.

Fortunately, the system includes a built-in "memory jogger" functionality which assist users in recalling a forgotten master password.

When the device is powered on, a careful observer will notice that the LED sequence played at startup is not only consistent but also loops afters some time.

By observing the LEDs more closely, subtle timing differences become noticeable between the moment when a pattern is displayed and the LEDs being totally off. Long story short, the sequence looks as follow:

| Timing    | Description       |
| --------- | ----------------- |
| [3600 ms] | Startup animation |
| [500  ms] | Pattern           |
| [200  ms] | Short pause       |
| [500  ms] | Pattern           |
| [500  ms] | Long pause        |
| [3600 ms] | Stop animation    |

The entire sequence last about ~50 seconds before repeating, meaning there are ~40 patterns to extract.

Assuming an ASCII string is being transmitted nibble by nibble (4-bits at a time), one of the LED should be lighten up rarely. Indeed, as characters range from `0x00` to `0x7F` the upper bit (bit7) will always be `0`. In the present case, this can be easily correlated to `LED4` on the board.

Here is how the flag could be obtained with some scripting on a prerecorded video:

```python
import cv2
import time

class LED:
    def __init__(self, name:str, x:int, y:int, w:int, h:int, thres_on:int,thres_off:int):
        self.name = name
        self.x = x
        self.y = y
        self.w = w
        self.h = h
        self.thres_on = thres_on
        self.thres_off = thres_off
        self.status = 0
        self.brightness = 0

def main():
    # Path to input video
    video_path = "video.mp4"
    
    # Output video path
    output_path = 'decoded_output.avi'

    # Open the video
    cap = cv2.VideoCapture(video_path)
    if not cap.isOpened():
        print(f"Error: Could not open video file: {video_path}")
        return
    
    # Get video properties
    fps    = 60
    width  = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
    fourcc = cv2.VideoWriter_fourcc(*'mp4v')

    # Set up video writer
    out = cv2.VideoWriter(output_path, fourcc, fps, (width, height))
    
    # define leds zones and threshold
    leds = [
        LED("LED1",366, 183, 5, 7, 215, 210),
        LED("LED2",368, 238, 5, 7, 215, 210),
        LED("LED3",329, 184, 5, 7, 219, 210),
        LED("LED4",330, 239, 5, 7, 219, 210),
    ]

    old_nibble = 0
    nibbles = []

    nb_frame = 0
    while cap.isOpened():
        nibble = 0
        ret, frame = cap.read()
        if not ret:
            break

        # skip some frames to get proper decoding
        if nb_frame < 155 or nb_frame in [788, 904, 905] or nb_frame > 1100:
            nb_frame += 1
            continue
        
        for i, led in enumerate(leds):
            gray = cv2.cvtColor(frame[led.y:led.y+led.h, led.x:led.x+led.w], cv2.COLOR_RGB2GRAY)
            cv2.rectangle(frame, (led.x, led.y), (led.x + led.w, led.y + led.h), (0, 0, 0), 1)
            led.brightness = gray.mean()
            
            if led.status == 0 and led.brightness >= led.thres_on:
                led.status = 1
            elif led.status == 1 and led.brightness < led.thres_off:
                led.status = 0
            
            if led.status == 1:
                cv2.putText(frame, led.name + " " + str(led.status), (10, 30*(i+1)), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
            else:
                cv2.putText(frame, led.name + " " + str(led.status), (10, 30*(i+1)), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)
            
            nibble |= led.status << i


        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

        # Print nibble in binary (4 bits) and decimal
        if old_nibble != nibble:
            old_nibble = nibble
            nibbles.append(hex(nibble)[2:])
        try:
            flag = bytes.fromhex("".join("".join(nibbles).split("0")))
            cv2.putText(frame, flag.decode(), (120,300), cv2.FONT_HERSHEY_SIMPLEX, 1, (255, 255, 0), 2)
        except:
            pass
        
        cv2.imshow("LEDs", frame)
        out.write(frame)
        nb_frame += 1
    
    time.sleep(5)
    cap.release()
    out.release()
    cv2.destroyAllWindows()
    
    print(f"Final flag => {flag}")
    
if __name__ == "__main__":
    main()
```

And here is the resulting video with opencv on-screen additions:

{%
  include embed/video.html
  src='/decoded_output.webm'  
  title='Decoding the flag using Python and OpenCV'
  autoplay=true
  loop=false
  muted=true
%}


### Flag

`INS{F1r5t_Ch4r_1s_(}`


## Passvault 2/3
### Description
```
Just before the CTF began, the PSIRT was made aware a critical vulnerability that lets you bypass the master password. ;(

Can you find it ?
```

### Solution
When reseting the device while being connected, one can observed that the master password is the following output:

```bash
__________                      ____   ____            .__   __
\______   \_____    ______ _____\   \ /   /____   __ __|  |_/  |_
|     ___/\__  \  /  ___//  ___/\   Y   /\__  \ |  |  \  |\   __\
|    |     / __ \_\___ \ \___ \  \     /  / __ \|  |  /  |_|  |
|____|    (____  /____  >____  >  \___/  (____  /____/|____/__|
                \/     \/     \/               \/
Version: SE-050C2
[I] Decrypting Master password
[I] Done decrypting Master password

$
```

In this challenge, the system uses an NXP SE050 secure element (SE) to store and protect the master password. Under normal conditions, the microcontroller fetches the master password over the I²C interface (`SDA` and `SCL` lines) from the SE050 and validates user input against it via the `unlock` command

However, if the SE050 is physically removed or rendered non-functional (e.g., by shorting SDA/SCL), the master password cannot be retrieved anymore.

Because of a lack of fail-safe mechanism, the firmware will simply continue booting and expose the shell to the user. Since the password has not been populated into the array, any `null` user input corresponding will be therefore treated as the correct one.

Input `unlock ""` to retrieve the flag on the shell.

> This technique can be useful for bypassing a BIOS password on a computer, as explained [here](https://cybercx.co.uk/blog/bypassing-bios-password/)
{: .prompt-tip }

### Flag

`INS{P4ssv4ul7_SE_Byp4ss3d}`
