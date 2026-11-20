# macscreen

Record mic audio and external monitor on mac with SoX and ffmpeg.

Status (19th November, 2026): Not production ready.


## The Problem

It's not possibly to record audio of satisfactory quality using ffmpeg on my Mac - there are annoying crackles in the recording every few seconds. I want to use ffmpeg to record the desktop _and_ the microphone and the poor audio quality means that I cannot use ffmpeg directly to capture audio.

Trying

```bash
ffmpeg -f avfoundation -list_devices true -i "" # use ot confirm mic is input ":0" 
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

So we use this string to tell sox which microphone to record

```bash
sox -t coreaudio "MacBook Neo Microphone" mic.wav
```
Note: it is _really annoying_ that Apple's CoreAudio identifies the mainboard mic using a model-specific string, rather than, say, the digit 0 for all Macs, but it is what it is. 

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
