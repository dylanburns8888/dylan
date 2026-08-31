#!/usr/bin/env python3
"""
Jarvis-lite prototype for Windows 11.

Features:
- open apps, folders (projects), and websites
- monitor YouTube channels via RSS and generic URLs for changes (games/patch pages)
- persistent memory and last-seen state (SQLite)
- text command mode (default) and optional voice mode (if SpeechRecognition/PyAudio installed)

Requirements:
pip install aiohttp feedparser pyttsx3 win10toast
Optional for voice: pip install SpeechRecognition pyaudio

Run:
python jarvis.py
"""
import asyncio
import aiohttp
import feedparser
import sqlite3
import hashlib
import json
import os
import subprocess
import webbrowser
import time
from datetime import datetime
import threading
import sys

# TTS and Windows toast (win10toast)
try:
    import pyttsx3
    tts_engine = pyttsx3.init()
except Exception as e:
    tts_engine = None
    print("pyttsx3 not available:", e)

try:
    from win10toast import ToastNotifier
    toaster = ToastNotifier()
except Exception as e:
    toaster = None
    print("win10toast not available:", e)

# Optional voice input
VOICE_AVAILABLE = False
try:
    import speech_recognition as sr
    VOICE_AVAILABLE = True
except Exception:
    VOICE_AVAILABLE = False

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
CONFIG_PATH = os.path.join(BASE_DIR, "config.json")
DB_PATH = os.path.join(BASE_DIR, "jarvis_memory.db")

# Default config if not present
DEFAULT_CONFIG = {
    "apps": {
        # friendly name: full path or shell command
        "vscode": r"C:\Users\%USERNAME%\AppData\Local\Programs\Microsoft VS Code\Code.exe",
        "chrome": r"C:\Program Files\Google\Chrome\Application\chrome.exe"
    },
    "projects": {
        # friendly name: folder path
        "mygame": r"C:\Users\%USERNAME%\Projects\MyGame"
    },
    "websites": {
        "youtube": "https://www.youtube.com",
        "google": "https://www.google.com"
    },
    "youtube_channels": [
        # you can put full RSS urls or channel ids (script will convert)
        # example: "UC_x5XG1OV2P6uZZ5FSM9Ttw"  (Google Developers)
    ],
    "monitors": [
        # {"label":"Steam-MyGame", "url":"https://store.steampowered.com/app/XXXX/YourGame"}
    ],
    "poll_interval_seconds": 60,
    "voice_mode": False
}

# ----------------------------
# Utility: config load/save
# ----------------------------
def load_config():
    if not os.path.exists(CONFIG_PATH):
        with open(CONFIG_PATH, "w", encoding="utf-8") as f:
            json.dump(DEFAULT_CONFIG, f, indent=2)
        return DEFAULT_CONFIG.copy()
    with open(CONFIG_PATH, "r", encoding="utf-8") as f:
        return json.load(f)

def save_config(cfg):
    with open(CONFIG_PATH, "w", encoding="utf-8") as f:
        json.dump(cfg, f, indent=2)

config = load_config()

# ----------------------------
# SQLite memory store
# ----------------------------
def init_db():
    conn = sqlite3.connect(DB_PATH, check_same_thread=False)
    cur = conn.cursor()
    cur.execute("""
    CREATE TABLE IF NOT EXISTS memory (
        key TEXT PRIMARY KEY,
        value TEXT,
        updated_at TEXT
    )
    """)
    cur.execute("""
    CREATE TABLE IF NOT EXISTS yt_state (
        channel TEXT PRIMARY KEY,
        last_video_id TEXT,
        updated_at TEXT
    )
    """)
    cur.execute("""
    CREATE TABLE IF NOT EXISTS url_state (
        label TEXT PRIMARY KEY,
        url TEXT,
        content_hash TEXT,
        updated_at TEXT
    )
    """)
    conn.commit()
    return conn

db = init_db()
db_lock = threading.Lock()

# ----------------------------
# Speak & notify
# ----------------------------
def speak(text):
    print("[Jarvis says]:", text)
    if tts_engine:
        try:
            tts_engine.say(text)
            tts_engine.runAndWait()
        except Exception as e:
            print("TTS error:", e)

def notify(title, msg, duration=5):
    # Windows toast
    print(f"[Notification] {title} - {msg}")
    if toaster:
        try:
            toaster.show_toast(title, msg, duration=duration, threaded=True)
        except Exception as e:
            print("Toast error:", e)
    speak(f"{title}: {msg}")

# ----------------------------
# App / project / website actions
# ----------------------------
def open_app(name):
    name = name.lower()
    apps = config.get("apps", {})
    if name not in apps:
        speak(f"I don't know an app named {name}.")
        return
    path = os.path.expandvars(apps[name])
    if not os.path.exists(path):
        # Try shell open (maybe it's on PATH)
        try:
            subprocess.Popen([path])
            speak(f"Opening {name}.")
            return
        except Exception as e:
            speak(f"Path {path} not found and failed to execute: {e}")
            return
    try:
        subprocess.Popen([path])
        speak(f"Opening {name}.")
    except Exception as e:
        speak(f"Failed to open {name}: {e}")

def open_project(name):
    name = name.lower()
    projects = config.get("projects", {})
    if name not in projects:
        speak(f"I don't know a project named {name}.")
        return
    folder = os.path.expandvars(projects[name])
    if not os.path.exists(folder):
        speak(f"Project folder {folder} not found.")
        return
    try:
        os.startfile(folder)
        speak(f"Opening project {name}.")
    except Exception as e:
        speak(f"Failed to open project {name}: {e}")

def open_website(token):
    token = token.lower()
    websites = config.get("websites", {})
    if token in websites:
        url = websites[token]
    else:
        url = token
        if not (url.startswith("http://") or url.startswith("https://")):
            url = "https://" + url
    try:
        webbrowser.open_new_tab(url)
        speak(f"Opening website {url}.")
    except Exception as e:
        speak(f"Failed to open website {url}: {e}")

# ----------------------------
# Memory (key/value)
# ----------------------------
def remember(key, value):
    now = datetime.utcnow().isoformat()
    with db_lock:
        cur = db.cursor()
        cur.execute("INSERT OR REPLACE INTO memory(key,value,updated_at) VALUES(?,?,?)", (key, value, now))
        db.commit()
    speak(f"Remembered {key} = {value}")

def recall(key):
    with db_lock:
        cur = db.cursor()
        cur.execute("SELECT value FROM memory WHERE key=?", (key,))
        r = cur.fetchone()
    if r:
        speak(f"{key} is {r[0]}")
    else:
        speak(f"I don't have a memory for {key}.")

# ----------------------------
# Monitoring: YouTube (RSS) + generic URL diff
# ----------------------------
async def fetch_url(session, url):
    try:
        async with session.get(url, timeout=20) as resp:
            resp.raise_for_status()
            return await resp.text()
    except Exception as e:
        print("fetch error:", e)
        return None

def channel_to_rss(channel_or_id):
    # If it's already a URL, return it
    if channel_or_id.startswith("http://") or channel_or_id.startswith("https://"):
        return channel_or_id
    # assume channel id
    return f"https://www.youtube.com/feeds/videos.xml?channel_id={channel_or_id}"

async def check_youtube_channel(session, channel_entry):
    rss = channel_to_rss(channel_entry)
    text = await fetch_url(session, rss)
    if not text:
        return
    parsed = feedparser.parse(text)
    if not parsed.entries:
        return
    latest = parsed.entries[0]
    # Unique id in feed - try id then link
    vid_id = getattr(latest, "id", None) or getattr(latest, "link", None) or latest.get("link")
    if not vid_id:
        return
    channel_key = rss
    with db_lock:
        cur = db.cursor()
        cur.execute("SELECT last_video_id FROM yt_state WHERE channel=?", (channel_key,))
        row = cur.fetchone()
        last = row[0] if row else None
        if last != vid_id:
            # update db
            now = datetime.utcnow().isoformat()
            cur.execute("INSERT OR REPLACE INTO yt_state(channel,last_video_id,updated_at) VALUES(?,?,?)", (channel_key, vid_id, now))
            db.commit()
    if last is None:
        print(f"[YouTube] Initialized channel {rss} latest {vid_id}")
        return
    if last != vid_id:
        title = latest.get("title", "New video")
        link = latest.get("link", rss)
        notify("YouTube: new video", f"{title} — {link}")

async def check_url_monitor(session, monitor):
    label = monitor.get("label")
    url = monitor.get("url")
    if not url or not label:
        return
    text = await fetch_url(session, url)
    if text is None:
        return
    h = hashlib.sha256(text.encode("utf-8", errors="ignore")).hexdigest()
    with db_lock:
        cur = db.cursor()
        cur.execute("SELECT content_hash FROM url_state WHERE label=?", (label,))
        row = cur.fetchone()
        last = row[0] if row else None
        if last != h:
            now = datetime.utcnow().isoformat()
            cur.execute("INSERT OR REPLACE INTO url_state(label,url,content_hash,updated_at) VALUES(?,?,?,?)", (label, url, h, now))
            db.commit()
    if last is None:
        print(f"[Monitor] Initialized {label}")
        return
    if last != h:
        notify("Page changed", f"{label} - {url}")

async def poll_loop():
    interval = max(10, int(config.get("poll_interval_seconds", 60)))
    async with aiohttp.ClientSession() as session:
        while True:
            try:
                # YouTube checks
                channels = config.get("youtube_channels", [])
                tasks = []
                for ch in channels:
                    tasks.append(check_youtube_channel(session, ch))
                # URL monitors
                for m in config.get("monitors", []):
                    tasks.append(check_url_monitor(session, m))
                if tasks:
                    await asyncio.gather(*tasks)
            except Exception as e:
                print("Poll loop error:", e)
            await asyncio.sleep(interval)

# ----------------------------
# CLI / Voice command parsing
# ----------------------------
def parse_and_execute(cmd_line):
    parts = cmd_line.strip().split()
    if not parts:
        return
    cmd = parts[0].lower()
    args = parts[1:]
    if cmd == "open":
        if not args:
            speak("Open what?")
            return
        subtype = args[0].lower()
        if subtype == "app" and len(args) >= 2:
            open_app(" ".join(args[1:]))
        elif subtype == "project" and len(args) >= 2:
            open_project(" ".join(args[1:]))
        elif subtype == "website" and len(args) >= 2:
            open_website(" ".join(args[1:]))
        else:
            # try infer: open <app/website/project> by name
            token = " ".join(args)
            # try apps
            if token.lower() in config.get("apps", {}):
                open_app(token)
            elif token.lower() in config.get("projects", {}):
                open_project(token)
            else:
                open_website(token)
    elif cmd == "remember" and len(args) >= 2:
        key = args[0]
        value = " ".join(args[1:])
        remember(key, value)
    elif cmd == "recall" and len(args) >= 1:
        key = args[0]
        recall(key)
    elif cmd == "subscribe" and len(args) >= 2 and args[0].lower() == "youtube":
        # subscribe youtube <channel-id-or-rss>
        ch = args[1]
        ch_rss = channel_to_rss(ch)
        if ch_rss not in config.get("youtube_channels", []):
            config.setdefault("youtube_channels", []).append(ch_rss)
            save_config(config)
            speak(f"Subscribed to YouTube RSS {ch_rss}")
        else:
            speak("Already subscribed.")
    elif cmd == "add" and len(args) >= 3 and args[0].lower() == "game_monitor":
        # add game_monitor <label> <url>
        label = args[1]
        url = args[2]
        config.setdefault("monitors", []).append({"label": label, "url": url})
        save_config(config)
        speak(f"Added monitor {label}")
    elif cmd == "list":
        if args and args[0].lower() == "monitors":
            for m in config.get("monitors", []):
                print("-", m)
            speak(f"{len(config.get('monitors',[]))} monitors listed.")
        elif args and args[0].lower() == "youtube":
            for ch in config.get("youtube_channels", []):
                print("-", ch)
            speak(f"{len(config.get('youtube_channels',[]))} youtube subscriptions listed.")
        else:
            speak("I can list monitors or youtube.")
    elif cmd == "help":
        print_help()
    elif cmd == "exit" or cmd == "quit":
        speak("Goodbye.")
        sys.exit(0)
    else:
        speak("I didn't understand that command. Say 'help' for examples.")

def print_help():
    print("""
Commands:
  open app <name>
  open project <name>
  open website <url-or-name>
  remember <key> <value>
  recall <key>
  subscribe youtube <channel_id_or_rss>
  add game_monitor <label> <url>
  list monitors
  list youtube
  help
  exit
""")

# ----------------------------
# Optional voice listener (simple)
# ----------------------------
def voice_listener_loop():
    if not VOICE_AVAILABLE:
        print("Voice mode unavailable (SpeechRecognition not installed).")
        return
    r = sr.Recognizer()
    mic = None
    try:
        mic = sr.Microphone()
    except Exception as e:
        print("Microphone init failed:", e)
        return
    with mic as source:
        r.adjust_for_ambient_noise(source)
    speak("Voice mode activated. Say a command after the beep.")
    while True:
        try:
            with mic as source:
                print("Listening...")
                audio = r.listen(source, phrase_time_limit=6)
            try:
                txt = r.recognize_google(audio)
                print("Heard:", txt)
                parse_and_execute(txt)
            except sr.UnknownValueError:
                print("Could not understand audio")
            except sr.RequestError as e:
                print("Speech recognition service error:", e)
        except KeyboardInterrupt:
            break
        except Exception as e:
            print("Voice loop error:", e)
            time.sleep(1)

# ----------------------------
# Main entry
# ----------------------------
def main():
    speak("Jarvis prototype starting.")
    # Start poll loop in background thread
    loop = asyncio.new_event_loop()
    t = threading.Thread(target=loop.run_until_complete, args=(poll_loop(),), daemon=True)
    t.start()
    # Optionally start voice mode
    if config.get("voice_mode", False) and VOICE_AVAILABLE:
        vthread = threading.Thread(target=voice_listener_loop, daemon=True)
        vthread.start()
    # CLI loop
    print("Jarvis console. Type 'help' for commands.")
    while True:
        try:
            cmd = input(">> ")
            parse_and_execute(cmd)
        except (EOFError, KeyboardInterrupt):
            speak("Exiting.")
            break

if __name__ == "__main__":
    main()
