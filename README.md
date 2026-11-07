[jarvis (3).py](https://github.com/user-attachments/files/34522062/jarvis.3.py)
# Jarvis-full
Put this into Claude and ask it to make it with this code
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
JARVIS 2.0  -  one file, runs on a Mac with plain Python.

  python3 jarvis.py             open the Jarvis window
  python3 jarvis.py --text      text-only mode in Terminal
  python3 jarvis.py --doctor    check what works (sound, mic, screen, key)
  python3 jarvis.py --set-key   save your OpenRouter key safely
  python3 jarvis.py --make-app  create Jarvis.app on your Desktop (needs Python)
  python3 jarvis.py --build-app build a standalone Jarvis.app + Jarvis.zip to share

Built-in commands need no key. AI chat and screen reading use a free
OpenRouter key (optional). Voice input needs the add-ons:
  python3 -m pip install numpy sounddevice SpeechRecognition
"""
import os, sys, re, ast, json, math, time, random, shutil, queue, secrets, string
import operator, subprocess, threading, base64, unicodedata, plistlib
import urllib.request, urllib.parse, urllib.error, webbrowser
import xml.etree.ElementTree as ET
from datetime import datetime, timedelta, date

try:
    from zoneinfo import ZoneInfo
except Exception:
    ZoneInfo = None

try:
    import numpy as np
    import sounddevice as sd
    import speech_recognition as sr
    VOICE_OK = True
    VOICE_ERR = ""
except Exception as _e:
    VOICE_OK = False
    VOICE_ERR = str(_e)

FROZEN = bool(getattr(sys, "frozen", False))

# Packaged apps have no system certificates; use certifi's bundle if present.
try:
    import ssl
    import certifi
    _ctx = ssl.create_default_context(cafile=certifi.where())
    urllib.request.install_opener(urllib.request.build_opener(urllib.request.HTTPSHandler(context=_ctx)))
except Exception:
    pass

HOME = os.path.expanduser("~")
DESK = os.path.join(HOME, "Desktop")
NOTES = os.path.join(DESK, "jarvis_notes.txt")
TASKS = os.path.join(DESK, "jarvis_tasks.txt")
CARDS = os.path.join(DESK, "jarvis_cards.txt")
SETTINGS = os.path.join(HOME, ".jarvis_settings.json")
KEYFILE = os.path.join(HOME, ".jarvis_openrouter_key")
SHOT = "/tmp/jarvis_screen.jpg"
OR_URL = "https://openrouter.ai/api/v1"
GREETING_LINES = (
    "Welcome back, sir. You're the boss, and the world is yours.",
    "{part}, sir. You run this world. Say the word.",
    "All systems online, sir. You're the boss. What's the plan?",
    "{part}, boss. I mean, sir. The world is waiting on you.",
    "Back in charge, sir. Everything is ready when you are.",
)


def greeting():
    h = datetime.now().hour
    part = "Good morning" if h < 12 else ("Good afternoon" if h < 18 else "Good evening")
    return random.choice(GREETING_LINES).format(part=part)


GREETING = "Welcome back, sir. You're the boss, and the world is yours."
NOKEY_FALLBACK = ("I don't know that one yet. Say help to see what I can do, "
                  "or add a free OpenRouter key to unlock AI chat.")

# ----------------------------------------------------------------------------
# settings
# ----------------------------------------------------------------------------
DEFAULTS = {"voice": "Daniel", "rate": 175, "theme": "cyan", "quiet": False, "wake": False}
S = dict(DEFAULTS)
try:
    with open(SETTINGS) as _f:
        S.update(json.load(_f))
except Exception:
    pass


def save_settings():
    try:
        with open(SETTINGS, "w") as f:
            json.dump(S, f)
    except Exception:
        pass


THEMES = {"cyan": "#00e5ff", "red": "#ff3d3d", "gold": "#ffb300", "green": "#39ff88",
          "purple": "#b388ff", "blue": "#4d9dff", "white": "#e8f4ff",
          "orange": "#ff7a1a", "pink": "#ff5fa8", "teal": "#1de9b6", "lime": "#c6ff00"}

# ----------------------------------------------------------------------------
# data
# ----------------------------------------------------------------------------
JOKES = [
    "Why do programmers prefer dark mode? Because light attracts bugs.",
    "Why was the computer cold? It left its Windows open.",
    "What do you call a fake noodle? An impasta.",
    "Why did the scarecrow win an award? He was outstanding in his field.",
    "I would tell you a UDP joke, but you might not get it.",
    "Why don't skeletons fight each other? They don't have the guts.",
    "What do you call cheese that isn't yours? Nacho cheese.",
    "Why did the math book look sad? It had too many problems.",
]
FACTS = [
    "Honey never spoils. Edible honey has been found in ancient Egyptian tombs.",
    "Octopuses have three hearts.",
    "A day on Venus is longer than a year on Venus.",
    "Bananas are berries, but strawberries are not.",
    "Sharks are older than trees.",
    "Wombat droppings are cube shaped.",
]
QUOTES = [
    "Small steps every day add up to big results.",
    "You do not have to be perfect, you just have to start.",
    "Hard work beats talent when talent does not work hard.",
    "Every expert was once a beginner.",
    "Done is better than perfect.",
]
RIDDLES = [
    ("What has keys but cannot open locks?", "A piano."),
    ("What gets wetter the more it dries?", "A towel."),
    ("What has hands but cannot clap?", "A clock."),
    ("What goes up but never comes down?", "Your age."),
    ("What has a head and a tail but no body?", "A coin."),
]
TRIVIA = [
    ("What is the capital of France?", "paris"),
    ("How many continents are there?", "seven|7"),
    ("Which planet is known as the Red Planet?", "mars"),
    ("What is the largest ocean on Earth?", "pacific"),
    ("How many sides does a hexagon have?", "six|6"),
    ("What gas do plants take in from the air?", "carbon dioxide|co2"),
    ("Who painted the Mona Lisa?", "leonardo|da vinci"),
    ("What is the largest planet in our solar system?", "jupiter"),
    ("What is the chemical symbol for gold?", "au"),
    ("What is the fastest land animal?", "cheetah"),
    ("In which country are the Pyramids of Giza?", "egypt"),
]
BALL = ["It is certain.", "Without a doubt.", "Yes, definitely.", "Most likely.",
        "Ask again later.", "Cannot predict now.", "Don't count on it.",
        "My sources say no.", "Very doubtful.", "Outlook not so good."]
SMALL = {
    "hello": "Hello! What can I do for you?", "hi": "Hi there!", "hey": "Hey! I'm listening.",
    "thanks": "You're welcome!", "thank you": "You're welcome, sir.",
    "how are you": "Running smoothly, thanks for asking.",
    "who are you": "I am Jarvis, your personal assistant.",
    "what is your name": "My name is Jarvis.", "what's your name": "My name is Jarvis.",
    "good night": "Good night, sir. Sleep well.",
    "what is the meaning of life": "Forty two, according to a very famous book.",
    "i love you": "That is very kind. I am a program, but I appreciate it.",
}
MUSIC = {"play music": "play", "resume music": "play", "pause music": "pause",
         "stop music": "pause", "next song": "next track", "skip": "next track",
         "skip song": "next track", "previous song": "previous track"}
FOLDERS = {"downloads": "~/Downloads", "documents": "~/Documents", "desktop": "~/Desktop",
           "pictures": "~/Pictures", "home": "~", "applications": "/Applications"}
SITE_ALIASES = {"google": "https://www.google.com", "youtube": "https://www.youtube.com",
                "amazon": "https://www.amazon.com", "netflix": "https://www.netflix.com",
                "wikipedia": "https://www.wikipedia.org", "facebook": "https://www.facebook.com",
                "instagram": "https://www.instagram.com", "twitter": "https://www.twitter.com",
                "x": "https://www.x.com", "reddit": "https://www.reddit.com",
                "github": "https://www.github.com", "canvas": "https://www.instructure.com"}
APP_ALIASES = {"chrome": "Google Chrome", "google chrome": "Google Chrome",
               "vscode": "Visual Studio Code", "vs code": "Visual Studio Code",
               "code": "Visual Studio Code", "word": "Microsoft Word", "excel": "Microsoft Excel",
               "powerpoint": "Microsoft PowerPoint", "settings": "System Settings",
               "system settings": "System Settings", "system preferences": "System Settings",
               "app store": "App Store", "facetime": "FaceTime", "imessage": "Messages",
               "messages": "Messages", "notes": "Notes", "calendar": "Calendar",
               "calculator": "Calculator", "terminal": "Terminal", "finder": "Finder",
               "safari": "Safari", "music": "Music", "spotify": "Spotify", "mail": "Mail",
               "photos": "Photos", "maps": "Maps", "discord": "Discord", "zoom": "zoom.us"}
HOLIDAYS = {"christmas": (12, 25), "halloween": (10, 31), "new year": (1, 1), "valentines": (2, 14)}
CITIES = {
    "tokyo": "Asia/Tokyo", "london": "Europe/London", "paris": "Europe/Paris", "berlin": "Europe/Berlin",
    "rome": "Europe/Rome", "madrid": "Europe/Madrid", "moscow": "Europe/Moscow",
    "new york": "America/New_York", "los angeles": "America/Los_Angeles", "chicago": "America/Chicago",
    "toronto": "America/Toronto", "mexico city": "America/Mexico_City", "sao paulo": "America/Sao_Paulo",
    "sydney": "Australia/Sydney", "delhi": "Asia/Kolkata", "mumbai": "Asia/Kolkata",
    "dubai": "Asia/Dubai", "beijing": "Asia/Shanghai", "shanghai": "Asia/Shanghai",
    "seoul": "Asia/Seoul", "singapore": "Asia/Singapore", "cairo": "Africa/Cairo",
    "san francisco": "America/Los_Angeles", "san jose": "America/Los_Angeles", "miami": "America/New_York",
    "denver": "America/Denver", "seattle": "America/Los_Angeles", "boston": "America/New_York",
    "jacksonville": "America/New_York", "orlando": "America/New_York", "houston": "America/Chicago",
    "dallas": "America/Chicago", "atlanta": "America/New_York", "hawaii": "Pacific/Honolulu",
}
UNIT_ALIASES = {
    "mile": "mi", "mi": "mi", "kilometer": "km", "kilometre": "km", "km": "km", "meter": "m",
    "metre": "m", "m": "m", "centimeter": "cm", "centimetre": "cm", "cm": "cm", "foot": "ft",
    "feet": "ft", "ft": "ft", "inch": "in", "inche": "in", "in": "in", "yard": "yd", "yd": "yd",
    "kilogram": "kg", "kg": "kg", "gram": "g", "g": "g", "pound": "lb", "lb": "lb",
    "ounce": "oz", "oz": "oz", "liter": "l", "litre": "l", "l": "l", "milliliter": "ml",
    "ml": "ml", "gallon": "gal", "gal": "gal", "cup": "cup", "celsius": "c", "c": "c",
    "fahrenheit": "f", "f": "f", "kelvin": "k", "k": "k",
}
UNIT_BASE = {
    "mi": ("len", 1609.344), "km": ("len", 1000), "m": ("len", 1), "cm": ("len", 0.01),
    "ft": ("len", 0.3048), "in": ("len", 0.0254), "yd": ("len", 0.9144),
    "kg": ("mass", 1000), "g": ("mass", 1), "lb": ("mass", 453.592), "oz": ("mass", 28.3495),
    "l": ("vol", 1000), "ml": ("vol", 1), "gal": ("vol", 3785.41), "cup": ("vol", 236.588),
}
CUR = {
    "dollar": "USD", "usd": "USD", "euro": "EUR", "eur": "EUR", "pound": "GBP", "gbp": "GBP",
    "yen": "JPY", "jpy": "JPY", "peso": "MXN", "mxn": "MXN", "rupee": "INR", "inr": "INR",
    "yuan": "CNY", "cny": "CNY", "franc": "CHF", "chf": "CHF", "cad": "CAD", "aud": "AUD",
    "canadian dollar": "CAD", "australian dollar": "AUD",
}
OPS = {ast.Add: operator.add, ast.Sub: operator.sub, ast.Mult: operator.mul,
       ast.Div: operator.truediv, ast.USub: operator.neg, ast.Pow: operator.pow,
       ast.Mod: operator.mod}
UNIT_RX = r"(sec\w*|min\w*|hour\w*|hr\w*)"
VISION_RX = re.compile(r"\b(my screen|the screen|on screen|what do you see|what am i looking at|"
                       r"look at this|read this|what does this say|explain this|describe this|"
                       r"what's this|whats this|what is this)\b")
VISION_SKIP = re.compile(r"\b(lock|sleep|brightness|screenshot|screen ?shot|record|saver|turn off|shut|dim)\b")

HELP = """
ASK ANYTHING (needs a free OpenRouter or Groq key: say set key)
  explain black holes | help me study | write a haiku about cats
  what's on my screen | explain this error | summarize this page
  summarize my clipboard | forget everything
MEMORY AND STUDY
  my name is Sam | remember that I like soccer | what do you know about me
  what did we talk about | what did I say about soccer | show my chat history
  clear my chat history | forget my info
  make flashcards about the water cycle | quiz me on fractions
  make cards from my clipboard | quiz cards | my study stats
SPORTS AND FANTASY
  nfl scores | how did the cowboys do
  how is my fantasy team doing | who am i playing | who's winning
  how did all my teams do | fantasy standings
  (fantasy commands need: set fantasy football to <your team page link>)
EMAIL AND WEB
  gmail | open my email | check my email | what's on my calendar
  save contact mom as mom@example.com
  email mom about dinner saying see you at six  (then say: send it)
  open ixl | set vericross to <web address> | open vericross
  play lofi on spotify | open spotify
YOUR DAY
  good morning (daily briefing) | focus mode | focus mode off
  organize my downloads | open my downloads folder | open file resume
  when I say gm, give me my briefing | list my shortcuts | forget shortcut gm
SETTINGS
  settings | set key | set groq key | stay on top | stop staying on top
  open at login | don't open at login | wake word on | wake word off
PRODUCTIVITY
  add task buy milk | show my tasks | done 2 | clear tasks
  timer for 5 minutes | remind me in 10 minutes to stretch
  set alarm for 7:30 am | start pomodoro | list timers | cancel timers
  start stopwatch | lap | stop stopwatch
  remember buy milk | what are my notes | clear my notes
  add card capital of france = paris | quiz cards
INFO
  time | date | weather | weather in Paris | what time is it in Tokyo
  news | battery | storage | system info | my ip
  who is Albert Einstein | define serendipity | how many days until Christmas
CALCULATORS
  what is 15 times 12 | 15 percent of 200 | square root of 144
  convert 5 miles to km | convert 10 dollars to euros
  spell necessary | generate a password
MAC CONTROLS
  volume 40 | volume up | volume down | mute | unmute
  play music | pause music | next song | pause spotify
  open Safari | close Safari | open downloads | go to wikipedia.org
  search cats | play lofi beats | directions to the airport | find pizza near me
  find file resume | take a screenshot | read my clipboard | copy hello
  lock screen | sleep my mac | dark mode | light mode | say hello
GAMES AND FUN
  guess the number | trivia | riddle | answer
  rock paper scissors rock | roll a d20 | roll 2d6 | choose pizza or tacos
  magic 8 ball | tell me a joke | tell me a fact | motivate me | flip a coin
VOICE AND LOOKS
  test voice | list voices | set voice to oliver | talk faster | talk slower
  stop talking | quiet mode | talk again | repeat that
  wake word on | wake word off  (then just say: Jarvis, what time is it)
  theme red | gold | green | purple | blue | white | cyan
  doctor (checks sound, mic, screen and key) | quit
In the window: Up and Down arrows recall old commands, Esc stops talking.
"""

# ----------------------------------------------------------------------------
# shared state and output
# ----------------------------------------------------------------------------
STATE = {"mode": "idle", "voice": False, "level": 0.0, "battery": "--", "say_warned": False}
LAST_REPLY = [""]
CUR_VOICE = [None]
WAKE = {"on": False, "fn": None, "gen": 0}
WAKE_RX = re.compile(r"\b(?:jarvis|jarvas|jarvus|jervis)\b", re.I)
GAME = {"fn": None, "target": 0, "tries": 0, "answer": "", "riddle": ""}
SW = [None]
TIMERS = []
_emit = [lambda who, text: print((who + ": " if who else "") + text)]


def emit(who, text):
    _emit[0](who, text)


def run(cmd, timeout=30, **kw):
    try:
        r = subprocess.run(cmd, capture_output=True, text=True, timeout=timeout, **kw)
        return r.stdout.strip()
    except Exception:
        return ""


def fmt(x):
    x = round(x, 2)
    return str(int(x)) if x == int(x) else str(x)


def fmt_time(s):
    m, sec = divmod(int(s), 60)
    return (str(m) + " minutes and " if m else "") + str(sec) + " seconds"


def notify(text):
    safe = text.replace("\\", "").replace('"', "'")
    run(["osascript", "-e", 'display notification "' + safe + '" with title "Jarvis"'])


# ----------------------------------------------------------------------------
# speaking
# ----------------------------------------------------------------------------
def installed_voices():
    found = []
    for line in run(["say", "-v", "?"]).splitlines():
        m = re.match(r"^(.+?)\s{2,}([a-z]{2,3}_[A-Za-z0-9]+)", line)
        if m:
            found.append((m.group(1).strip(), m.group(2)))
    return found


def resolve_voice():
    names = [n for n, _ in installed_voices()]
    if not names:
        return None
    low = {n.lower(): n for n in names}
    for want in (S.get("voice", ""), "Daniel", "Oliver", "Arthur", "Alex", "Samantha"):
        if want and want.lower() in low:
            return low[want.lower()]
    return None


def _clean_for_speech(t):
    t = t.replace("\u00b0", " degrees ")
    t = "".join(ch for ch in t if unicodedata.category(ch) not in ("So", "Sk", "Cs", "Cn")
                and ch not in "\ufe0f\u200d")
    t = re.sub(r"[`*_#>|]", " ", t)
    return re.sub(r"\s+", " ", t).strip()[:700]


def _say(text, voice):
    cmd = ["say"]
    if voice:
        cmd += ["-v", voice]
    cmd += ["-r", str(int(S.get("rate", 175))), "--", text]
    try:
        r = subprocess.run(cmd, capture_output=True, text=True)
        return (r.returncode == 0 or r.returncode < 0), (r.stderr or "").strip()
    except FileNotFoundError:
        return False, "the say command was not found"
    except Exception as e:
        return False, str(e)


_speak_lock = threading.Lock()


def speak(text):
    LAST_REPLY[0] = text
    emit("JARVIS", text)
    if S.get("quiet"):
        STATE["mode"] = "idle"
        return
    clean = _clean_for_speech(text)
    if not clean:
        return
    with _speak_lock:
        STATE["mode"] = "speaking"
        try:
            ok, err = _say(clean, CUR_VOICE[0])
            if not ok and CUR_VOICE[0]:
                ok, err = _say(clean, None)
            if not ok and not STATE["say_warned"]:
                STATE["say_warned"] = True
                emit("", "Sound problem: " + (err or "unknown") + "  (say: doctor)")
        finally:
            STATE["mode"] = "idle"


def stop_talking():
    run(["killall", "say"])


def set_voice(name):
    name = name.strip().lower()
    for n, _ in installed_voices():
        if n.lower() == name or n.lower().startswith(name):
            CUR_VOICE[0] = n
            S["voice"] = n
            save_settings()
            return "This is my new voice, " + n
    return "I don't have a voice called " + name + ". Say list voices to see them."


def list_voices():
    english = [n for n, lang in installed_voices() if lang.startswith("en_")]
    if not english:
        return "I couldn't read the voice list"
    emit("", ", ".join(english))
    return "I have " + str(len(english)) + " English voices. The list is on your screen."


def set_rate(change):
    S["rate"] = max(100, min(300, int(S.get("rate", 175)) + change))
    save_settings()
    return "Speaking faster" if change > 0 else "Speaking slower"


# ----------------------------------------------------------------------------
# listening
# ----------------------------------------------------------------------------
def listen(max_seconds=12):
    if not VOICE_OK:
        return ""
    rate, step = 16000, 0.2
    chunk = int(rate * step)
    frames, started, quiet = [], False, 0
    with sd.InputStream(samplerate=rate, channels=1, dtype="int16") as stream:
        amb = []
        for _i in range(3):
            d, _o = stream.read(chunk)
            amb.append(float(np.abs(d).mean()))
        thr = max(250.0, (sum(amb) / len(amb)) * 2.5)
        for i in range(int(max_seconds / step)):
            data, _o = stream.read(chunk)
            frames.append(data)
            if float(np.abs(data).mean()) > thr:
                started, quiet = True, 0
            elif started:
                quiet += 1
                if quiet >= 5:
                    break
            elif i > 25:
                return ""
    audio = sr.AudioData(np.concatenate(frames).tobytes(), rate, 2)
    try:
        return sr.Recognizer().recognize_google(audio)
    except sr.UnknownValueError:
        return ""
    except sr.RequestError:
        emit("", "Speech service unreachable. Voice input needs the internet.")
        time.sleep(2)
        return ""


# ----------------------------------------------------------------------------
# wake word: say "Jarvis, what time is it" or just "Jarvis" and then your command
# ----------------------------------------------------------------------------
def chime():
    if sys.platform == "darwin":
        try:
            subprocess.Popen(["afplay", "/System/Library/Sounds/Tink.aiff"],
                             stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        except OSError:
            pass


def _wake_loop(gen):
    while WAKE["on"] and WAKE["gen"] == gen:
        if STATE["mode"] in ("speaking", "thinking"):
            time.sleep(0.3)
            continue
        if STATE["mode"] == "idle":
            STATE["mode"] = "listening"
        try:
            heard = listen(8)
        except Exception as e:
            if STATE["mode"] == "listening":
                STATE["mode"] = "idle"
            WAKE["on"] = False
            emit("JARVIS", "Wake word stopped: microphone problem (" + str(e)[:80] + "). Say doctor.")
            return
        if STATE["mode"] == "listening":
            STATE["mode"] = "idle"
        if not heard or not (WAKE["on"] and WAKE["gen"] == gen):
            continue
        m = WAKE_RX.search(heard)
        if not m:
            continue
        chime()
        cmd = heard[m.end():].strip(" ,.!?")
        if not cmd:
            speak("Yes, sir?")
            if STATE["mode"] == "idle":
                STATE["mode"] = "listening"
            try:
                cmd = listen(10)
            except Exception:
                cmd = ""
            if STATE["mode"] == "listening":
                STATE["mode"] = "idle"
            if not cmd:
                continue
        fn = WAKE.get("fn")
        if fn:
            fn(cmd)


def start_wake():
    if not VOICE_OK:
        return "I need the voice add-ons for that. Run: python3 -m pip install numpy sounddevice SpeechRecognition"
    if WAKE["on"]:
        return "The wake word is already on. Just say Jarvis and then your command."
    STATE["voice"] = False
    WAKE["on"] = True
    WAKE["gen"] += 1
    S["wake"] = True
    save_settings()
    threading.Thread(target=_wake_loop, args=(WAKE["gen"],), daemon=True).start()
    return "Wake word on. Say Jarvis, then your command. Sound that I hear is sent to Google's speech service."


def stop_wake():
    was = WAKE["on"]
    WAKE["on"] = False
    WAKE["gen"] += 1
    S["wake"] = False
    save_settings()
    if STATE["mode"] == "listening":
        STATE["mode"] = "idle"
    return "Wake word off" if was else "The wake word was already off"


# ----------------------------------------------------------------------------
# web helpers (all free, no accounts)
# ----------------------------------------------------------------------------
def get_json(url):
    req = urllib.request.Request(url, headers={"User-Agent": "JarvisHomeProject/2.0"})
    return json.loads(urllib.request.urlopen(req, timeout=8).read().decode())


def weather(city=""):
    try:
        url = "https://wttr.in/" + urllib.parse.quote(city) + "?format=3"
        w = urllib.request.urlopen(url, timeout=8).read().decode().strip()
        return re.sub(r"\+(\d)", r"\1", w)
    except Exception:
        return None


def news(n=5):
    try:
        req = urllib.request.Request("https://feeds.bbci.co.uk/news/rss.xml",
                                     headers={"User-Agent": "JarvisHomeProject/2.0"})
        root = ET.fromstring(urllib.request.urlopen(req, timeout=8).read())
        return [i.findtext("title") for i in root.iter("item")][:n]
    except Exception:
        return None


def wiki(topic):
    try:
        res = get_json("https://en.wikipedia.org/w/rest.php/v1/search/title?q="
                       + urllib.parse.quote(topic) + "&limit=1")
        pages = res.get("pages", [])
        if not pages:
            return "I couldn't find anything on " + topic
        s = get_json("https://en.wikipedia.org/api/rest_v1/page/summary/"
                     + urllib.parse.quote(pages[0]["key"]))
        sentences = re.split(r"(?<=[.!?])\s+", s.get("extract", ""))
        return " ".join(sentences[:2]) or "I couldn't find anything on " + topic
    except Exception:
        return "I couldn't look that up right now"


def define(word):
    try:
        data = get_json("https://api.dictionaryapi.dev/api/v2/entries/en/" + urllib.parse.quote(word))
        return word + ": " + data[0]["meanings"][0]["definitions"][0]["definition"]
    except Exception:
        return "I couldn't find a definition for " + word


def cur_code(w):
    w = w.strip().lower()
    if w in CUR:
        return CUR[w]
    if w.endswith("s") and w[:-1] in CUR:
        return CUR[w[:-1]]
    return None


def currency(n, a, b):
    ca, cb = cur_code(a), cur_code(b)
    if not ca or not cb:
        return None
    try:
        data = get_json("https://api.frankfurter.app/latest?amount=" + str(n) + "&from=" + ca + "&to=" + cb)
        return fmt(n) + " " + a.strip() + " is about " + fmt(data["rates"][cb]) + " " + b.strip()
    except Exception:
        return "I couldn't get exchange rates right now"


# ----------------------------------------------------------------------------
# AI (OpenRouter free models): chat and screen reading
# ----------------------------------------------------------------------------
AI_SYSTEM = ("You are Jarvis, a witty, polite British butler-style voice assistant who occasionally "
             "says sir. Reply in 1 to 4 short sentences. Plain text only: no markdown, no lists, no emoji. "
             "You run as an app on the user's Mac and DO have real abilities: opening apps, files and folders, "
             "reading the screen, playing music, timers, notes, reading unread email, sending email after the "
             "user confirms, calendar, summarizing web pages, flashcards, and more. Never claim you cannot access "
             "their computer; if unsure what you can do, tell them to say: help.")
SV_SYSTEM = ("You are Jarvis, a polite British butler-style assistant looking at a screenshot of the "
             "user's computer screen. Answer their question in 1 to 4 short sentences. Plain text only. "
             "If you only see a wallpaper or blank screen, say screen recording permission may be missing.")
AI_HISTORY = []
_MODELS = {"text": [], "vision": [], "t": 0.0}

# long-term memory: facts about you + the recent chat survive closing the app
MEMFILE = os.path.join(HOME, ".jarvis_memory.json")
MEM = {"facts": [], "history": []}


def mem_load():
    try:
        with open(MEMFILE) as f:
            d = json.load(f)
        MEM["facts"] = [str(x) for x in d.get("facts", [])][:60]
        AI_HISTORY[:] = [m for m in d.get("history", []) if isinstance(m, dict)
                         and m.get("role") in ("user", "assistant") and isinstance(m.get("content"), str)][-20:]
    except Exception:
        pass


def mem_save():
    try:
        with open(MEMFILE, "w") as f:
            json.dump({"facts": MEM["facts"], "history": AI_HISTORY[-20:]}, f)
        os.chmod(MEMFILE, 0o600)
    except OSError:
        pass


def ai_system():
    if not MEM["facts"]:
        return AI_SYSTEM
    return AI_SYSTEM + " Things you know about the user: " + "; ".join(MEM["facts"][-30:]) + "."


def mem_command(c):
    """Handle 'my name is...', 'remember that i...', 'what do you know about me'. Returns text or None."""
    m = re.match(r"^(?:my name is|call me|i am called|i'm called)\s+([a-z][a-z' -]{0,30})$", c)
    if m:
        name = m.group(1).strip().title()
        MEM["facts"] = [f for f in MEM["facts"] if not f.startswith("Name is ")] + ["Name is " + name]
        mem_save()
        return "Pleased to meet you, " + name + ". I'll remember that."
    m = re.match(r"^remember (?:that )?(i .+|my .+)$", c)
    if m:
        fact = m.group(1).strip()
        fact = re.sub(r"^i (am|have|like|love|hate|want|need|play|study|live|go)\b",
                      lambda mm: "User " + mm.group(1) + ("s" if mm.group(1) not in ("am", "have") else ""), fact)
        fact = fact.replace("User am", "User is").replace("User have", "User has")
        fact = re.sub(r"^my ", "User's ", fact)
        MEM["facts"].append(fact[:160])
        mem_save()
        return "Noted, sir. I'll remember that."
    if c in ("what do you know about me", "what do you remember about me", "what do you remember"):
        if not MEM["facts"]:
            return "Nothing yet. Tell me: my name is ... or: remember that I like ..."
        return "Here is what I know: " + "; ".join(MEM["facts"][-10:]) + "."
    if c in ("forget my info", "forget what you know about me", "forget about me", "clear my memory"):
        MEM["facts"] = []
        mem_save()
        return "Done. I've forgotten what I knew about you."
    return None


CHATLOG = os.path.join(HOME, ".jarvis_chats.jsonl")


def log_chat(you, jarvis):
    """Append every exchange to a private chat log so Jarvis can recall it later."""
    try:
        with open(CHATLOG, "a") as f:
            f.write(json.dumps({"t": datetime.now().isoformat(timespec="minutes"), "you": you[:500],
                                "jarvis": jarvis[:800]}) + "\n")
        os.chmod(CHATLOG, 0o600)
        if os.path.getsize(CHATLOG) > 3000000:
            with open(CHATLOG) as f:
                keep = f.readlines()[-3000:]
            with open(CHATLOG, "w") as f:
                f.writelines(keep)
    except OSError:
        pass


def chat_entries():
    out = []
    try:
        with open(CHATLOG) as f:
            for line in f:
                try:
                    out.append(json.loads(line))
                except ValueError:
                    pass
    except OSError:
        pass
    return out


def _when(t):
    try:
        return datetime.fromisoformat(t).strftime("%a %I:%M %p").replace(" 0", " ")
    except ValueError:
        return t


def chat_search(term):
    term = term.lower().strip()
    hits = [e for e in chat_entries() if term in e.get("you", "").lower() or term in e.get("jarvis", "").lower()]
    if not hits:
        return "I don't see anything about " + term + " in our chats."
    parts = [_when(e["t"]) + ": you said " + e["you"][:90] for e in hits[-4:]]
    return "Found " + str(len(hits)) + ". " + ". ".join(parts)


def chat_recap():
    ents = chat_entries()
    if not ents:
        return "We haven't chatted yet that I can remember."
    recent = ents[-12:]
    if ai_ready():
        text = "\n".join("User: " + e["you"] + "\nJarvis: " + e["jarvis"][:200] for e in recent)
        try:
            reply, _m = _chat(False, [{"role": "system", "content":
                                       "Summarize this conversation in 2 or 3 short plain sentences, "
                                       "addressed to the user as 'we' or 'you'. No markdown."},
                                      {"role": "user", "content": text}], 200)
            if reply:
                return _tidy(reply)
        except AIError:
            pass
    return "Recently you asked: " + "; ".join(e["you"][:50] for e in recent[-5:])


def chat_command(c):
    if re.search(r"what (?:did|have) we (?:talk(?:ed)?|chat(?:ted)?|discuss(?:ed)?|been talking) (?:about|for)|"
                 r"(?:recap|summari[sz]e) (?:our|my) (?:chat|chats|conversation|conversations)|remind me what we (?:talked|discussed)", c):
        return chat_recap()
    m = re.match(r"^(?:what did i (?:say|ask|tell you) about|search (?:my |our )?(?:chats?|conversations?|history) (?:for|about)|"
                 r"find in (?:my |our )?chats?|did i (?:ever )?(?:talk|ask) (?:to you )?about) (.+)$", c)
    if m:
        return chat_search(m.group(1))
    if re.fullmatch(r"(?:show|read|what(?:'s| is)) (?:me )?(?:my |our )?(?:recent )?(?:chat )?(?:history|chats|conversations)", c):
        ents = chat_entries()
        if not ents:
            return "No chat history yet."
        return "Your last messages: " + ". ".join(_when(e["t"]) + ", " + e["you"][:60] for e in ents[-5:])
    if re.fullmatch(r"(?:clear|delete|erase|wipe) (?:my |our |all )?(?:chat )?(?:history|chats|conversations)", c):
        try:
            os.remove(CHATLOG)
        except OSError:
            pass
        AI_HISTORY[:] = []
        mem_save()
        return "Done. I've erased our chat history."
    return None


mem_load()


class AIError(Exception):
    def __init__(self, code, msg=""):
        Exception.__init__(self, msg)
        self.code = code
        self.msg = msg


def _key():
    k = os.environ.get("OPENROUTER_API_KEY", "").strip()
    if not k:
        try:
            with open(KEYFILE) as f:
                k = f.read().strip()
        except OSError:
            k = ""
    return k


GROQ_URL = "https://api.groq.com/openai/v1"
GKEYFILE = os.path.join(HOME, ".jarvis_groq_key")


def _gkey():
    k = os.environ.get("GROQ_API_KEY", "").strip()
    if not k:
        try:
            with open(GKEYFILE) as f:
                k = f.read().strip()
        except OSError:
            k = ""
    return k


def ai_ready():
    return bool(_key() or _gkey())


_UAS = ("Jarvis/2.0", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 Jarvis/2.0")


def _or(path, body=None, timeout=60):
    data = json.dumps(body).encode() if body is not None else None
    last = None
    for ua in _UAS:
        req = urllib.request.Request(
            OR_URL + path, data=data,
            headers={"Authorization": "Bearer " + _key(), "Content-Type": "application/json",
                     "HTTP-Referer": "https://localhost", "X-Title": "Jarvis", "User-Agent": ua})
        try:
            with urllib.request.urlopen(req, timeout=timeout) as r:
                return json.loads(r.read().decode())
        except urllib.error.HTTPError as e:
            last = AIError(e.code, e.read()[:200].decode("utf-8", "ignore"))
            if e.code != 403:
                raise last
        except Exception as e:
            raise AIError(0, str(e))
    raise last


def _models(vision):
    kind = "vision" if vision else "text"
    if _MODELS[kind] and time.time() - _MODELS["t"] < 1800:
        return _MODELS[kind]
    text, vis = [], []
    for m in _or("/models").get("data", []):
        mid = m.get("id", "")
        if not mid.endswith(":free"):
            continue
        arch = m.get("architecture") or {}
        ins = arch.get("input_modalities") or ["text"]
        outs = arch.get("output_modalities") or ["text"]
        if "text" not in outs:
            continue
        ctx = m.get("context_length") or 0
        text.append((ctx, mid))
        if "image" in ins:
            vis.append((ctx, mid))
    _MODELS["text"] = [i for _c, i in sorted(text, reverse=True)]
    _MODELS["vision"] = [i for _c, i in sorted(vis, reverse=True)]
    _MODELS["t"] = time.time()
    return _MODELS[kind]


_REASON_RX = re.compile(r"(?i)(analy[sz]e the request|determine the response|draft(?:ing)? (?:the )?(?:response|\d)|"
                        r"\bconstraints?:|\bpersona:|\bthe user (?:asks|is asking|wants|said)\b.*\n|^\s*1\.\s+\*{0,2}(?:analy|understand|identify))")


def _is_reasoning(reply):
    """Some free models print their thinking instead of an answer. Spot that so we can skip it."""
    return bool(_REASON_RX.search(reply)) and (len(reply) > 220 or reply.count("\n") >= 3)


def _chat_or(vision, messages, max_tokens=350, budget=60, per_try=25):
    models = _models(vision)
    if not models:
        return None, None
    errors = []
    give_up = time.time() + budget
    for model in list(models[:30]):
        if time.time() > give_up:
            break
        try:
            r = _or("/chat/completions", {"model": model, "messages": messages, "max_tokens": max_tokens},
                    timeout=per_try)
        except AIError as e:
            errors.append(e)
            if e.code == 429:
                time.sleep(1)
            continue
        msg = ((r.get("choices") or [{}])[0].get("message")) or {}
        reply = (msg.get("content") or "").strip()
        if reply and _is_reasoning(reply):
            continue
        if reply:
            lst = _MODELS["vision" if vision else "text"]
            if model in lst:
                lst.remove(model)
                lst.insert(0, model)
            return reply, model
    if errors:
        codes = set(e.code for e in errors)
        if len(codes) == 1:
            raise errors[0]
        if 429 in codes:
            raise AIError(429, "")
        raise errors[-1]
    return None, None


_GM = {"text": [], "vision": [], "t": 0.0}


def _groq(path, body=None, timeout=30):
    data = json.dumps(body).encode() if body is not None else None
    req = urllib.request.Request(
        GROQ_URL + path, data=data,
        headers={"Authorization": "Bearer " + _gkey(), "Content-Type": "application/json", "User-Agent": _UAS[1]})
    try:
        with urllib.request.urlopen(req, timeout=timeout) as r:
            return json.loads(r.read().decode())
    except urllib.error.HTTPError as e:
        raise AIError(e.code, e.read()[:200].decode("utf-8", "ignore"))
    except Exception as e:
        raise AIError(0, str(e))


def _groq_models(vision):
    kind = "vision" if vision else "text"
    if _GM[kind] and time.time() - _GM["t"] < 1800:
        return _GM[kind]
    ids = [m.get("id", "") for m in _groq("/models").get("data", []) if m.get("active", True)]
    skip = ("whisper", "guard", "tts", "playai", "orpheus", "embed", "distil")
    ids = [i for i in ids if i and not any(w in i.lower() for w in skip)]

    def rank(i):
        i = i.lower()
        for n, key in enumerate(("llama-3.3-70b", "gpt-oss-120b", "70b", "gpt-oss", "llama-4", "qwen", "8b")):
            if key in i:
                return n
        return 20
    vis = [i for i in ids if any(w in i.lower() for w in ("scout", "maverick", "vision"))]
    _GM["text"] = sorted(ids, key=rank)
    _GM["vision"] = sorted(vis, key=rank)
    _GM["t"] = time.time()
    return _GM[kind]


def _chat_groq(vision, messages, max_tokens=350):
    models = _groq_models(vision)
    last = None
    for model in models[:4]:
        try:
            r = _groq("/chat/completions", {"model": model, "messages": messages, "max_tokens": max_tokens})
        except AIError as e:
            last = e
            if e.code in (401, 403):
                break
            if e.code == 429:
                time.sleep(1)
            continue
        msg = ((r.get("choices") or [{}])[0].get("message")) or {}
        reply = (msg.get("content") or "").strip()
        if reply and _is_reasoning(reply):
            continue
        if reply:
            return reply, "groq " + model
    if last:
        raise last
    return None, None


def _chat(vision, messages, max_tokens=350):
    """Reply fast: if both free services are set up, race them and use whichever answers first.
    Otherwise use whichever one key you have."""
    if _key() and _gkey():
        import concurrent.futures
        pool = concurrent.futures.ThreadPoolExecutor(max_workers=2)
        fut_or = pool.submit(_chat_or, vision, messages, max_tokens, 14, 8)
        fut_gq = pool.submit(_chat_groq, vision, messages, max_tokens)
        winner = None
        try:
            for fut in concurrent.futures.as_completed((fut_or, fut_gq), timeout=20):
                try:
                    r = fut.result()
                except AIError:
                    continue
                if r[0]:
                    winner = r
                    break
        except concurrent.futures.TimeoutError:
            pass
        pool.shutdown(wait=False)  # don't wait for the slower/losing call; let it finish quietly later
        if winner:
            return winner
        # neither answered within 20s total; give OpenRouter a longer last try
        try:
            r = _chat_or(vision, messages, max_tokens, budget=20, per_try=15)
            if r[0]:
                return r
        except AIError:
            pass
        return None, None
    err = None
    if _key():
        try:
            r = _chat_or(vision, messages, max_tokens)
            if r[0]:
                return r
        except AIError as e:
            err = e
    if _gkey():
        try:
            r = _chat_groq(vision, messages, max_tokens)
            if r[0]:
                return r
        except AIError as e:
            err = err or e
    if err:
        raise err
    return None, None


def _ai_error(e):
    detail = (e.msg or "").replace("\n", " ")
    try:
        detail = json.loads(e.msg)["error"]["message"]
    except Exception:
        pass
    detail = detail[:140]
    if e.code == 401:
        return "OpenRouter says unauthorized: " + detail + " (If it is about the key, type: set key)"
    if e.code == 402:
        return "Those free models want credits on your OpenRouter account right now. " + detail
    if e.code == 429:
        return ("All the free AI models are rate-limited right now. They are shared by lots of people, "
                "so wait a minute and ask again." +
                ("" if _gkey() else " For a backup that rarely runs out, say: set groq key"))
    if e.code == 0:
        return "I couldn't reach the AI service. Check your internet, or the school filter. " + detail[:80]
    return "The AI service said " + str(e.code) + ": " + detail


def _tidy(reply):
    reply = re.sub(r"(?is)<think(?:ing)?>.*?(?:</think(?:ing)?>|$)", "", reply)
    reply = reply.replace("*", "").replace("#", "").replace("`", "")
    reply = re.sub(r"^\s*(?:jarvis|assistant)\s*:\s*", "", reply, flags=re.I)
    reply = re.sub(r"[ \t]+", " ", reply)
    reply = re.sub(r"\n{3,}", "\n\n", reply)
    return reply.strip()


def ask_ai(text):
    if not ai_ready():
        return None
    AI_HISTORY.append({"role": "user", "content": text})
    msgs = [{"role": "system", "content": ai_system()}] + AI_HISTORY[-12:]
    try:
        reply, _m = _chat(False, msgs)
    except AIError as e:
        AI_HISTORY.pop()
        return _ai_error(e)
    if not reply:
        AI_HISTORY.pop()
        return "The free AI models are busy right now. Try again in a minute."
    reply = _tidy(reply)
    AI_HISTORY.append({"role": "assistant", "content": reply})
    mem_save()
    return reply


UI_ACTION = [None]


def ui_action(name, msg):
    if UI_ACTION[0]:
        UI_ACTION[0](name)
        return msg
    return "That only works in the Jarvis window."


def daily_briefing():
    now = datetime.now()
    part = "Good morning" if now.hour < 12 else ("Good afternoon" if now.hour < 18 else "Good evening")
    out = [part + ", sir. It is " + now.strftime("%I:%M %p").lstrip("0") + " on " + now.strftime("%A, %B ") + str(now.day) + "."]
    box = {}

    def job(key, fn):
        try:
            box[key] = fn()
        except Exception:
            box[key] = None
    threads = [threading.Thread(target=job, args=(k, f), daemon=True) for k, f in
               (("weather", weather), ("cal", calendar_today), ("mail", email_summary))]
    for t in threads:
        t.start()
    for t in threads:
        t.join(25)
    if box.get("weather"):
        out.append(box["weather"])
    tasks = _read_lines(TASKS)
    out.append(("You have " + str(len(tasks)) + " task" + ("" if len(tasks) == 1 else "s") + " on your list.") if tasks else "Your task list is empty.")
    cal = box.get("cal") or ""
    if cal.startswith("Today you have"):
        out.append(cal)
    mail = box.get("mail") or ""
    if mail.startswith("You have"):
        out.append(mail)
    elif mail.startswith("You have no"):
        out.append(mail)
    if TIMERS:
        out.append(str(len(TIMERS)) + " timer" + ("" if len(TIMERS) == 1 else "s") + " running.")
    return " ".join(out)


def make_cards(source, is_text=False):
    """Ask the free AI to write flashcards, save them, return a message."""
    if not ai_ready():
        return "Making flashcards needs the free AI key. Type: set key"
    if is_text:
        ask = "Make 6 flashcards from these notes:\n" + source[:3500]
    else:
        ask = "Make 6 flashcards to study this topic: " + source
    msgs = [{"role": "system", "content": "You write flashcards. Output exactly 6 lines, each formatted as: "
             "question = short answer. No numbering, no extra text, no markdown."},
            {"role": "user", "content": ask}]
    try:
        reply, _m = _chat(False, msgs, 500)
    except AIError as e:
        return _ai_error(e)
    if not reply:
        return "The free AI models are busy right now. Try again in a minute."
    new = []
    for line in reply.splitlines():
        line = line.replace("**", "").strip()
        line = re.sub(r"^[^A-Za-z0-9]*(?:\d+[.)]\s*)?", "", line)
        if "=" in line:
            q, a = line.split("=", 1)
            if q.strip() and a.strip():
                new.append(q.strip().rstrip("?") + "|||" + a.strip())
    if not new:
        return "I couldn't make cards from that. Try a simpler topic."
    _write_lines(CARDS, _read_lines(CARDS) + new)
    return "Made " + str(len(new)) + " flashcards. Say: quiz cards"


DL_TYPES = {"Images": (".png", ".jpg", ".jpeg", ".gif", ".heic", ".webp", ".svg"),
            "Documents": (".pdf", ".doc", ".docx", ".txt", ".pages", ".rtf", ".ppt", ".pptx", ".key"),
            "Spreadsheets": (".xls", ".xlsx", ".csv", ".numbers"),
            "Music and Video": (".mp3", ".m4a", ".wav", ".mp4", ".mov", ".mkv"),
            "Installers and Zips": (".dmg", ".pkg", ".zip", ".tar", ".gz"),
            "Code": (".py", ".js", ".html", ".css", ".json")}


def organize_downloads():
    base = os.path.join(HOME, "Downloads")
    if not os.path.isdir(base):
        return "I can't find your Downloads folder."
    moved = 0
    for name in os.listdir(base):
        src = os.path.join(base, name)
        if name.startswith(".") or not os.path.isfile(src):
            continue
        ext = os.path.splitext(name)[1].lower()
        for folder, exts in DL_TYPES.items():
            if ext in exts:
                dest = os.path.join(base, folder)
                dst = os.path.join(dest, name)
                if os.path.exists(dst):
                    break
                try:
                    os.makedirs(dest, exist_ok=True)
                    shutil.move(src, dst)
                    moved += 1
                except OSError:
                    pass
                break
    return ("Tidied " + str(moved) + " files into folders inside Downloads. Nothing was deleted."
            if moved else "Your Downloads are already tidy.")


def focus_mode(on):
    if on:
        run(["osascript", "-e", "set volume with output muted"])
        run(["open", "-a", "Notes"])
        return "Focus mode on. Sound is muted and Notes is open. Say: focus mode off when you're done."
    run(["osascript", "-e", "set volume without output muted"])
    return "Focus mode off. Sound is back on."


DEFAULT_LINKS = {"ixl": "https://www.ixl.com/signin", "gmail": "https://mail.google.com",
                 "email": "https://mail.google.com",
                 "spotify web": "https://open.spotify.com"}


def get_link(name):
    return (S.get("links") or {}).get(name) or DEFAULT_LINKS.get(name)


def email_summary():
    script = ('tell application "Mail"\n'
              'set unreadMsgs to (messages of inbox whose read status is false)\n'
              'set n to count of unreadMsgs\n'
              'set out to (n as string) & "|"\n'
              'if n > 5 then set n to 5\n'
              'repeat with i from 1 to n\n'
              'set m to item i of unreadMsgs\n'
              'set out to out & (extract name from sender of m) & ": " & (subject of m) & "; "\n'
              'end repeat\nreturn out\nend tell')
    out = run(["osascript", "-e", script], timeout=40)
    if "|" not in out:
        return ("I couldn't read your email. Open the Mail app and add your account first "
                "(System Settings > Internet Accounts > Google), then allow Jarvis when asked. "
                "Or say: open gmail")
    count, rest = out.split("|", 1)
    if count.strip() == "0":
        return "You have no unread emails."
    return "You have " + count.strip() + " unread. Newest: " + rest.strip().rstrip(";")


PENDING = {"email": None}

FOLDERS = {"downloads": "Downloads", "documents": "Documents", "desktop": "Desktop", "pictures": "Pictures",
           "photos": "Pictures", "music": "Music", "movies": "Movies", "home": ""}


def _resolve_recipient(who):
    who = who.strip().strip(".")
    if "@" in who:
        return who.replace(" ", "").lower()
    s = re.sub(r"\s+at\s+", "@", who.lower())
    s = re.sub(r"\s+dot\s+", ".", s)
    if "@" in s and "." in s:
        return s.replace(" ", "")
    return (S.get("contacts") or {}).get(who.lower())


def prepare_email(who, subject, body):
    addr = _resolve_recipient(who)
    if not addr:
        return ("I don't have an email address for " + who + ". Say: save contact " + who.lower() +
                " as name@example.com")
    PENDING["email"] = (addr, subject.strip()[:120], body.strip())
    return ("Ready to email " + addr + ". Subject: " + subject.strip()[:120] + ". Message: " + body.strip()[:300] +
            ". Say: send it, to send, or: cancel.")


def send_pending_email():
    p = PENDING["email"]
    if not p:
        return "There's no email waiting to be sent."
    PENDING["email"] = None
    script = ('on run {addr, subj, body}\ntell application "Mail"\n'
              'set m to make new outgoing message with properties {subject:subj, content:body & return, visible:false}\n'
              'tell m to make new to recipient at end of to recipients with properties {address:addr}\n'
              'send m\nend tell\nend run')
    try:
        r = subprocess.run(["osascript", "-e", script, p[0], p[1], p[2]], capture_output=True, text=True, timeout=40)
    except Exception as e:
        return "I couldn't send it: " + str(e)[:60]
    if r.returncode != 0:
        return ("I couldn't send it. Make sure the Mail app has your account (System Settings > Internet Accounts) "
                "and that you allowed Jarvis to control Mail.")
    return "Sent to " + p[0] + "."


def open_path(kind, name):
    name = name.strip().strip(".")
    if kind == "folder" and name.lower() in FOLDERS:
        run(["open", os.path.join(HOME, FOLDERS[name.lower()])])
        return "Opening your " + name.lower() + " folder."
    out = run(["mdfind", "-onlyin", HOME, "-name", name], timeout=15)
    hits = [h for h in out.splitlines() if h and "/Library/" not in h and "/." not in h]
    if kind == "folder":
        hits = [h for h in hits if os.path.isdir(h)]
    elif kind == "file":
        hits = [h for h in hits if os.path.isfile(h)]
    if not hits:
        return "I couldn't find a " + kind + " called " + name + "."
    hits.sort(key=lambda h: (name.lower() not in os.path.basename(h).lower(), len(h)))
    run(["open", hits[0]])
    return "Opening " + os.path.basename(hits[0]) + "."


def open_link(name):
    url = get_link(name)
    if not url:
        return ("I don't have a link for " + name + " yet. Open it in your browser, copy the address, "
                "then say: set " + name + " to <the address>")
    webbrowser.open(url)
    if name == "email":
        return "Opening your email."
    extra = " Once you're signed in, say: summarize this page, or: what's on my screen." if name in ("vericross", "ixl") else ""
    return "Opening " + ("IXL" if name == "ixl" else name.title()) + "." + extra


def nfl_scores(team=None):
    try:
        d = get_json("https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard")
    except Exception as e:
        return "I couldn't reach NFL scores right now (" + str(e)[:60] + ")."
    events = d.get("events") or []
    if not events:
        return "No NFL games are scheduled right now."
    lines = []
    for ev in events:
        comp = (ev.get("competitions") or [{}])[0]
        teams = comp.get("competitors") or []
        if len(teams) != 2:
            continue
        a = next((t for t in teams if t.get("homeAway") == "away"), teams[0])
        h = next((t for t in teams if t.get("homeAway") == "home"), teams[1])
        an, hn = a["team"]["abbreviation"], h["team"]["abbreviation"]
        adn, hdn = a["team"].get("displayName", "").upper(), h["team"].get("displayName", "").upper()
        if team and team.upper() not in (an, hn) and team.upper() not in adn and team.upper() not in hdn:
            continue
        asc, hsc = a.get("score", "0"), h.get("score", "0")
        status = (comp.get("status") or {}).get("type", {}).get("shortDetail", "")
        lines.append(an + " " + asc + ", " + hn + " " + hsc + " (" + status + ")")
    if not lines:
        return "I don't see a game for that team right now."
    return "; ".join(lines[:10])


def fantasy_vision(question, extra=""):
    if not ai_ready():
        return "That needs the free AI key. Type: set key"
    url = get_link("fantasy football")
    if not url:
        return "I don't have your fantasy football page saved. Say: set fantasy football to <the web address>"
    webbrowser.open(url)
    time.sleep(4)
    return look_at_screen(question + extra)


def front_page_text():
    """Text of the page open in Safari or Chrome's front tab. Returns (url, text) or (None, error)."""
    url = ""
    for script in ('tell application "Safari" to get URL of front document',
                   'tell application "Google Chrome" to get URL of active tab of front window'):
        url = run(["osascript", "-e", script], timeout=8)
        if url.startswith("http"):
            break
    if not url.startswith("http"):
        return None, "I couldn't find an open web page in Safari or Chrome."
    try:
        req = urllib.request.Request(url, headers={"User-Agent": _UAS[1]})
        with urllib.request.urlopen(req, timeout=15) as r:
            html = r.read(600000).decode("utf-8", "ignore")
    except Exception as e:
        return None, "I couldn't read that page (" + str(e)[:60] + ")."
    html = re.sub(r"(?is)<(script|style|noscript|svg)[^>]*>.*?</\1>", " ", html)
    text = re.sub(r"(?s)<[^>]+>", " ", html)
    text = re.sub(r"&[a-z#0-9]+;", " ", text)
    text = re.sub(r"\s+", " ", text).strip()
    return url, text[:6000]


def summarize_page():
    if not ai_ready():
        return "Summaries need the free AI key. Type: set key"
    url, text = front_page_text()
    if not url:
        return text
    if len(text) < 80:
        return "That page has hardly any readable text for me."
    return ask_ai("Summarize this web page in 3 short sentences. Page text: " + text) or NOKEY_FALLBACK


def calendar_today():
    script = ('set d to current date\nset time of d to 0\n'
              'tell application "Calendar"\nset out to ""\n'
              'repeat with cal in calendars\n'
              'try\nset evs to (every event of cal whose start date >= d and start date < (d + 1 * days))\n'
              'repeat with e in evs\nset out to out & (summary of e) & " at " & time string of (start date of e) & "; "\n'
              'end repeat\nend try\nend repeat\nreturn out\nend tell')
    out = run(["osascript", "-e", script], timeout=40)
    if not out:
        return "Nothing on your calendar today, or Calendar access is not allowed yet (System Settings > Privacy > Automation)."
    return "Today you have: " + out.strip().rstrip(";")


LOGIN_PLIST = os.path.join(HOME, "Library", "LaunchAgents", "com.jarvis.assistant.plist")


def login_item(on):
    if sys.platform != "darwin":
        return "That only works on a Mac."
    if not on:
        try:
            os.remove(LOGIN_PLIST)
        except OSError:
            return "Jarvis wasn't set to open at login."
        return "Okay, Jarvis will no longer open at login."
    prog = [sys.executable] if FROZEN else [sys.executable, os.path.abspath(__file__), "--app"]
    try:
        os.makedirs(os.path.dirname(LOGIN_PLIST), exist_ok=True)
        with open(LOGIN_PLIST, "wb") as f:
            plistlib.dump({"Label": "com.jarvis.assistant", "ProgramArguments": prog, "RunAtLoad": True}, f)
    except OSError as e:
        return "I couldn't set that up: " + str(e)[:60]
    return "Done. Jarvis will open by itself the next time you log in. Say: don't open at login to undo it."


def look_at_screen(question):
    if not ai_ready():
        return "I need a free OpenRouter key to read your screen. Type: set key"
    try:
        if os.path.exists(SHOT):
            os.remove(SHOT)
    except OSError:
        pass
    run(["screencapture", "-x", "-t", "jpg", SHOT])
    if not os.path.exists(SHOT) or os.path.getsize(SHOT) == 0:
        return "I couldn't capture the screen. Turn on Screen Recording for this app in System Settings."
    run(["sips", "-Z", "1280", SHOT])
    with open(SHOT, "rb") as f:
        img = base64.b64encode(f.read()).decode()
    try:
        os.remove(SHOT)
    except OSError:
        pass
    content = [{"type": "text", "text": question},
               {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64," + img}}]
    msgs = [{"role": "system", "content": SV_SYSTEM}, {"role": "user", "content": content}]
    try:
        reply, _m = _chat(True, msgs, 450)
    except AIError as e:
        return _ai_error(e)
    return _tidy(reply) if reply else "The free vision models are busy right now. Try again in a minute."


# ----------------------------------------------------------------------------
# features
# ----------------------------------------------------------------------------
TIMERS_FILE = os.path.join(HOME, ".jarvis_timers.json")
MISSED = []


def save_timers():
    try:
        data = [{"due": t[1].isoformat(timespec="seconds"), "message": t[3], "label": t[2]} for t in TIMERS]
        with open(TIMERS_FILE, "w") as f:
            json.dump(data, f)
    except (OSError, IndexError):
        pass


def schedule(seconds, message, label=None, _save=True):
    def fire():
        notify(message)
        speak(message)
        TIMERS[:] = [t for t in TIMERS if t[0] is not timer]
        save_timers()
    timer = threading.Timer(seconds, fire)
    timer.daemon = True
    timer.start()
    TIMERS.append((timer, datetime.now() + timedelta(seconds=seconds), label or message, message))
    if _save:
        save_timers()


def restore_timers():
    """Bring back timers and alarms that were running when Jarvis was closed."""
    try:
        with open(TIMERS_FILE) as f:
            saved = json.load(f)
    except (OSError, ValueError):
        return
    now = datetime.now()
    for t in saved:
        try:
            rem = (datetime.fromisoformat(t["due"]) - now).total_seconds()
        except (KeyError, ValueError):
            continue
        if rem > 0:
            schedule(rem, t.get("message", "Timer"), t.get("label"), _save=False)
        elif rem > -6 * 3600:
            MISSED.append(t.get("label") or "timer")
    save_timers()


def dur(n, u):
    if u.startswith("sec"):
        secs, name = 1, "second"
    elif u.startswith("min"):
        secs, name = 60, "minute"
    else:
        secs, name = 3600, "hour"
    return n * secs, str(n) + " " + name + ("" if n == 1 else "s")


def safe_eval(node):
    if isinstance(node, ast.Expression):
        return safe_eval(node.body)
    if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
        return node.value
    if isinstance(node, ast.BinOp) and type(node.op) in OPS:
        a, b = safe_eval(node.left), safe_eval(node.right)
        if isinstance(node.op, ast.Pow) and abs(b) > 100:
            raise ValueError("too big")
        return OPS[type(node.op)](a, b)
    if isinstance(node, ast.UnaryOp) and type(node.op) in OPS:
        return OPS[type(node.op)](safe_eval(node.operand))
    raise ValueError("bad math")


def calc(text):
    e = " " + text.lower() + " "
    for a, b in ((" plus ", "+"), (" minus ", "-"), (" times ", "*"), (" multiplied by ", "*"),
                 (" divided by ", "/"), (" over ", "/"), (" x ", "*"), (" to the power of ", "**"),
                 (" squared", "**2"), (" cubed", "**3")):
        e = e.replace(a, b)
    e = e.replace("?", "").replace("^", "**").strip()
    if not re.fullmatch(r"[\d\s.+\-*/()%]+", e) or not re.search(r"\d\s*(?:\*\*|[+\-*/%])\s*[\d(]|\*\*\d", e):
        return None
    try:
        r = safe_eval(ast.parse(e, mode="eval"))
    except ZeroDivisionError:
        return "I can't divide by zero"
    except Exception:
        return None
    return "The answer is " + fmt(r)


def norm_unit(u):
    u = u.strip().lower().replace("degrees ", "").replace("degree ", "")
    if u in UNIT_ALIASES:
        return UNIT_ALIASES[u]
    if u.endswith("s") and u[:-1] in UNIT_ALIASES:
        return UNIT_ALIASES[u[:-1]]
    return None


def convert(n, a_raw, b_raw):
    a, b = norm_unit(a_raw), norm_unit(b_raw)
    if a and b:
        if a in ("c", "f", "k") and b in ("c", "f", "k"):
            c = n if a == "c" else (n - 32) * 5 / 9 if a == "f" else n - 273.15
            r = c if b == "c" else c * 9 / 5 + 32 if b == "f" else c + 273.15
        elif a in UNIT_BASE and b in UNIT_BASE and UNIT_BASE[a][0] == UNIT_BASE[b][0]:
            r = n * UNIT_BASE[a][1] / UNIT_BASE[b][1]
        else:
            return None
        return fmt(n) + " " + a_raw.strip() + " is " + fmt(r) + " " + b_raw.strip()
    return currency(n, a_raw, b_raw)


def days_until(s):
    s = s.strip().lower().replace("'", "")
    today = date.today()
    target = None
    for k, (mo, d) in HOLIDAYS.items():
        if k in s:
            target = date(today.year, mo, d)
            break
    if target is None:
        s = re.sub(r"(\d)(st|nd|rd|th)\b", r"\1", s).replace(",", " ")
        s = " ".join(s.split())
        has_year = False
        for f, hy in (("%B %d %Y", True), ("%b %d %Y", True), ("%B %d", False),
                      ("%b %d", False), ("%d %B", False), ("%d %b", False)):
            try:
                p = datetime.strptime(s, f)
            except ValueError:
                continue
            has_year = hy
            try:
                target = date(p.year if hy else today.year, p.month, p.day)
            except ValueError:
                return None
            break
        if target is None:
            return None
        if not has_year and target < today:
            target = date(today.year + 1, target.month, target.day)
    elif target < today:
        target = date(today.year + 1, target.month, target.day)
    n = (target - today).days
    if n == 0:
        return "That's today!"
    return "There are " + str(n) + (" day" if n == 1 else " days") + " until then"


def _read_lines(path):
    if not os.path.exists(path):
        return []
    with open(path) as f:
        return [l.strip() for l in f.read().splitlines() if l.strip()]


def _write_lines(path, items):
    with open(path, "w") as f:
        f.write("\n".join(items) + ("\n" if items else ""))


def show_tasks():
    t = _read_lines(TASKS)
    if not t:
        return "Your to do list is empty"
    for i, x in enumerate(t, 1):
        emit("", "  " + str(i) + ". " + x)
    return ("You have " + str(len(t)) + (" task. " if len(t) == 1 else " tasks. ")
            + ". ".join(str(i) + ", " + x for i, x in enumerate(t, 1)))


def get_volume():
    try:
        return int(run(["osascript", "-e", "output volume of (get volume settings)"]))
    except ValueError:
        return 50


def set_volume(v):
    v = max(0, min(100, v))
    run(["osascript", "-e", "set volume output volume " + str(v)])
    return v


def open_app(name):
    key = name.strip().lower()
    real = APP_ALIASES.get(key, name.strip().title())
    try:
        r = subprocess.run(["open", "-a", real], capture_output=True, text=True)
    except Exception:
        return "I couldn't open " + real
    return "Opening " + real if r.returncode == 0 else "I couldn't find an app called " + real


def briefing():
    now = datetime.now()
    greet = "Good morning" if now.hour < 12 else "Good afternoon" if now.hour < 18 else "Good evening"
    parts = [greet + ", sir. It is " + now.strftime("%I:%M %p").lstrip("0") + " on "
             + now.strftime("%A, %B ") + str(now.day) + "."]
    w = weather()
    if w:
        parts.append("Weather: " + w + ".")
    t = _read_lines(TASKS)
    parts.append("You have " + str(len(t)) + (" task" if len(t) == 1 else " tasks") + " on your list.")
    heads = news(3)
    if heads:
        parts.append("Top headlines. " + ". ".join(heads) + ".")
    return " ".join(parts)


def start_guess():
    GAME["target"], GAME["tries"] = random.randint(1, 50), 0
    GAME["fn"] = guess_step
    return "I'm thinking of a number from 1 to 50. Take a guess, or say give up."


def guess_step(c):
    m = re.search(r"\d+", c)
    if not m:
        return None
    g = int(m.group())
    GAME["tries"] += 1
    if g == GAME["target"]:
        GAME["fn"] = None
        return "Correct! You got it in " + str(GAME["tries"]) + (" try" if GAME["tries"] == 1 else " tries")
    return "Higher" if g < GAME["target"] else "Lower"


def start_trivia():
    q, a = random.choice(TRIVIA)
    GAME["answer"] = a
    GAME["fn"] = trivia_step
    return q


def trivia_step(c):
    GAME["fn"] = None
    if any(a in c for a in GAME["answer"].split("|")):
        return "Correct! Nice one."
    return "Not quite. The answer was " + GAME["answer"].split("|")[0]


def _stat(right):
    st = S.setdefault("stats", {"right": 0, "wrong": 0})
    st["right" if right else "wrong"] = st.get("right" if right else "wrong", 0) + 1
    save_settings()


def study_stats():
    st = S.get("stats") or {}
    r, w = st.get("right", 0), st.get("wrong", 0)
    if r + w == 0:
        return "No quiz answers yet. Say: quiz cards"
    return "You've answered " + str(r + w) + " flashcards: " + str(r) + " right, " + str(w) + " wrong, " + str(round(100 * r / (r + w))) + " percent."


def start_cards():
    cards = [l.split("|||", 1) for l in _read_lines(CARDS) if "|||" in l]
    if not cards:
        return "You have no flashcards yet. Say: add card capital of france = paris"
    q, a = random.choice(cards)
    GAME["answer"] = a
    GAME["fn"] = card_step
    return q + "?"


def card_step(c):
    GAME["fn"] = None
    ans = GAME["answer"].lower()
    words = [w for w in re.findall(r"[a-z0-9]+", ans) if len(w) > 2] or re.findall(r"[a-z0-9]+", ans)
    if words and all(w in c for w in words):
        _stat(True)
        return "Correct! Say quiz cards for another."
    _stat(False)
    return "Not quite. The answer is " + GAME["answer"] + ". Say quiz cards for another."


# ----------------------------------------------------------------------------
# the brain: returns a reply string, "QUIT", or None (not a built-in command)
# ----------------------------------------------------------------------------
def handle(text):
    text = re.sub(r"^(?:hey |ok |okay )?jarvis[,.!]?\s*", "", text.strip(), flags=re.I).strip()
    c = text.lower().replace("\u2019", "'").strip().rstrip("?!. ")
    if not c:
        return "Yes, sir?"
    if c in ("quit", "exit", "bye", "goodbye", "good bye", "close jarvis", "shut down jarvis"):
        return "QUIT"

    if GAME["fn"]:
        if c in ("stop", "stop game", "quit game", "give up", "cancel", "exit game"):
            GAME["fn"] = None
            return "Okay, game over"
        r = GAME["fn"](c)
        if r:
            return r

    if c in SMALL:
        return SMALL[c]
    if re.fullmatch(r"(?:help|commands|features|abilities|skills|show (?:me )?(?:your )?(?:commands|features)|"
                    r"list (?:your )?(?:commands|features)|what (?:all |else )?(?:can|could) you (?:do|help (?:me )?with)|"
                    r"what (?:all )?do you (?:do|know)(?: how to do)?|what are your (?:features|commands|abilities|skills)|"
                    r"what are you (?:able|capable) (?:to do|of|of doing)|what can jarvis do)", c):
        for line in HELP.strip("\n").splitlines():
            emit("", line)
        return ("I can chat about anything, read your screen, manage tasks, timers and alarms, check the "
                "weather and news, do math, control your Mac, and play games. The full list is on your screen.")
    if c in ("good morning", "morning briefing", "briefing", "daily briefing", "brief me"):
        return daily_briefing()
    if c in ("doctor", "run diagnostics", "diagnostics", "check yourself", "self test", "run a self test"):
        lines = doctor()
        for l in lines:
            emit("", l)
        bad = sum(1 for l in lines if l.startswith("[FAIL]"))
        return ("Diagnostics done. Everything looks fine." if not bad else
                "Diagnostics done. " + str(bad) + " thing" + ("" if bad == 1 else "s") +
                " need attention. The details are on your screen.")
    if c in ("repeat that", "say that again", "repeat", "what did you say"):
        return LAST_REPLY[0] or "I haven't said anything yet"
    if c in ("forget everything", "forget our conversation", "new conversation", "reset chat", "clear chat"):
        AI_HISTORY[:] = []
        mem_save()
        return "Chat memory cleared. (I still know your facts. Say: forget my info, to clear those.)"
    r = mem_command(c) or chat_command(c)
    if r:
        return r
    if re.search(r"what (?:can|do) you (?:access|have access to|control|see|open)|can you (?:access|control) my (?:computer|mac)|what (?:can|could) you do on my (?:computer|mac)", c):
        return ("I can open apps, files and folders, read your screen, play and pause music, set timers, "
                "take notes, read your unread email and send emails after you confirm, check your calendar, "
                "summarize a web page, make flashcards and tidy your downloads. I can't delete files or run "
                "secret commands. Say help for the full list.")
    if PENDING["email"]:
        if re.fullmatch(r"(?:yes )?(?:send it|send|confirm|yes|yes send it|go ahead|do it)", c):
            return send_pending_email()
        if re.fullmatch(r"cancel|no|never ?mind|cancel (?:the |that )?e-?mail|don'?t send(?: it)?", c):
            PENDING["email"] = None
            return "Okay, I won't send it."
    m = re.match(r"^(?:save|add) (?:contact|email) (.+?) (?:as|is|=) (\S+@\S+|\S+ at \S+)$", text.strip().rstrip("."), re.I)
    if m:
        addr = _resolve_recipient(m.group(2))
        if addr and "@" in addr:
            S.setdefault("contacts", {})[m.group(1).lower().strip()] = addr
            save_settings()
            return "Saved " + m.group(1).strip() + " as " + addr + "."
        return "That doesn't look like an email address."
    m = re.match(r"^(?:send |write |compose )?(?:an? )?(?:e-?mail|message)(?: to)? (.+?)(?: (?:about|subject|with subject) (.+?))?"
                 r" (?:saying|that says|and say|with (?:the )?message|body) (.+)$", text.strip(), re.I)
    if m:
        body = m.group(3)
        subj = m.group(2) or (body[:40] + ("..." if len(body) > 40 else ""))
        return prepare_email(m.group(1), subj, body)
    m = re.fullmatch(r"(?:open|show|go to)(?: my)? (?:the )?(downloads|documents|desktop|pictures|photos|music|movies|home)(?: folder)?", c)
    if m:
        return open_path("folder", m.group(1))
    m = re.match(r"^(?:open|find and open|show me) (?:the |my )?(file|document|folder|pdf|presentation|spreadsheet) (?:called |named )?(.+)$", c)
    if m:
        kind = "folder" if m.group(1) == "folder" else "file"
        return open_path(kind, m.group(2))
    if re.fullmatch(r"(?:check|read|show)(?: me)?(?: my)?(?: new| unread)? (?:e-?mails?|inbox|mail)|(?:do i have|any)(?: any)?(?: new| unread)? e-?mails?(?: today)?|what(?:'s| is) in my inbox", c):
        return email_summary()
    if re.fullmatch(r"(?:open|launch|go to|start)(?: the)?(?: apple)? mail app|(?:open|launch|start) (?:the )?mail", c):
        run(["open", "-a", "Mail"])
        return "Opening the Mail app."
    if re.fullmatch(r"(?:(?:open|launch|go to|show|pull up|take me to|get)(?: me)?(?: to)? )?(?:my |the )?(?:school |work |personal )?(?:g-?mail|gmial|gamil|gmal|gmaill|gmali|e-?mail|emial|emal|emails|inbox|mail)(?: (?:inbox|website|site|page|account))?(?: please)?", c):
        return open_link("gmail" if re.search(r"\bg-?m|\bgam", c) else "email")
    m = re.fullmatch(r"(?:open|launch|go to|start)(?: my)? (ixl|vericross|gmail|spotify web|spotify website)(?: website| site| page)?", c)
    if m:
        return open_link(m.group(1).replace("spotify website", "spotify web"))
    m = re.match(r"^(?:set|save|change) (?:my )?([a-z ]{2,20}?)(?: link| address| url| website)? (?:to|as) (https?://\S+|www\.\S+|\S+\.\S+)$", text.strip().rstrip("."), re.I)
    if m and m.group(1).lower().strip() not in ("voice", "theme", "key"):
        url = m.group(2)
        if not url.lower().startswith("http"):
            url = "https://" + url
        S.setdefault("links", {})[m.group(1).lower().strip()] = url
        save_settings()
        return "Saved. Say: open " + m.group(1).lower().strip() + ", any time."
    m = re.fullmatch(r"play (.+) on spotify", c)
    if m:
        run(["open", "spotify:search:" + urllib.parse.quote(m.group(1))])
        return "Opening Spotify search for " + m.group(1) + ". Press play on the top result."
    if re.fullmatch(r"(?:open|launch|start) spotify", c):
        run(["open", "-a", "Spotify"])
        return "Opening Spotify."
    m = re.match(r"^(?:make|create|write|generate)(?: me)?(?: some)? (?:flash ?)?cards? (?:about|on|for) (.+)$", c)
    if m:
        topic = m.group(1).strip()
        if re.fullmatch(r"(?:my |the )?clipboard", topic):
            clip = run(["pbpaste"])
            return make_cards(clip, True) if clip else "Your clipboard is empty"
        return make_cards(topic)
    if re.fullmatch(r"(?:make|create) (?:me )?(?:some )?(?:flash ?)?cards? from (?:my |the )?clipboard", c):
        clip = run(["pbpaste"])
        return make_cards(clip, True) if clip else "Your clipboard is empty"
    m = re.match(r"^quiz me on (?!cards$|my cards$)(.+)$", c)
    if m and ai_ready():
        r = make_cards(m.group(1).strip())
        return start_cards() if r.startswith("Made") else r
    if re.fullmatch(r"(?:organi[sz]e|clean up|tidy)(?: up)? my downloads(?: folder)?", c):
        return organize_downloads()
    if re.fullmatch(r"(?:focus mode|focus mode on|start focus mode|i need to focus)", c):
        return focus_mode(True)
    if re.fullmatch(r"(?:focus mode off|stop focus mode|end focus mode|i'm done focusing)", c):
        return focus_mode(False)
    if re.fullmatch(r"(?:explain|what(?:'s| is)) (?:this|the|that) error|what(?:'s| is) wrong with my code|read (?:this|my screen) (?:out loud|aloud|to me)|help me with this (?:problem|question|homework)", c):
        return look_at_screen(text)
    if re.fullmatch(r"(?:open|start|launch) (?:jarvis )?(?:at|on) (?:login|startup|start up)|open (?:jarvis )?when i log in", c):
        return login_item(True)
    if re.fullmatch(r"(?:don'?t|do not|stop|never) (?:open|start|launch) (?:jarvis )?(?:at|on) (?:login|startup|start up)", c):
        return login_item(False)
    if re.fullmatch(r"(?:summarize|summarise|sum up|tldr) (?:this |the )?(?:page|web ?page|website|site|tab|article)", c):
        return summarize_page()
    if re.fullmatch(r"(?:what(?:'s| is)(?: on)? (?:my )?(?:calendar|schedule)(?: today)?|my (?:calendar|schedule)(?: today)?|what do i have today)", c):
        return calendar_today()
    m = re.fullmatch(r"(?:nfl|football) scores?(?: for (.+))?|(?:what are the )?(?:nfl|football) scores|"
                    r"how (?:did|are) the ([a-z]+(?: [a-z]+)?) (?:do|doing)(?: today)?", c)
    if m:
        team = next((g for g in m.groups() if g), None)
        return nfl_scores(team)
    if re.fullmatch(r"(?:how (?:is|are) my fantasy(?: football)? team(?: doing)?|what(?:'s| is) my fantasy(?: football)? score|"
                    r"how many points (?:did|do) i (?:get|have)(?: in fantasy)?(?: football)?)", c):
        return fantasy_vision("Look at this fantasy football page. Tell me my team's total points this week in one short sentence.")
    if re.fullmatch(r"who am i (?:playing|up against|going against)(?: this week)?(?: in fantasy(?: football)?)?|"
                    r"who(?:'s| is) (?:winning|beating me)(?: in fantasy(?: football)?)?|"
                    r"(?:what(?:'s| is)|show me) my fantasy(?: football)? matchup", c):
        return fantasy_vision("Look at this fantasy football matchup page. Tell me who I'm playing this week, "
                              "both scores, and who is currently ahead, in 2 short sentences.")
    if re.fullmatch(r"how did (?:all (?:my )?(?:fantasy )?teams?|everyone)(?: in my league)? do(?: this week)?|"
                    r"fantasy(?: football)? (?:standings|rankings|rank|league)|"
                    r"who(?:'s| is) ranked where(?: in (?:my )?fantasy(?: football)?)?|"
                    r"what(?:'s| is) the fantasy(?: football)? standings", c):
        return fantasy_vision("Look at this fantasy football page. If it shows a list of teams with scores, standings "
                              "or rank, tell me each team's rank or score in order, in plain sentences, no lists. "
                              "If this isn't a standings page, tell me to open the league standings page and ask again.")

    # vision
    if VISION_RX.search(c) and not VISION_SKIP.search(c):
        return look_at_screen(text)
    m = re.match(r"^(?:summarize|summarise|explain|translate|fix|proofread) (?:my |the )?clipboard$", c)
    if m:
        clip = run(["pbpaste"])
        if not clip:
            return "Your clipboard is empty"
        if not ai_ready():
            return "I need a free OpenRouter key for that. Type: set key"
        verb = c.split()[0]
        return ask_ai(verb.capitalize() + " this: " + clip[:3000]) or NOKEY_FALLBACK

    # voice and looks
    m = re.match(r"^(?:set|change|switch|use)(?: the| your)? voice(?: to)?\s+(.+)$", c)
    if m:
        return set_voice(m.group(1))
    if re.fullmatch(r"(?:turn on|enable|start|activate)(?: the)? wake word|wake word on|wake word|hands free mode|hands free", c):
        return start_wake()
    if re.fullmatch(r"(?:turn off|disable|stop|deactivate)(?: the)? wake word|wake word off|hands free off", c) \
            or (c in ("stop listening", "stop wake word listening") and WAKE["on"]):
        return stop_wake()
    if c in ("set key", "set my key", "change key", "change my key", "new key", "enter key", "set api key", "set openrouter key"):
        if KEY_PROMPT[0]:
            KEY_PROMPT[0]()
            return "Opening the key box."
        return "In Terminal run: python3 jarvis.py --set-key"
    m = re.match(r"^(?:when|if) i say (.+?),? (?:then )?(?:you )?(?:should |please )?(?:do |run |just )?(.+)$", text.strip().rstrip("."), re.I)
    if m and m.group(1).strip() and m.group(2).strip():
        trig = m.group(1).strip().lower().strip("\"'")
        S.setdefault("aliases", {})[trig] = m.group(2).strip()
        save_settings()
        return "Done. When you say " + trig + ", I'll run: " + m.group(2).strip() + "."
    if c in ("list my shortcuts", "my shortcuts", "show my shortcuts", "list shortcuts"):
        al = S.get("aliases") or {}
        return ("Your shortcuts: " + ". ".join(k + " means " + v for k, v in al.items())) if al else "You have no shortcuts yet. Say: when I say gm, give me my briefing"
    m = re.match(r"^(?:forget|delete|remove) (?:the )?shortcut (.+)$", c)
    if m:
        gone = (S.get("aliases") or {}).pop(m.group(1).strip().strip("\"'"), None)
        save_settings()
        return "Shortcut removed." if gone else "I don't have that shortcut."
    if c in ("my study stats", "study stats", "my score", "how am i doing", "my quiz score", "flashcard stats"):
        return study_stats()
    if re.fullmatch(r"(?:good morning|morning|daily briefing|morning briefing|brief me|give me (?:my |the )?(?:daily |morning )?briefing|what(?:'s| is) my day (?:look )?like|start my day)(?: jarvis)?", c):
        return daily_briefing()
    if c in ("settings", "open settings", "show settings", "preferences", "open preferences") :
        return ui_action("settings", "Opening settings.")
    if re.fullmatch(r"(?:stay on top|keep (?:this |the )?window on top|always on top|float)", c):
        return ui_action("top_on", "I'll stay on top of your other windows.")
    if re.fullmatch(r"(?:stop staying on top|stay (?:in )?back|not on top|don'?t stay on top|stop floating)", c):
        return ui_action("top_off", "Okay, I'll stay behind your other windows.")
    if c in ("set groq key", "set backup key", "add groq key", "change groq key", "set my groq key"):
        if KEY_PROMPT[1]:
            KEY_PROMPT[1]()
            return "Opening the Groq key box. Get a free key at console.groq.com > API Keys."
        return "Set the GROQ_API_KEY setting, or use the app: say set groq key."
    if c in ("wake word status", "is the wake word on"):
        return "The wake word is " + ("on" if WAKE["on"] else "off")
    if c in ("list voices", "what voices do you have"):
        return list_voices()
    if c in ("test voice", "test your voice", "say something", "test sound", "can you hear me"):
        return "Good day, sir. If you can hear this, my voice is working."
    if c in ("talk faster", "speak faster", "faster"):
        return set_rate(25)
    if c in ("talk slower", "speak slower", "slower"):
        return set_rate(-25)
    if c in ("stop talking", "be quiet", "shut up", "stop", "silence"):
        stop_talking()
        return "Quiet now."
    if c in ("quiet mode", "quiet mode on", "text only mode", "mute your voice"):
        S["quiet"] = True
        save_settings()
        return "Quiet mode on. I will only show text. Say talk again to hear me."
    if c in ("talk again", "quiet mode off", "unmute your voice", "voice on"):
        S["quiet"] = False
        save_settings()
        return "Voice is back on"
    if c in ("list themes", "themes", "what themes do you have"):
        return "My themes are " + ", ".join(THEMES) + "."
    for name in THEMES:
        if re.fullmatch(r"(?:set |change |switch )?(?:the )?(?:theme|color|colour)(?: to)? " + name
                        + "|" + name + r" (?:theme|mode)", c):
            S["theme"] = name
            save_settings()
            return "Theme set to " + name + ". The colors show in the window."

    # timers, alarms, reminders
    if "pomodoro" in c:
        schedule(25 * 60, "Focus time is over. Take a five minute break.")
        schedule(30 * 60, "Break is over. Ready for another round?")
        return "Pomodoro started. Focus for 25 minutes."
    if c in ("list timers", "my timers", "show timers", "what timers do i have"):
        if not TIMERS:
            return "You have no timers running"
        now = datetime.now()
        parts = []
        for _t, due, label, *_r in TIMERS:
            parts.append(label + " in " + fmt_time(max(0, (due - now).total_seconds())))
        for p in parts:
            emit("", "  " + p)
        return "You have " + str(len(TIMERS)) + " running. " + ". ".join(parts)
    if c in ("cancel timers", "cancel all timers", "stop timers", "clear timers", "cancel timer"):
        for t, *_r in TIMERS:
            t.cancel()
        n = len(TIMERS)
        TIMERS[:] = []
        save_timers()
        return "Cancelled " + str(n) + (" timer" if n == 1 else " timers")
    m = (re.search(r"timer (?:for )?(\d+)\s*" + UNIT_RX, c)
         or re.search(r"(\d+)[ -]*" + UNIT_RX + r"\s+timer", c))
    if m:
        secs, label = dur(int(m.group(1)), m.group(2))
        schedule(secs, "Your " + label + " timer is done", label + " timer")
        return "Timer set for " + label
    m = re.search(r"remind me in (\d+)\s*" + UNIT_RX + r"\s+to\s+(.+)", c)
    if m:
        secs, label = dur(int(m.group(1)), m.group(2))
        schedule(secs, "Reminder: " + m.group(3), "reminder to " + m.group(3))
        return "Okay, I will remind you in " + label
    m = re.search(r"remind me to\s+(.+?)\s+in (\d+)\s*" + UNIT_RX + r"$", c)
    if m:
        secs, label = dur(int(m.group(2)), m.group(3))
        schedule(secs, "Reminder: " + m.group(1), "reminder to " + m.group(1))
        return "Okay, I will remind you in " + label
    m = re.search(r"alarm (?:for|at) (\d{1,2})(?::(\d{2}))?\s*(am|pm)?", c.replace(".", ""))
    if m:
        h, mi, ap = int(m.group(1)), int(m.group(2) or 0), m.group(3)
        if ap == "pm" and h < 12:
            h += 12
        if ap == "am" and h == 12:
            h = 0
        if h > 23 or mi > 59:
            return "That time doesn't look right"
        now = datetime.now()
        target = now.replace(hour=h, minute=mi, second=0, microsecond=0)
        if target <= now:
            target += timedelta(days=1)
        schedule((target - now).total_seconds(), "Your alarm is going off", "alarm")
        return "Alarm set for " + target.strftime("%I:%M %p").lstrip("0")
    if c in ("start stopwatch", "start the stopwatch", "stopwatch start"):
        SW[0] = time.time()
        return "Stopwatch started"
    if c in ("lap", "stopwatch", "check stopwatch", "stop stopwatch", "stop the stopwatch"):
        if SW[0] is None:
            return "The stopwatch isn't running"
        out = fmt_time(time.time() - SW[0])
        if c.startswith("stop"):
            SW[0] = None
            return "Stopped at " + out
        return "Elapsed time is " + out

    # tasks, notes, cards
    m = re.match(r"^(?:add task|add a task|add to do|todo|to do|add to my list|add to my to do list)\s+(.+)$",
                 text.rstrip("?!. "), re.I)
    if m:
        items = _read_lines(TASKS) + [m.group(1)]
        _write_lines(TASKS, items)
        return "Added. You now have " + str(len(items)) + (" task" if len(items) == 1 else " tasks")
    m = re.match(r"^(?:done|complete|finish|remove|delete)(?: task)?\s+(\d+)$", c)
    if m:
        items = _read_lines(TASKS)
        i = int(m.group(1))
        if 1 <= i <= len(items):
            gone = items.pop(i - 1)
            _write_lines(TASKS, items)
            return "Done. Removed " + gone
        return "I don't have a task number " + str(i)
    if c in ("my tasks", "show my tasks", "show tasks", "list tasks", "to do list", "my to do list",
             "show my to do list", "what do i need to do", "tasks"):
        return show_tasks()
    if c in ("clear tasks", "clear my tasks", "clear my to do list"):
        _write_lines(TASKS, [])
        return "Your task list is cleared"
    m = re.match(r"^add (?:a )?(?:flash)?card\s+(.+?)\s*(?:=|->| is )\s*(.+)$", text.rstrip("?!. "), re.I)
    if m:
        items = _read_lines(CARDS) + [m.group(1).strip() + "|||" + m.group(2).strip()]
        _write_lines(CARDS, items)
        return "Card saved. You have " + str(len(items)) + (" card" if len(items) == 1 else " cards")
    if c in ("quiz cards", "quiz me on cards", "flashcards", "study cards", "quiz me on my cards"):
        return start_cards()
    if c.startswith("remember "):
        with open(NOTES, "a") as f:
            f.write(datetime.now().strftime("%Y-%m-%d %H:%M") + "  " + text[9:] + "\n")
        return "Got it, I saved that"
    if "clear my notes" in c:
        if os.path.exists(NOTES):
            os.remove(NOTES)
        return "Your notes are cleared"
    if re.fullmatch(r"(?:what are |show |read )?(?:me )?my notes|what did i tell you", c):
        if os.path.exists(NOTES):
            return "Here are your notes. " + open(NOTES).read().replace("\n", ". ")
        return "You have no notes yet"

    # conversions and math
    m = re.match(r"^(?:convert|what is|what's|whats|how much is|how many)\s+(-?[\d.]+)\s*([a-z ]+?)\s+(?:to|into|in)\s+([a-z ]+)$", c)
    if m:
        try:
            r = convert(float(m.group(1)), m.group(2), m.group(3))
        except ValueError:
            r = None
        if r:
            return r
    m = re.search(r"(\d+(?:\.\d+)?)\s*(?:percent|%)\s+of\s+(\d+(?:\.\d+)?)", c)
    if m:
        return "That is " + fmt(float(m.group(1)) * float(m.group(2)) / 100)
    m = re.search(r"square root of (\d+(?:\.\d+)?)", c)
    if m:
        return "The square root is " + fmt(math.sqrt(float(m.group(1))))
    m = re.match(r"^(?:calculate|calc|what is|what's|whats|how much is)\s+(.+)$", c)
    if m:
        ans = calc(m.group(1))
        if ans:
            return ans
    m = re.match(r"^(?:define|definition of)\s+(\w+)$|^what does (\w+) mean$", c)
    if m:
        return define(m.group(1) or m.group(2))
    m = re.match(r"^(?:how do you )?spell\s+(\w+)$", c)
    if m:
        return ", ".join(m.group(1).upper())
    m = re.search(r"(?:how many days|days|how long) (?:until|till|to|before) (?:the )?(.+)$", c)
    if m:
        r = days_until(m.group(1))
        if r:
            return r
    if re.search(r"\b(generate|make|create|new)\b.*\bpassword\b", c) or c == "password":
        n = re.search(r"(\d+)\s*(?:character|char|digit|letter)", c)
        length = max(8, min(64, int(n.group(1)))) if n else 16
        pw = "".join(secrets.choice(string.ascii_letters + string.digits + "!@#$%^&*-_") for _ in range(length))
        emit("", "Password: " + pw)
        try:
            subprocess.run(["pbcopy"], input=pw, text=True)
        except Exception:
            pass
        return "I made a " + str(length) + " character password. It's on your screen and copied to your clipboard."

    # info
    m = re.search(r"\btime (?:is it )?in ([a-z .]+)$", c)
    if m:
        city = m.group(1).strip()
        if city in CITIES and ZoneInfo:
            try:
                now = datetime.now(ZoneInfo(CITIES[city]))
                return "In " + city.title() + " it is " + now.strftime("%I:%M %p").lstrip("0") + " on " + now.strftime("%A")
            except Exception:
                return "I couldn't get the time there"
        return "I don't know that city yet"
    if re.fullmatch(r"(?:what(?:'s| is)(?: the)?(?: current)? time(?: is it)?|what time is it|tell me the time|"
                    r"time|current time|the time)(?: right now| now| please)?", c):
        return "It is " + datetime.now().strftime("%I:%M %p").lstrip("0")
    if re.fullmatch(r"(?:what(?:'s| is)(?: the)?(?: today's)? date(?: today)?|what day is it(?: today)?|"
                    r"today's date|the date|date|what is today|what day is today)", c):
        now = datetime.now()
        return "Today is " + now.strftime("%A, %B ") + str(now.day)
    m = re.fullmatch(r"(?:(?:what(?:'s| is)|how(?:'s| is)) (?:the )?)?(?:weather|forecast)(?: like)? (?:in|for|at) (.+)", c)
    if m:
        return weather(m.group(1)) or "I couldn't get the weather"
    if re.fullmatch(r"(?:(?:what(?:'s| is)|how(?:'s| is)) (?:the )?)?(?:weather|forecast)(?: like)?"
                    r"(?: today| outside| right now| now| here)?", c):
        return weather() or "I couldn't get the weather"
    if c in ("news", "the news", "headlines", "whats in the news", "what's in the news", "tell me the news"):
        heads = news(5)
        if not heads:
            return "I couldn't get the news right now"
        for h in heads:
            emit("", "  - " + h)
        return "Here are the top headlines. " + ". ".join(heads)
    if (c in ("battery", "check battery", "my battery", "how much battery do i have")
            or re.search(r"\b(battery level|battery percent\w*|battery status|how much battery|battery life)\b", c)):
        out = run(["pmset", "-g", "batt"])
        m = re.search(r"(\d+)%", out)
        if not m:
            return "I couldn't read the battery"
        return "Battery is at " + m.group(1) + " percent" + (" and plugged in" if "AC Power" in out else "")
    if re.search(r"how much (storage|space)|disk space", c):
        free = shutil.disk_usage(HOME).free / 1e9
        return "You have about " + str(round(free)) + " gigabytes free"
    if c in ("system info", "about my mac", "my mac", "mac info"):
        chip = run(["sysctl", "-n", "machdep.cpu.brand_string"])
        mem = int(run(["sysctl", "-n", "hw.memsize"]) or 0) / 1e9
        return "macOS " + run(["sw_vers", "-productVersion"]) + ", " + chip + ", " + str(round(mem)) + " gigabytes of memory"
    if c in ("my ip", "my ip address", "what is my ip", "what is my ip address", "whats my ip", "what's my ip"):
        try:
            return "Your public IP address is " + urllib.request.urlopen("https://api.ipify.org", timeout=8).read().decode()
        except Exception:
            return "I couldn't get your IP address"
    if re.fullmatch(r"(?:read |what(?:'s| is) (?:in )?)?my clipboard|read (?:the )?clipboard|what(?:'s| is) on my clipboard", c):
        clip = run(["pbpaste"])
        return "Your clipboard says: " + clip[:300] if clip else "Your clipboard is empty"

    # mac controls
    if c in ("volume up", "louder", "turn it up", "turn up the volume", "turn the volume up"):
        return "Volume " + str(set_volume(get_volume() + 10))
    if c in ("volume down", "quieter", "turn it down", "turn down the volume", "turn the volume down"):
        return "Volume " + str(set_volume(get_volume() - 10))
    m = re.search(r"volume (?:to )?(\d+)", c)
    if m:
        return "Volume set to " + str(set_volume(int(m.group(1))))
    if c in ("what is the volume", "whats the volume", "what's the volume"):
        return "The volume is " + str(get_volume())
    if c in ("mute", "unmute"):
        run(["osascript", "-e", "set volume output muted " + ("true" if c == "mute" else "false")])
        return "Muted" if c == "mute" else "Unmuted"
    if re.search(r"\b(take|capture|grab)\b.*\bscreen ?shot\b", c) or c in ("screenshot", "screen shot"):
        path = os.path.join(DESK, "jarvis_shot_" + datetime.now().strftime("%H%M%S") + ".png")
        run(["screencapture", path])
        return "Screenshot saved to your Desktop"
    m = re.fullmatch(r"(play|pause|resume|next|skip|previous)(?: the)? spotify(?: music| song)?", c)
    if m:
        verb = {"play": "play", "resume": "play", "pause": "pause", "next": "next track",
                "skip": "next track", "previous": "previous track"}[m.group(1)]
        run(["osascript", "-e", 'tell application "Spotify" to ' + verb])
        return "Okay"
    if c in MUSIC:
        run(["osascript", "-e", 'tell application "Music" to ' + MUSIC[c]])
        return "Okay"
    if c in ("lock screen", "lock my screen", "lock the screen", "lock"):
        run(["pmset", "displaysleepnow"])
        return "Locking the screen"
    if c in ("sleep my mac", "sleep computer", "put the computer to sleep", "sleep my computer"):
        run(["pmset", "sleepnow"])
        return "Going to sleep"
    if c in ("dark mode", "light mode", "switch to dark mode", "switch to light mode"):
        val = "true" if "dark" in c else "false"
        run(["osascript", "-e",
             'tell application "System Events" to tell appearance preferences to set dark mode to ' + val])
        return "Switching to " + ("dark" if val == "true" else "light") + " mode"

    # open, search, maps
    m = re.match(r"^(?:go to|open website|open site|visit)\s+(\S+)$", c)
    if m:
        u = m.group(1)
        if not u.startswith("http"):
            u = "https://" + (u if "." in u else u + ".com")
        webbrowser.open(u)
        return "Opening " + m.group(1)
    if c.startswith("open "):
        name = c[5:].replace(" folder", "").strip()
        if name in FOLDERS:
            run(["open", os.path.expanduser(FOLDERS[name])])
            return "Opening " + name
        if "." in name and " " not in name:
            webbrowser.open("https://" + name)
            return "Opening " + name
        if name in SITE_ALIASES:
            webbrowser.open(SITE_ALIASES[name])
            nice = {"youtube": "YouTube", "x": "X"}.get(name, name.title())
            return "Opening " + nice + "."
        if get_link(name):
            return open_link(name)
        return open_app(name)
    if c.startswith(("close ", "quit ")):
        name = text.split(" ", 1)[1].strip()
        app = APP_ALIASES.get(name.lower(), name.title()).replace('"', "")
        run(["osascript", "-e", 'tell application "' + app + '" to quit'])
        return "Closing " + app
    m = re.match(r"^find (?:a )?files? (?:named |called )?(.+)$", c)
    if m:
        hits = run(["mdfind", "-name", m.group(1)]).splitlines()
        if not hits:
            return "I couldn't find any files called " + m.group(1)
        for h in hits[:5]:
            emit("", "  " + h)
        return "I found " + str(len(hits)) + " matches. The first few are on your screen."
    if c.startswith(("search ", "google ")):
        q = text.split(" ", 1)[1]
        webbrowser.open("https://www.google.com/search?q=" + urllib.parse.quote_plus(q))
        return "Searching now"
    if c.startswith("youtube "):
        webbrowser.open("https://www.youtube.com/results?search_query=" + urllib.parse.quote_plus(text[8:]))
        return "Searching YouTube"
    m = re.match(r"^play (.+?)(?: on youtube)?$", c)
    if m:
        webbrowser.open("https://www.youtube.com/results?search_query=" + urllib.parse.quote_plus(m.group(1)))
        return "Searching YouTube for " + m.group(1)
    m = re.match(r"^(?:directions to|navigate to|take me to)\s+(.+)$", c)
    if m:
        webbrowser.open("https://www.google.com/maps/dir/?api=1&destination=" + urllib.parse.quote_plus(m.group(1)))
        return "Getting directions"
    m = re.match(r"^(?:map of|show me a map of|show map of)\s+(.+)$|^find (.+) near me$", c)
    if m:
        q = m.group(1) or (m.group(2) + " near me")
        webbrowser.open("https://www.google.com/maps/search/" + urllib.parse.quote(q))
        return "Opening maps"
    if c.startswith("say "):
        return text[4:]
    if c.startswith("copy "):
        try:
            subprocess.run(["pbcopy"], input=text[5:], text=True)
        except Exception:
            return "I couldn't copy that"
        return "Copied to your clipboard"

    # games and fun
    if c in ("guess the number", "play guess the number", "number game"):
        return start_guess()
    if c in ("trivia", "quiz me", "ask me a question", "play trivia"):
        return start_trivia()
    if c in ("riddle", "tell me a riddle", "give me a riddle"):
        q, GAME["riddle"] = random.choice(RIDDLES)
        return q + " Say answer when you give up."
    if c in ("answer", "riddle answer", "what is the answer", "i give up") and GAME["riddle"]:
        a, GAME["riddle"] = GAME["riddle"], ""
        return "The answer is: " + a
    m = re.search(r"rock paper scissors\s*(rock|paper|scissors)?", c)
    if m:
        if not m.group(1):
            return "Say rock paper scissors followed by rock, paper, or scissors"
        me, you = random.choice(["rock", "paper", "scissors"]), m.group(1)
        beats = {"rock": "scissors", "paper": "rock", "scissors": "paper"}
        if me == you:
            return "I chose " + me + ". It's a tie!"
        return "I chose " + me + ". " + ("You win!" if beats[you] == me else "I win!")
    m = re.match(r"^(?:choose|pick|decide)(?: between)?\s+(.+ or .+)$", c)
    if m:
        return "I choose " + random.choice(re.split(r"\s+or\s+|,\s*", m.group(1)))
    if "8 ball" in c or "eight ball" in c:
        return random.choice(BALL)
    m = re.search(r"roll (?:a |an )?(\d*)\s*d(\d+)\b", c)
    if m:
        n = max(1, min(20, int(m.group(1) or 1)))
        sides = max(2, min(1000, int(m.group(2))))
        rolls = [random.randint(1, sides) for _ in range(n)]
        return "You rolled " + ", ".join(map(str, rolls)) + (" for a total of " + str(sum(rolls)) if n > 1 else "")
    if re.search(r"roll (a |the )?(dice|die)", c):
        return "You rolled a " + str(random.randint(1, 6))
    m = re.search(r"random number(?: between)? (\d+) (?:and|to) (\d+)", c)
    if m:
        lo, hi = sorted((int(m.group(1)), int(m.group(2))))
        return "Your number is " + str(random.randint(lo, hi))
    if re.fullmatch(r"(?:give me |pick )?(?:a )?random number", c):
        return "Your number is " + str(random.randint(1, 100))
    if "flip a coin" in c or "coin flip" in c:
        return random.choice(["Heads", "Tails"])
    if re.search(r"\b(?:tell me|give me|got|know) (?:a |another )?joke|^(?:another )?joke$|make me laugh", c):
        return random.choice(JOKES)
    if re.fullmatch(r"(?:(?:tell me|give me|say) )?(?:a |another )?(?:random |fun )?fact", c):
        return random.choice(FACTS)
    if c in ("motivate me", "quote", "inspire me", "give me a quote", "motivation"):
        return random.choice(QUOTES)

    # Wikipedia only when there is no AI key (the AI answers these better)
    m = re.match(r"^(?:who is|who was|who's|tell me about|what is an?|what are|what is the|wiki|wikipedia)\s+(.+)$", c)
    if m and not ai_ready():
        return wiki(m.group(1))
    return None


KEY_PROMPT = [None, None]


def save_groq_key(k):
    k = (k or "").strip().strip('"').strip("'")
    if len(k) < 20:
        return "That looks too short, so nothing was saved."
    with open(GKEYFILE, "w") as f:
        f.write(k)
    os.chmod(GKEYFILE, 0o600)
    _GM["text"], _GM["vision"] = [], []
    try:
        _groq("/models", timeout=10)
        return "Groq key saved and accepted. I'll use it whenever the other free models are busy."
    except AIError as e:
        return "Saved, but Groq said code " + str(e.code) + ". If chat fails, make a new key."


def save_key(k):
    """Save an OpenRouter key. Returns a message for the user."""
    k = (k or "").strip().strip('"').strip("'")
    if len(k) < 20:
        return "That looks too short, so nothing was saved."
    with open(KEYFILE, "w") as f:
        f.write(k)
    os.chmod(KEYFILE, 0o600)
    try:
        r = _or("/auth/key", timeout=10)
        if (r.get("data") or {}).get("is_management_key"):
            return "Saved, but that is a Management key and it can't chat. Make a regular API key and say: set key"
        return "Key saved and OpenRouter accepted it. AI chat and screen reading are on."
    except AIError as e:
        return "Saved, but OpenRouter check failed (code " + str(e.code) + "). If chat fails, make a new key."


def respond(text):
    """Always returns a string (or 'QUIT')."""
    try:
        al = S.get("aliases") or {}
        key = text.strip(" .!?,").lower()
        if key in al:
            text = al[key]
        r = handle(text)
        if r is None:
            r = ask_ai(text)
        if r is None:
            r = NOKEY_FALLBACK
    except Exception as e:
        r = "Something went wrong with that command. " + str(e)[:100]
    if r != "QUIT":
        log_chat(text, r)
    return r


# ----------------------------------------------------------------------------
# doctor: shows exactly what works
# ----------------------------------------------------------------------------
def doctor():
    out = []

    def ok(t):
        out.append("[ OK ] " + t)

    def bad(t):
        out.append("[FAIL] " + t)

    def info(t):
        out.append("[ .. ] " + t)

    info("Python " + sys.version.split()[0] + " on " + sys.platform)
    try:
        import tkinter
        ok("Window support (tkinter " + str(tkinter.TkVersion) + ")")
    except Exception as e:
        bad("Window support missing (" + str(e)[:60] + "). Use: python3 jarvis.py --text")

    vs = run(["osascript", "-e", "get volume settings"])
    m = re.search(r"output volume:(\d+)", vs)
    if m:
        vol = int(m.group(1))
        if "output muted:true" in vs or vol == 0:
            bad("Mac sound is muted or at 0. Say: unmute, or: volume 50")
        else:
            ok("Mac volume is " + str(vol))
    else:
        info("Couldn't read the Mac volume (normal if this isn't a Mac)")

    names = [n for n, _l in installed_voices()]
    if not names:
        bad("No voices found. The say command may be missing.")
    else:
        want = S.get("voice", "Daniel")
        if CUR_VOICE[0]:
            ok("Using voice: " + CUR_VOICE[0])
            if want.lower() != CUR_VOICE[0].lower():
                info("Your saved voice '" + want + "' isn't installed, so I picked " + CUR_VOICE[0])
        else:
            info("Using the Mac's default voice")
        good, err = _say("Sound check. If you can hear this, my voice works.", CUR_VOICE[0])
        if good:
            ok("The say command ran. Did you hear it? If not, check volume and output device.")
        else:
            bad("The say command failed: " + (err or "unknown"))

    if VOICE_OK:
        ok("Voice add-ons installed")
        try:
            d = sd.rec(int(16000 * 0.4), samplerate=16000, channels=1, dtype="int16")
            sd.wait()
            level = float(np.abs(d).mean())
            ok("Microphone opened (level " + str(int(level)) + ")")
            if level == 0:
                info("Mic level is exactly 0. Allow the microphone in System Settings > Privacy & Security.")
        except Exception as e:
            bad("Microphone failed: " + str(e)[:80] + ". Allow it in System Settings > Privacy & Security > Microphone.")
        try:
            flac = sr.audio.get_flac_converter()
            r = subprocess.run([flac, "--version"], capture_output=True, timeout=10)
            if r.returncode == 0:
                ok("Speech encoder (FLAC) works")
            else:
                bad("Speech encoder failed. On an Apple Silicon Mac run: softwareupdate --install-rosetta")
        except Exception as e:
            bad("Speech encoder (FLAC) can't run: " + str(e)[:60] + ". Apple Silicon Mac? Run: softwareupdate --install-rosetta")
    else:
        bad("Voice add-ons missing (" + VOICE_ERR[:60] + "). Run: python3 -m pip install numpy sounddevice SpeechRecognition")

    run(["screencapture", "-x", "-t", "jpg", SHOT])
    if os.path.exists(SHOT) and os.path.getsize(SHOT) > 0:
        ok("Screen capture works. If the AI only sees your wallpaper, turn on Screen & System Audio Recording for this app.")
        try:
            os.remove(SHOT)
        except OSError:
            pass
    else:
        bad("Screen capture failed. Turn on Screen & System Audio Recording in System Settings > Privacy & Security.")

    try:
        _or("/models", timeout=10)
        ok("Can reach openrouter.ai")
        net = True
    except Exception as e:
        bad("Can't reach openrouter.ai (" + str(e)[:50] + "). A school filter may block it.")
        net = False
    if ai_ready():
        info("Key saved (" + str(len(_key())) + " characters)")
        if net:
            try:
                _or("/auth/key", timeout=10)
                ok("OpenRouter accepts your key")
                try:
                    reply, model = _chat(False, [{"role": "user", "content": "Reply with one word: ready"}], 20)
                    if reply:
                        ok("A free AI model answered (" + model + ")")
                    else:
                        bad("No free AI model answered. Try again in a minute.")
                except AIError as e2:
                    bad("Chat test failed: " + _ai_error(e2))
            except AIError as e:
                bad("OpenRouter rejected the key (code " + str(e.code) + "). Type: set key")
    else:
        info("No key saved. AI chat and screen reading are off. Type: set key")
    return out


# ----------------------------------------------------------------------------
# text mode
# ----------------------------------------------------------------------------
def text_main():
    def wake_handle(cmd):
        emit("YOU", cmd)
        STATE["mode"] = "thinking"
        reply = respond(cmd)
        if reply == "QUIT":
            speak("Goodbye, sir")
            os._exit(0)
        speak(reply)
        if STATE["mode"] == "thinking":
            STATE["mode"] = "idle"

    WAKE["fn"] = wake_handle
    speak(greeting())
    if S.get("wake") and VOICE_OK:
        emit("JARVIS", start_wake())
    voice_mode = False
    while True:
        try:
            if voice_mode:
                print("(listening...)")
                heard = listen()
                if not heard:
                    continue
                print("You said:", heard)
                if "stop listening" in heard.lower():
                    voice_mode = False
                    speak("Back to typing")
                    continue
                text = heard
            else:
                text = input("You: ").strip()
                if not text:
                    continue
                if text.lower() == "voice":
                    if VOICE_OK:
                        voice_mode = True
                        speak("Voice mode on. Say stop listening to go back.")
                    else:
                        speak("The voice add-ons are not installed. Run: python3 jarvis.py --doctor")
                    continue
            reply = respond(text)
            if reply == "QUIT":
                speak("Goodbye, sir")
                break
            speak(reply)
        except (KeyboardInterrupt, EOFError):
            print()
            break


# ----------------------------------------------------------------------------
# the window
# ----------------------------------------------------------------------------
def rgb(h):
    return tuple(int(h[i:i + 2], 16) for i in (1, 3, 5))


def hexc(c):
    return "#%02x%02x%02x" % tuple(max(0, min(255, int(v))) for v in c)


def mix(a, b, t):
    return tuple(a[i] + (b[i] - a[i]) * t for i in range(3))


def run_gui(app_mode=False):
    try:
        import tkinter as tk
        root = tk.Tk()
    except Exception as e:
        if app_mode:
            sys.stderr.write("Window problem: " + str(e) + "\n")
            return
        print("I couldn't open a window (" + str(e)[:80] + "). Switching to text mode.")
        return text_main()

    W, H = 1180, 680
    CX, CY = W // 2, 318
    K = 0.94
    BG = "#03070d"
    BGC = rgb(BG)
    TINT = {"idle": None, "speaking": None, "listening": rgb("#39ff88"), "thinking": rgb("#ffb300")}
    LABEL = {"idle": "STANDING BY", "listening": "LISTENING", "thinking": "PROCESSING", "speaking": "SPEAKING"}
    SPEED = {"idle": 1.0, "listening": 1.5, "thinking": 3.0, "speaking": 1.8}
    ui_q = queue.Queue()
    hist = [0.0] * 60
    cur = list(rgb(THEMES.get(S.get("theme"), THEMES["cyan"])))
    parts = [[random.uniform(0, W), random.uniform(0, H), random.uniform(-0.2, 0.2),
              random.uniform(-0.6, -0.1), random.uniform(0.15, 0.5), random.choice((1, 2, 2, 3))]
             for _ in range(50)]
    tick = [0]
    boot = time.time()
    t0 = time.time()
    history, hpos = [], [0]

    _emit[0] = lambda who, text: ui_q.put(("log", who, text))

    root.title("Jarvis")
    root.configure(bg=BG)
    root.resizable(False, False)
    cv = tk.Canvas(root, width=W, height=H, bg=BG, highlightthickness=0)
    cv.pack()
    for gx in range(0, W, 40):
        cv.create_line(gx, 0, gx, H, fill="#07101c")
    for gy in range(0, H, 40):
        cv.create_line(0, gy, W, gy, fill="#07101c")
    for sy in range(0, H, 4):
        cv.create_line(0, sy, W, sy, fill="#040a12")

    log = tk.Text(root, bg="#050e1a", fg="#bfeeff", relief="flat", font=("Menlo", 11), wrap="word",
                  padx=8, pady=6, state="disabled", highlightthickness=0, insertwidth=0)
    log.tag_config("jarvis", foreground="#00e5ff")
    log.tag_config("you", foreground="#ffffff")
    log.tag_config("sys", foreground="#6fb7c7")
    cv.create_window(W - 290, 136, anchor="nw", width=262, height=H - 136 - 96, window=log)

    entry = tk.Entry(root, bg="#07111f", fg="#ffffff", insertbackground="#00e5ff", relief="flat",
                     font=("Menlo", 14), highlightthickness=1, highlightbackground="#0a3a47",
                     highlightcolor="#00e5ff")
    cv.create_window(20, H - 52, anchor="nw", width=W - 280, height=36, window=entry)

    def add_log(who, msg):
        tag = who.lower() if who.lower() in ("jarvis", "you") else "sys"
        log.config(state="normal")
        log.insert("end", (who + ": " if who else "") + msg + "\n", tag)
        if int(log.index("end-1c").split(".")[0]) > 400:
            log.delete("1.0", "60.0")
        log.see("end")
        log.config(state="disabled")

    def process(text):
        STATE["mode"] = "thinking"
        try:
            reply = respond(text)
            if reply == "QUIT":
                speak("Goodbye, sir")
                ui_q.put(("quit",))
                return
            speak(reply)
        finally:
            if STATE["mode"] == "thinking":
                STATE["mode"] = "idle"

    def voice_loop():
        while STATE["voice"]:
            if STATE["mode"] in ("speaking", "thinking"):
                time.sleep(0.3)
                continue
            STATE["mode"] = "listening"
            try:
                heard = listen()
            except Exception as ex:
                STATE["mode"] = "idle"
                STATE["voice"] = False
                ui_q.put(("log", "JARVIS", "Microphone problem: " + str(ex)[:120] + ". Say doctor, or allow the mic in System Settings."))
                return
            if STATE["mode"] == "listening":
                STATE["mode"] = "idle"
            if not heard or not STATE["voice"]:
                continue
            ui_q.put(("log", "YOU", heard))
            if "stop listening" in heard.lower():
                STATE["voice"] = False
                speak("Back to typing")
                return
            process(heard)

    def ask_key(which="openrouter"):
        from tkinter import simpledialog
        if which == "groq":
            k = simpledialog.askstring(
                "Jarvis - Groq backup key",
                "Paste your free Groq key (starts with gsk_).\nGet one at console.groq.com > API Keys.",
                show="*", parent=root)
            if k:
                add_log("JARVIS", save_groq_key(k))
            return
        k = simpledialog.askstring(
            "Jarvis - OpenRouter key",
            "Paste your free OpenRouter key (starts with sk-or-).\nGet one at openrouter.ai > Keys.",
            show="*", parent=root)
        if k:
            add_log("JARVIS", save_key(k))

    KEY_PROMPT[0] = lambda: ui_q.put(("askkey",))
    KEY_PROMPT[1] = lambda: ui_q.put(("askkey", "groq"))
    UI_ACTION[0] = lambda name: ui_q.put(("ui", name))

    def open_settings():
        import tkinter as tk
        from tkinter import ttk
        bg, fg = "#0a1018", "#9fe8ff"
        win = tk.Toplevel(root)
        win.title("Jarvis settings")
        win.configure(bg=bg)
        win.resizable(False, False)
        pad = {"padx": 14, "pady": 7}

        def lab(text, r):
            tk.Label(win, text=text, bg=bg, fg=fg, anchor="w").grid(row=r, column=0, sticky="w", **pad)
        lab("Voice", 0)
        voices = sorted(n for n, lang in installed_voices() if lang.startswith("en_")) or ["Daniel"]
        vv = tk.StringVar(value=S.get("voice", "Daniel"))
        ttk.Combobox(win, textvariable=vv, values=voices, state="readonly", width=22).grid(row=0, column=1, **pad)
        lab("Color theme", 1)
        tv = tk.StringVar(value=S.get("theme", "cyan"))
        ttk.Combobox(win, textvariable=tv, values=sorted(THEMES), state="readonly", width=22).grid(row=1, column=1, **pad)
        wv = tk.BooleanVar(value=bool(WAKE["on"]))
        tk.Checkbutton(win, text="Wake word (say Jarvis)", variable=wv, bg=bg, fg=fg, selectcolor=bg,
                       activebackground=bg, activeforeground=fg).grid(row=2, column=0, columnspan=2, sticky="w", **pad)
        lv = tk.BooleanVar(value=os.path.exists(LOGIN_PLIST))
        tk.Checkbutton(win, text="Open Jarvis at login", variable=lv, bg=bg, fg=fg, selectcolor=bg,
                       activebackground=bg, activeforeground=fg).grid(row=3, column=0, columnspan=2, sticky="w", **pad)
        bar = tk.Frame(win, bg=bg)
        bar.grid(row=4, column=0, columnspan=2, **pad)
        tk.Button(bar, text="OpenRouter key", command=lambda: ask_key("openrouter")).pack(side="left", padx=4)
        tk.Button(bar, text="Groq key", command=lambda: ask_key("groq")).pack(side="left", padx=4)

        def apply_close():
            if vv.get() != S.get("voice"):
                add_log("JARVIS", set_voice(vv.get()))
            if tv.get() != S.get("theme"):
                add_log("JARVIS", handle("theme " + tv.get()) or "")
            if wv.get() != bool(WAKE["on"]):
                add_log("JARVIS", start_wake() if wv.get() else stop_wake())
            if lv.get() != os.path.exists(LOGIN_PLIST):
                add_log("JARVIS", login_item(lv.get()))
            win.destroy()
        tk.Button(win, text="Save and close", command=apply_close).grid(row=5, column=0, columnspan=2, pady=12)

    def ui_do(name):
        try:
            if name == "settings":
                open_settings()
            elif name == "top_on":
                root.attributes("-topmost", True)
            elif name == "top_off":
                root.attributes("-topmost", False)
        except Exception as e:
            add_log("JARVIS", "That didn't work: " + str(e)[:80])

    def wake_handle(cmd):
        ui_q.put(("log", "YOU", cmd))
        process(cmd)

    WAKE["fn"] = wake_handle

    def toggle_voice():
        if not VOICE_OK:
            add_log("JARVIS", "Voice add-ons missing: run  python3 -m pip install numpy sounddevice SpeechRecognition  then reopen me.")
            return
        if WAKE["on"]:
            stop_wake()
            add_log("JARVIS", "Wake word turned off so the mic button can use the microphone.")
        STATE["voice"] = not STATE["voice"]
        if STATE["voice"]:
            add_log("JARVIS", "Voice mode on. Say stop listening, or click the mic, to turn it off.")
            threading.Thread(target=voice_loop, daemon=True).start()

    def on_enter(event=None):
        text = entry.get().strip()
        entry.delete(0, "end")
        if not text:
            return
        history.append(text)
        hpos[0] = len(history)
        add_log("YOU", text)
        if text.lower() == "voice":
            toggle_voice()
            return
        threading.Thread(target=process, args=(text,), daemon=True).start()

    def hist_move(delta):
        if not history:
            return "break"
        hpos[0] = max(0, min(len(history), hpos[0] + delta))
        entry.delete(0, "end")
        if hpos[0] < len(history):
            entry.insert(0, history[hpos[0]])
        return "break"

    def quit_app(event=None):
        STATE["voice"] = False
        WAKE["on"] = False
        stop_talking()
        root.destroy()

    entry.bind("<Return>", on_enter)
    entry.bind("<Up>", lambda e: hist_move(-1))
    entry.bind("<Down>", lambda e: hist_move(1))
    root.bind("<Escape>", lambda e: threading.Thread(target=stop_talking, daemon=True).start())
    root.protocol("WM_DELETE_WINDOW", quit_app)

    def on_click(e):
        if W - 230 <= e.x <= W - 20 and H - 52 <= e.y <= H - 16:
            toggle_voice()
    cv.bind("<Button-1>", on_click)

    def poll():
        while True:
            try:
                item = ui_q.get_nowait()
            except queue.Empty:
                break
            if item[0] == "log":
                add_log(item[1], item[2])
            elif item[0] == "ui":
                ui_do(item[1])
            elif item[0] == "askkey":
                ask_key(item[1] if len(item) > 1 else "openrouter")
            elif item[0] == "quit":
                quit_app()
                return
        root.after(80, poll)

    def battery_loop():
        while True:
            out = run(["pmset", "-g", "batt"])
            m = re.search(r"(\d+)%", out)
            if m:
                STATE["battery"] = m.group(1) + "%" + (" CHG" if "AC Power" in out else "")
            time.sleep(25)

    def bracket(x0, y0, x1, y1, col, s=18):
        for (x, y, dx, dy) in ((x0, y0, 1, 1), (x1, y0, -1, 1), (x0, y1, 1, -1), (x1, y1, -1, -1)):
            cv.create_line(x, y + dy * s, x, y, x + dx * s, y, fill=col, width=2, tags="dyn")

    def frame():
        now = time.time()
        t = now - t0
        mode = STATE["mode"]
        base = rgb(THEMES.get(S.get("theme"), THEMES["cyan"]))
        tint = TINT.get(mode)
        target = mix(base, tint, 0.75) if tint else base
        for i in range(3):
            cur[i] += (target[i] - cur[i]) * 0.08
        main = tuple(cur)
        M = hexc(main)
        D = hexc(mix(BGC, main, 0.38))
        F = hexc(mix(BGC, main, 0.16))
        soft = hexc(mix(BGC, main, 0.6))
        goal = {"speaking": 0.45 + 0.55 * abs(math.sin(t * 7) * math.sin(t * 3.1)),
                "listening": 0.35 + 0.25 * math.sin(t * 4), "thinking": 0.25, "idle": 0.06}[mode]
        STATE["level"] += (goal - STATE["level"]) * 0.25
        lvl = STATE["level"]
        tick[0] += 1
        if tick[0] % 3 == 0:
            hist.append(lvl)
            hist.pop(0)
        p = min(1.0, (now - boot) / 3.0)
        ez = (1 - (1 - p) ** 3) * K
        spd = SPEED[mode]
        cv.delete("dyn")

        for pt in parts:
            pt[0] += pt[2] * spd
            pt[1] += pt[3] * (1 + (spd - 1) * 0.5)
            if pt[1] < -5:
                pt[1], pt[0] = H + 5, random.uniform(0, W)
            if pt[0] < -5 or pt[0] > W + 5:
                pt[0] = random.uniform(0, W)
            cv.create_rectangle(pt[0], pt[1], pt[0] + pt[5], pt[1] + pt[5],
                                fill=hexc(mix(BGC, main, pt[4])), outline="", tags="dyn")

        R = 215 * ez
        sweep = (t * 60 * spd) % 360
        for k in range(14):
            col = hexc(mix(BGC, main, 0.28 * (1 - k / 14) ** 2))
            cv.create_arc(CX - R, CY - R, CX + R, CY + R, start=(sweep - k * 5) % 360, extent=5.5,
                          style="pieslice", fill=col, outline="", tags="dyn")
        for i in range(7):
            rr = (140 - i * 16) * ez
            col = hexc(mix(BGC, main, 0.03 + 0.025 * i + 0.05 * lvl))
            cv.create_oval(CX - rr, CY - rr, CX + rr, CY + rr, fill=col, outline="", tags="dyn")

        r0 = 236 * ez
        for i in range(90):
            a = math.radians(i * 4 + t * 6 * spd)
            big = i % 5 == 0
            r2 = r0 + (14 if big else 7)
            cv.create_line(CX + r0 * math.cos(a), CY + r0 * math.sin(a),
                           CX + r2 * math.cos(a), CY + r2 * math.sin(a),
                           fill=M if big else D, width=2 if big else 1, tags="dyn")

        def ring(r, start, n, ext, col, w):
            r *= ez
            for k in range(n):
                cv.create_arc(CX - r, CY - r, CX + r, CY + r, start=(start + k * 360 / n) % 360,
                              extent=ext, style="arc", outline=col, width=w, tags="dyn")

        ring(222, t * 25 * spd, 3, 80, M, 3)
        ring(198, -t * 45 * spd, 5, 40, M, 2)
        ring(172, t * 70 * spd, 2, 150, D, 6)
        ring(150, -t * 90 * spd, 8, 20, M, 2)
        for rr, col in ((205, F), (160, D)):
            rr *= ez
            cv.create_oval(CX - rr, CY - rr, CX + rr, CY + rr, outline=col, tags="dyn")

        base_r = 104 * ez
        for i in range(72):
            a = 2 * math.pi * i / 72 + t * 0.3
            amp = lvl * (0.5 + 0.5 * math.sin(t * 7 + i * 0.9)) * (0.6 + 0.4 * math.sin(t * 3.3 + i * 0.37))
            L = 4 + amp * 38
            cv.create_line(CX + base_r * math.cos(a), CY + base_r * math.sin(a),
                           CX + (base_r + L) * math.cos(a), CY + (base_r + L) * math.sin(a),
                           fill=M, width=2, tags="dyn")

        for j, rad in enumerate((222, 198, 172)):
            ang = t * (0.9 + 0.4 * j) * spd + j * 2.1
            for k in range(6):
                a2 = ang - k * 0.08
                x, y = CX + rad * ez * math.cos(a2), CY + rad * ez * math.sin(a2)
                s = 5 - k * 0.7
                cv.create_oval(x - s, y - s, x + s, y + s, fill=hexc(mix(BGC, main, 1 - k * 0.15)),
                               outline="", tags="dyn")

        cr = 46 * (1 + 0.06 * math.sin(t * 2 * spd) + lvl * 0.12) * ez / K
        cv.create_oval(CX - cr - 14, CY - cr - 14, CX + cr + 14, CY + cr + 14, fill=F, outline=D, tags="dyn")
        cv.create_oval(CX - cr - 4, CY - cr - 4, CX + cr + 4, CY + cr + 4, fill=D, outline=M, width=2, tags="dyn")
        cv.create_oval(CX - cr + 8, CY - cr + 8, CX + cr - 8, CY + cr - 8, fill=M, outline="", tags="dyn")
        gl = hexc(mix(main, (255, 255, 255), 0.55))
        cv.create_oval(CX - cr * 0.55, CY - cr * 0.6, CX - cr * 0.15, CY - cr * 0.3, fill=gl, outline="", tags="dyn")
        cv.create_text(CX, CY + 2, text="J", fill=BG, font=("Menlo", 26, "bold"), tags="dyn")

        label = ("INITIALIZING " + str(int(p * 100)) + "%") if p < 1 else LABEL[mode]
        cv.create_text(CX, H - 76, text=label + ("_" if int(t * 2) % 2 else " "), fill=M,
                       font=("Menlo", 13, "bold"), tags="dyn")

        dt = datetime.now()
        cv.create_text(28, 30, anchor="w", text="J.A.R.V.I.S", fill=M, font=("Menlo", 22, "bold"), tags="dyn")
        cv.create_text(28, 58, anchor="w", text="JUST A RATHER VERY INTELLIGENT SYSTEM", fill=soft,
                       font=("Menlo", 9), tags="dyn")
        cv.create_text(W - 28, 30, anchor="e", text=dt.strftime("%H:%M:%S"), fill=M,
                       font=("Menlo", 22, "bold"), tags="dyn")
        cv.create_text(W - 28, 58, anchor="e", text=dt.strftime("%A %d %B %Y").upper(), fill=soft,
                       font=("Menlo", 9), tags="dyn")
        cv.create_line(20, 80, W - 20, 80, fill=D, tags="dyn")
        hx = (t * 260) % W
        cv.create_line(hx - 90, 80, hx + 90, 80, fill=M, width=2, tags="dyn")
        bracket(8, 8, W - 8, H - 8, M, 34)
        bracket(20, 100, 290, H - 96, D)
        bracket(W - 296, 100, W - 20, H - 96, D)

        x0 = 28
        cv.create_text(x0 + 12, 120, anchor="w", text="SYSTEM STATUS", fill=M, font=("Menlo", 12, "bold"), tags="dyn")
        up = int(t)
        rows = (("CORE", "ONLINE"), ("MODE", LABEL[mode]),
                ("MIC", "ON" if STATE["voice"] else "OFF"), ("WAKE", "ON" if WAKE["on"] else "OFF"),
                ("AI", "READY" if ai_ready() else "NO KEY"),
                ("POWER", STATE["battery"]), ("THEME", str(S.get("theme", "cyan")).upper()),
                ("UPTIME", "%02d:%02d:%02d" % (up // 3600, up % 3600 // 60, up % 60)))
        for i, (k, v) in enumerate(rows):
            y = 154 + i * 27
            cv.create_text(x0 + 12, y, anchor="w", text=k, fill=soft, font=("Menlo", 11), tags="dyn")
            cv.create_text(x0 + 250, y, anchor="e", text=v, fill="#e8f4ff", font=("Menlo", 11, "bold"), tags="dyn")
            cv.create_line(x0 + 12, y + 13, x0 + 250, y + 13, fill=F, tags="dyn")
        try:
            load = os.getloadavg()[0] / (os.cpu_count() or 4)
        except Exception:
            load = 0
        load = max(0, min(1, load))
        cv.create_text(x0 + 12, 372, anchor="w", text="CPU LOAD  " + str(int(load * 100)) + "%", fill=soft,
                       font=("Menlo", 11), tags="dyn")
        cv.create_rectangle(x0 + 12, 386, x0 + 250, 398, outline=D, tags="dyn")
        cv.create_rectangle(x0 + 12, 386, x0 + 12 + 238 * load, 398, fill=M, outline="", tags="dyn")
        cv.create_text(x0 + 12, 428, anchor="w", text="VOICE ACTIVITY", fill=soft, font=("Menlo", 11), tags="dyn")
        cv.create_rectangle(x0 + 12, 444, x0 + 250, 524, outline=D, tags="dyn")
        pts = []
        for i, v in enumerate(hist):
            pts += [x0 + 12 + i * 238 / 59, 520 - v * 72]
        cv.create_line(*pts, fill=M, width=2, smooth=True, tags="dyn")

        cv.create_text(W - 280, 120, anchor="w", text="COMMS LOG", fill=M, font=("Menlo", 12, "bold"), tags="dyn")

        on = STATE["voice"]
        cv.create_rectangle(W - 230, H - 52, W - 20, H - 16, fill=M if on else "#0b1b2e", outline=M, width=2, tags="dyn")
        cv.create_text(W - 125, H - 34, text="MIC ON" if on else "MIC OFF", fill=BG if on else M,
                       font=("Menlo", 13, "bold"), tags="dyn")
        root.after(36, frame)

    entry.focus_set()
    root.lift()
    root.attributes("-topmost", True)
    root.after(500, lambda: root.attributes("-topmost", False))
    for i, line in enumerate(("[ CORE ] ONLINE",
                              "[ AI ] " + ("READY" if ai_ready() else "NO KEY (built-in commands only)"),
                              "[ VOICE ] " + ("READY" if VOICE_OK else "ADD-ONS MISSING"))):
        root.after(500 + i * 600, lambda s=line: add_log("", s))
    threading.Thread(target=battery_loop, daemon=True).start()
    for _i, _lbl in enumerate(MISSED):
        root.after(4200 + _i * 600, lambda l=_lbl: add_log("JARVIS", "While I was closed, your " + l + " finished."))
    if not ai_ready():
        root.after(3400, lambda: add_log("JARVIS", "No AI key yet. Type  set key  to add a free OpenRouter key (optional)."))
    root.after(80, poll)
    root.after(36, frame)
    root.after(2800, lambda: threading.Thread(target=speak, args=(greeting(),), daemon=True).start())
    if S.get("wake") and VOICE_OK:
        root.after(5200, lambda: add_log("JARVIS", start_wake()))
    root.mainloop()


# ----------------------------------------------------------------------------
# setup helpers
# ----------------------------------------------------------------------------
def make_app():
    app = os.path.join(DESK, "Jarvis.app")
    script = os.path.abspath(__file__)
    if os.path.exists(app):
        shutil.rmtree(app)
    macos = os.path.join(app, "Contents", "MacOS")
    os.makedirs(macos)
    launcher = os.path.join(macos, "Jarvis")
    with open(launcher, "w") as f:
        f.write('#!/bin/zsh\nexec "' + sys.executable + '" "' + script + '" --app >> "$HOME/jarvis_app.log" 2>&1\n')
    os.chmod(launcher, 0o755)
    plist = {
        "CFBundleName": "Jarvis", "CFBundleDisplayName": "Jarvis",
        "CFBundleIdentifier": "com.local.jarvis", "CFBundleExecutable": "Jarvis",
        "CFBundlePackageType": "APPL", "CFBundleVersion": "2.0", "CFBundleShortVersionString": "2.0",
        "NSHighResolutionCapable": True,
        "NSMicrophoneUsageDescription": "Jarvis listens to your voice commands.",
        "NSAppleEventsUsageDescription": "Jarvis controls Music and system settings when you ask.",
    }
    with open(os.path.join(app, "Contents", "Info.plist"), "wb") as f:
        plistlib.dump(plist, f)
    run(["xattr", "-cr", app])
    run(["codesign", "--force", "--deep", "--sign", "-", app])
    print("Done! Jarvis.app is on your Desktop.")
    print("It runs this file, so updating Jarvis = replace " + script + " and reopen the app.")
    print("If the app ever does nothing, run:  cat ~/jarvis_app.log")


def set_key():
    import getpass
    k = getpass.getpass("Paste your OpenRouter key, then press Enter (nothing will show): ").strip()
    if len(k) < 20:
        print("That looks too short. Nothing saved.")
        return
    print(save_key(k))


def make_icon_png(path, n=1024):
    """Draw a glowing arc-reactor icon with only the standard library."""
    import zlib, struct
    c = (n - 1) / 2.0
    rows = []
    for y in range(n):
        row = bytearray([0])
        dy = y - c
        for x in range(n):
            dx = x - c
            r = math.sqrt(dx * dx + dy * dy) / (n / 2.0)
            a = math.atan2(dy, dx)
            # rounded-square mask
            ax, ay = abs(dx) / (n / 2.0), abs(dy) / (n / 2.0)
            m = max(ax, ay)
            if m > 0.94:
                row += bytes((0, 0, 0, 0))
                continue
            edge = min(1.0, (0.94 - m) / 0.01)
            g = 0.0
            for rad, w, s in ((0.80, 0.020, 0.9), (0.62, 0.030, 1.0), (0.40, 0.012, 0.8)):
                d = abs(r - rad)
                g += s * math.exp(-(d / w) ** 2) + 0.18 * s * math.exp(-(d / (w * 5)) ** 2)
            seg = (math.sin(a * 6) > 0.35) and 0.46 < r < 0.58
            if seg:
                g += 0.9
            if r < 0.24:
                g += 1.1 * math.exp(-(r / 0.16) ** 2) + 0.15
            if 0.24 < r < 0.80 and abs(math.sin(a * 3)) < 0.012 and r > 0.40:
                g += 0.5
            g = min(1.0, g)
            bgv = 8 + 14 * (1 - r)
            rr = bgv + g * (0 - bgv) + g * g * 150
            gg = bgv + g * (229 - bgv) + g * g * 26
            bb = bgv + g * (255 - bgv)
            row += bytes((int(max(0, min(255, rr))), int(max(0, min(255, gg))), int(max(0, min(255, bb))), int(255 * edge)))
        rows.append(bytes(row))
    raw = b"".join(rows)

    def chunk(t, d):
        return struct.pack(">I", len(d)) + t + d + struct.pack(">I", zlib.crc32(t + d) & 0xffffffff)
    png = b"\x89PNG\r\n\x1a\n" + chunk(b"IHDR", struct.pack(">IIBBBBB", n, n, 8, 6, 0, 0, 0)) \
        + chunk(b"IDAT", zlib.compress(raw, 6)) + chunk(b"IEND", b"")
    with open(path, "wb") as f:
        f.write(png)


def build_standalone():
    """Make a self-contained Jarvis.app (no Python needed) plus Jarvis.zip for sharing."""
    if FROZEN:
        print("This is already the packaged app.")
        return
    if sys.platform != "darwin":
        print("Building the app only works on a Mac.")
        return
    script = os.path.abspath(__file__)
    work = os.path.join(HOME, ".jarvis_build")
    venv = os.path.join(work, "venv")
    os.makedirs(work, exist_ok=True)
    py = os.path.join(venv, "bin", "python")

    def step(msg):
        print("\n=== " + msg + " ===", flush=True)

    def sh(cmd, **kw):
        r = subprocess.run(cmd, **kw)
        if r.returncode != 0:
            print("\nA build step failed: " + " ".join(cmd[:4]) + " ...")
            print("Scroll up to see the error, and send me the last 15 lines.")
            raise SystemExit(1)

    step("0/5  Checking your Mac is ready")
    chk = subprocess.run(["xcode-select", "-p"], capture_output=True, text=True)
    if chk.returncode != 0 or not os.path.exists(os.path.join((chk.stdout or "").strip() or "/nonexistent", "usr", "bin", "lipo")):
        print("Your Mac needs Apple's free Command Line Tools first.")
        subprocess.run(["xcode-select", "--install"])
        print("A window popped up. Click Install and wait until it finishes (5-15 minutes),")
        print("then run this same command again:  python3 ~/Desktop/jarvis.py --build-app")
        raise SystemExit(1)
    print("Command Line Tools: OK")
    try:
        urllib.request.urlopen("https://pypi.org/simple/pip/", timeout=10).close()
        print("Internet: OK")
    except Exception as e:
        print("I can't reach pypi.org (" + str(e)[:60] + "). The build needs internet and the school filter may block it.")
        raise SystemExit(1)

    step("1/5  Making a clean build environment (needs internet, ~2 minutes)")
    if not os.path.exists(py):
        sh([sys.executable, "-m", "venv", venv])
    sh([py, "-m", "pip", "install", "--quiet", "--upgrade", "pip"])
    sh([py, "-m", "pip", "install", "--quiet", "pyinstaller", "numpy", "sounddevice",
        "SpeechRecognition", "certifi"])

    step("2/5  Drawing the icon")
    png = os.path.join(work, "icon.png")
    iconset = os.path.join(work, "Jarvis.iconset")
    icns = os.path.join(work, "Jarvis.icns")
    icon_args = []
    try:
        make_icon_png(png)
        if os.path.exists(iconset):
            shutil.rmtree(iconset)
        os.makedirs(iconset)
        for sz in (16, 32, 128, 256, 512):
            for scale in (1, 2):
                px = sz * scale
                name = "icon_%dx%d%s.png" % (sz, sz, "@2x" if scale == 2 else "")
                sh(["sips", "-z", str(px), str(px), png, "--out", os.path.join(iconset, name)],
                   stdout=subprocess.DEVNULL)
        sh(["iconutil", "-c", "icns", iconset, "-o", icns])
        icon_args = ["--icon", icns]
    except SystemExit:
        raise
    except Exception as e:
        print("Icon skipped (" + str(e)[:60] + ").")

    step("3/5  Packaging Jarvis (this is the slow part, 2-5 minutes)")
    dist = os.path.join(work, "dist")
    for d in (dist, os.path.join(work, "build")):
        if os.path.exists(d):
            shutil.rmtree(d)
    sh([py, "-m", "PyInstaller", "--noconfirm", "--clean", "--windowed", "--name", "Jarvis",
        "--osx-bundle-identifier", "com.jarvis.assistant",
        "--distpath", dist, "--workpath", os.path.join(work, "build"), "--specpath", work,
        "--collect-all", "speech_recognition", "--collect-all", "sounddevice",
        "--collect-all", "_sounddevice_data", "--collect-all", "certifi",
        "--hidden-import", "numpy", "--hidden-import", "tkinter.simpledialog"] + icon_args + [script])
    built = os.path.join(dist, "Jarvis.app")
    if not os.path.exists(built):
        print("The build finished but Jarvis.app is missing.")
        raise SystemExit(1)

    step("4/5  Adding permission messages and signing")
    pl = os.path.join(built, "Contents", "Info.plist")
    with open(pl, "rb") as f:
        info = plistlib.load(f)
    info.update({
        "CFBundleDisplayName": "Jarvis", "CFBundleShortVersionString": "2.0", "CFBundleVersion": "2.0",
        "NSHighResolutionCapable": True,
        "NSMicrophoneUsageDescription": "Jarvis listens to your voice commands.",
        "NSAppleEventsUsageDescription": "Jarvis controls Music and system settings when you ask.",
    })
    with open(pl, "wb") as f:
        plistlib.dump(info, f)
    subprocess.run(["xattr", "-cr", built])
    subprocess.run(["codesign", "--force", "--deep", "--sign", "-", built])

    step("5/5  Putting Jarvis on your Desktop")
    final = os.path.join(DESK, "Jarvis.app")
    zipf = os.path.join(DESK, "Jarvis.zip")
    for p in (final, zipf):
        if os.path.isdir(p):
            shutil.rmtree(p)
        elif os.path.exists(p):
            os.remove(p)
    sh(["ditto", built, final])
    sh(["ditto", "-c", "-k", "--keepParent", final, zipf])
    print("\nDONE!")
    print("  Double-click Jarvis on your Desktop to open it.")
    print("  Share this file with friends:  " + zipf)
    print("  (Your own AI key is NOT inside it. Everyone adds their own with: set key)")


def main():
    args = sys.argv[1:]
    if FROZEN:
        try:
            lg = open(os.path.join(HOME, "jarvis_app.log"), "a", buffering=1)
            sys.stdout = lg
            sys.stderr = lg
        except OSError:
            pass
    CUR_VOICE[0] = resolve_voice()
    restore_timers()
    if FROZEN and not args:
        run_gui(app_mode=True)
        return
    if "--build-app" in args:
        build_standalone()
    elif "--doctor" in args:
        for line in doctor():
            print(line)
    elif "--make-app" in args:
        make_app()
    elif "--set-key" in args:
        set_key()
    elif "--text" in args:
        text_main()
    elif "--app" in args:
        run_gui(app_mode=True)
    else:
        run_gui()


if __name__ == "__main__":
    main()
