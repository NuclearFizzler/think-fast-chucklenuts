---
title: Think Fast Chucklenuts!
feature_text: |
  ## Think Fast Chucklenuts!
  An IoT Remote Wakeup Device with a bright flare
feature_image: "assets/img/mask.png"
excerpt: "A CC3200-based IoT wakeup machine with a sleeping mask, speaker, OLED, and mobile app."
---

*Think Fast Chucklenuts!* consists of a sleeping mask that has extremely bright LEDs sewn into the lining, which will be triggered by a CC3200-based Main Controller at a certain time. The Main Controller will also display a submitted image and play a submitted sound, retrieved from a custom backend. Finally, users can use a custom mobile app to create wakeups and submit wakeup media to.   


### Video Demo

#### Actual Demos
<br/>
<div style="display:flex; gap:10px; margin-top:10px;">

  <div style="width:50%;">
    <div style="font-weight:bold; margin-bottom:6px;">Scuffed Full Product Demo</div>
    <video style="width:100%;" controls>
      <source src="{{ site.baseurl }}/assets/video/IMG_8130.mp4" type="video/mp4">
      Your browser does not support the video tag.
    </video>
  </div>

  <div style="width:50%;">
    <div style="font-weight:bold; margin-bottom:6px;">Extra App Features</div>
    <video style="width:100%;" controls>
      <source src="{{ site.baseurl }}/assets/video/IMG_8131_rotated.mp4" type="video/mp4">
      Your browser does not support the video tag.
    </video>
  </div>

</div>

#### Cursed I2S Behavior
<br/>
<video width="100%" height="400" controls>
  <source src="{{ site.baseurl }}/assets/video/IMG_8127.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
---

### Functional Specification
![Functional Spec]({{ site.baseurl }}/assets/img/functional_diagram.png)

### System Architecture

![System Architecture Diagram]({{ site.baseurl }}/assets/img/architecture.png)

The project has four major components working together:

<table style="border-collapse: collapse; width: 100%;">
  <tr>
    <th style="border: 1px solid #333; padding: 8px; text-align: left;">Component</th>
    <th style="border: 1px solid #333; padding: 8px; text-align: left;">Role</th>
  </tr>
  <tr>
    <td style="border: 1px solid #333; padding: 8px;">Sleeping Mask</td>
    <td style="border: 1px solid #333; padding: 8px;">Passive IR-triggered LED flash mask worn by the person being woken up</td>
  </tr>
  <tr>
    <td style="border: 1px solid #333; padding: 8px;">Main Controller (CC3200)</td>
    <td style="border: 1px solid #333; padding: 8px;">FreeRTOS base station that polls backend, drives LEDs, sound, and OLED</td>
  </tr>
  <tr>
    <td style="border: 1px solid #333; padding: 8px;">Backend App</td>
    <td style="border: 1px solid #333; padding: 8px;">Custom FastAPI REST server storing wakeup state and media</td>
  </tr>
  <tr>
    <td style="border: 1px solid #333; padding: 8px;">Frontend App</td>
    <td style="border: 1px solid #333; padding: 8px;">Flutter app for creating/viewing/modifying wakeups</td>
  </tr>
</table>

<br/>

**Operation Overview:**
1. The **Frontend App** lets a user schedule a wakeup with a time, an image URL, and a YouTube audio link. 
2. That wakeup is stored in the **Backend**, which serves that information to the Main Controller, downloads the requested media, and converts it to a compatible format. 
3. The **Main Controller** polls the backend for wakeup information; when wakeup time arrives, plays sound through the MAX98357A amplifier, displays the image on the OLED screen, and flashes the IR LED to trigger the sleeping mask continually until the target presses the E-Stop button.

---

## Implementation

#### Main Controller (CC3200 + FreeRTOS)

<br/>

| ![Main Controller CAD]({{ site.baseurl }}/assets/img/BombCad.png) | ![Main Controller Real]({{ site.baseurl }}/assets/img/BombWiring.png)  |

The main control flow uses TI's OSI layer over FreeRTOS with three tasks:

- **`vIdleTask`**: Polls the backend via HTTP, parses wakeup time/ID, waits until the scheduled time, then pauses itself and spawns the other two tasks while displaying the image on the OLED.
- **`vLEDTask`**: Blinks an IR LED at ~38 kHz (half-period delay of 173 cycles ≈ 13 µs at 80 MHz) to activate the sleeping mask's IR receiver. Blink rate progressively increases over time. Monitors a sync object for the E-Stop signal to self-delete.
- **`vSoundTask`**: Drives a MAX98357A I2S amplifier connected to a 3W 8Ω speaker. Due to `I2SDataPut` blocking issues with TI's HAL, the task buzzes the data line at an abrasive frequency by deliberately misusing a non-I2S pin that happened to output data correctly. SD pin left floating, Vin/GND decoupled with a 1000 µF capacitor from the 5V rail, gain set to GND for 12 dB.

The I2S pins used are `McASP0_McACLK` → LRC, `McASP0_McASFX` → BCLK, `McASP0_McAXR0` → DIN on the amplifier.

Both `vLEDTask` and `vSoundTask` call `osi_SyncObjWait(OSI_NO_WAIT)` in their loops; `vIdleTask` calls it with `OSI_WAIT_FOREVER`. When the E-Stop GPIO interrupt fires, it signals the sync object to make the LED and sound tasks self-delete and the idle task resumes.

All electronics are housed inside a **custom 3D-printed box** with cutouts for the OLED, speaker, E-Stop button, USB flashing port, and power jack.

---

#### Sleeping Mask

<br/>

| ![Mask Transparent Background]({{ site.baseurl }}/assets/img/mask.png) | <img src="{{ site.baseurl }}/assets/img/ripgrannygothitbyabazooka.jpg" style="width: 50%; height: auto;" alt="RIP granny got hit by a bazooka">  |

The sleeping mask is a passive, battery-powered circuit:

- **3× AAA batteries** (one holder modified with a solder bridge to fit a single cell in a 2-cell holder)
- **1× IR Receiver**: outputs active-low when it detects ~38 kHz modulated IR at ~980 nm
- **2× PNP transistors**: driven by the IR receiver output (active-low → PNP is correct polarity)
- **2× Camera flash LEDs** with 3.3 Ω current-limiting resistors



---

#### Backend App

![Backend code sample]({{ site.baseurl }}/assets/img/backend_code.png)
![Backend Live Docker]({{ site.baseurl }}/assets/img/backend_docker.png)

A **FastAPI** REST service, containerized with Docker Compose. Each wakeup record stores:

```json
  {
    "id": 0,
    "description": "string",
    "wakeup_time": 0,
    "image_url": "string",
    "sound_url": "string",
    "image_path": "string",
    "sound_path": "string",
    "awake": true
  }
```

Two API surfaces:
- **Frontend API**: CRUD API that allows the Frontend to create/update/delete wakeups
- **Controller API**: minimal endpoint returning only the soonest wakeup time + ID and serving converted media, to minimize JSON parsing complexity on the microcontroller

Media is fetched from provided URLs and converted to device-compatible formats on the server side.

---

#### Frontend App (Flutter / Android)

<br/>

| ![App beta]({{ site.baseurl }}/assets/img/prelim_app.png) |  <img src="{{ site.baseurl }}/assets/img/app.jpg" style="height: 600px; width: auto;" alt="App running on phone"> |

Built with Flutter for cross-platform portability; deployed as an Android APK for the demo. 

**Features:**
- Create, view, edit, and delete wakeups
- Real-time status updates when the woken user presses E-Stop
- Refresh and notification bell UI for live wakeup monitoring

---

## Bill of Materials

| Item | Source | Unit Cost | Qty | Total |
|---|---|---|---|---|
| TI CC3200 Launchpad | Lab | $0 | 1 | $0 |
| OLED Screen | Lab | $0 | 1 | $0 |
| IR Receiver | Lab | $0 | 1 | $0 |
| E-Stop Button | Amazon | $10.00 | 1 | $10.00 |
| MAX98357A Amplifier | Amazon | $3.33 | 1 | $3.33 |
| 3W 8Ω Speaker | Amazon | $2.50 | 1 | $2.50 |
| IR LED | Amazon | $0.60 | 1 | $0.60 |
| Camera Flash LEDs | LCSC | $0.33 | 2 | $0.66 |
| PNP Transistors | Amazon | $0.022 | 2 | $0.044 |
| AAA Battery Holders | Amazon | $1.25 | 2 | $2.50 |
| Breadboard Power Supply | Amazon | $1.80 | 1 | $1.80 |
| Sleeping Mask | Amazon | $2.30 | 1 | $2.30 |
| Power Supply Adapter | Amazon | $5.00 | 1 | $5.00 |
| **Total** | | | | **~$28.73** |