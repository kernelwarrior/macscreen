# macscreen

Record mic audio and external monitor on mac with SoX and ffmpeg.

Status (19th November, 2026): Not production ready.


## The Problem

It's not possibly to record audio of satisfactory quality using ffmpeg on my Mac - there are annoying crackles in the recording every few seconds. I want to use ffmpeg to record the desktop _and_ the microphone and the poor audio quality means that I cannot use ffmpeg directly to capture audio.

Trying

```bash
ffmpeg -f avfoundation -list_devices true -i "" # use to confirm mic is input ":0" 
ffmpeg -f avfoundation -i ":0"  -c:a  aac_at -global_quality:a 0 -aac_at_quality 0 -aac_at_mode vbr -q:a 14  -y mic.m4a
```
results in "stuttering" where the audio disappears briefly maybe once every second or so - not even cloe to acceptable quality 

### Recording audio

However, SoX will produce good quality audio. First of all, we need to find out what microphone to use. In the below scripts, I am using the bash shell, as installed with homebrew.


```bash
#!/opt/homebrew/bin//bash
src="$(system_profiler SPAudioDataType | grep -m 2 Microphone | tail -n 1| cut -d : -f2 | sed 's/^[[:space:]]*//')"
echo "$src"
```

This will print, e.g., "MacBook Neo Microphone" if your machine is a Neo.

Alternatively,

```bash
src="$(ox -V -t coreaudio null -n 2>&1 | grep "found audio" | cut -d"'" -f2 | grep Mac)"
echo "$src"
```

So we use this string to tell sox which microphone to record

```bash
sox -t coreaudio "MacBook Neo Microphone" mic.wav
```
Note: it is _really annoying_ that when picking the audio input in soxApple's CoreAudio identifies the mainboard mic using a model-specific string, rather than, say, the digit 0 for all Macs, but it is what it is. 

And this worked to record crackle-free audio with sox, and pipe it through to gffmpeg to convert to mp3.
```bash
sox -t coreaudio "MacBook Neo Microphone" -t wav - | ffmpeg -i - -c:a libmp3lame mic.mp3
```
and this creates an m4a file.
```bash
sox -t coreaudio "MacBook Neo Microphone" -t wav - | ffmpeg -i - -c:a aac -b:a 192k mic.m4a
```

### Recording Video

This is straightforward as per the usual instructions. First figure out which display to record.

```bash
ffprobe -hide_banner -f avfoundation -list_devices true -i - 2> >( grep -i screen ) | sed 's/\[AV.*\] \[/\[/'
```
This will return a result such as

```bash
[1] Capture screen 0
[2] Capture screen 1
```
so there are two displays, and the numbers in brackets are the ones to use below ([0] typically refers to the builtin webcam on a MacBook).

Now I can record the external display as follows (replace '2:none' with '1:none' to record the native display).

```bash
ffmpeg -f avfoundation -framerate 30 -i "2:none" -c:v libx264 -pix_fmt yuv420p screen.mp4
```


## Combining Audio and Video

Let's try

```bash
sox -t coreaudio "MacBook Neo Microphone" -t wav - | ffmpeg -i -  -f avfoundation -framerate 30 -i "2:none" -map 0:a:0 -map 1:v:0 -c:a aac -b:a 192k  -c:v libx264 -pix_fmt yuv420p mic_screen.mp4
```

This worked except that audio was being dropped intermittently and sox often reported that

```bash
sox WARN coreaudio: unhandled buffer overrun. Data discarded.
```
Tried increasing the buffer from the default 8192 but even 64k didn't help.

```bash
buf=65536; sox -t coreaudio "MacBook Neo Microphone" -b $buf -t wav - | ffmpeg -i -  -f avfoundation -framerate 30 -i "2:none" -map 0:a:0 -map 1:v:0 -c:a aac -b:a 192k  -c:v libx264 -pix_fmt yuv420p -y mic_screen.mp4
```

The overruns don't happen if only audio is being recorded. CPU load in Activity monitor is showing at about 47% when  recording audio and screen, and GPU % is negligible so looks like there is no hardware acceleration on the video compression. So tried


```bash
sox -t coreaudio "MacBook Neo Microphone" -b $buf -t wav - | ffmpeg -i -  -f avfoundation -framerate 30 -i "2:none" -map 0:a:0 -map 1:v:0 -c:a aac -b:a 192k  -c:v h264_videotoolbox -pix_fmt yuv420p -y mic_screen.mp4
```

This lowered the CPU % to about 10% but there were still overruns, mostly in the first ten seconds of recording. Tried using the Apple codec for audio but no improvement. Changing the buffer size did not seem to help.


```bash
buf=4096; sox -t coreaudio "MacBook Neo Microphone" -b $buf -t wav - | ffmpeg -i -  -f avfoundation -framerate 30 -i "2:none" -map 0:a:0 -map 1:v:0 -c:a aac_at -global_quality:a 0 -aac_at_quality 0 -aac_at_mode vbr -q:a 14   -c:v h264_videotoolbox -pix_fmt yuv420p -y mic_screen.mp4 
```
Setting the quality of aac_at in ffmpeg is poorly documented, to put it mildly.

### Next steps

I'll try recording mic with sox and screen with ffmpeg in concurrent processes and mux them afterwards. No lip-sync issues since no webcam used and if by end of recording both streams are within 0.5s or so of sync should be ok.

```bash
ffmpeg -f avfoundation -framerate 30 -i "1:" -c:v libx264 -pix_fmt yuv420p -y screen.mp4 &
 sox -t coreaudio "MacBook Neo Microphone" mic.wav &
```
This script didn't work because screen.mp4 is not created (the relevant line executes as expected if typed directly into the termoinal. Head scratching so will revisit when less bemused.


I suspect that the piping from sox to ffmpeg is the cause of the problem. So next thing to try is recording audio (with sox) and video (with ffmpeg) in separate processed, and muxing them afterwards. Video of screen only so no lip-sync issues will arise.

First attempt did not work - the ffmpeg process was running but producing no output. Suspected it was a mac permissions issue (I had given the terminal access to the screen to record it but this permission might not be inherited by processes it spawns).

This suspicion seemed to be confirmed whan a script which runs sox first in the background, and then runs ffmpeg in  the foreground produced vidoe of the screen. The audio was free of overruns as well. Result!

```bash
sox -t coreaudio "MacBook Neo Microphone" mic.wav & 
ffmpeg -f avfoundation -framerate 30 -i "1:" -c:v libx264 -pix_fmt yuv420p -y screen.mp4
```

Trying


```bash
sox -t coreaudio "MacBook Neo Microphone" mic.wav & 
ffmpeg -f avfoundation -framerate 30 -i "1:" -c:v h264_videotoolbox -pix_fmt yuv420p -y screen.mp4
```

That worked too. Compressing the audio to ogg also worked. I changed the pixel format from yuv420p to suppress a complaint from ffmpeg on my test device but it persisted so I dropped that. Specified main profile just for fun.

```bash
sox -t coreaudio "MacBook Neo Microphone" mic.ogg & 
ffmpeg -hide_banner -f avfoundation -framerate 30 -i "2:" -c:v h264_videotoolbox  -profile main  -y screen.mp4
```


### Muxing video and audio

```bash
ffmpeg -i mic.ogg -i screen.mp4 -c copy combo.mp4
```
This seemed to work ok - yet to test for being web-friendly

### Keyboard shortcut for ending recording

The recording can be ended by executing the command `killall sox ffmpeg`. This needs to be triggered by a keyboard shortcut so that the final few seconds of video do not feature a terminal where this is typed.

On the Mac, this is set up with Automator.

- Open Automator, choose Quick Action, Pick Utilities, then Run Shell Script
- Choose the options so that Workflow receives `no input` in `any application`
   - this means that the "workflow" works in any application and does not expect to read text
   - I initially forgot the 'no input' part and the script just refused to run outside Automator
- pick the shell (/bin/sh will do)
  - user-installed shells are not listed among the options so no recent bash is available
- set "Pass Input' to stdin
  - other choice is 'as arguments'
  - meaningless setting when you've already configured 'no input'
  - enter the command as `killall sox ffmpeg`
- File/Save and name it "Zap Recording"
  - saves with file extension .workflow somewhere
  - not at all clear that this saved the action or whether some other step was required

Now set up A Keyboard Shortcut as follows

- System Settings/Keyboard
- Click "Keyboard Shortcuts" button
- Select Services in the column on left
- The new Quick action is under "General"
  - if you neglected to configure it with "no input" it would be under "text"
- you set a new shortcut by selecting the action (Zap Recording in this case) and double clicking it somewhere/somehow until the existing shortcut icons turn into a text box
  - this was an almighty faff as there seemed to be no discernible logic to where/when to double click to create this prompt
- Enter the shortcut in the text box and click 'done'
  - I chose Opt|Sh|Com|\

You will not be warned if your chosen shortcut is already in use. As far as I could tell, any such conflict results in neither the new purpose of the shortcut nor the existing one surviving your click of the 'done' button. There is no way back except to select "Restore Defaults" which will erase any existing custom shortcuts. Yet another triumph of UI design by the esteemed Apple Inc.

If there is some error in your Quick Action configuration, typing the shortcut generates an error beep, but no other diagnostic information is supplied - this is especially frustrating when the Quick Action works inside Automator (when clicking the 'Run' button in the Editor.

Setting up the equivalent functionality to the above in Debian Linux is _way_ more straightforward.


### Running the record script from file finder

Copying the approach in https://stackoverflow.com/questions/56169040/run-bash-script-on-selected-files-in-finder
but running a shell script rather than an Apple script. Selecting that you want to pass the inputs as arguments results in the auto-generation of this code

```bash
for f in "$@"
do
	echo "$f"
done
```

Set the workflow to receive "PDF files" in "Finder.app". Now the name of the workflow appears under "Quick actions" when you right click a PDF file. The output of the echo statement above disappears into a black hole, as a terminal is not opened to run the script.

Can work around this using a clone of zenity

```bash
brew install ncruces/tap/zenity
```
_`brew install zenity` will install the X11 version, which is not what we need in this context_

Changed the script to
```bash
#!/opt/homebrew/bin//bash
for f in "$@"
do
	/opt/homebrew/bin/zenity --question --text "$f"
done
```

I had to allow some "runner" permission on the first run to allow the script to complete execution. The PATH variable for the script is `/usr/bin:/bin:/usr/sbin:/sbin`, and so any utility installed with homebrew must have an explicit path

Best strategy seems to be to invoke another script from this one. For the moment, I will leave it as

```bash
#!/opt/homebrew/bin//bash
file=$(basename "$1")
name="${file%.*}"
extension="${file##*.}"
/opt/homebrew/bin/zenity --question --text " Record '$file'?"
exit 0 # otherwise Automator will report an error if you answer 'N'
```



