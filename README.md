# Audio-Reactive Polygon with Dorothy
It listens to live music playback and translates sound features (like loudness and bass energy) into changes in a dynamic polygon drawn on screen. The main idea is to connect sound analysis with generative graphics in real time.

At the start, I only had a blank Dorothy window — no music playback, no visuals, just a static screen.
From there, I focused on building the core that makes the app work:
	1.	Load a music file.
	2.	Read its audio data (amplitude and frequency).
	3.	Use those values to animate a shape on screen.
Everything else in the code supports those three steps.

after imports and page set up and the first major function I wrote was:
'''def find_first_track():
    # scans MUSIC_DIR, returns first .mp3 or None with clear prints

This looks inside a folder of songs (/Users/stonesavage/Desktop/Coding for media/data/MP3s) and finds the first .mp3 file.
Once found, the setup() function plays the track using Dorothy’s built-in audio player.



https://github.com/user-attachments/assets/ad14842c-d353-4e53-bfd2-5f6bc7b4ae8a

Song: Like sugar, by Chaka Khan.
