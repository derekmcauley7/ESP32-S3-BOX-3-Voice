# AOOSTAR MACO R7 PRO 6850H → Local LLM for Home Assistant (Pi 5)

Setup: Proxmox VE on the mini PC, an Ubuntu LXC container with the iGPU passed through,
Ollama as the inference backend, Qwen3 4B as the model, connecting into your existing
Home Assistant on the Pi 5.

---

## 1. BIOS setup (mini PC)

Before installing anything, boot into BIOS (Del/F2 at startup) and:

- Enable **virtualization** (look for "SVM Mode" or "AMD-V") — required for Proxmox.
- Set **UMA Frame Buffer Size** (or "iGPU Memory") to a fixed value like **8GB–16GB**, if
  the option exists. This reserves memory the iGPU can use for models. If your BIOS only
  offers "Auto," skip this.
- Enable **Above 4G Decoding** if present.
- IOMMU/VT-d style passthrough settings are **not needed** — the LXC approach below shares
  the host kernel and doesn't require PCI passthrough.

## 2. Install Proxmox VE

1. On another computer, download the [Proxmox VE ISO](https://www.proxmox.com/en/downloads)
   and flash it to a USB drive with [Rufus](https://rufus.ie) (Windows) or
   [balenaEtcher](https://etcher.balena.io) (Mac/Linux).
2. Boot the mini PC from the USB drive and run the Proxmox installer. Give the host a static
   IP during setup (or reserve one on your router afterward via its MAC address) so it's
   always reachable at the same address.
3. Once installed, reboot, remove the USB, and access the web UI from another device on your
   network:
   ```
   https://<mini-pc-ip>:8006
   ```
   Log in as `root` with the password you set during install.
4. (Optional but recommended) Remove the enterprise repo nag / switch to the no-subscription
   repo so `apt update` works without a subscription key — Proxmox's docs cover this under
   "Package Repositories."

## 3. Create the LXC container

1. In the web UI, download an Ubuntu template: click your node → **local (storage)** →
   **CT Templates** → **Templates**, search for `ubuntu-24.04-standard`, click **Download**.
2. Right-click your node → **Create CT**:
   - **General**: hostname `llm-box`, set a root password.
   - **Template**: the Ubuntu 24.04 template you just downloaded.
   - **Disks**: 32GB+ (models are several GB each; more if you plan to try bigger ones later).
   - **CPU**: allocate 6 of the 8 cores (leaves headroom for Proxmox itself).
   - **Memory**: 16–20GB out of your 24GB (leave some for the host and iGPU UMA buffer).
   - **Network**: bridge to `vmbr0`, static IP or DHCP reservation.
   - On the **General** tab, **uncheck "Unprivileged container."** Privileged is the simpler
     path for GPU passthrough on a single-purpose home appliance like this. (Unprivileged is
     possible with extra GID-mapping steps if you'd rather have that isolation — the Proxmox
     forum has walkthroughs — but it's more fiddly for no real benefit here since this box
     isn't exposed to the internet.)
3. Don't start it yet — first pass the GPU through.

### Pass the iGPU into the container

Proxmox 8.1+ supports this from the GUI: select the container → **Resources** → **Add** →
**Device Passthrough**. Add two entries:
- `/dev/dri/renderD128`
- `/dev/dri/card0`

If your Proxmox version doesn't have that option, edit the config directly on the host shell
instead:

```
nano /etc/pve/lxc/<CTID>.conf
```

Add:

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

Then start the container (**Start** in the GUI, or `pct start <CTID>`).

### Verify the passthrough

Enter the container:

```
pct enter <CTID>
```

Check the device is visible:

```
ls -l /dev/dri
```

You should see `card0` and `renderD128`. If it's missing, double check the passthrough
entries above and that you selected a **privileged** container.

## 4. Install GPU drivers (Vulkan) — inside the container

```
apt update && apt install -y mesa-vulkan-drivers vulkan-tools mesa-utils
```

Verify:

```
vulkaninfo --summary
```

You should see `AMD Radeon Graphics (RADV REMBRANDT)` (or similar) listed. If missing,
`apt install -y linux-firmware` and restart the container (`pct reboot <CTID>` from the host).

## 5. Install Ollama — inside the container

```
curl -fsSL https://ollama.com/install.sh | sh
```

This installs Ollama as a systemd service (`ollama.service`).

### Enable Vulkan + network access

```
systemctl edit ollama.service
```

Add:

```ini
[Service]
Environment="OLLAMA_VULKAN=1"
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Save, then:

```
systemctl daemon-reload
systemctl restart ollama
systemctl status ollama
```

### Open the firewall (if enabled inside the container)

```
ufw allow 11434/tcp
```

## 6. Pull the model — inside the container

```
ollama pull qwen3:4b
```

**Important gotcha:** Qwen3 defaults to "thinking mode," which wraps tool calls in `<think>`
tags that Home Assistant doesn't parse — commands get planned but never executed. Create a
variant with thinking disabled:

```
cat <<'EOF' > /tmp/Modelfile
FROM qwen3:4b
PARAMETER num_ctx 4096
SYSTEM /no_think
EOF

ollama create qwen3-ha:4b -f /tmp/Modelfile
```

Use `qwen3-ha:4b` (not the base `qwen3:4b`) in Home Assistant.

### Test it

```
ollama run qwen3-ha:4b "Say hello in one sentence."
```

Should respond fast with no thinking output. If it's slow, confirm Vulkan is actually being
used:

```
journalctl -u ollama -f
```

Run a prompt again and watch for a GPU/Vulkan reference in the logs — if it only mentions
CPU, the `OLLAMA_VULKAN=1` env var didn't take (recheck step 5), or the passthrough in step 3
isn't working (recheck `ls -l /dev/dri` inside the container).

## 7. Connect Home Assistant (Pi 5) to the container

1. On the Pi 5, open Home Assistant → **Settings → Devices & Services → Add Integration**.
2. Search for **Ollama**.
3. Enter the container's URL: `http://<container-ip>:11434`.
4. HA will detect available models — select `qwen3-ha:4b`.
5. In the integration's options, set:
   - **Control Home Assistant**: enabled.
   - **Max history messages**: keep low (e.g. 5) to reduce prompt size and speed up responses.

## 8. Wire it into the Assist pipeline

1. **Settings → Voice assistants → Add assistant** (or edit your existing one).
2. Set **Conversation agent** to the Ollama entry you just created.
3. Enable **"Prefer handling commands locally"** — routes simple commands through HA's
   built-in instant intent matcher and only falls back to the LLM for what it can't
   pattern-match. This is what keeps things feeling fast.
4. Under **Settings → Voice assistants → Expose**, expose the entities you want voice
   control over.

## 9. Test it

- Text test: **Settings → Voice assistants → your assistant → chat bubble icon** — try
  "turn on the office lights" (instant/local) and something conversational like "what's a
  good bedtime routine" (routes to the LLM, a couple seconds).
- Voice test: use any configured satellite (HA Voice Preview Edition, ESPHome satellite, or
  the mobile app's Assist mic).

## 10. Optional tuning

- **Snapshot the container** once it's working (Proxmox GUI → container → **Backup** or
  **Snapshots**) — cheap insurance before trying bigger models or config changes.
- **Keep the model warm**: `OLLAMA_KEEP_ALIVE=-1` alongside the other env vars in step 5
  keeps the model resident in memory instead of unloading after 5 minutes idle.
- **Reduce context window**: `num_ctx 4096` in the Modelfile is already fairly tight —
  smaller context = faster responses.
- **Monitor GPU load**: `apt install -y radeontop` inside the container, then run
  `radeontop` to confirm Vulkan is doing the work during a query.
- **Resource limits**: if Proxmox's host summary shows memory pressure, lower the
  container's RAM allocation slightly rather than letting it compete with Proxmox itself.
- **If responses still feel slow**: drop to `qwen3:1.7b` as a fallback, or reconsider
  whether a given command really needs the LLM vs. a local intent/sentence trigger defined
  directly in HA (instant, no LLM involved at all).
