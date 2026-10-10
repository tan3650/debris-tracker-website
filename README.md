# Debris Tracker web console

**Site:** https://tan3650.github.io/debris-tracker-website/frontend/

**Code:** https://github.com/tan3650/debris-tracker-website

**Deployed at** https://tan3650.github.io/debris-tracker-website/frontend/

**TOKEN** (random, changes every start(server)): U47AKzQ7SZU62ymOfBuodcIi

## For judges
This website is using a PC as a server through tunneling since MATLAB Web App Server is a paid service. So, we thought of MATLAB runs on a local PC and the website reaches it through a tunnel.
Open two command prompt windows.

The site has two modes, shown in its Connection window.

**Demo mode (always available).** The site reads the saved results in `data/results/` from a real MATLAB run. It works at any time, on any device, and needs nothing running on our side.

**Live mode (during the presentation only).** Pressing Run sends frames to MATLAB on the presenter's own laptop, and the page then shows the fresh results. This only works while that laptop is on and running MATLAB, a small bridge server (`server/server.py`) and a secure tunnel. The tunnel address and access token change every session, so they are started just before presenting and stopped afterwards. At any other time the site shows demo mode, which is expected, not a fault.

Why a laptop acts as the server: MATLAB Web App Server is a paid service, so MATLAB runs on a local PC and the website reaches it through a tunnel.

### Sample images for testing

Because BMP images are hard to find, a set of real star-field images (Orion) is provided for judges to test with: **[add folder or link here]**. These images have no synthetic debris added and were not used to tune or test the algorithms or models. Upload 3 to 12 BMP files of the same size in live mode. Uploaded images have no ground truth, so the site shows detections and tracks but no accuracy score.

## How it works

```
Browser (GitHub Pages)  ->  tunnel  ->  server/server.py (your PC)  ->  matlab -batch "main; track_debris"
        ^                                                                       |
        +------------------ data/results/*.json, *.csv, *.png <----------------+
```

## Running live mode (server, token and tunnel)

You need two Command Prompt windows. No Cloudflare account or Cloudflare token is needed. These commands are for Command Prompt, not PowerShell.

**One-time setup**

1. Install Python, MATLAB R2021a or newer with the Image Processing Toolbox, and cloudflared:
   ```bat
   winget install --id Cloudflare.cloudflared
   ```
2. Install the server's requirement from the project folder:
   ```bat
   cd server
   pip install -r requirements.txt
   ```
3. In `debris_defaults.m`, make sure `cfg.blink = false;` and `tc.blink = false;`. The blink tool opens an interactive window that hangs batch mode, and the server refuses to run while either is true.
4. Test MATLAB once by hand from the project folder: `matlab -batch "main; track_debris"`. If it does not finish and write `data\results\tracks.json`, the server cannot work either.

**Every time you present**

1. **Window 1: start the server.**
   ```bat
   cd C:\path\to\debris-tracker\server
   set ALLOWED_ORIGINS=https://tan3650.github.io
   python server.py
   ```
   The server prints a random access token on the line `TOKEN (random, changes every start)`. Copy it. To use your own fixed token instead, run `set DEBRIS_TOKEN=your-long-random-secret` (20 or more characters) before `python server.py`. If MATLAB is not found, set its full path first, for example `set MATLAB_EXE=C:\Program Files\MATLAB\R2024b\bin\matlab.exe`.

2. **Window 2: start the tunnel.**
   ```bat
   cloudflared tunnel --url http://127.0.0.1:5000
   ```
   It prints an address like `https://random-words.trycloudflare.com`. This address changes every time the tunnel starts.

3. **Check the tunnel.** Open the tunnel address followed by `/api/health` in a browser. It should show text starting with `{"ok":true`.

4. **Connect the site.** Open the site and, in the Connection window, enter the full tunnel address (nothing after `.com`) and the token. Press **Connect**, then run on the sample data or upload BMP frames.

5. **Stop everything when you finish.** Press Ctrl+C in both windows. Anyone with the tunnel address and the token can start MATLAB on this PC, so the tunnel should not stay open.

**Never write the tunnel address or the token in this README or in any file in the repository.**

**If something fails**

- **"Failed to fetch":** check that the server window is running, that `ALLOWED_ORIGINS` was set before `python server.py`, and that the Server URL has no trailing path.
- **502 error on the tunnel address:** the server is not running on port 5000.
- **MATLAB not found:** set `MATLAB_EXE` as shown above.

### Test locally first (no tunnel)

```bat
:: window 1
cd C:\path\to\debris-tracker\server
set DEBRIS_TOKEN=choose-a-long-random-secret
set ALLOWED_ORIGINS=https://tan3650.github.io,http://localhost:8000
python server.py
```

```bat
:: window 2, from the project folder
python -m http.server 8000
```

Open `http://localhost:8000/frontend/`, enter `http://127.0.0.1:5000` and your token, press Connect, then Run on the sample data.

## Publishing the site

Push the repository, then on GitHub go to Settings, Pages, and deploy from branch `main`, folder `/ (root)`. Demo mode reads `data/results/` and `data/frames_bmp_debris/`, so commit those folders.

## What happens on an upload run

Uploaded frames are saved to a separate run folder under your temp directory, and MATLAB is pointed at it through the `DEBRIS_IN_DIR`, `DEBRIS_OUT_DIR` and `DEBRIS_NO_TRUTH` environment variables (read by `debris_paths.m`). The sample frames, `debris_truth.mat` and the saved results are never moved or overwritten, and only the newest 3 upload runs are kept. Uploaded runs have no ground truth, so the accuracy panel says so.

## Safety (this endpoint can run MATLAB on your PC)

- Use a long random token and never commit it. Anyone with the tunnel address and the token can start a MATLAB run.
- Only run the tunnel while presenting, and stop it afterwards.
- Do not widen `ALLOWED_ORIGINS` to `*`. The server only accepts the listed sites.
- The server never runs anything the page sends: the command is fixed (`MATLAB_CMD`). Uploads are checked as real BMPs, limited to 12 files of 8 MB each, and stored under names the server picks.
- It listens on 127.0.0.1 only, so the tunnel is the only way in.

## Known limits

- Live mode only works while the PC, MATLAB, the server and the tunnel are all running. Otherwise use demo mode.
- One run at a time. A run times out after 20 minutes (`MATLAB_TIMEOUT`).
- The track window reads `tracks.csv` for the paths and `tracks.json` for speed, heading, predictions, ellipses and PINN. If an overlay is missing, check the structure of that file.
- Coordinates are image pixels, not sky coordinates.
