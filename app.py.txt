import os
import random
import requests
import streamlit as st
from moviepy.editor import *
from openai import OpenAI

st.set_page_config(page_title="CoreCore Cloud Factory", layout="centered")
st.title("🎬 CoreCore Cloud Factory")

pexels_key = st.text_input("Pexels API Key", type="password")
openai_key = st.text_input("OpenAI API Key", type="password")
query = st.text_input("Suchbegriff", value="city night lonely")
uploaded_music = st.file_uploader("Ambient / LoFi Musik hochladen (mp3)", type=["mp3"])

def fetch_videos(query, api_key):
    url = f"https://api.pexels.com/videos/search?query={query}&per_page=5"
    headers = {"Authorization": api_key}
    response = requests.get(url, headers=headers).json()
    return [v["video_files"][0]["link"] for v in response["videos"]]

def download_video(url, filename):
    r = requests.get(url)
    with open(filename, "wb") as f:
        f.write(r.content)
    return filename

def extract_clip(path):
    clip = VideoFileClip(path)
    if clip.duration < 4:
        return None
    start = random.uniform(0, clip.duration - 4)
    end = start + random.uniform(2, 4)
    return clip.subclip(start, end)

def generate_text(api_key):
    client = OpenAI(api_key=api_key)
    prompt = """
    Schreibe 3 minimalistische, melancholische Core-Core Sätze
    über Zeit, Einsamkeit und moderne Gesellschaft.
    Maximal 10 Wörter pro Satz.
    """
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content.split("\n")

def build_video(clips, text_lines, music_path):
    total = 0
    selected = []

    for c in clips:
        if total >= 30:
            break
        selected.append(c)
        total += c.duration

    video = concatenate_videoclips(selected, method="compose")
    audio = AudioFileClip(music_path).subclip(0, video.duration)
    video = video.set_audio(audio.volumex(0.6))

    txt_clips = []
    for i, line in enumerate(text_lines):
        txt = TextClip(line, fontsize=60, color='white')
        txt = txt.set_position("center").set_duration(4).set_start(i*8)
        txt_clips.append(txt)

    final = CompositeVideoClip([video, *txt_clips])
    output_path = "corecore_output.mp4"
    final.write_videofile(output_path, fps=24)
    return output_path

if st.button("🎬 Generate Video"):
    if not all([pexels_key, openai_key, uploaded_music]):
        st.error("Bitte alle Felder ausfüllen.")
    else:
        with st.spinner("Erstelle dein CoreCore Video..."):
            urls = fetch_videos(query, pexels_key)
            clips = []
            for i, url in enumerate(urls):
                video_path = download_video(url, f"video_{i}.mp4")
                clip = extract_clip(video_path)
                if clip:
                    clips.append(clip)
            text_lines = generate_text(openai_key)
            with open("music.mp3", "wb") as f:
                f.write(uploaded_music.read())
            output_path = build_video(clips, text_lines, "music.mp3")
            with open(output_path, "rb") as f:
                st.download_button(
                    label="⬇️ Download Video",
                    data=f,
                    file_name="corecore_video.mp4",
                    mime="video/mp4"
                )
            st.success("Fertig!")
