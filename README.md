# 3D Shadow & Light Lab 🧊💡

A lightweight, web-based 3D scene editor and light laboratory. Build scenes using primitives, apply physically-based materials (Glass, Mirror, Wood, Matte), and manipulate lighting with realistic real-time shadows.

Try the live [demo](https://patrickwolf.github.io/3dLab)

## Features ✨
* **Intuitive Transform Tools:** Move, rotate, scale, and box-select objects with ease.
* **Advanced Snapping:** Magnetically snap objects together by their edges, or deform geometry using Point (Vertex) Snapping.
* **Dynamic Mirrors:** Reflective surfaces accurately mirror the surrounding environment using a dynamic cube camera.
* **AI Magic:** Connect your own Google Gemini API key to automatically generate literal 3D scenes (like cities or forests) from text prompts, or have the AI act as an art critic for your creations.
* **Instant URL Sharing:** Compress your entire 3D scene into a single, shareable URL. No database required!
* **Import/Export:** Save your masterpieces locally as JSON files and load them later.

## Tech Stack 🛠️
* **Three.js** - 3D rendering engine
* **Tailwind CSS** - UI styling
* **LZ-String** - URL compression for sharing
* **Google Gemini API** - AI scene generation and critique

## Setup & Deployment 🚀
Because this application is 100% client-side and bundled into a single file, deployment is instant:
1. Clone or download this repository.
2. Open `index.html` in any modern web browser.
3. To host on the web, simply push to a GitHub repository and enable **GitHub Pages** from the repository settings.

## License 📄
This project is licensed under the MIT License - see the LICENSE file for details. Created by [Patrick Wolf](https://github.com/patrickwolf).
