# SelfhostedSmartHomeGuide
### This is not done yet. In-progress. You assume all responsibility for if *anything* goes wrong for *any reason*.

# Table of Contents
[**Prerequisites**](README.md#prerequisites)

[**Setting up the Pi**](README.md#setting-up-the-pi)

[**Initally setting up Rhasspy**](README.md#initally-setting-up-rhasspy)

[**Making it smart**](README.md#making-it-smart)

[**Adding features (skills)**](README.md#features)

[Controlling Lights](README.md#controlling-devices)

[Basic maths](README.md#doing-basic-maths)

[Setting timers](README.md#setting-timers)

[Getting the weather](README.md#the-weather)

[Getting the time](README.md#getting-the-time-but-better)

[Getting the date](README.md#getting-the-date)

[Giving greetings](README.md#giving-greetings)

[Using as bluetooth speaker](README.md#bluetooth-audio-streaming-highly-imperfect)

[Adding natural and varied responses](README.md#natural-and-varied-responses)

[Converting Units](README.md#converting-units)


# Prerequisites

## *What's this?*
A project that aims to make a self-hosted smart speaker with a reasonable amount of functionality, speed, and ease of use once complete. It won't be perfect, but it'll work. We'll use [Rhasspy](https://rhasspy.readthedocs.io/en/latest/)  to tie together all of the separate bits we need to assemble an assistant (TTS, STT, Wake Word, etc), and [Homeassistant](https://www.home-assistant.io/docs/) to interface with any smart devices. Our equivalent of an Alexa Skill would be an Intent, and whenever Rhasspy recognises one of these, it'll be passed along to a Python script to be handled. This means we can do anything Python can do, assuming we've got the patience to program it.

## What's needed
* Raspberry Pi 4

* Micro SD Card >=16GB (or other boot device, USB drives work too)

===BELOW IS FOR TESTING ONLY===

* USB Mic

* 3.5mm Speakers

# Setting Up the Pi

## Flashing the OS

### Download the [Raspberry Pi Imager](https://www.raspberrypi.com/software/), and install/run it.

Choose OS, then **Raspberry Pi OS (other)**

![Other](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/otherOS.png)

then **Raspberry Pi OS 64-bit lite**

![64-bit-lite](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/lite64.png)

### Now, click the settings icon in the bottom right corner.

* Set hostname to whatever you want to call it. I chose assistant-main-node
* Enable SSH 
* Set username and password
* Configure Wifi if you're going to use it instead of ethernet. Ethernet is more stable, but Wifi allows for more freedom in positioning.
* Scroll down, save.

![Settings page](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/settings.png)

### Choose your boot device (what you'll be keeping in the Pi, typically a MicroSD card)
![Boot device list](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/selectboot.png)

### And finally, press write, and wait for it to finish.

![Writing Media](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/writingprogress.png)

## Booting and initial install
Assuming you have a Micro-HDMI cable, you can turn on the Pi without anything inserted to ensure it boots fine. You'll likely get a screen like this.

![Boot with no device](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/bootnodevice.png)

But if - like most people - you don't own a Micro HDMI cable, that's alright. We can run *headlessly* (without a display)

### Regardless of whether you'll be connecting a keyboard and monitor to the Pi or not, make sure it's off, then insert your boot drive, and plug it in. Then, wait for one of two things:

If you've got a display connected, wait until there's a line with your username in green at the bottom of the screen, like this.

![Linux bootup scene](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/withdisplaybootup.png)

If you don't have a display connected, go to a computer on the same network as your Pi (so, if you didn't set up Wifi, connect an ethernet cable). Then, run **"terminal"** (same on Linux, Windows 11, or MacOS, but on Windows 10, run cmd). Now, type ```ssh yourusername@yourhostname.local```, and replace 'yourusername' with your Pi username, and 'yourhostname' with the hostname you typed in the settings page. At first, it'll likely error out, since the Pi isn't done booting yet, but you can press the up arrow and enter to run the command again. You know you've succeeded once you see this page.
![SSH fingerprint question](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/fingerprint.png)

Type yes, enter, then type your password (which won't show up onscreen as a security measure, but it *is* still going through). 

### From here, we can run commands.

First, run
```
sudo apt update && sudo apt upgrade -y
```
to get up to date

### Now, set a static local IP

In your terminal, type ```ip route | grep default```. Then, note down three things: the first IP, the network device, and the second IP. The IPs will likely be 192.168.x.x, but may be different. In the image, I've highlighted these things in yellow so you know where to look.

![my private IP for the Pi](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/ipdefault.png)

Now, run
```
sudo nano /etc/dhcpcd.conf
```
and navigate down to the bottom line using the arrow keys, then press enter a few times to add some lines. You should get to here:

![dhcpcd](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/dhcpcd.png)

Then, paste this in:

```
interface
static ip_address=/24
static routers=
static domain_name_servers=1.1.1.1
```

Next to interface, add a space, then the network device (either **eth0** or **wlan0**). Now, for static ip_address, type the second IP before the */24*. Finally, add the first IP from earlier directly after **static routers=**. Then, press CTRL+X, then Y, then Enter, to save and exit. Finally, run ```sudo reboot``` to restart. Your SSH will disconnect, and you can just keep trying to reconnect until it works to check if you're booted.

## Optimisations

### We can make our pi run better, and use less power on things we're not using.

#### Running better

The Pi can be overclocked. Run this:
```
sudo nano /boot/config.txt
```
then go down to where it says **"#uncomment to overclock the arm"**

![default boot config](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/defaultarmover.png)

Remove the **#** from the line beginning ```#arm_freq```, and change the number to 1750. Add another line below the **"#uncomment to overclock the arm"** bit, and copy this in:
```
over_voltage=2
```
Then, if you're not going to be using the HDMI ports, below the ```#arm_freq``` line, paste this:
```
gpu_freq=0
```
In the end, it'll look like this:

![better boot config](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/betterarmover.png)

If you want to put in the effort **and extra cooling**, you can tune this for better performance at the cost of more heat, however the above config should be doable by practically all Pi 4s, and remain at safe temperatures. I do **over_voltage=4**, **arm_freq=2000**, and have not had any cooling or stability issues despite running the Pi bare.

#### Using less energy

Now, to disable all LEDs on the board, go down to the section starting with **```[pi4]```** and paste what's below:
```
# Disable the power LED
dtparam=pwr_led_trigger=none
dtparam=pwr_led_activelow=off
# Disable the Activity LED
dtparam=act_led_trigger=none
dtparam=act_led_activelow=off
# Disable ethernet LEDs
dtparam=eth_led0=4
dtparam=eth_led1=4
```

And, if you're not using the GPU, you can also add ```gpu_mem=16``` to the **"[all]"** section above. It likely won't affect anything though. Something that will help power consumption is the line **```dtoverlay=disable-bt```**. If you're not using wifi either, you can duplicate that line and change **bt** to **wifi**. However, if you intend on using this like a regular smart-speaker (including the ability to play music as a bluetooth speaker, and not needing to run an ethernet cable to it), I suggest leaving both Wifi and bluetooth enbaled.

In the end, it'll look like this:

![better led config](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/betterled.png)

You can now do CTRL+X, Y, ENTER, to save and exit, then run ```sudo reboot``` to restart your pi. Once you're back, continue with the next section.

## Installing things
Run
```
curl -sSL https://get.docker.com | sh
sudo apt-get install -y uidmap libffi-dev libssl-dev python3 python3-pip python3-dev
sudo pip3 install docker-compose
sudo gpasswd -a $USER docker
```
to install docker and docker compose. This may take a while.

Then run

Then, to install rhasspy, run
```
mkdir assistant
cd assistant
nano docker-compose.yml
```
and paste in the following. This will be much easier if being run from another PC through SSH, since indentation must be exactly correct.
```
version: '3.3'
services:
    rhasspy:
        container_name: rhasspy
        ports:
            - '12101:12101'
            - '12183:12183'
        volumes:
            - './profiles:/profiles'
            - '/etc/localtime:/etc/localtime:ro'
        devices:
            - '/dev/snd:/dev/snd'
        image: rhasspy/rhasspy
        command: --user-profiles /profiles --profile en
        restart: unless-stopped
```
Then, press CTRL+X, Y, Enter, to save and exit. After which, you can just run
```
sudo docker-compose up -d
```
to begin installing. This, again, may take a while. You'll know it's done once you see this:
![Install complete](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/donedocker.png)

# Initally setting up rhasspy
In a browser on the same network as your Pi, go to this site, changing 'yourhostname' to your hostname.
```
http://yourhostname.local:12101
```
Your browser *should* complain that this site is not secure. If it was a site on the internet, you wouldn't want to access it, however it's just local, so we can tell the browser that we want to continue.
![https issue](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/httpsonly.png)

### And now you'll be here!
![Rhasspy main page](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/rhasspyhome.png)

Then, go to the settings page using the left-side menu. Go through each service, and - for now - just select the default from the dropdown menu. Then, press the save settings button below the list, and restart. Once restarted, go to the top of the page, and press download. After that's done, things should look like this.

![Settings page with defaults selected](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/settingdefault.png)

## Testing things

### Testing audio output
#### Plug something in using the Pi's 3.5mm jack.

To test audio output, go back to the home page, and type something into the 'speak' box, and see if it comes out of your speakers. 

![AudioTest](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/audiotest.png)

It will likely sound quite bad, but should work.

### Testing audio input
#### Plug a USB microphone into a USB port

To test audio input, press 'Wake Up' on the home page, and say "What time is it?". If it hears you, you'll get a response, and the UI will update to show this:

![AudioInputWorks](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/audioinputworks.png)

If there's no response, try relaunching rhasspy, by going back to your terminal, and typing
```
sudo docker-compose up -d
```
This may get your mic detected if it wasn't before.

## Some improvements
### TTS
I reccommend going back to the settings page, switching your **Text To Speech** to ```Larynx```, pressing refresh, and choosing a voice you think sounds good. **Southern-english-female** is - at this point in writing - my chosen voice, since higher-pitched voices will work better for voice assistants due to them often using small speakers with little bass response, and I believe it to be the most natural sounding. **Low Quality Vocoder** is perfectly fine, as you'll see when you test it, and is necesary for fast responses on a Pi. Though, **Larynx takes around 15 seconds to initialise each time you reboot, and doesn't do this automatically,** meaning the first question you ask will be highly delayed. 

### Remember to save your settings and restart afterwards.

![TTSsettings](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/Texttospeech.png)

### Wake word
To wake things without using the web UI, you *could* set a custom word using **Rhasspy Raven,** however I had trouble with being recognised. Instead, I use **Porcupine**. I just went into porcupine's dropdown, pressed refresh, and selected one from the list, and I'd suggest you do the same. I also increased the sensitivity to **0.85** so it can pick me up when I'm quieter. Save and restart, and it should work.

# Making it smart
## Setting up Homeassistant
### If you've already got a homassistant instance, scroll down until we need our access tokens.
To set up homeassistant, first we need a docker-compose file, just like what we had for rhasspy.
So, run:
```
mkdir ~/hass
cd ~/hass
nano docker-compose.yml
```
And paste in:
```
version: '3'
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
```
Then, press CTRL+X, Y, Enter, to save and exit. After which, run
```
sudo docker-compose up -d
```
Once that's done, go to a browser on another machine on your network, and go to this URL: (or your already existing server's URL)
```
http://yourhostname.local:8123
```
Replacing 'yourhostname' with your hostname. It should look like this: (after the same security prompt as before)

![hass onboarding](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/hassonboarding.png)

Just type a name, username, and make a password, and press 'Create Account'

![hass make account](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/hassmakeaccoun.png)

Now, give your home a name and location, (plus elevation, if you'd like)

![hass name home](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/hassnamehome.png)

And choose which data to *opt-in* for. These are all disabled by default, however I'd ask that you consider turning on anything you're comfortable with, since it can help the devs.

![hass opt in](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/hassoptin.png)

Finally for the onboarding, just press finish! This page shows any services that homeassistant automatically found, but we'll set things up later.

![hass auto find](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/hassautofind.png)

Now, you should be on the main homeassistant page. Click your name in the bottom left, then scroll down to long-lived tokens.

![llat](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/llat.png)

Create one, name it whatever you'd like, and save it for a minute.

Go back to your rhasspy tab, then settings, scroll down to intent handler, and select local command.

![local intent handler](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/localcommandintents.png)

Then, press the green dropdown, and set the program to ```/profiles/intentHandler```

Now, you can press save and restart.

While it's restarting, run this:
```
sudo nano ~/assistant/profiles/intentHandler
```
Then, paste this in [(we're building from the Rhasspy example)](https://github.com/synesthesiam/rhasspy/blob/master/bin/mock-commands/handle.py):
```
#!/usr/bin/env python

import sys
import json
import random
import datetime
import requests

def speech(text):
    global o
    o["speech"] = {"text": text}

# Set Homeassistant URL
hassurl = "http://"
hassauth = ""
hassheaders = {"Authorization": "Bearer " + hassauth, "content-type": "application/json",}

# get json from stdin and load into python dict
o = json.loads(sys.stdin.read())

intent = o["intent"]["name"]

if intent == "GetTime":
    now = datetime.datetime.now()
    speech("It's %s %d %s." % (now.strftime('%H'), now.minute, now.strftime('%p')))

elif intent == "Greet":
    replies = ['Hi!', 'Hello!', 'Hey there!', 'Greetings.']
    speech(random.choice(replies))

# convert dict to json and print to stdout
print(json.dumps(o))
```
Now, add your Pi's IP (adding ```:8123``` to the end) to the hassurl section, and your auth token to the hassauth section.

CTRL+X, Y, ENTER.

Then, run
```
sudo chmod +x ~/assistant/profiles/intentHandler
```
which will allow the script to be executed.

Now, go to rhasspy's web UI, click the wake button, and say out loud, "What time is it?". It should respond with the current time. If not, give it about 20 seconds, the TTS may be doing first-time setup.

Go to your terminal (still SSH'd into the Pi), and type 
```
sudo nano ~/hass/config/configuration.yaml
```

Go right to the top of the file, and look for the line ```default_config:```. Go one line below it, and add exactly:
```
api:
```

Also, I highly reccomend going some lines below, and pasting this, which will prevent homeassistant from taking up lots of space on your (presumably quite limited) Pi storage, and reduce disk usage, prolonging life:
```
# Remove history to save space
recorder:
  auto_purge:
  purge_keep_days: 14
  commit_interval: 30
```
Then, CTRL+X, Y, ENTER.

You can also run ```sudo docker restart homeassistant``` now too.

# Features

## Controlling devices

So you know, there are more ways to accomplish things. I'll be describing the methods I use, but if there's a better method, please feel free to share, I'd appreciate it.

### First, we need a thing to control. 

This isn't a homeassistant tutorial, but if you've got any WLED devices, they should automatically appear in the devices section to be configured like this:

![Autoadd1](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/Autoadd1.png)
![Autoadd2](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/Autoadd2.png)
![Autoadd3](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/Autoadd3.png)

To check the device name, go to settings, then entities in Homeassistant. Then, click on the device you're intending to control. Note down the name at the bottom.

![Wled device in entities](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/wledentities.png)

Back in Rhasspy, click the circular "slots" tab,

![slot tab](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/slottab.png)

then make a new slot called lights. Within regular brackets, put the name you'd like to speak. If there are multiple, such as "Bed LEDS" and "Bedside LEDs", separate them with a pipe symbol (|). Then, immediately after the brackets, add a colon (:), and **without adding a space**, add the entity id from homeassistant. Here's what mine looks like with two lights.

![Light slot](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/lightslot.png)

Then, head back to your sentences section, and remove what you've already got. If you want to use my setup, paste this in:
```
[SetSpecificLightPower]
light_name = ($lights){entity}
light_state = (on | off){state}
(set | turn) <light_state> [the] <light_name>
(set | turn) [the] <light_name> <light_state>
```
What's within the top square brackets is what the intent handler will recognise when checking what event is being sent. Then, we set two variables. light_name equals what's in our lights slot (we know we're talking about a slot because of the $), and light_state can be on or off. Again, **or** is represented by a pipe (|). [The next bit was taught to me by a post on the rhasspy community page. Credit to them for this config!](https://community.rhasspy.org/t/access-from-home-assistant-the-raw-value-in-slots-array/3497) Then, we make some sentences. I made two, so I can say things in different orders. The first would allow me to say **"Turn on the bedside light"**, and the second allows **"Turn the bedside light on"**. Arrow brackets reference variables, regular brackets reference groups of words, square brackets reference optional words, and curly brackets reference the name that the sent data will have in the JSON file that Rhasspy sends to homeassistant. When saving, remember to allow training.

While SSH'd into the pi, run
```
sudo nano ~/hass/config/automations.yaml
```
Then, paste this in some empty space (remove square brackets if there):
```
- alias: "Turn on/off specific light"
  trigger:
  - event_data: {}
    platform: event
    event_type: assistant_SetSpecificLightPower
  action:
     - service: light.turn_{{trigger.event.data.state}}
       data:
         entity_id: "{{trigger.event.data.entity}}"
```
If you changed what's within the square brackets in the sentence section, change what's after ```assistant_``` to match. Otherwise, things should just work. Now, go to homeassistant's dev tools, YAML, and reload automations. 

Now, run:
```
sudo nano ~/assistant/profiles/intentHandler
```
And paste this below the last elif statement:
```
elif intent == "SetSpecificLightPower":
    entity = o["slots"]["entity"]
    state = o["slots"]["state"]
    speech("Alright, I'll turn it " + state )
    requests.post(hassurl+"/api/events/assistant_"+intent, headers = hassheaders, json = {"entity": entity,"state": state})
```
Things should look like this:
![add intents](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/addintents.png)

Anything within ```speech()``` will be spoken. 

### Now, we'll learn how to add colour.

Go back to your slots, add a new one called ```colours``` (the British spelling), and paste this:
```
(aqua | aquamarine | beige | black | blue | brown | chocolate | coral | crimson | cyan | firebrick | forest green | gold | gray | green | hot pink | indigo | khaki | lavender | light blue | light coral | light cyan | light green | light pink | light salmon | light yellow | lime | lime green | magenta | maroon | navy | olive | orange | orchid | pink | purple | red | salmon | tomato | violet | white | yellow)
```
It actually supports all colours in [this list](https://www.w3.org/TR/css-color-3/#svg-color), so if I omitted your favourite colour, you can add it as long as it's in the page on that link.

![Colour slot](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/colourslotcorrected.png)

Then, go to your sentences, and duplicate your power control section. Change Power to Colour (or apply your own naming convention), change ```light_state``` to ```light_colour```, change ```on | off``` to ```$colours```, and change ```{entity}``` to ```{colour}```. Remember to also change light_state in the actual sentence too, along with correcting the layout of the sentence so it makes sense when you say it. In the end, I've got this:

![Colour sentence](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/coloursentencecorrection.png)

Finally, go back to your terminal with automations.yaml open, and paste this below your power config:
```
- alias: "Set specific light colour"
  trigger:
  - event_data: {}
    platform: event
    event_type: assistant_SetSpecificLightColour
  action:
     - service: light.turn_on
       data:
         entity_id: "{{trigger.event.data.entity}}"
         color_name: "{{trigger.event.data.colour}}"
```

All you should need to change is the event_type if you decided to name things differently. Save and exit (CTRL+X, Y, ENTER), then reload your automations in homeassistant. 

Then, run:
```
sudo nano ~/assistant/profiles/intentHandler
```
We'll paste another elif block, very similar to our last:
```
elif intent == "SetSpecificLightColour":
    entity = o["slots"]["entity"]
    colour = o["slots"]["colour"]
    speech("Alright, I'll make it " + colour )
    requests.post(hassurl+"/api/events/assistant_"+intent, headers = hassheaders, json = {"entity": entity,"colour": colour})
```
Once you've saved an exited, it should work immediately. 

### Adding brightness settings

It would be nice to be able to control brightness, but we can't yet, so let's add it.

First, go back into your intentHandler file:
```
sudo nano ~/assistant/profiles/intentHandler
```
And add this elif statement, just like the colour one:
```
elif intent == "SetSpecificLightBrightness":
    entity = o["slots"]["entity"]
    brightness = o["slots"]["brightness"]
    speech("Alright, I'll make it " + str(brightness) + " percent")
    requests.post(hassurl+"/api/events/assistant_"+intent, headers = hassheaders, json = {"entity": entity,"brightness": brightness})
```
You can probably see how things work now, based on how little has changed from the version of that code which modifies colour instead.

Now, go to your automations.yaml file:
```
sudo nano ~/hass/config/automations.yaml
```
and at the bottom, paste this:
```
- alias: "Set specific light brightness"
  trigger:
  - event_data: {}
    platform: event
    event_type: assistant_SetSpecificLightBrightness
  action:
     - service: light.turn_on
       data:
         entity_id: "{{trigger.event.data.entity}}"
         brightness_pct: "{{trigger.event.data.brightness}}"
```

All that's left is to reload your automations.yaml (if you don't remember, that's **developer tools --> yaml --> reload automations**) and add your Rhasspy sentences.

Go to rhasspy's web ui at ```yourip:12101```, then click sentences on the left, and add this:

```
[SetSpecificLightBrightness]
light_name = ($lights){entity}
(set | turn | make) [the] <light_name> [to] (1..100){brightness} percent [brightness]
```

Save and retrain rhasspy, and things should work.

## Doing basic maths

What if we want to ask the assistant to perform calculations? I'll explain the basic multiplication, subtraction, and addition stuff, and if you want to make it better, you should be able to figure it out from what you learn here.

First, go back to the slots section in the left menu.

![slot tab](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/slottab.png)

Then, we'll define our operations. Make a new slot called **"operations"**, then paste this in:
```
(add | plus):+
(times | multiplied by):*
(minus | subtract | take | take away):-
(divided by | over):/
```
If you'd like to be able to call a certain operation with another word, just add it within the brackets, along with a pipe (|) symbol to separate it from the other words. Save this.

Now, go to the sentences tab, and add a [DoMaths] section. Paste in what's below:
```

[DoMaths]
operator = ($operators){operator}
what is (-1000..1000){num1} <operator> (-1000..1000){num2}

```
This lets us perform those three operations on two numbers between -1000 and 1000. You can increase the range by changing the numbers at either end of the two dots (**".."**), but I was concerned that the assistant may find it harder to tell exactly what number you're saying as the range of numbers increases, so 1000 seemed an alright compromise.

Finally, head over to the intentHandler by running:
```
sudo nano ~/assistant/profiles/intentHandler
```
And paste this below the last elif section:
```
elif intent == "DoMaths":
    operator, num1, num2 = o["slots"]["operator"], o["slots"]["num1"], o["slots"]["num2"]
    if operator == "*":
        calcResult = str(num1*num2)
    elif operator == "+":
        calcResult = str(num1+num2)
    elif operator == "-":
        calcResult = str(num1-num2)
    elif operator == "/":
        calcResult = str(num1/num2)
    if num1 == 9 and num2 == 10 and operator == "+":
        speech("That's 21")
    else:
        speech("That's " + calcResult.replace("." , " point "))
```


Basically, we make variables for the operator and both numbers from the incoming JSON, then just perform the operation, speaking the result. Once you've saved and exited, it should just work. Keep in mind, you've got to say your numbers quite quickly. Once your sentence is perceived to be complete, it will stop listening, even if you're still speaking. This means that if you say - for example - **"twenty seven"** too slowly, it may cut you off before you've said seven.

## Setting timers
What if you want to set a timer? It's not a super complex one, you can only pick minutes **or** seconds, meaning you couldn't ask for a 2 minute and 17 second timer, but it works well enough.

Go to your sentences section, and add this:
```
[DoTimer]
(set | make | start) a (1..60){time} (second | minute){unit} timer
```
Remember to save and train.

Then, go to the intentHandler script (```sudo nano ~/assistant/profiles/intentHandler```) and paste this below the last elif statement:

```
elif intent == "DoTimer":
    number, unit = o["slots"]["time"], o["slots"]["unit"]
    if unit == "second":
        timerLength = number-1
    elif unit == "minute":
        timerLength = (number*60)-1
    while timerLength:
        time.sleep(1)
        timerLength -=1
    speech("Timer complete")
```
It receives a number between 1-60, and the unit (whether you said "Seconds" or "Minutes"). It then sets a variable to the correct number of seconds, either by taking 1 away from the number you said, or multiplying it by 60, then still removing one. Afterwards, it just runs a timer, and will speak once it's complete. You can still run other voice commands while the timer is running.

### But that's not a great way of announcing the completion of a timer

We want a sound. Let's get one. If you've got something on your computer (a .wav file), you can copy it over to the Pi by running the first section ON YOUR PC, NOT THE PI's SSH SESSION(and if you've not got one yet, you can find one at the bottom of this section)
:

```
scp pathtoyourfile piusername@pihostname:/home/assistant-main-node/
```
Then, on the pi, you can do
```
sudo cp /home/assistant-main-node/yourfile.wav ~/assistant/profiles
```

Replace pathtoyourfile with the path to your wav file. Replace piusername with the username you picked for your Pi. Replace hostname with the hostname you picked for your Pi. If you're using the same file structure and docker compose files as me, you can keep the rest of the command the same. When you press enter, it'll ask for your Pi's password. This copies your file to the Pi over ssh. If you choose another method to get the file to the Pi, that's fine, just make sure it's in a directory accessible from the docker container, which is why I chose the profiles folder.

Now, go to the top of your intentHandler script, and add ```from subprocess import call```.

And on another line, add 
```
import os
import time
```

Then, go back down to your DoTimer section, and add these lines right to the top of the elif statement (ensure indentation matches the rest of the code):
```
timerFinishedAudio = workingDir+"yourfile.wav"
if os.path.exists(stopTimerSoundFilePath):
    os.remove(stopTimerSoundFilePath)
```
If you understand what this is doing, you can probably tell where we're going from here. We'll be looping a small piece of audio until we detect a file that tells us to stop, which will be made when we say the voice command **"stop"**. This bit of code was necessary to make sure the file isn't already there incase the user was detected to be saying **"stop"** while a sound wasn't going off. Remember again to replace **"yourfile.wav"** with the name of your file.

Now, you can replace the ```speech("Timer complete")``` line with this:
```
while not os.path.exists(stopTimerSoundFilePath):
    call(["aplay", timerFinishedAudio])
if os.path.exists(stopTimerSoundFilePath):
    os.remove(stopTimerSoundFilePath)
```

In this image, I've also cleaned up assigning the number and unit variables to be on one line, but your code should otherwise now look like this:

![Timer statement](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/timerCode.png)

If you were now to ask for a timer, it would finish by infinitely repeating whatever your sound is. We can fix this by making a new elif statement below:
```
elif intent == "StopTimerSound":
    with open(stopTimerSoundFilePath, 'w') as stopTimerSoundFile:
        pass
```

Now, go to the top of the file, and paste this:
```
# Set paths
workingDir = "/profiles/"
stopTimerSoundFilePath = workingDir+"stopTimerSoundFile"
```

If you're using a different file structure, you can change the data inside the workingDir variable. Remember to save and exit (CTRL+X, Y, ENTER)

Now, go to your rhasspy sentences section, and make a new section that looks like this:
```
[StopTimerSound]
stop [the] [alarm | timer | sound]
```
Remember to save and retrain Rhasspy once done. Now, you should be able to ask for a quick one second timer, then while the audio is looping, ask it to stop. Once the current loop is over, it will finish. 

### Some notes about the audio
Due to it finishing the current audio loop before stopping, I suggest having a simple <5 second sound. Anything long will take a very long time to stop after you ask it to. It's not ideal, but it works, and even this solution took me hours to figure out. I just used an [electronic chime licensed under the Public Domain.](https://soundbible.com/1598-Electronic-Chime.html) Though, there was quite a bit of empty space at the end of that audio file, so I've trimmed it, [and uploaded it here.](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/resources/sounds/timerchime.wav)

### Check Timer Progress While Running
Ideally, you could ask the assistant how far along the timer is. Let's make that.

First, add a variable to the top of your intentHandler in the ```# Set paths``` section called timerLeftPath, and set it to ```workingDir+"timerLeft"```. Then, within your ```while timerLength``` loop, add this to the bottom (ensure indentation stays correct):
```
with open(timerLeftPath, "w") as timerLeft:
    timerLeft.write(str(timerLength))
```

Then, at the bottom of your **DoTimer** intent, duplicate your already existing section which deletes your stopfile, and change ```stopFilePath``` to ```workingDir+"timerLeft"```. It should look like this:
```
if os.path.exists(timerLeftPath):
    os.remove(timerLeftPath)
```
Now we've got a file that contains the number of seconds remaining, and it'll delete itself once the timer is done.

Go to the bottom of your intentHandler, and paste this:
```
elif intent == "TimerRemaining":
    if os.path.exists(timerLeftPath):
        timerRemainingNumber = int(open(timerLeftPath, "r").read()) - 3
        if timerRemainingNumber >= 60:
            speech("There are " + str(math.trunc(timerRemainingNumber/60)) + " minutes and " + str(timerRemainingNumber % 60) + " seconds left")
        else:
            speech("There are " + str(timerRemainingNumber) + " seconds left")
    else:
        speech("You've got no timers set")
```
This checks if the file exists, and if it does, checks whether the time remaining should be measured in minutes (number is > 60) or seconds (number is < 60). Then, if minutes are needed, we divide the number of seconds by 60, then truncate (remove **but not round** the decimals) it, as well as telling us the number of seconds by finding the remainder when dividing by 60. If just seconds are needed, we only need to speak the number we've gotten from the file. Also, since I expect the words to be spoken 3-ish seconds after reading the value, I remove 3 from the number we get at the start.

In here, we use the **math** library, so you'll want to go to the top of your file, and add ```import math``` too.

Now, in rhasspy's sentences tab, you just need to add this:
```
[TimerRemaining]
how long left on timer
what is [the] timer [at]
```
The broken English is intentional, since our speech-to-text system will turn whatever you say into the *closest* sentence possible, so having a shorter sentence that misses words is alright, and I believe *(with no tested evidence)* that it could speed things up.

### Cancelling a timer

But what if you decide that you don't want your timer anymore? (or, more likely, STT picks up the wrong number)

First, add a variable to your ```# Set Paths``` section called cancelFilePath, with the value ```workingDir+"cancelFile"```.

Now, to account for any errors or remaining files, add this line to the other sections where we delete files. One at the start of the timer **elif** statement, one at the end.

```
if os.path.exists(cancelFilePath):
    os.remove(cancelFilePath)
```

Within the ```while timerLength``` section, at the end, add this, which will stop the timer if it detects the cancel file:

```
if os.path.exists(cancelFilePath):
    break
```

We'll now be working within the ```while not os.path.exists(stopFilePath)``` section.

At the same level of indentation as the ```call aplay``` line, right at the top, add:
```
if os.path.exists(cancelFilePath):
    speech("Timer cancelled")
    break
else:
````
Then, correct your ```call aplay``` line to be at the right level of indentation.

Now, add a new elif statement for making the cancelFile:

```
elif intent == "CancelTimer":
    with open(cancelFilePath, "w") as cancelFile:
        pass
```

All that's left to do is go to your Rhasspy sentences, and add this new one:

```
[CancelTimer]
(stop | cancel) [the] timer
```

And remember, if you don't like *anything at all* about how I handle things, **you can change it**. All of the code is free for you to make exactly how you like it, even for things as basic as how you phrase sentences. You don't need to follow everything verbatim, but you can, it's up to you.

### The end result
This timer section was massive. My code at the end of it looks like this:
```
elif intent == "DoTimer":
    timerFinishedAudio = workingDir+"timerchime.wav"
    number, unit = o["slots"]["time"], o["slots"]["unit"]
    if os.path.exists(stopTimerSoundFilePath):
        os.remove(stopTimerSoundFilePath)
    if os.path.exists(cancelFilePath):
        os.remove(cancelFilePath)
    if unit == "second":
        timerLength = number-1
    elif unit == "minute":
        timerLength = (number*60)-1
    while timerLength:
        time.sleep(1)
        timerLength -=1
        with open(timerLeftPath, "w") as timerLeft:
            timerLeft.write(str(timerLength))
        if os.path.exists(cancelFilePath):
            break
    while not os.path.exists(stopTimerSoundFilePath):
        if os.path.exists(cancelFilePath):
            speech("Timer cancelled")
            break
        else:
            call(["aplay", timerFinishedAudio])
    if os.path.exists(stopTimerSoundFilePath):
        os.remove(stopTimerSoundFilePath)
    if os.path.exists(timerLeftPath):
        os.remove(timerLeftPath)
    if os.path.exists(cancelFilePath):
        os.remove(cancelFilePath)
```
Here's the StopTimerSound section:
```
elif intent == "StopTimerSound":
    with open(stopTimerSoundFilePath, 'w') as stopTimerSoundFile:
        pass
```
Here's the cancelTimer section:
```
elif intent == "CancelTimer":
    with open(cancelFilePath, "w") as cancelFile:
        pass
```
And here's the TimerRemaining section:
```
elif intent == "TimerRemaining":
    if os.path.exists(timerLeftPath):
        timerRemainingNumber = int(open(timerLeftPath, "r").read()) - 3
        if timerRemainingNumber >= 60:
            speech("There are " + str(math.trunc(timerRemainingNumber/60)) + " minutes and " + str(timerRemainingNumber % 60) + " seconds left")
        else:
            speech("There are " + str(timerRemainingNumber) + " seconds left")
    else:
        speech("You've got no timers set")
```
## The weather
What if I want it to tell me the weather?

First, go to [openweathermap](https://openweathermap.org/) and sign up for a free account. Then, check your email for an API key. It's important to do this first, since they key will take a little bit to properly activate. Then, you can go to your rhasspy sentences tab, and add this:
```
[GetWeather]
(what's | tell me) the (weather | temperature)
how (hot | cold | warm ) is it [today | right now]
```
If you want to customise the way you speak to it, you can do it from here. Remember to save and train.

Then, go to your intent handler, and below the last elif statement, paste this:
```
elif intent == "GetWeather":
    weather = requests.get(opnwthrurl+"lat="+opnwthrlat+"&lon="+opnwthrlon+"&units="+opnwthrunits+"&appid="+opnwthrauth).json()
    currentTemp = weather["main"]["temp"]
    currentDesc = weather["weather"][0]["main"]
    speech("It's currently " + str(round(currentTemp)) + " degrees and " + currentDesc)
```
However, we're not done. You'll also want to go back to the top (near where your homeassistant details are), and add a line below them, called ```#Set OpenWeatherMap data```. Below it, paste this:
```
opnwthrurl = "https://api.openweathermap.org/data/2.5/weather?"
opnwthrauth = "YOURAUTHKEY"
opnwthrlat, opnwthrlon = "LAT" , "LONG"
opnwthrunits = "metric"
```
Change **YOURAUTHKEY** to your api key from openweathermap, and **LAT** / **LONG** to your current latitude and longitude. They don't have to be exactly on your location, but you can use a tool [like this](https://www.latlong.net/) to get the numbers for your general area.

Then, save and exit, and ask your assistant **"What's the weather"**, and it should tell you the current temperature, along with a word to describe it, like **Sun** or **Clouds**.

## Getting the time (but better)

### The examples provided by Rhasspy can already do this...

but we can do it better. It normally responds in 24-hour (at least, it does for me, though my system is set to 24-hour), which is great for reading the time, but not for speaking it. Also, despite technically telling our assistant to say whether it's AM or PM, it sends the strings "am" and "pm", meaning that they're pronounced very awkwardly. To fix this, you can replace the GetTime intent near the top of the intentHandler with this:

```
if intent == "GetTime":
    now = datetime.datetime.now()
    if now.strftime('%p') == "PM":
        apm = "peey em"
    else:
        apm = "ey em"
    if now.strftime('%M') == 00:
        speech("Its " + now.strftime('%I') + " " + apm)
    else:
        speech("Its " + now.strftime('%I') + " " + now.strftime('%M') + " " + apm)
```

I know that **"Its"** should have an apostrophe to represent a contraction, and it annoys me too, a lot, however I'm trying to avoid extra symbols when necessary.

Because of the interesitng methods of writing AM ("ey em") and PM ("peey em"), this might not sound right if you use a different TTS voice to me. However, on the southern british female voice for larynx, they sound much better than the deault, and it now speaks in 12-hour.

Also, I earlier asked you to remove all of the predone sentences in rhasspy, which would include the GetTime ones. Here's what to add to your sentences:
```
[GetTime]
what's [the] time
tell [me the] time
what time [is it]
```

## Getting the date
To get the date, we'll need to import something new from datetime, so add this to the top of your intentHandler:
```
from datetime import datetime
```

Now, you can add this elif statement:
```
elif intent == "GetDate":
    months = [" January ", " February ", " March ", " April ", " May ", " June ", " July ", " August ", " September ", " October ", " November ", " December "]
    weekdays = [" Monday ", " Tuesday ", " Wednesday ", " Thursday ", " Friday ", " Saturday ", " Sunday "]
    dayNum = datetime.now().day
    month = months[(datetime.now().month)-1]
    weekday = weekdays[datetime.today().weekday()]
    speech(random.choice(currentlyResponse) + weekday + "the " + str(dayNum) + " of" + month)
```
We get a number for the day, day of week, and month, then convert these to words using lists. Then, we speak a sentence which puts it all together.

Go to your Rhasspy sentences section, and add this:
```
[GetDate]
what date [is it]
whats [the] date
tell me [the] date
whats today
```
Save and retrain, and it should work.

## Giving greetings

This is **part of the Rhasspy-provided example script,** however it's a feature nonetheless. 

In your Rhasspy sentences section, add this:
```
[Greet]
Hello
Hi
Good (morning | afternoon | evening)
```
and remember to save and retrain.

It should work immediately, since it was part of the example we pasted in earlier, however we'll look at the code anyway.

```
elif intent == "Greet":
    replies = ['Hi!', 'Hello!', 'Hey there!', 'Greetings.']
    speech(random.choice(replies))
```

If the intent is **"Greet"**, we make a list of items, each of which is a string. In this case, they're just different ways of greeting the user. Then, we randomly pick one of the items and say t. If you want to add extra things to say, just add a string to the list. 

I soon intend to make it aware of the time so it can correct you if you mistakenly say **Good Morning** in the **Evening** (or vice versa).

## Bluetooth Audio Streaming (highly imperfect)
You can use an Echo or Google Home as a bluetooth speaker, why not this?

First, run this to install all of the (originally difficult for me to find) dependencies

```
sudo apt-get install libdbus-glib-1-2 libdbus-glib-1-dev python3-gi python3-gst-1.0
sudo pip install dbus-python
```

Then, put this into your terminal
```
sudo hostnamectl --pretty set-hostname ""
```
and put what you want the speaker to appear as within the quotes, then run it

Then, run 
```
sudo nano /etc/bluetooth/main.conf
```
and go to the ```#DiscoverableTimeout``` line. Remove the #, and set it to ```DiscoverableTimeout = 30```

Then ```CTRL+X, Y, ENTER``` to save and exit.

Now, run ```sudo reboot now``` to reboot and apply these changes.

Once you're back in, run 
```
cd ~/assistant/
sudo curl -O https://raw.githubusercontent.com/elwint/bt-audio/master/bt-audio.py
```

Now, run this:
```
sudo apt install pulseaudio-module-bluetooth
```

Then, run 
```
sudo nano ~/assistant/profiles/intentHandler
```
and go to near the bottom, where you'll add another elif statement:
```
elif intent == "BluetoothPairing":
    speech("Turning on bluetooth pairing")
    with open(bluetoothFilePath, "w") as bluetoothFile:
        time.sleep(1)
        os.remove(bluetoothFilePath)
```

Then, go to the ```# Set paths``` section at the top, and add 
```
bluetoothFilePath = workingDir+"bluetoothFile"
```
Save and exit.

Now, run
```
sudo nano /etc/systemd/system/speakerbluetoothpair.service
```
and paste in
```
[Unit]
Description=Starts bluetooth pairing script

[Service]
Type=oneshot
ExecStart=/home/assistant-main-node/assistant/bt-audio.py
```
(replace assistant-main-node with your username if different).

Save and exit.

Then, make practically the same thing by running:
```
sudo nano /etc/systemd/system/speakerbluetoothpairstop.service
```
and pasting:
```
[Unit]
Description=Stops bluetooth pairing script after 30s

[Service]
Type=oneshot
ExecStart=/home/assistant-main-node/assistant/stop-bluetooth-pairing.sh
```
(replace assistant-main-node with your username if different).

Then, run
```
sudo nano /etc/systemd/system/speakerbluetoothpair.path
```
And paste in:
```
[Unit]
Description=Checks for bluetooth pairing file from rhasspy to start pairing

[Path]
PathExists=/home/assistant-main-node/assistant/profiles/bluetoothFile

[Install]
WantedBy=multi-user.target
```

And again run
```
sudo nano /etc/systemd/system/speakerbluetoothpairstop.path
```
And paste in:
```
[Unit]
Description=Checks for bluetooth pairing file from rhasspy to stop pairing

[Path]
PathExists=/home/assistant-main-node/assistant/profiles/bluetoothFile

[Install]
WantedBy=multi-user.target
```


CTRL+X+Y to save and exit.

Run
```
sudo nano ~/assistant/stop-bluetooth-pairing.sh
```
and add
```
sleep 30
systemctl stop speakerbluetoothpair.service
systemctl reset-failed speakerbluetoothpair.service
```

Now, run 
```
sudo chmod +x ~/assistant/stop-bluetooth-pairing.sh
sudo chmod +x ~/assistant/bt-audio.py
sudo systemctl enable speakerbluetoothpair.path
sudo systemctl enable speakerbluetoothpairstop.path
```

Then, go to your rhasspy sentences section, and paste this at the bottom:
```
[BluetoothPairing]
turn on bluetooth [pairing]
```

```sudo reboot now``` to reboot.


**It's a mess, but it works.**

Except for if you repair your phone. It likely won't let you re-pair.

To fix that, there's no elegant solution right now. Open the terminal, run ```bluetoothctl```, then type ```remove ```, press tab, and it'll either fill something in, or give you a list of options. If it fills something in, just press enter and you're done. If you've got a list, type the first letter of one, press tab, then enter, and do that for each item in the list.

### Optimal.

## Natural and varied responses

Right now, the assistant will always respond in the same way to a given request. This is easy to program, but not very natural, and we can make it better.

If you were paying attention to the **greetings** section, you'll probably understand how we'll implement this. All we need is a list of potential appropriate phrases, and then we pick a random one to speak each time.

First, we need to figure out what we want it to say, and which possible situations they're appropriate for. For example, after turning on a light, it would make sense to preface "I'll turn it on" with **Ok / Alright / Will do / Got it / Sure**, but it wouldn't make sense to have that same list of words when answering a maths question.

So, get into your intentHandler:
```
sudo nano ~/assistant/profiles/intentHandler
```
and go to just below your ```# Set paths``` section.

Add a new section called ```# Set responses```. Then, add your responses below - here's how you would add the example from before:
```
agreeResponse = ["Okay, ", "Alright, ", "Will do. ", "Got it, ", "Sure, "] 
```
In this case, I want a small pause after the phrase, so I've added a comma and a space within the quotes for all of them, and used a comma after the parenthesis to separate each one. 

Now, I can go down to the ```SetSpecificLight``` sections, and change this:
```
speech("Alright, I'll make it " + colour)
```
to this (doing the same change for all light sections):
```
speech(random.choice(agreeResponse) + "I'll make it " + colour)
```
All we've actually done is make it pick a random string from the list we made instead of just saying **"Alright, "**. If we just saved and exited here, it would work.

### But we can add it to other situations, like the time and weather

Back in the ```# Set responses``` section, I've added this line:
```
currentlyResponse = ["Right now it's ", "Its ", "Currently its ", "At the moment its "]
```
Then, in the ```GetTime``` and ```GetWeather``` sections, we can replace the ```"Its "``` with
```
random.choice(currentlyResponse)
```


### We could also implement different options for individual responses. 

For example, when cancelling a timer, I could add:
```
timerCancelResponse = ["Timer cancelled", "Cancelling timer", "I'll cancel the timer"]
```
and then just set it to pick one of those:
```
speech(random.choice(timerCancelResponse))
```

### Or different ways of saying AM / PM

We can add ```morningResponse = [" in the morning", " ey em"]``` and then ```eveningResponse = [" in the afternoon", " in the evening", " peey em"]```

**(remember from the timer bit, AM and PM are spelt weirdly so that they're spoken correctly with my TTS choice. They might need changing for whatever voice you choose personally)**

Now, we go down to the ```if intent == "GetTime":``` section, and make the a/pm variable sections with:
```
apm = random.choice(eveningResponse)
```
then the next one is
```
apm = random.choice(morningResponse)
```
and now your time announcements should be a bit more varied than before.

The way we're doing this is really simple and flexible, but makes the responses less repetitive. I like it.

## Converting units

First, add a slot file called "units", and paste this in:
```
kilometres
metres
centimetres
millimetres

miles
feet
inches
yards


tons
(kilograms | kilos):kilograms
grams

ounces
pounds
stones


litres
millilitres
centilitres

fluid ounces
```
I've separated the "types" of units (length, mass, volume) with two lines, and the metric / imperial units by one line. These are all of the units that I thought of immediately that could be useful, but if you ever need anything else, you could just add it here.


Now, go to the sentences section, and paste this:
```
[UnitConversion]
unit = ($units)
(whats | convert) (1..1000){number} <unit>{unit1} (to | in) <unit>{unit2}
```

All that's left is the awkward bit, coding the conversions. To simplify things, we'll be converting into an intermediary unit. So, rather than having to separately code the conversions between kilograms and tons, then grams and ounces, etc for every unit of mass, we can instead just turn every unit1 into kilograms, then go from kilograms to unit2.

The code isn't nice, and it's massive. It works though. Paste this elif statement into your intentHandler:
```
elif intent == "UnitConversion":
    number, unit1, unit2 = float(o["slots"]["number"]), o["slots"]["unit1"], o["slots"]["unit2"]
# Length
    if unit1 == "metres":
        toBeConverted = number
    elif unit1 == "kilometres":
        toBeConverted = number*1000
    elif unit1 == "centimetres":
        toBeConverted = number/100
    elif unit1 == "millimetres":
        toBeConverted = number/1000
    elif unit1 == "miles":
        toBeConverted = number*1609.344
    elif unit1 == "feet":
        toBeConverted = number*0.3048
    elif unit1 == "inches":
        toBeConverted = number*0.0254
    elif unit1 == "yards":
        toBeConverted = number*0.9144

# Mass
    elif unit1 == "kilograms":
        toBeConverted = number
    elif unit1 == "tons":
        toBeConverted = number*1000
    elif unit1 == "grams":
        toBeConverted = number/1000
    elif unit1 == "ounces":
        toBeConverted = number*0.02834
    elif unit1 == "pounds":
        toBeConverted = number*0.4535
    elif unit1 == "stones":
        toBeConverted = number*6.35029

# Volume
    elif unit1 == "litres":
        toBeConverted = number
    elif unit1 == "millilitres":
        toBeConverted = number/1000
    elif unit1 == "centilitres":
        toBeConverted = number/100
    elif unit1 == "fluid ounces":
        toBeConverted = number*0.02841

# Doing the conversion
    if unit2 == "kilometres":
        finalValue = toBeConverted/1000
    elif unit2 == "metres":
        finalValue = toBeConverted
    elif unit2 == "centimetres":
        finalValue = toBeConverted*100
    elif unit2 == "millimetres":
        finalValue = toBeConverted*1000
    elif unit2 == "miles":
        finalValue = toBeConverted/1609.344
    elif unit2 == "feet":
        finalValue = toBeConverted*3.28084
    elif unit2 == "inches":
        finalValue = toBeConverted*39.37
    elif unit2 == "yards":
        finalValue = toBeConverted*1.093613

    elif unit2 == "tons":
        finalValue = toBeConverted/1000
    elif unit2 == "kilograms":
        finalValue = toBeConverted
    elif unit2 == "grams":
        finalValue = toBeConverted*1000
    elif unit2 == "ounces":
        finalValue = toBeConverted*35.27396
    elif unit2 == "pounds":
        finalValue = toBeConverted*2.2046
    elif unit2 == "stones":
        finalValue = toBeConverted*0.15747

    elif unit2 == "litres":
        finalValue = toBeConverted
    elif unit2 == "millilitres":
        finalValue = toBeConverted*1000
    elif unit2 == "centilitres":
        finalValue = toBeConverted*100
    elif unit2 == "fluid ounces":
        finalValue = toBeConverted*35.195

    speech(str(number) + " " + unit1 + " is " + str(round(finalValue,3)).replace("." , " point ") + " " + unit2)
```

# Adding satellites
But you might want more than one smart speaker, and setting them up as full instances seems somewhat wasteful when we've got a perfectly good CPU on our main node. Why not use the power of *that* node to manage ***all*** speakers?

## Raspberry Pi 2/3

Just so you know, using a Pi 3 for this is huge excess, and even a Pi 2 is a bit much, I'm just using it here because I already had one to test on. We don't need something this powerful.

### Flashing with Raspbian

This is just like with the main node, however this time we can use 32-bit Pi OS Lite, since we don't need to take advantage of all of the power of the Pi, and this allows it to run on the Pi 2 too.

![Selecting Pi OS 32 bit lite](https://github.com/IssacDowling/SelfhostedSmartHomeGuide/blob/0eef6842d9a259c6076ee01c0aae0d80759c0078/images/Raspbian%20%2032bit%20lite.png)

Just remember that the Pi 2 and earlier do not have Wifi, so don't set that up, you'll be relying on Ethernet, or a separately set up USB Wifi dongle. In my case, for initial setup, I'll just use ethernet. I'm going to make my username/hostname **assistant-secondary-node** to go along with my **assistant-main-node** from before, however in the future I would probably name these based on their rooms/positions instead.

### Installing software

Once you've flashed the SD card, insert it, boot the Pi, and - in your terminal - ssh into your new hostname. So, I would run
```
ssh assistant-secondary-node@assistant-secondary-node.local
```
to get in.

Then, run an update like this:
```
sudo apt update && sudo apt upgrade -y
```
and then install docker just like before, by pasting these lines:
```
curl -sSL https://get.docker.com | sh
sudo apt-get install -y uidmap libffi-dev libssl-dev python3 python3-pip python3-dev
sudo pip3 install docker-compose
sudo gpasswd -a $USER docker
```
and also get Rhasspy ready by running
```
mkdir assistant
cd assistant
nano docker-compose.yml
```
and pasting the following:
```
version: '3.3'
services:
    rhasspy:
        container_name: rhasspy
        ports:
            - '12101:12101'
            - '12183:12183'
        volumes:
            - './profiles:/profiles'
            - '/etc/localtime:/etc/localtime:ro'
        devices:
            - '/dev/snd:/dev/snd'
        image: rhasspy/rhasspy
        command: --user-profiles /profiles --profile en
        restart: unless-stopped
```
Then do CTRL+X, Y, ENTER to save, and run:
```
sudo docker-compose up -d
```
to start the install. Just like before, it's done once you see this:
![Done docker](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/donedocker.png)

### Setting a static IP

And again, as before, we'll want to set a static IP. I'm going to copy and paste the section from earlier.

In your terminal, type ```ip route | grep default```. Then, note down three things: the first IP, the network device, and the second IP. The IPs will likely be 192.168.x.x, but may be different. In the image, I've highlighted these things in yellow so you know where to look.

![my private IP for the Pi](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/ipdefault.png)

Now, run
```
sudo nano /etc/dhcpcd.conf
```
and navigate down to the bottom line using the arrow keys, then press enter a few times to add some lines. You should get to here:

![dhcpcd](https://github.com/IssacDowling/SelfhostedVoiceAssistantGuide/blob/main/images/dhcpcd.png)

Then, paste this in:

```
interface
static ip_address=/24
static routers=
static domain_name_servers=1.1.1.1
```

Next to interface, add a space, then the network device (either **eth0** or **wlan0**). Now, for static ip_address, type the second IP before the */24*. Finally, add the first IP from earlier directly after **static routers=**. Then, press CTRL+X, then Y, then Enter, to save and exit. Finally, run ```sudo reboot``` to restart. Your SSH will disconnect, and you can just keep trying to reconnect until it works to check if you're booted.

### Setting it up

First, go to your main node's settings page, and set the **"siteId"** to something other than default. I picked **"main-node"**. Next, do the same with your secondary node, calling it something else. I picked **"secondary-node"**. Remember to save your settings.


Next, set audio recording to pyaudio, wakeword to porcupine, audio recording to aplay, disable intent handling, set dialogue management to Rhasspy, and then everything else to Remote HTTP.

MQTT would be better, but I haven't figured it all out yet.

Now, 

# Resources
There's a folder called resources in this git repo. It contains any files of mine (or somebody else's, if they're ok with it) that you might want. Any API keys or related stuff in code will be blocked out, however they're otherwise unmodified.








# Extras
[Rhasspy Documentation](https://rhasspy.readthedocs.io)

[Homeassistant Documentation](https://www.home-assistant.io/docs/)
