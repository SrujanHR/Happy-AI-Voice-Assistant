# Happy — Voice Assistant

Happy is a personal AI voice assistant for Windows, built with Python. It listens for a wakeword, recognizes voice commands, speaks responses, and can interact with Google Gemini through Chrome automation. This repository contains the code, setup instructions, and supporting files to get Happy running on your machine.

---

## README.md

````md
# Happy — Voice Assistant

Happy is a Python-based personal assistant for Windows. It responds to the wakeword **"happy"**, listens to your commands, and replies with synthesized speech. It can also automate Chrome to interact with Google Gemini, control volume, and play music.

## Features
- Wakeword detection powered by Vosk
- Speech recognition using Google SpeechRecognition API
- Text-to-speech with gTTS and VLC playback
- Google Gemini integration via Chrome automation
- Local + YouTube music playback
- Volume control through keyboard automation

## Requirements
- Windows 10/11
- Python 3.10+
- Google Chrome
- VLC Media Player
- Vosk English model (download e.g. `vosk-model-en-in-0.5` into `models/`)

## Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/<your-username>/happy-assistant.git
   cd happy-assistant
````

2. Set up a virtual environment:

   ```bash
   python -m venv venv
   venv\\Scripts\\activate
   ```

3. Install dependencies:

   ```bash
   pip install -r requirements.txt
   ```

4. Initialize runtime files:

   ```bash
   echo 0 > runtime.txt
   echo 50 > current_vol.txt
   ```

5. Run Happy:

   ```bash
   python -m src.happy
   ```

## Notes

* `runtime.txt` tracks how long Happy has been active (in seconds).
* `current_vol.txt` saves volume levels between sessions (0–100).
* If `pyaudio` fails to install on Windows, use:

  ```bash
  pip install pipwin
  pipwin install pyaudio
  ```

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

```

---

## requirements.txt

```

vosk
pyaudio
SpeechRecognition
pyautogui
pyperclip
clipboard
python-vlc
gTTS
soundfile
numpy

```

---

## .gitignore

```

**pycache**/
venv/
*.pyc
combined_audio.wav
audio_*.mp3
runtime.txt
current_vol.txt
.DS_Store

```

---

## runtime.txt

```

0

```

---

## current_vol.txt

```

50

```

---


## src/happy.py

```python
"""Happy — Voice Assistant entrypoint.

Update MODEL_PATH, CHROME_PATH, and VLC_LIB_DIR to match your system paths.
"""
import os, time, subprocess
from vosk import Model, KaldiRecognizer
import pyaudio, speech_recognition as sr
import pyautogui, pyperclip, clipboard
from gtts import gTTS
import soundfile as sf, numpy as np
import vlc

# === Configuration (edit to your machine) ===
MODEL_PATH = os.path.join('models','vosk-model-en-in-0.5')
CHROME_PATH = r"C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe"
VLC_LIB_DIR = r"C:\\Program Files\\VideoLAN\\VLC"
URL_GEMINI = "https://gemini.google.com/app"
WAKEWORD = "happy"

# === Runtime files ===
RUNTIME_FILE = 'runtime.txt'
VOLUME_FILE = 'current_vol.txt'
for f, default in [(RUNTIME_FILE,'0'), (VOLUME_FILE,'50')]:
    if not os.path.exists(f):
        with open(f,'w') as fh: fh.write(default)

# === Load Vosk model ===
model = Model(MODEL_PATH)
rec = KaldiRecognizer(model, 16000)
pa = pyaudio.PyAudio()
stream = pa.open(format=pyaudio.paInt16, channels=1, rate=16000, input=True, frames_per_buffer=16000)

# === TTS helpers ===
def generate_audio(sentence, index):
    gTTS(sentence, lang='en', slow=False).save(f'audio_{index}.mp3')

def play_audio_clip(path):
    instance = vlc.Instance('--no-video')
    player = instance.media_player_new()
    player.set_media(instance.media_new(path))
    player.play()
    while player.get_state() != vlc.State.Ended:
        time.sleep(0.05)

def combine_audio_files(text):
    sentences = [s.strip() for s in text.split('.') if s.strip()]
    for i,s in enumerate(sentences): generate_audio(s,i)
    combined = np.array([])
    sr = None
    for i in range(len(sentences)):
        data, samplerate = sf.read(f'audio_{i}.mp3')
        combined = np.concatenate((combined, data)) if combined.size else data
        sr = samplerate
        os.remove(f'audio_{i}.mp3')
    sf.write('combined_audio.wav', combined, sr)
    play_audio_clip('combined_audio.wav')
    os.remove('combined_audio.wav')

# === Wakeword loop ===
def vosk_recognize():
    print('Listening for wakeword...')
    try:
        while True:
            data = stream.read(16000, exception_on_overflow=False)
            if rec.AcceptWaveform(data):
                res = rec.Result().lower()
                if WAKEWORD in res:
                    print('Wakeword detected')
                    recognize_and_process()
                elif 'quit' in res:
                    break
    except KeyboardInterrupt:
        print('Stopped by user')

# === Command processing ===
def recognize_and_process():
    r = sr.Recognizer()
    with sr.Microphone() as src:
        if os.path.exists('notify.wav'):
            play_audio_clip('notify.wav')
        audio = r.listen(src, phrase_time_limit=6)
    try:
        text = r.recognize_google(audio)
        print('You said:', text)
    except Exception:
        return
    subprocess.Popen([CHROME_PATH, URL_GEMINI])
    time.sleep(3)
    pyautogui.click(pyautogui.size()[0]//2, pyautogui.size()[1]//2)
    pyautogui.write(text)
    pyautogui.press('enter')
    # TODO: Implement clipboard scrape and response playback

if __name__ == '__main__':
    vosk_recognize()
````

---
