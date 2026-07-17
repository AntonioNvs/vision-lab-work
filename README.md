# VisionLab — Multi-Phone Motion Capture

Scaling Google Research's libsoftwaresync beyond the 11-device cap using a Raspberry Pi 5 access point.

📄 **[Read the full report](https://antoniocaetanonevesneto.github.io/visionlab/)**

## Quick Links

- [Full Academic Report](docs/index.md)
- [libsoftwaresync (patched)](libsoftwaresync/)

## How to Set Up GitHub Pages

### 1. Create the GitHub repository

```bash
cd ~/visionlab
git init
git add .
git commit -m "Initial commit: Pi 5 AP for multi-phone motion capture"
```

Then create a repo on GitHub (e.g., `visionlab`) and push:

```bash
git remote add origin git@github.com:antoniocaetanonevesneto/visionlab.git
git branch -M main
git push -u origin main
```

### 2. Enable GitHub Pages

1. Go to `https://github.com/antoniocaetanonevesneto/visionlab/settings/pages`
2. Under **"Build and deployment"**:
   - **Source:** "Deploy from a branch"
   - **Branch:** `main`
   - **Folder:** `/docs`
3. Click **Save**.

Your site will be live at `https://antoniocaetanonevesneto.github.io/visionlab/` within ~1 minute.

### 3. Verify

Visit `https://antoniocaetanonevesneto.github.io/visionlab/` — you should see the formatted report.

## Project Structure

```
visionlab/
├── docs/
│   ├── _config.yml    # Jekyll theme config
│   └── index.md        # Full academic report
├── libsoftwaresync/    # Patched Android app
└── README.md           # This file
```
