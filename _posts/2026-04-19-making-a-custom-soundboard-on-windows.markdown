---
layout: post
title: "Making a custom soundboard on Windows"
date: 2026-04-19 19:15:59 +08:00
categories: audio tutorial vac voicemeeter
published: true
---

My fascination with soundboards for a bit of fun started with Half-Life DJ around a decade ago, where I would trawl around CS:GO with various sound effects that I'm sure weren't grating to anyone else in the server. Sometimes you would come across another player with HLDJ installed and the etiquette of the time was to outdo one another by unleashing the larger (or louder) torrent of memes than the other person. HLDJ worked entirely within the console of CS:GO and you could simply list the directory of sounds you have saved from there, and play them too. Some time later, SLAM (Source Live Audio Mixer) would allow you to do the same thing, came with a nice external GUI interface and keybind support. I always preferred HLDJ, however, since I never had to leave the game at any point and I could play all of my sounds without needing keybinds.

Ever since then, I've wanted that functionality, but everywhere. It was sure fun back in the day, but limited to source engine games with SLAM. It turns out you can do this, of course, but the available tutorials on the internet leave much to be desired, so I figured I'd write a (probably worse) tutorial of my own, for my specific use-case, so that I don't forget the process ever again. Maybe someone else will find this useful too, who knows? (who cares?)

# Framing the problem
Okay, so we specifically want the following scenario:
- Normal system sounds should be played through my primary audio device (headphones).
- My microphone should not be played through my headphones, I don't want to hear my own voice.
- My soundboard effects should play both through my microphone, and through my headphones.

Easy, as it turns out, there's a program that allows us to do this: [Voicemeeter Banana](https://vb-audio.com/Voicemeeter/banana.htm). It's "an Advanced Audio Mixer Application", meaning that we can mix hardware audio devices with virtual (software) audio devices fairly freely. Note that Voicemeeter Banana also comes with it's own (admittedly overcomplicated) soundboard support so I'll be using that instead of an external program, but I will show that setup, too.

<!-- Check if VAC can be uninstalled!-->

Install Voicemeeter Banana, restart your computer and let's get started.

## Preparation
You may have lost audio at this point, let's fix that real quick. Open the "Sound Control Panel". For me, I can open this in the taskbar using the sound icon:
<figure>
  <img src="/assets/images/soundboard/taskbar.png" style="margin: auto">
  <figcaption style="text-align: right">Opening the sound control panel from the taskbar</figcaption>
</figure>

Alternatively, you can also navigate to **System->Sound(s)->Sound Control Panel** to open the menu as well:
<figure>
  <img src="/assets/images/soundboard/sound_settings_control_panel.png">
  <figcaption style="text-align: right">Opening the sound control panel from the sound settings</figcaption>
</figure>

After that, you can set your microphone to be both the default device and the default communication device by right clicking it and selecting those options:
<figure>
  <img src="/assets/images/soundboard/control_panel_set_default_recording.png" style="margin: auto">
  <figcaption style="text-align: right">Setting my physical microphone as the default recording device</figcaption>
</figure>

You should also reset the default playback device, too, while you're at it, so both input and output sounds work again:
<figure>
  <img src="/assets/images/soundboard/control_panel_set_default_playback.png" style="margin: auto">
  <figcaption style="text-align: right">Setting my default headphones as the default playback device</figcaption>
</figure>

Now that we've fixed that issue, keep this menu in mind, because we'll need it again later. You should have noticed the various voicemeeter input and voicemeeter out devices that now exist. They'll be instrumental later on. Since we'll need to mix which sounds output to our headphones, we'll need the voicemeeter input playback device. Since we need to mix which sounds are played through our microphone, we'll need the voicemeeter output device.

## The Voicemeeter UI
It's time to understand the voicemeeter UI, at least a little bit. A little side-note, make sure you've opened Voicemeeter Banana, not base voicemeeter, as the banana install also comes with the standard version. If you've opened the wrong version, fully close voicemeeter (from taskbar if need be) and ensure that you open the version which says *Banana* in yellow at the top-left of the window.

The image below shows all the important buttons that we'll make use of for this tutorial:
<figure>
  <img src="/assets/images/soundboard/voicemeeter_ui.png" style="margin: auto">
  <figcaption style="text-align: right">The voicemeeter UI with important buttons highlighted</figcaption>
</figure>

- **Stereo Input 1** is where we will enter our microphone device for mixing. For this tutorial, stereo input 2 and stereo input 3 will not be needed.
- The buttons below labelled **A1-A3,B1,B2** are our *buses*, and they determine whether a sound is forwarded into that "lane" on the master track, a.k.a the final output of the mixing process.
- The virtual inputs **Voicemeeter Input** and **Voicemeeter AUX I(n)** will be the source of our system sounds and soundboard sounds, respectively. This is essentially how we will play soundboard effects without accidentally mixing in system audio. Note that this is only needed if you're using an external soundboard program, because the internal voicemeeter soundboard functionality can bypass the need to use the auxiliary input.
- The **A1 Box** is where we will output our sound to our normal playback device, in this case my headphones.
- **A1** and **B1** in the master section represent what **we** hear, and what **others** hear, respectively
- The *yellow* underscored **Physical** and **Virtual** labels in our case represent mixing that is output to the playback device, and merging of multiple sources into a recording device, respectively

That's it, let's set it all up!

## Setting it all up
Note that you will lose sound again, at which point you will set your default playback device to **Voicemeeter Input** and your default recording device to **Voicemeeter Out B1**.

1. Set **Stereo Input 1** to your microphone.
2. In the *buses* underneath **Stereo Input 1**, only enable **B1**
3. Disable the *buses* for **Stereo Inputs 2 and 3**.
4. For **Voicemeeter Input**, only enable bus **A1**.
5. For **Voicemeeter AUX I(n)**, enable *buses* **A1** and **B1**, so that we both hear our soundboard and send it to our microphone.
6. In the **Hardware Out** section, set the **A1 Box** to your normal playback device.

Your UI should look mostly similar to this (except for the casette buttons that I've configured, the **intellipan** square in **Stereo Input 1** and some device names):
<figure>
  <img src="/assets/images/soundboard/voicemeeter_ui_configured.png" style="margin: auto">
  <figcaption style="text-align: right">The configured voicemeeter UI</figcaption>
</figure>

That should be all you need to do to configure the correct mixing/merging. As long as your programs are set to use the default windows audio device, you should be able to hear again once you change your default playback/recording devices to **Voicemeeter Input** and **Voicemeeter Out B1**.

If you play some sounds, like a video or song, you should notice that the amplitude of the **A1** master track should jump up in unison with the sounds, whereas **B1** will not jump up, and the converse is true when speaking into the microphone. Hopefully this is enough proof that we have separated our system sounds from our microphone input.

Let's use VLC as our example "soundboard" program.

Open VLC, and give it an audio file to play. I'm going to use the vine boom 💀 sound effect.
Once the sound file is loaded, change the audio device to use **Voicemeeter AUX Input**, which, if you remember, we designated earlier to be used for our soundboard.

<figure>
  <img src="/assets/images/soundboard/vlc_device_configuration.png" style="margin: auto">
  <figcaption style="text-align: right">VLC Configured to work as a soundboard</figcaption>
</figure>

Hitting play now should show the **A1** and **B1** tracks in voicemeeter jumping in unison, and you should hear the sound effect that you played. That means you're done.

## Going a bit further
Everything else in this tutorial was getting it working. This section is for getting it working *nicely*. Note that the MacroButtons configuration below is an *alternative* to something like VLC, not complementary (although you could use both, if you're ambitious like that).

Click the **Menu** button in voicemeeter and then enable the options: **System Tray**, **Run on Windows Startup** and **Run MacroButtons on Voicemeeter start**. That final button should open a window called **Voicemeeter.Remote - Macro.Buttons**, or something like that (if not, just search for **macro buttons** in the windows start menu). We can use these macro buttons as a soundboard. Let's save our configuration: In **Menu** press **Save Settings...**, give the XML file a good name and then press **Load Settings on Startup:**, selecting the file you just made.

<figure>
  <img src="/assets/images/soundboard/macrobuttons_ui.png" style="margin: auto">
  <figcaption style="text-align: right">The MacroButtons UI</figcaption>
</figure>

## Macro Buttons
I'm only going to configure one button for the vine boom 💀 sound effect, but I'll touch on anything important from there.

Right click on a button to begin configuring it, this should present you with the UI below:

<figure>
  <img src="/assets/images/soundboard/macrobuttons_button_ui.png" style="margin: auto">
  <figcaption style="text-align: right">The UI for configuring a button in MacroButtons, with important fields highlighted</figcaption>
</figure>

Let's configure our button so that if we click it, or press **Numpad 0** our sound effect will play:
<figure>
  <img src="/assets/images/soundboard/macrobuttons_button_push.png" style="margin: auto">
  <figcaption style="text-align: right">Our configured soundboard button</figcaption>
</figure>

Pressing **OK** at the bottom of the menu will create the button, which should work, but won't function as we want it to just yet.

What if the sound effect was 10 minutes long? Every time we press the button, it will start playing the sound effect again, i.e. we can't actually *stop* playing the sound unless we wait for it to finish. An easy solution is to set our button type to **2 Position**, and make the **Button OFF** trigger stop our sound effect:
<figure>
  <img src="/assets/images/soundboard/macrobuttons_button_2position.png" style="margin: auto">
  <figcaption style="text-align: right">Our configured 2-position soundboard button</figcaption>
</figure>

Now if we click the vine boom 💀 button twice in quick succession, the sound starts and stops, restarting each time. Perfect! You could potentially add a toggle for *recorder.ff* (fast-forward, 1 is enabled, 0 is disabled) if you wanted a way to speed through tracks, too. Lovely, let's save our configuration. Right click on the Title bar at the top of the window and press **Save Button Map**, giving the XML file a good name. I also like to enable **System Tray (Close = Hide)** in the same menu.

Moving back to Voicemeeter, we can make our soundboard work as expected by selecting the correct tracks by the casette player, so that our headphones and microphone receive the soundboard effects.
<figure>
  <img src="/assets/images/soundboard/voicemeeter_casette_configured.png" style="margin: auto">
  <figcaption style="text-align: right">Voicemeeter, highlighting our configured casette player</figcaption>
</figure>

If you **right-click** the Casette, you can configure the **Recorder Options**. Here is where you can disable some unneeded functionality, ensure soundboard buttons work correctly, and adjust the volume of soundboard clips:

<figure>
  <img src="/assets/images/soundboard/voicemeeter_recorder_options_configured.png" style="margin: auto">
  <figcaption style="text-align: right">A configured example of Voicemeeter's recorder options</figcaption>
</figure>

The figure above shows my configuration for recorder options. I've disabled all the **I/O arming** inputs, I've set the **Playback Gain** to **-15.2 dB**, and I've ensured that **Play On Load** is set to **Yes**.

That's enough for this tutorial. Keep in mind if you don't like the MacroButtons workflow, nor the VLC workflow, you can find another soundboard program and tell it to output it's audio to the **Voicemeeter AUX Input** playback device, which should already integrate perfectly with our setup.
