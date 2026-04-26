# LScript v3 — Full Development Documentation

**Original script by:** Aris Melachroinos  
**Modernization & bug fixes by:** Dalcots  

---

## Project Overview

LScript (The Lazy Script) is a Bash-based WiFi penetration testing toolkit originally written by Aris Melachroinos for Kali Linux. This document covers all modernization and bug-fix work performed by Dalcots to make it functional on modern Kali Linux.

---

## Phase 1 — Modern Kali Compatibility (v3.0.0)

The original script was written around 2018 and had numerous broken dependencies and deprecated syntax.

**Fixes applied across all scripts (`l`, `lh1`–`lh4`, `lh21`, `lh31`, `lh41`–`lh43`, `install.sh`, `uninstall.sh`, `ls/l131–133.sh`):**

| Issue | Fix |
|-------|-----|
| `python` → Python 2 | Changed to `python3` |
| `pip install` | Changed to `pip3 install` |
| `apt-get` | Changed to `apt` |
| `leafpad` (removed from Kali) | Changed to `mousepad` |
| `gnome-terminal -e` / `--geometry` | Updated to `gnome-terminal --` (new syntax) |
| `ncurses-dev` | Renamed to `libncurses-dev` |
| `openvas-start/stop/setup` | Changed to `gvm-start/stop/setup` |
| `msfupdate` (removed) | Changed to `apt install metasploit-framework` |
| `ngrok` zip download (dead) | Updated to official apt package install |
| `ngrok authtoken` | Changed to `ngrok config add-authtoken` (v3 API) |
| AngryIP 3.5.2 download link (dead) | Updated to 3.9.1 |
| `easy_install` (deprecated) | Changed to `pip3` |
| `python setup.py install` (deprecated) | Changed to `pip3 install .` |
| `fern-wifi-cracker` SVN checkout (dead) | Changed to `git clone` |

---

## Phase 2 — Runtime Bug Fixes

### tempscan.txt Race Condition
**Problem:** The network scan function called `arp-scan` via `open_term` (which opens a new terminal window asynchronously), then immediately tried to `cat tempscan.txt` before `arp-scan` had written anything. Result: `cat: /root/lscript/tempscan.txt: No such file or directory` and "No hosts found."

**Fix:** Replaced the async `open_term` call with an inline blocking `arp-scan` call that writes to the file before the script continues reading it.

### Hardcoded `/root/lscript/` Paths
**Problem:** All runtime file references (`tempscan.txt`, `wlanmon.txt`, `sqltemp`, `hashlog.txt`, `tempairodump`, etc.) were hardcoded to `/root/lscript/`. Running from any other directory caused file-not-found errors.

**Fix:** Replaced all hardcoded paths with `$LPATH/filename` throughout all scripts.

---

## Phase 3 — Interface Selection Fix

### Problem
The `interface` command defaulted to `wlan0` / `wlan0mon` regardless of what interfaces actually existed on the machine. The original code read from a saved `wlan.txt` file and returned early without ever prompting the user if that file existed.

### Fix
Rewrote `set_interface_number` to:
1. Detect available wireless interfaces using `iw dev` and `/sys/class/net/*/wireless`
2. Display the detected interfaces
3. **Always prompt** the user to type both the managed and monitor interface names manually
4. Accept the typed values and save them to `wlan.txt` and `wlanmon.txt`
5. Never silently auto-select

Also rewrote `interface_menu` to skip the "do you want to change?" confirmation and go directly to the picker.

---

## Phase 4 — Launch Command & PATH Issues

### lh Scripts Not Found at Runtime
**Problem:** When a menu option launched a sub-script (e.g. option 10 for WPA handshake), the script ran `gnome-terminal -- lh1`. This only works if `/root/lscript` is in the system PATH — which it isn't in a fresh terminal or when running from a different directory.

**Fix:** Changed all `gnome-terminal -- lhX` calls to use the full path:
```bash
gnome-terminal -- bash -c "bash \"$LPATH/lh1\""
```

### exec bash "$0" Breaking on Non-Standard Paths
**Problem:** When a sub-menu finished, it ran `exec bash "$0"` to restart the main script. `$0` is the path you originally used to launch the script — if you ran it from the Desktop, it re-ran the Desktop copy, which didn't have the installed files around it.

**Fix:** Changed all `exec bash "$0"` to `exec bash "$LPATH/l"` so it always restarts from the installed location.

### Dynamic LPATH Resolution
**Problem:** `LPATH="/root/lscript"` was hardcoded at the top of every script. Running from Desktop or any other path caused all file references to break.

**Fix:** Added dynamic resolution at the top of every script:
```bash
LPATH="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
[[ -d "/root/lscript" && -f "/root/lscript/lh1" ]] && LPATH="/root/lscript"
export LPATH
```

This means the script always knows where it is, whether run from the Desktop, Downloads, or the installed location.

### Rename Launch Command: `l` → `lzsc`
**Problem:** The launch command was `l` — a single letter that conflicts with other commands and aliases on many systems.

**Fix:** Renamed to `lzsc`. The installer now creates `/bin/lscript/lzsc` alongside the original `l` binary.

### PATH Not Applied for Root User
**Problem:** The installer wrote `export PATH=/bin/lscript:$PATH` to `~/.bashrc`, but when running as root, the home directory is `/root`, not `/home/kali`. So the PATH was written to `/home/kali/.bashrc` and never loaded in root sessions.

**Fix:** The installer now writes the PATH entry to:
- `/root/.bashrc`
- `~/.bashrc` (current user fallback)
- `/etc/profile.d/lscript.sh` (applies to all users and all shells)

### install.sh Broken gnome-terminal Call
**Problem:** The re-launch line at the end of installation was:
```bash
gnome-terminal -- "bash /root/lscript/install.sh"
```
The entire command was wrapped in quotes, so the shell tried to execute `bash /root/lscript/install.sh` as a single binary name — which doesn't exist.

**Fix:**
```bash
gnome-terminal -- bash /root/lscript/install.sh
```

### ls/ Subdirectory Not Copied During Install
**Problem:** The `ls/` subdirectory containing `l131.sh`, `l132.sh`, `l133.sh` was never copied to `/bin/lscript/` during installation. Tools that depended on these scripts (MITMf, dns2proxy, arpspoof automation) silently failed.

**Fix:** Added explicit copy of the `ls/` subdirectory to the install script.

---

## Installation Instructions

```bash
# 1. Open terminal as root
sudo -i

# 2. Go into extracted folder
cd ~/Desktop/lscript_clean

# 3. Run installer
bash install.sh
# Press 'i' when asked first-time or update

# 4. Create symlink so lzsc works from anywhere
ln -sf /bin/lscript/lzsc /usr/local/bin/lzsc

# 5. Launch
lzsc
```

**Every subsequent use:**
```bash
sudo -i
lzsc
```

**If symlink not working (fallback that always works):**
```bash
sudo bash /bin/lscript/lzsc
```

---

## File Structure After Install

```
/root/lscript/              ← source files
    l, lh1–lh4, lh21, lh31, lh41–lh43
    install.sh, uninstall.sh
    ls/l131.sh, l132.sh, l133.sh

/bin/lscript/               ← installed binaries (in PATH)
    l, lzsc, lh1–lh4, lh21, lh31, lh41–lh43
    ls/l131.sh, l132.sh, l133.sh

/usr/local/bin/lzsc         ← symlink for easy access
/etc/profile.d/lscript.sh  ← PATH entry for all shells
```

---

## Credits

**Original author:** Aris Melachroinos — [github.com/arismelachroinos/lscript](https://github.com/arismelachroinos/lscript)  
**Modernization, bug fixes & documentation:** [github.com/Dalcots](https://github.com/Dalcots)
