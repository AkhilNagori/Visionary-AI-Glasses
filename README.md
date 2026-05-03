# Visionary AI Glasses

**AI-powered smart glasses that convert printed text into spoken audio for visually impaired students.**

Visionary AI Glasses is a low-cost wearable assistive technology project designed to help visually impaired students access printed classroom materials more independently. The device captures text using a camera, processes it with OCR, converts the recognized text into speech, and plays the audio through a speaker or headphones.

---

## Overview

Many visually impaired students rely on teachers, aides, or digital versions of documents to access printed classroom materials. Visionary was built to make that process faster, cheaper, and more independent.

The glasses use a compact embedded system with a camera, OCR software, text-to-speech processing, and audio output. The goal is to allow a student to point toward printed text and hear it read aloud in real time.

---

## Features

- Real-time text detection and reading
- Text-to-speech audio output
- Low-cost hardware design
- Wearable glasses-based form factor
- Works with printed worksheets, textbooks, and classroom documents
- Designed for visually impaired students
- Built using Python, OpenCV, OCR, and embedded hardware
- Tested across different lighting conditions, font sizes, and document types

---

## Hardware

The prototype uses:

- Raspberry Pi Zero 2 W
- Camera module
- Speaker or audio output device
- Battery power source
- 3D-printed glasses frame
- Basic wiring and mounting components

---

## Software

The system uses:

- Python
- OpenCV
- OCR/text recognition pipeline
- Text-to-speech engine
- Audio playback system
- Raspberry Pi OS

---

## How It Works

1. The camera captures an image of the printed text.
2. The image is processed using computer vision techniques.
3. OCR extracts the text from the image.
4. The extracted text is converted into speech.
5. The audio is played through the connected speaker or headphones.

---

## Project Structure

```txt
Visionary-AI-Glasses/
│
├── src/
│   ├── main.py
│   ├── camera.py
│   ├── ocr.py
│   ├── text_to_speech.py
│   └── audio.py
│
├── hardware/
│   ├── frame_designs/
│   └── wiring_diagram.png
│
├── tests/
│   └── performance_tests.md
│
├── assets/
│   ├── prototype_images/
│   └── demo_videos/
│
├── requirements.txt
└── README.md
