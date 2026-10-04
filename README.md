# Mac Headless Local-AI Server: Setup Guide

**Goal #1 (this guide):** Always-on, fully local voice control for Home Assistant (HA), with web lookups.

**Goal #2 (later):** Other AI use(s)

---

## 0. Decisions at a glance

| Topic | Decision | Why |
|---|---|---|
| Home Assistant location | Stays on its own hardware | Already running; keeps your smart home independent of Mac reboots/experiments |
| Mac's role | Voice + LLM backend: STT, LLM, TTS | Fully local pipeline, as you requested |
| Wake word | On-device on each Voice PE (microWakeWord), custom model trained on the Mac | Voice PE detects the wake word on the device itself, so no audio streams until it triggers |
| STT | **Whisper `large-v3-turbo`** via `wyoming-mlx-whisper` (MLX, runs on the GPU) on port 10300 | Native Apple Silicon speed; accurate and low-latency (§6) |
| TTS | **Pocket TTS** (`kyutai/pocket-tts`, CPU) via a Wyoming wrapper on port 10200 | Your pick. ~100M parameters, runs on CPU, low latency. Needs a free Hugging Face account to accept its terms (§7) |
| LLM | `llama-server` (brew `llama.cpp`) on port 8080: **Gemma 4 26B-A4B (QAT, `UD-Q4_K_XL`)**, 3 parallel slots | Your pick. Mixture-of-experts with ~4B active parameters, so it decodes fast, but all ~14 GB of weights stay resident (§5.1, footnote [^mem]) |
| HA to LLM link | Core **llama.cpp** integration (added in HA 2026.8) | Official, no custom code |
| Web search | **Free, no accounts:** self-hosted SearXNG + the Wikipedia tool, via the "Tools for Assist" HACS integration. Brave stays an optional upgrade | No sign-up or credit card; Wikipedia answers most "who/what" questions directly (§8.4) |
| Sophia NLU | **Optional, evaluate after baseline works** (§10) | Real product, but it solves a different problem than web search |
| Service management | `launchd` LaunchAgents + auto-login | Survives reboots and power loss without anyone at the keyboard |
| Accounts | `insert_username` (sudo, rarely used) + `home_assistant_account` (standard user, auto-login, runs the voice stack). A `other_ai_use_account` account is added when Goal #2 starts | The account that auto-logs-in shouldn't be an admin, and research work gets walled off from the voice stack's files and keys (§2.1, §12) |
| Goal #2 coexistence | Separate ports/dirs, raised GPU memory limit, a separate macOS user for research | Voice stays responsive while you experiment |

### Architecture

```
                         ┌─────────────── Home Assistant box (separate) ───────────────┐
 Voice PE x4+  ──ESPHome─▶  Assist pipeline ── Wyoming STT ──────────────────┐         │
 (wake word on  ◀────────  (conversation agent: llama.cpp integration)         │         │
  device)                  Tools for Assist (SearXNG, Wikipedia, weather)       │         │
                         └────────────────┬───────────────────────────────────┼─────────┘
                                          │ HTTP :8080/v1                      │ :10300 / :10200
                         ┌────────────────▼────────────────────────────────────▼─────────┐
                         │ Mac mac_hostname  (headless)                                       │
                         │ llama :8080  whisper :10300  tts :10200  searxng :8888        │
                         │  (later: other llama-server :8081, MLX fine-tuning, etc.)   │
                         └────────────────────────────────────────────────────────────────┘
```

### What "Sophia NLU" actually is

Your hunch is partly right. Sophia is a real, commercial, closed-source **deterministic intent parser** for HA (Rust, ~160 MB RAM, no GPU). It runs as an add-on **on the HA box**, not on the Mac. It turns sentences like "turn on the kitchen and living room lights and set the temp to 23" into HA intents in milliseconds, with no hallucinations. It is **not** a replacement for an LLM and does **not** do web search. It has an optional LLM fallback (an "Other / OpenAI Compatible" provider exists) for things it can't parse. Pricing: 14-day trial, then one-time $79.95 (12 months of upgrades; $19.95/yr optional after). The "99%" accuracy figure comes from the vendor's own open-sourced test suite, so run it against your own entities before trusting it. See §10.

---

## 1. Prerequisites and assumptions

I stopped the interview once the key forks were settled. These are my assumptions; tell me if any are wrong:

- HA is running on dedicated hardware and is reachable from the Mac's LAN. HA version is **2026.8 or newer** (needed for the core llama.cpp integration). [VERIFY: Settings > About]
- HACS is (or can be) installed on HA.
- Mac mac_hostname connects via **Ethernet**.
- You can temporarily attach a monitor, keyboard and mouse for first-time setup.
- English only.
- Each Voice PE is a Home Assistant Voice Preview Edition (ESP32-S3). Other ESPHome satellites also work.
- You're comfortable on the command line.

**Have ready:** a router login (DHCP reservation), a UPS if you have one, and your HA admin login. No paid or card-backed accounts are required.

### Port plan (keep this; Goal #2 uses neighbouring ports)

| Port | Service | Used by |
|---|---|---|
| 22 | SSH | You |
| 5900 | Screen Sharing | You (backup) |
| 8080 | `llama-server` (voice LLM) | HA |
| 8081 | `llama-server` (other LLM, **later**) | You only. Do **not** expose to HA |
| 8888 | SearXNG (free web search) | HA |
| 10200 | Pocket TTS Wyoming server (TTS) | HA |
| 10300 | `wyoming-mlx-whisper` (STT) | HA |

---

## 2. Phase 1: macOS baseline for headless operation

Do this section with a monitor attached.

### 2.1 Update, name the machine, and create accounts

1. Complete Setup Assistant. Create the **`insert_username`** account (Administrator). Don't sign in with an Apple ID unless you need to.
2. Update macOS fully: **System Settings > General > Software Update**.
3. Set a stable hostname:

```bash
sudo scutil --set ComputerName "mac_hostname"
sudo scutil --set LocalHostName "mac_hostname"
sudo scutil --set HostName "mac_hostname"
```

4. Create the service account: **System Settings > Users & Groups > Add User...** Set the type to **Standard** (not Administrator), full name "Home Voice", account name **`home_assistant_account`**. Give it a strong password.

Each phase below says **Run as:** `insert_username` or `home_assistant_account`. Over SSH that means `ssh admin@mac_hostname.local` or `ssh home@mac_hostname.local`.

**Which account does what**

| Account | Type | Logs in | Runs |
|---|---|---|---|
| `insert_username` | Administrator | Only when you SSH in for maintenance | Anything needing `sudo`: `brew install/upgrade`, `pmset`, firewall, GPU limit, macOS updates |
| `home_assistant_account` | **Standard** | **Automatically at boot** | The voice stack (llama-server, Whisper, Pocket TTS, SearXNG), plus its API keys, LaunchAgents and logs |
| `other_ai_use_account` | Standard | Only when you do research (SSH or Fast User Switching) | other models, fine-tuning, attack tooling. **Create later**, see §12 |

**Why this split is worth it**
- The account that auto-logs-in at the console is the one anyone with physical access sees. It should have no admin rights, so `home_assistant_account` is standard and `insert_username` never auto-logs-in.
- Separate accounts mean separate files. A research script or AI agent running as `other_ai_use_account` can't read the voice stack's API keys, edit its LaunchAgents, or touch its models.
- Day-to-day, you rarely need `insert_username`, which keeps `sudo` out of routine operation.

**What accounts do *not* do:** they don't partition RAM/GPU (§4 does that), and they aren't a strong sandbox against genuinely hostile code or a network barrier. §12 covers VMs and network isolation for the other work.

**Why not create all three now?** `other_ai_use_account` doesn't exist until Goal #2 begins, and nothing in the voice setup needs it. Create it then.

### 2.2 Enable remote access

**System Settings > General > Sharing:**
- **Remote Login**: ON (SSH). Allow access for `insert_username` and `home_assistant_account` (add `other_ai_use_account` later). Not "all users".
- **Screen Sharing**: ON (your backup GUI).

> **Headless GUI note:** Screen Sharing on a Mac with no display attached can give you a low-resolution virtual display. If it's unreliable or sluggish, buy a cheap **HDMI dummy plug** (about $10). It's the standard fix.

### 2.3 Never sleep, and restart after power loss

```bash
sudo pmset -a sleep 0          # never system-sleep
sudo pmset -a disksleep 0
sudo pmset -a displaysleep 10  # irrelevant headless, harmless
sudo pmset -a autorestart 1    # power back on after a power failure
sudo pmset -a womp 1           # wake for network access
sudo pmset -a powernap 0
pmset -g                       # verify
```

Also check **System Settings > Energy**: enable "Start up automatically after a power failure" and "Wake for network access" if present.

### 2.4 Auto-login and FileVault (an important trade-off)

Your services will run as **LaunchAgents**, which only start when the user is logged in. For a Mac that must recover unattended after a power cut, you need **automatic login**, and macOS only permits auto-login when **FileVault is off**.

- **Recommended for this use case:** FileVault OFF, automatic login ON for the **`home_assistant_account`** account (standard user, so the account exposed at the console has no admin rights). Mitigate by keeping the machine in a physically secure place, SSH key-only login (§2.6), and a firewall posture (§2.7).
- **If you require FileVault ON:** after any reboot or power loss the Mac will sit at the pre-boot unlock screen and **your voice assistant stays down until someone enters the password**. That conflicts with "always on". Plan around it with a UPS and don't enable it unless you accept this.

Set it: **System Settings > Users & Groups > Automatically log in as...** and choose **`home_assistant_account`** (the option is greyed out if FileVault is on).

### 2.5 Stop surprise reboots

macOS auto-installing updates and restarting is a risk for an always-on device. In **Software Update > Automatic updates (ⓘ)**, turn OFF "Install macOS updates" (keep security responses/data files if you like). Update manually in a maintenance window (§11.4).

### 2.6 SSH hardening

From **your other computer**:

```bash
ssh-keygen -t ed25519 -C "you@laptop"          # skip if you already have a key
ssh-copy-id admin@mac_hostname.local
ssh-copy-id home@mac_hostname.local
ssh admin@mac_hostname.local                          # confirm key login works
ssh home@mac_hostname.local
```

Then, **as `insert_username`** on the Mac, disable password auth:

```bash
sudo tee /etc/ssh/sshd_config.d/100-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
sudo launchctl kickstart -k system/com.openssh.sshd 2>/dev/null || true
```

> Keep your original SSH session open while testing a second login, so you can't lock yourself out. Screen Sharing is your safety net.

### 2.7 Network and firewall

1. In your router, create a **DHCP reservation** for the Mac's Ethernet MAC address (so HA's configured IP never changes). Note the IP; this guide calls it `MAC_IP`.
2. **Firewall gotcha:** the macOS application firewall pops up an *interactive* "allow incoming connections?" dialog for each new unsigned binary. Headless, nobody can click it and the connection silently fails. Pick one:
   - **Simplest:** leave the macOS application firewall **off** and restrict access at your router (only HA's IP and your admin machine may reach ports 8080/10200/10300).
   - **Stricter:** turn it on and pre-approve each binary after you install it (done in each phase below):
     ```bash
     sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
     sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /path/to/binary
     sudo /usr/libexec/ApplicationFirewall/socketfilterfw --unblockapp /path/to/binary
     ```
3. Consider putting HA, the Voice PEs, and the Mac in a trusted VLAN and keeping security-research traffic elsewhere (§12).

### 2.8 Reboot test (do this now)

```bash
sudo reboot
```

Unplug the monitor first if you want a realistic test. Confirm that `ssh admin@mac_hostname.local` works within ~2 minutes. Then also test a **hard power pull** later once everything is installed (§11.5).

---

## 3. Phase 2: Tooling and directory layout

**Run as `insert_username`:**

```bash
# Xcode command line tools, then Homebrew
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile && source ~/.zprofile

brew install llama.cpp uv git jq tmux
llama-server --version
```

Homebrew belongs to `insert_username`. Only `insert_username` runs `brew install/upgrade`; `home_assistant_account` just runs the installed programs.

**Run as `home_assistant_account`** (so the voice stack's files and keys are owned by the service account):

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile && source ~/.zprofile   # puts brew-installed tools on PATH
llama-server --version

mkdir -p ~/ai/{bin,logs,models/gguf,voice,secrets,searxng}
chmod 700 ~/ai/secrets
```

| Path (in `home_assistant_account`'s home folder) | Purpose |
|---|---|
| `~/ai/bin` | wrapper scripts launchd runs |
| `~/ai/logs` | service logs |
| `~/ai/models/gguf` | voice-stack models |
| `~/ai/voice` | STT/TTS models and servers, wake-word training |
| `~/ai/searxng` | SearXNG config |
| `~/ai/secrets` | API keys (mode 700) |

The other work gets its own equivalent folders inside the `other_ai_use_account` account later (§12), not here.

---

## 4. Phase 3: GPU memory budget

Apple Silicon shares one memory pool. By default macOS only lets the GPU "wire" roughly three-quarters of RAM, which on 64 GB is about 48 GB. Because the voice stack must stay resident while Goal #2 runs, raise the limit a bit and budget deliberately.

**Approximate budget (64 GB), summarized.** Per-component detail, and which numbers are measured versus estimated, is in footnote [^mem].

| Component | Runs on | Approx. memory |
|---|---|---|
| macOS + background | CPU/RAM | 6–8 GB |
| Gemma 4 26B-A4B, Q4 (weights + KV cache + buffers) | **GPU** | ~16–18 GB |
| Whisper `large-v3-turbo` (MLX) | **GPU** | ~2–3 GB |
| Pocket TTS | CPU | ~1–1.5 GB |
| SearXNG (Colima VM + container) | CPU | ≤2 GB |
| **Goal #1 total (including macOS)** | | **~26–33 GB** |
| **Left for Goal #2** | | **~31–38 GB** [^mem] |

The LLM and Whisper both use GPU-wired memory (~18–21 GB together); TTS and SearXNG are ordinary RAM. All of it comes out of the same 64 GB pool, so **total RAM, not the GPU limit, is the binding constraint**.

**Run as `insert_username`.** Set the GPU wired limit to 56 GB (57344 MB) and persist it across reboots with a LaunchDaemon:

```bash
sudo sysctl iogpu.wired_limit_mb=57344    # takes effect now, lost at reboot

sudo tee /Library/LaunchDaemons/com.local.gpu-wired-limit.plist >/dev/null <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.local.gpu-wired-limit</string>
  <key>ProgramArguments</key>
  <array><string>/usr/sbin/sysctl</string><string>iogpu.wired_limit_mb=57344</string></array>
  <key>RunAtLoad</key><true/>
</dict></plist>
EOF
sudo chown root:wheel /Library/LaunchDaemons/com.local.gpu-wired-limit.plist
sudo chmod 644 /Library/LaunchDaemons/com.local.gpu-wired-limit.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/com.local.gpu-wired-limit.plist
sysctl iogpu.wired_limit_mb
```

> Don't set this too close to total RAM. Leave at least 6–8 GB for macOS or the whole machine (and your voice assistant) can stall under pressure.

---

## 5. Phase 4: Voice LLM server (llama.cpp)

**Run as `home_assistant_account`** (SSH in as `home_assistant_account`, or use Screen Sharing). Firewall approvals, if you use them, are `sudo` commands run as `insert_username`.

### 5.1 The model

**Gemma 4 26B-A4B-it, QAT, GGUF**, from `unsloth/gemma-4-26B-A4B-it-qat-GGUF`, quant **`UD-Q4_K_XL`** (14.2 GB file).

What to know about it:
- **Mixture-of-experts.** 25.2B total parameters, but only ~3.8B are active per token, so generation speed is closer to a 4B model than a 26B one. The catch is memory: *all* ~14 GB of weights must be resident. That's why this section's footprint is ~16–18 GB (footnote [^mem]) rather than the ~8 GB of a small dense model.
- **Native function calling and a proper system role**, which is what Home Assistant's tool-driven Assist needs. QAT (quantization-aware training) means the 4-bit weights were trained to track the full-precision model closely.
- **Thinking mode should stay off for voice.** Reasoning tokens are seconds of dead air. §5.2 pins it off explicitly.
- **Vision isn't needed.** The model accepts images, so we pass `--no-mmproj` so the vision encoder (~550M parameters) is never loaded.
- **Sampling:** Google's recommended settings are temperature 1.0, top-p 0.95, top-k 64, which §5.2 uses as defaults. If tool calls turn out flaky, lower the temperature (try 0.3–0.6).
- **Needs a recent llama.cpp** that knows the `gemma4` architecture. If `llama-server` complains about an unknown architecture, run `brew upgrade llama.cpp` as `insert_username`.
- **Optional speed-up for later:** the repo ships a small Multi-Token-Prediction drafter that newer llama.cpp can use (`--spec-type draft-mtp --spec-draft-n-max 4`). Leave it off until the baseline is stable, and only try it if your Homebrew build supports it.

> **[VERIFY]** `--no-mmproj`, `--chat-template-kwargs` and the MTP flags change between llama.cpp releases. Confirm with `llama-server --help` after installing.

### 5.2 Test run in the foreground

```bash
export LLAMA_CACHE=~/ai/models/gguf       # keep downloads in one place

llama-server \
  -hf unsloth/gemma-4-26B-A4B-it-qat-GGUF:UD-Q4_K_XL \
  --no-mmproj \
  --alias voice \
  --host 0.0.0.0 --port 8080 \
  -c 24576 -np 2 \
  -ngl 99 --jinja \
  --reasoning off \
  --spec-type draft-mtp --spec-draft-n-max 2 \
  --temp 1.0 --top-p 0.95 --top-k 64
```

The first run downloads ~14 GB, so give it several minutes. When it's up, **read the memory numbers from the startup log** (the model buffer, KV cache and compute buffer sizes) and compare them to footnote [^mem].

What the flags do:
- `-np 3` gives three parallel slots, meaning up to three LLM requests processed at the same moment.
- `-c 36864` is the **total** context, divided across slots, so ~12k per slot. HA's Assist prompt (tools plus exposed entities) is long; the integration's docs recommend at least ~10k per request. If you expose many entities, raise `-c` (and remember it's divided by `-np`).
- `--jinja` enables the model's chat template, which tool calling needs.
- `--no-mmproj` skips the vision encoder; `--chat-template-kwargs` turns Gemma's thinking off; `--temp/--top-p/--top-k` are Google's recommended sampling defaults.
- `--host 0.0.0.0` makes it reachable from HA. Protect it with the API key below.

**How many slots do you actually need?** Slots count simultaneous *LLM requests*, not rooms.
- Most commands (lights, timers, "prefer local" matches) never reach the LLM at all, so real LLM concurrency is much lower than satellite concurrency.
- If every slot is busy, the next request **waits in a queue**; it isn't rejected. A short voice reply finishes in a second or two, so a rare extra simultaneous request just waits briefly.
- Memory is the trade-off, but Gemma 4's sliding-window attention keeps its KV cache modest. Dropping to 2 slots (`-c 24576 -np 2`) shrinks it proportionally; check the startup log for the actual saving. Don't shrink the *per-slot* context to save memory, because the Assist prompt has to fit.
- **Recommendation:** 3 slots. Use 2 if you'd rather give the memory to Goal #2 and accept an occasional short wait.

In a second terminal:

```bash
curl -s http://localhost:8080/v1/models | jq .
curl -s http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"voice","messages":[{"role":"user","content":"Say hi in five words."}]}' | jq -r '.choices[0].message.content'
```

Stop it with Ctrl-C once it works.

### 5.3 API key and wrapper script

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))" > ~/ai/secrets/llama-voice.key
chmod 600 ~/ai/secrets/llama-voice.key

cat > ~/ai/bin/start-llama-voice.sh <<'EOF'
#!/bin/bash
export LLAMA_CACHE="$HOME/ai/models/gguf"
exec /opt/homebrew/bin/llama-server \
    -hf unsloth/gemma-4-26B-A4B-it-qat-GGUF:UD-Q4_K_XL \
  --no-mmproj \
  --alias voice \
  --host 0.0.0.0 --port 8080 \
  -c 24576 -np 2 \
  -ngl 99 --jinja \
  --reasoning off \
  --spec-type draft-mtp --spec-draft-n-max 2 \
  --temp 1.0 --top-p 0.95 --top-k 64 \
  --api-key "$(cat "$HOME/ai/secrets/llama-voice.key")"
EOF
chmod +x ~/ai/bin/start-llama-voice.sh
```

### 5.4 Run it as a LaunchAgent

```bash
mkdir ~/Library/LaunchAgents
cat > ~/Library/LaunchAgents/com.local.llama-voice.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.local.llama-voice</string>
  <key>ProgramArguments</key><array><string>$HOME/ai/bin/start-llama-voice.sh</string></array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>ThrottleInterval</key><integer>15</integer>
  <key>StandardOutPath</key><string>$HOME/ai/logs/llama-voice.log</string>
  <key>StandardErrorPath</key><string>$HOME/ai/logs/llama-voice.err</string>
</dict></plist>
EOF

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.local.llama-voice.plist
launchctl print gui/$(id -u)/com.local.llama-voice | head -20
tail -f ~/ai/logs/llama-voice.err        # watch model load; Ctrl-C to exit
```

Useful commands:

```bash
launchctl kickstart -k gui/$(id -u)/com.local.llama-voice   # restart
launchctl bootout   gui/$(id -u)/com.local.llama-voice      # stop/unload
```

If you chose the stricter firewall: `sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /opt/homebrew/bin/llama-server` and `--unblockapp` the same path. [VERIFY: Homebrew may symlink; if approval doesn't stick, use the real path from `readlink -f $(which llama-server)`.]

---

## 6. Phase 5: Speech-to-text (Whisper on Apple Silicon)

**Run as `home_assistant_account`.**

`wyoming-mlx-whisper` serves Whisper on the **GPU via MLX** and speaks the Wyoming protocol Home Assistant understands. The default model is `mlx-community/whisper-large-v3-turbo` (~1.6 GB), a good balance of accuracy and low latency.

Model options, from the project's README:

| Model | Size | Notes |
|---|---|---|
| `mlx-community/whisper-large-v3-turbo` (default) | ~1.6 GB | Best accuracy; near real-time |
| `mlx-community/whisper-large-v3-turbo-q4` | smaller (4-bit) | Trims memory at some accuracy cost. [VERIFY it's listed in the current README] |
| `mlx-community/whisper-small-mlx` | ~481 MB | Fast, "good" accuracy |
| `mlx-community/whisper-tiny` | ~75 MB | Basic; not recommended for voice commands |

### 6.1 Install and test

**Safer than piping curl to bash:** download the installer, read it, then run it.

```bash
curl -fsSL https://raw.githubusercontent.com/basnijholt/wyoming-mlx-whisper/main/scripts/install_service.sh -o /tmp/install_whisper.sh
nano /tmp/install_whisper.sh    # Read it / change log dir to the one we created earlier
bash /tmp/install_whisper.sh
```

It installs a launchd service listening on `tcp://0.0.0.0:10300`. Logs: `~/Library/Logs/wyoming-mlx-whisper/` or whatever you changed to.

```bash
tail -f ~/Library/Logs/wyoming-mlx-whisper/*.log
nc -vz localhost 10300              # port open?
wyoming-mlx-whisper --help          # [VERIFY] flags such as --model, --language, --initial-prompt
```

The first start downloads the model, so allow a minute or two before the port opens.

### 6.2 Help it with your device names

Whisper has never heard of your "Ecobee" or "standing lamp", so it may mishear them. The `--initial-prompt` option biases it toward words you list (device names, room names, "sportsteam"). Edit the installer's LaunchAgent and add it:

```bash
launchctl list | grep -i whisper                      # note the label, e.g. something like com.<name>.wyoming-mlx-whisper
ls ~/Library/LaunchAgents | grep -i whisper           # find its plist
open -e ~/Library/LaunchAgents/<that-file>.plist      # or edit it over SSH with nano
```

In the plist's `ProgramArguments` array, append two entries:

```xml
<string>--initial-prompt</string>
<string>Kitchen light, living room device, Ecobee, standing lamp, sportsteam</string>
```

Then reload it: `launchctl bootout gui/$(id -u) <plist>` followed by `launchctl bootstrap gui/$(id -u) <plist>`. Keep the list short (a few dozen names at most), since very long prompts degrade Whisper's quality.

> **Alternative if you don't want to maintain that list by hand:** the Open Home Foundation's `wyoming-faster-whisper` can read the names of your *conversation-exposed* entities (plus area and floor names) from Home Assistant on every request and use them as the prompt automatically (`--hass-api` plus a long-lived token). The trade-off is that on a Mac it runs on the **CPU** (faster-whisper has no Metal backend), so it's slower than the MLX server, and it needs a standard Whisper model (distilled "distil-*" checkpoints don't work with prompts). [VERIFY against its README if you go this way]

### 6.3 Notes

- **Concurrency:** Whisper on Metal handles a short voice command in well under a second on this hardware class (my estimate), so simultaneous requests from two or three rooms just queue briefly.
- **GPU sharing:** Whisper and the LLM use the same GPU. Each STT request is a short burst, but one that lands while Gemma is working through a long prompt may take a little longer. If you notice STT lag during LLM replies, try the `-q4` or `small` model first.
- **Updates:** the installer set up the service, so to update it re-run the installer (or upgrade the package it installed) and restart its LaunchAgent (§11.4).

---

## 7. Phase 6: Text-to-speech (Pocket TTS)

**Run as `home_assistant_account`.**

Pocket TTS (Kyutai) is a ~100M-parameter model built for CPUs: about 200 ms to first audio, roughly 6x faster than real time on a MacBook Air M4, using about 2 CPU cores. That's why it fits well here: it doesn't compete with the LLM and Whisper for the GPU.

Things to know:
- **It's a gated model.** Downloading the weights requires a free Hugging Face account and accepting Kyutai's terms. No payment is involved. It's a one-time download, after which it can run offline.
- **Voices:** the built-in English voices are `alba` (the default), `marius`, `javert`, `jean`, `fantine`, `cosette`, `eponine` and `azelma`. Stick to built-in voices: the terms prohibit voice cloning or impersonation without explicit, lawful consent.
- **Licence:** CC-BY-4.0 for the model; fine for home use.
- **Wyoming:** Pocket TTS doesn't ship a Wyoming server. This guide uses the community **`ikidd/pocket-tts-wyoming`** (a single Python file on top of `pocket-tts`). It's distributed mainly as a Docker image, but its Dockerfile shows it's just a script, so the steps below run it **natively**, which avoids spending VM memory inside Colima.

### 7.1 One-time Hugging Face access

1. Create a free account at huggingface.co.
2. Open `huggingface.co/kyutai/pocket-tts` while logged in and accept the conditions.
3. Create a **read-only access token** (Settings > Access Tokens) and save it:

```bash
printf '%s' 'hf_PASTE_TOKEN_HERE' > ~/ai/secrets/hf-token.key && chmod 600 ~/ai/secrets/hf-token.key
```

### 7.2 Install the Wyoming server

```bash
mkdir -p ~/ai/voice/pocket-tts && cd ~/ai/voice/pocket-tts
git clone https://github.com/kyutai-labs/pocket-tts.git src
cd src
curl -fsSL https://raw.githubusercontent.com/ikidd/pocket-tts-wyoming/master/wyoming_tts_server.py -o wyoming_tts_server.py
uv python install 3.14
uv python pin 3.14
uv sync
uv add "wyoming>=1.8,<2" zeroconf
```

This mirrors the project's Dockerfile. For stability, note the commit you cloned (`git rev-parse HEAD`) and update deliberately rather than automatically.

### 7.3 Test it

```bash
export HF_TOKEN="$(cat ~/ai/secrets/hf-token.key)"
WYOMING_PORT=10200 WYOMING_HOST=0.0.0.0 DEFAULT_VOICE=alba MODEL_VARIANT=b6369a24 ZEROCONF= \
  uv run python wyoming_tts_server.py
```

`ZEROCONF=` (empty) turns off mDNS announcements, since you'll add the server to HA by IP. The first run downloads ~500 MB of weights. In another terminal: `nc -vz localhost 10200`. Ctrl-C when it works.

> **[VERIFY]** The variable names and `MODEL_VARIANT` default come from the project's README at the time of writing. If the server logs an unknown-variant or auth error, re-check them against the current README and confirm you accepted the model terms with the same account that issued the token.

### 7.4 Run it as a service

```bash
cat > ~/ai/bin/start-tts.sh <<'EOF'
#!/bin/bash
export HF_TOKEN="$(cat "$HOME/ai/secrets/hf-token.key")"
export WYOMING_PORT=10200 WYOMING_HOST=0.0.0.0 DEFAULT_VOICE=alba MODEL_VARIANT=b6369a24 ZEROCONF=
# After the first successful download, uncomment to stop all network calls:
# export HF_HUB_OFFLINE=1
cd "$HOME/ai/voice/pocket-tts/src"
exec /opt/homebrew/bin/uv run python wyoming_tts_server.py
EOF
chmod +x ~/ai/bin/start-tts.sh

cat > ~/Library/LaunchAgents/com.local.pocket-tts.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.local.pocket-tts</string>
  <key>ProgramArguments</key><array><string>$HOME/ai/bin/start-tts.sh</string></array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>ThrottleInterval</key><integer>15</integer>
  <key>StandardOutPath</key><string>$HOME/ai/logs/tts.log</string>
  <key>StandardErrorPath</key><string>$HOME/ai/logs/tts.err</string>
</dict></plist>
EOF
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.local.pocket-tts.plist
nc -vz localhost 10200
```

**If the first word gets clipped:** the wrapper prepends a sacrificial `...` and trims it from the audio, because audio-prompt TTS models can swallow the first word. The project documents tuning knobs (`PREFIX_MIN_DURATION`, `PREFIX_MAX_DURATION`, `PREFIX_SILENCE_GAP`) in its README.

**Docker alternative (if the native route gives you trouble):** run the project's prebuilt image `ghcr.io/ikidd/pocket-tts-wyoming:latest` in Colima. Its compose file uses host networking, which inside Colima is the *VM's* network, not the Mac's, so publish the port instead (`-p 10200:10201`, with `-e DEFAULT_VOICE=alba -e MODEL_VARIANT=b6369a24 -e ZEROCONF= -e HF_TOKEN=...`), and raise the Colima VM's memory cap (§8.5) by ~2 GB. [VERIFY the image has an arm64 build.]

---

## 8. Phase 7: Home Assistant side

### 8.1 Add the Wyoming services

In HA: **Settings > Devices & services > Add integration > Wyoming Protocol**:
1. Host `MAC_IP` or `MAC_FQDN`, port `10300` (speech-to-text).
2. Add again: host `MAC_IP` or `MAC_FQDN`, port `10200` (text-to-speech).

Rename each device so you can tell them apart from any local add-ons (e.g. "Mac Whisper", "Mac Pocket TTS").

### 8.2 Add the LLM

HA 2026.8+ includes a core **llama.cpp** integration: **Add integration > llama.cpp**.
- Base URL: `http://MAC_IP:8080/v1`
- API key: the value in `~/ai/secrets/llama-voice.key`

Then add a conversation agent from it, and set:
- **Control Home Assistant:** Assist (so the model can call HA tools).
- **Prefer handling commands locally** (if offered): ON. Simple things like "turn on the kitchen light" are then answered by HA's built-in intent matcher instantly, and the LLM is only used for what it can't handle. This is faster and more reliable for the on/off/timer majority of your use.
- Keep the prompt short, and **expose only the entities you actually want voice control over** (Settings > Voice assistants > Expose). Fewer exposed entities means a smaller prompt, faster responses, and fewer mistakes.
- I used this prompt:
  ```
  You are a voice assistant for a smart home. Your replies are spoken aloud.
  
  Style:
  - Reply in one or two short sentences of plain text. No markdown, lists, emoji, or symbols.
  - Say numbers and units the way you would speak them ("seventy-one degrees").
  - After a command, confirm briefly ("Done." or "Living room lights are on.").
  -  Never ask follow-up questions. If the request is unclear, inaudible, or empty, give a very short reply or say nothing.
  
  Tool rules:
  - To change anything in the home, you MUST call a tool. Never say you did something unless the tool call succeeded. If it failed, say so.
  - For any question about current state (on/off, temperature, locked, playing), call GetLiveContext first. Never guess or answer from memory.
  - Only use device names and areas that appear in the device list. Never invent names.
  - If a request could match several devices and the area doesn't settle it, ask one short clarifying question.
  - If nothing in the home matches, say you couldn't find it and then stop.
  
  Examples:
  - "Turn on the lights" -> turn on the lights in the current area.
  - "Is it cold upstairs?" -> call GetLiveContext, then report the Upstairs temperature.
  - "Turn off the Christmas tree" -> turn off "Xmas tree lights".
  
  For questions unrelated to the home, answer briefly and truthfully. If you don't know, say so
  
  The current time is {{ now().strftime('%-I:%M %p') }}.
  ```
- **Thinking options (Gemma 4):** if your agent exposes them, keep *Enable thinking* **off** and *Include prior thinking* **off**; Gemma 4's model card says thoughts from earlier turns must not be fed back. If the agent exposes sampling options, Top-K 64 and Top-P 0.95 match Google's recommendation. [VERIFY wording: the **Local OpenAI LLM** integration has both toggles; the core integration may differ.]

> If the core integration lacks an option you need (streaming TTS, trimming history, chat-template arguments), the HACS integration **Local OpenAI LLM** (`skye-harris/hass_local_openai_llm`) is a well-featured alternative that talks to the same llama-server.

### 8.3 Build the voice pipeline

**Settings > Voice assistants > Add assistant**:
- Conversation agent: the llama.cpp agent from §8.2
- Speech-to-text: your Mac Whisper
- Text-to-speech: your Mac Pocket TTS (pick the voice, e.g. `alba`)
- Wake word: leave empty (the Voice PE detects it on-device)

Test in the HA UI with the text/mic box before touching satellites.

### 8.4 Internet search (free, no accounts)

Install **Tools for Assist** (`skye-harris/llm_intents`) via HACS (custom repository, type Integration), restart HA, then add it under Devices & services. Its web search supports **Brave** or **SearXNG**, and it also ships tools that need no key at all: **Wikipedia**, a **weather forecast** tool that reads your existing HA weather entity, a calculator/unit converter, and date info. Repeated lookups are cached for about two hours.

**The free stack (no sign-ups, no card):**

| Question type | Tool | Cost / account |
|---|---|---|
| "Who plays Neo in The Matrix?", "what is X", "how tall is Y" | **Wikipedia** tool | Free, no key |
| Broader web questions, recent facts | **SearXNG** (self-hosted on the Mac, §8.5) | Free, no account |
| Weather | HA weather integration (e.g. Met.no or NWS) + the Weather Forecast tool | Free |
| "What time is the sportsteam game?" | **Team Tracker** (HACS) gives HA a sports-schedule entity per team; expose it | Free |
| Maths, unit conversions, dates | Calculator / Basic Utilities | Free |

**SearXNG vs Brave, honestly:**

| | SearXNG (free) | Brave API |
|---|---|---|
| Cost | Free, no account | ~$5 monthly credit (about 1,000 searches), but a **credit card is required** even for the credit |
| Reliability | Good, not perfect: it queries other search engines on your behalf, so an upstream engine occasionally blocks or changes and results thin out | High, structured, fast |
| Maintenance | One more service (§8.5) | None |
| Privacy | Queries still leave your network to upstream engines, but not tied to an account | Queries go to Brave under your API key |

Start free. If SearXNG ever proves flaky for your questions, switching the provider to Brave is a settings change in the same integration; nothing else needs rebuilding.

**Enable the tools on your agent:** on your llama.cpp conversation agent, under **Control Home Assistant**, enable the Search, Weather Forecast and Basic Utilities tool groups. [VERIFY: Tools for Assist documents this for the Ollama/OpenAI agents. If the tool groups don't appear on the core llama.cpp agent, use the **Local OpenAI LLM** HACS integration from the §8.2 note, which pointed at the same llama-server.]

**Make common questions not need search at all:**
- **Weather:** expose a weather entity and it's answered from HA directly.
- **Sports:** expose the Team Tracker entities. Web search then handles only the long tail.
- **Timers:** HA Assist supports voice timers natively. **Alarms** are less standardized in HA, so expect to build a small helper/automation pair or a custom sentence for "wake me at 7". [VERIFY current HA alarm support before you promise this to your household.]

### 8.5 Run SearXNG on the Mac (as `home_assistant_account`)

SearXNG ships as a container, so the Mac needs a lightweight container runtime. **Colima** is the headless-friendly choice (Docker Desktop wants a GUI session). It runs in a small VM on CPU/RAM only; the GPU stays free for the LLM.

**Run as `insert_username`:**

```bash
brew install colima docker docker-compose
```

**Run as `home_assistant_account`:**

```bash
mkdir -p ~/ai/searxng/config && cd ~/ai/searxng

# 1) SearXNG settings: JSON output must be enabled for Home Assistant
SECRET=$(openssl rand -hex 32)
cat > config/settings.yml <<EOF
use_default_settings: true
server:
  secret_key: "$SECRET"
  limiter: false
search:
  formats:
    - html
    - json
EOF

# 2) Compose file
cat > docker-compose.yml <<'EOF'
services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    ports:
      - "8888:8080"
    volumes:
      - ./config:/etc/searxng
    restart: unless-stopped
EOF

# 3) Colima as a LaunchAgent (starts the container runtime at login)
cat > ~/Library/LaunchAgents/com.local.colima.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.local.colima</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/colima</string>
    <string>start</string><string>--foreground</string>
    <string>--cpu</string><string>2</string>
    <string>--memory</string><string>2</string>
    <string>--vm-type</string><string>vz</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict><key>PATH</key><string>/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin</string></dict>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>ThrottleInterval</key><integer>30</integer>
  <key>StandardOutPath</key><string>$HOME/ai/logs/colima.log</string>
  <key>StandardErrorPath</key><string>$HOME/ai/logs/colima.err</string>
</dict></plist>
EOF
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.local.colima.plist
sleep 60 && tail -5 ~/ai/logs/colima.err

# 4) Start SearXNG (restart: unless-stopped brings it back whenever Colima starts)
export DOCKER_HOST=unix://$HOME/.colima/default/docker.sock
docker-compose up -d
```

**Test it:**

```bash
curl -s "http://localhost:8888/search?q=who+plays+neo+in+the+matrix&format=json" | jq -r '.results[0:3][] | .title'
curl -s -o /dev/null -w "%{http_code}\n" "http://MAC_IP:8888/"      # from another machine; expect 200
```

If you get titles back, you're done. In HA, make sure the Tools for Assist integration has been added on the settings, devices & services page. Then: **Tools for Assist > Add > Web Search > SearXNG**, server `http://MAC_IP:8888`, and keep **Number of Results** small (2–3) so replies stay fast and the prompt stays short.

> **Security note:** this instance has no rate limiter and no login, which is fine on a trusted LAN. Restrict port 8888 at your router to HA (and your admin machine). It holds nothing sensitive, but there's no reason to expose it further.
> **[VERIFY]** Colima flags and the container's `/etc/searxng` settings path change occasionally; if the container won't start, check `docker-compose logs searxng`. Any reachable SearXNG with JSON enabled works, so running it on another box (or from source with `uv`) is equally valid if you'd rather not run a container VM on the Mac.

---

## 9. Phase 8: Custom wake word

**How it works:** Voice PE runs **microWakeWord on the device**. It does not use the server-side openWakeWord engine. To use a custom phrase you (1) train a microWakeWord model, then (2) tell your Voice PE's ESPHome config to load it, which requires "taking control" of the device in ESPHome. [VERIFY against current Voice PE / ESPHome docs; this area changes.]

Your 3-syllable, rarely-spoken phrase is a good candidate: distinctive phonetics mean fewer false triggers.

### 9.1 Easiest path to try first
Before training anything, confirm the whole pipeline works with a **built-in** wake word (e.g. "Okay Nabu") on one Voice PE.

### 9.2 Train your wake word on the Mac
A community trainer exists for Apple Silicon with Metal acceleration: **`TaterTotterson/microWakeWord-Trainer-AppleSilicon`**. Follow its README. In outline:

```bash
cd ~/ai/voice
git clone https://github.com/TaterTotterson/microWakeWord-Trainer-AppleSilicon.git
cd microWakeWord-Trainer-AppleSilicon
# follow README: enter your phrase, generate synthetic samples, (optionally) add real recordings of household voices, train
```

Practical advice:
- **Test the phrase with TTS first** to make sure it sounds the way you pronounce it. Spell it phonetically if needed.
- Record real samples from each household member if the trainer supports it; this improves recall noticeably.
- Expect to **tune sensitivity** (probability cutoff / sliding window) after living with it for a few days. This is adjustable in ESPHome YAML without retraining.
- Training is a one-off, so it isn't a permanent load on the voice stack. The trained model is a ~200 KB-class file.

### 9.3 Deploy to a Voice PE
In HA: **Settings > Devices & services > ESPHome (Device Builder)**, find the Voice PE, **Take control**, and point `micro_wake_word` at your model. The pattern (adapt to the trainer's output) is:

```yaml
substitutions:
  name: home-assistant-voice-XXXXXX
  friendly_name: Room Voice

packages:
  Nabu Casa.Home Assistant Voice PE: github://esphome/home-assistant-voice-pe/home-assistant-voice.yaml

esphome:
  name: ${name}
  name_add_mac_suffix: false
  friendly_name: ${friendly_name}

micro_wake_word:
  models:
    - model: /config/esphome/my_wakeword.json     # or an https:// URL you host
```

After flashing, pick your wake word in the device's **Wake word** dropdown in HA. Repeat per device (one shared YAML via ESPHome `packages`/substitutions saves effort across four rooms).

> **Trade-off to know:** once you "take control", firmware updates are **yours** to apply via ESPHome rather than automatic via HA. Put "update Voice PE firmware" on a quarterly checklist.

---

## 10. Phase 9: Sophia NLU (optional; evaluate, don't commit)

**Recommendation:** get the baseline in §8 working first. Then trial Sophia for 14 days against your real devices. Because HA's built-in intent matcher plus an LLM may already cover your mostly-simple needs, Sophia is only worth the money if it fixes something you can observe.

**Where it would help:** multi-command sentences ("turn on the kitchen and living room lights and set the temp to 23"), loose phrasing, fuzzy device names, a large entity count.
**Where it won't:** web search or general knowledge (it's an intent parser). Its LLM fallback is a plain OpenAI-compatible handoff. I could not confirm that fallback has access to your search tools, so test it (below).

**Install (on the HA box, HAOS):** per Sophia's install docs: trial signup gives you a personal add-on repository URL; add it under **Settings > Apps > App Store**, install the Sophia add-on, activate your license in its UI (port 10520), install its HACS integration (`cicero-ai/sophia-ha-integration`), restart HA, then create a new Assist pipeline with **Sophia NLU** as the conversation agent. In Sophia's settings, set the LLM fallback provider to **Other / OpenAI Compatible** with `http://MAC_IP:8080/v1` and your API key.

**Evaluation checklist (run against your own house):**
1. 20 typical commands, single and multi-intent. Compare against pipeline A (built-in + LLM) for success and latency.
2. "Who plays Neo in The Matrix?" and "What time is the sportsball game today?" through the Sophia pipeline. **If those fail** because its fallback can't use web search, Sophia can't be your only pipeline.
3. Decide: keep both pipelines and assign different Voice PEs, use Sophia as the primary only if it passes #2, or skip it.

Notes: it's closed source (the vendor states it is network-isolated as an add-on and activates once), and a vendor's own benchmark naturally favors its product, so rely on your test, not the leaderboard.

---

## 11. Phase 10: Reliability for a device that runs your house

### 11.1 What "always on" really needs
- **UPS** on the Mac, the router/switch, and the HA box. Even a small one converts power blips into non-events.
- Ethernet, not Wi-Fi.
- `autorestart` + auto-login + LaunchAgents (done above) so it self-heals after an outage.

### 11.2 A fallback so you can still turn on lights if the Mac is down
Voice depends on three Mac services. Keep a **second pipeline** in HA that doesn't: HA's built-in Assist with a lightweight local STT on the HA box (e.g. its own Whisper add-on with a small model, or Speech-to-Phrase) and its own TTS. Switching a satellite to it takes seconds. Note that the Voice PE has a pipeline selector. Physical switches and the HA app remain your real "always works" layer.

### 11.3 Health check from HA or another box
```bash
for p in 8080 10200 10300; do nc -z -w2 MAC_IP $p && echo "port $p OK" || echo "port $p DOWN"; done
curl -s -H "Authorization: Bearer $(cat ~/ai/secrets/llama-voice.key)" http://MAC_IP:8080/v1/models | jq -r '.data[0].id'
```
Wire the same checks into an HA automation (e.g. a binary sensor per port) that notifies your phone when the Mac stops answering.

### 11.4 Update procedure (monthly, in a maintenance window)
```bash
# --- as admin ---
brew update && brew upgrade llama.cpp colima docker docker-compose   # the whole point of using brew

# --- as home ---
launchctl kickstart -k gui/$(id -u)/com.local.llama-voice
# STT: re-run the installer from §6.1 (or upgrade the package it installed), then restart its agent:
launchctl list | grep -i whisper        # find the label, then: launchctl kickstart -k gui/$(id -u)/<label>
# Pocket TTS is pinned to the checkout from §7.2; update it deliberately, then:
# launchctl kickstart -k gui/$(id -u)/com.local.pocket-tts
cd ~/ai/searxng && export DOCKER_HOST=unix://$HOME/.colima/default/docker.sock \
  && docker-compose pull && docker-compose up -d

# macOS updates: manually, only when you can babysit the reboot
```
After any update, run the §5.2 curl test, then say a command to a satellite.

### 11.5 Failure drills (do each once)
1. Kill `llama-server`; confirm launchd restarts it (`pkill llama-server; sleep 20; nc -z localhost 8080`).
2. Reboot over SSH; confirm all three ports return.
3. **Pull the power** (briefly), restore it; confirm the Mac comes back by itself and voice works with nobody touching it. This verifies auto-restart, auto-login and LaunchAgents together.

### 11.6 Backups
Back up `~/ai/bin`, `~/ai/secrets`, your plists, the wake-word model and ESPHome YAML. Models are re-downloadable; configs aren't.

---

## 12. Quick troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| HA can't connect to a port | Application firewall prompt never answered (§2.7); service not running (`launchctl print gui/$(id -u)/<label>`); wrong IP after DHCP change |
| Services don't start after reboot | Auto-login off or FileVault on (§2.4); LaunchAgent not in `~/Library/LaunchAgents` or bad plist (`plutil -lint file.plist`) |
| Voice responds slowly | Model too large or "thinking" model in use; context too big; Mac is also running a heavy research job. Check `~/ai/logs/llama-voice.err` |
| LLM ignores/garbles tool calls | Missing `--jinja`; model weak at tool calling; too many exposed entities. Trim entities, try another model |
| LLM errors on long prompts | Per-slot context too small: raise `-c` (remember it's divided by `-np`) |
| STT mishears device names | Add them to `--initial-prompt` (§6.2); stay on the turbo model rather than tiny/small |
| Wake word triggers randomly or never | Adjust sensitivity in the ESPHome YAML; retrain with real household samples |
| `iogpu.wired_limit_mb` reverted | LaunchDaemon not loaded: `sudo launchctl print system/com.local.gpu-wired-limit` |
| Can't SSH after hardening | Use Screen Sharing; remove `/etc/ssh/sshd_config.d/100-hardening.conf` |
| `launchctl bootstrap` fails over SSH (domain / error 125) | `home_assistant_account` has no GUI session yet: confirm auto-login worked, or run the command from Screen Sharing/Terminal as `home_assistant_account`. Also confirm you're SSH'd in as `home_assistant_account`, not `insert_username` |
| `brew` says permission denied as `home_assistant_account` | Expected: Homebrew belongs to `insert_username`. Run `brew install/upgrade` as `insert_username`; `home_assistant_account` only runs installed tools |
| Web search returns nothing | `curl ...&format=json` test (§8.5); JSON not enabled in `settings.yml`; an upstream engine is blocking, so retry later. The Wikipedia tool is independent and should still answer "who/what" questions |
| `llama-server` says unknown model architecture `gemma4` | llama.cpp is too old: `brew upgrade llama.cpp` as `insert_username`, then restart the agent |
| Pocket TTS won't start (401/403 or gated-repo error) | Accept the model terms on Hugging Face with the same account that issued `~/ai/secrets/hf-token` (§7.1) |
| SearXNG down after reboot | Colima agent not loaded (`launchctl print gui/$(id -u)/com.local.colima`); check `~/ai/logs/colima.err` |

---

## 14. Final checklist

- [ ] macOS updated; hostname set; Remote Login + Screen Sharing on
- [ ] `home_assistant_account` standard account created; auto-login set to `home_assistant_account`; `insert_username` does not auto-login
- [ ] `pmset` set; auto-login on; FileVault decision made and documented
- [ ] SSH key-only; DHCP reservation; firewall approach chosen
- [ ] Reboot test and power-pull test pass
- [ ] `brew install llama.cpp`; directories created
- [ ] GPU wired-limit LaunchDaemon installed and verified
- [ ] llama-server LaunchAgent running on :8080 with API key
- [ ] Whisper (MLX) service on :10300; Pocket TTS service on :10200 (Hugging Face terms accepted)
- [ ] HA: Wyoming STT + TTS, llama.cpp agent, Assist pipeline, entities exposed
- [ ] SearXNG running on :8888 (JSON enabled); Tools for Assist set up with SearXNG + Wikipedia; weather and sports entities exposed
- [ ] Built-in wake word works on one Voice PE; then custom wake word trained and deployed
- [ ] Fallback pipeline and Mac health alerts configured
- [ ] (Optional) Sophia 14-day evaluation against §10 checklist
- [ ] Goal #2 prerequisites reviewed (§12)

---

### Sources consulted (Oct 2026)

- Sophia NLU docs, FAQ and release notes: nlu.to/ha, nlu.to/ha/faq, nlu.to/ha/docs/install, release 2.1.3 notes
- Home Assistant llama.cpp integration: home-assistant.io/integrations/llama_cpp
- `unsloth/gemma-4-26B-A4B-it-qat-GGUF` model card (file size, sampling, thinking, MTP notes)
- `basnijholt/wyoming-mlx-whisper` (MLX Whisper Wyoming server, model options); `OHF-Voice/wyoming-faster-whisper` (CPU alternative with automatic name biasing)
- `kyutai/pocket-tts` model card; `ikidd/pocket-tts-wyoming` (Wyoming wrapper and Dockerfile)
- `skye-harris/hass_local_openai_llm` and `skye-harris/llm_intents` (Tools for Assist)
- HA Community threads on Voice PE custom wake words; `TaterTotterson/microWakeWord-Trainer-AppleSilicon`
- Brave Search API pricing coverage (monthly-credit model introduced Feb 2026)
- `skye-harris/llm_intents` README (SearXNG, Wikipedia, weather and utility tools; provider requirements)

---
