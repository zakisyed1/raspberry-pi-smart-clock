# Raspberry Pi Kiosk Dashboard - Technical Specification

**Target Hardware:** Raspberry Pi 3B+ (1GB RAM)

**Display:** Waveshare 8.8” DSI LCD (1920×480)

**OS:** Raspberry Pi OS Lite (64-bit, Bookworm)

**UI Framework:** React + Framer Motion

**Version:** 1.0

**Last Updated:** January 2026

---

## Table of Contents

1. [Hardware & Display Configuration](about:blank#1-hardware--display-configuration)
2. [Operating System Setup](about:blank#2-operating-system-setup)
3. [Window Manager Comparison](about:blank#3-window-manager-comparison)
4. [Chromium Kiosk Configuration](about:blank#4-chromium-kiosk-configuration)
5. [System Memory Optimization](about:blank#5-system-memory-optimization)
6. [React Dashboard Implementation](about:blank#6-react-dashboard-implementation)
7. [Framer Motion Performance Optimization](about:blank#7-framer-motion-performance-optimization)
8. [UI/UX Performance Optimization](about:blank#8-uiux-performance-optimization)
9. [Project Structure](about:blank#9-project-structure)
10. [Autostart Configuration](about:blank#10-autostart-configuration)
11. [CM4 Migration Planning (Phase 2)](about:blank#11-cm4-migration-planning-phase-2)
12. [Data Integration Options](about:blank#12-data-integration-options)
13. [Monitoring & Maintenance](about:blank#13-monitoring--maintenance)
14. [Production Deployment Checklist](about:blank#14-production-deployment-checklist)

---

## 1. Hardware & Display Configuration

### 1.1 Raspberry Pi 3B+ Specifications

**Critical Constraints:**
- **RAM:** 1GB LPDDR2 SDRAM (shared with GPU)
- **CPU:** Broadcom BCM2837B0, Quad-core Cortex-A53 @ 1.4GHz
- **GPU:** VideoCore IV @ 400MHz (250MHz base, 400MHz turbo)
- **Storage:** microSD (recommend Class 10 UHS-I minimum)

**Performance Implications:**
- Limited RAM requires aggressive optimization
- GPU shares system memory - careful balance needed
- CPU adequate for React rendering at 1920×480
- Storage I/O can become bottleneck with SD cards

### 1.2 Waveshare 8.8” DSI LCD

**Display Specifications:**
- **Resolution:** 1920×480 pixels
- **Aspect Ratio:** 4:1 (ultra-wide)
- **Interface:** DSI (Display Serial Interface)
- **Touch:** Capacitive touch (optional, not needed for kiosk)
- **Refresh Rate:** 60Hz
- **Connector:** 15-pin FFC to Pi DSI port

**Layout Advantages:**
- Perfect for dashboard panels (4 equal 480×480 sections)
- High DPI text rendering
- No HDMI overhead (DSI is more efficient)

### 1.3 Boot Configuration (`/boot/firmware/config.txt`)

**Production-Ready Configuration:**

```
# GPU Memory Allocation
# For 1920×480 display: 64MB is optimal balance
# 16MB = too constrained, stuttering
# 128MB = wastes system RAM
gpu_mem=64

# Display Configuration
# Native KMS driver for Waveshare 8.8" DSI panel
dtoverlay=vc4-kms-dsi-waveshare-panel,8_8_inch

# Disable unused features to save RAM
dtparam=audio=off
camera_auto_detect=0
display_auto_detect=0

# Performance optimizations
arm_freq=1400
gpu_freq=400
over_voltage=2
force_turbo=0

# Disable rainbow splash (faster boot)
disable_splash=1

# Console settings (disable console on DSI)
# Prevents kernel messages on display
dtparam=console=serial0,115200

# Reduce HDMI memory overhead (no HDMI used)
hdmi_blanking=2

# Enable 64-bit mode
arm_64bit=1
```

**GPU Memory Split Decision Matrix:**

| GPU Memory | System RAM | Use Case | Performance |
| --- | --- | --- | --- |
| 16MB | 1008MB | Text-only UI | Poor rendering, stuttering |
| 32MB | 992MB | Minimal graphics | Adequate for static content |
| **64MB** | **960MB** | **React dashboard** | **Optimal for 1920×480** |
| 128MB | 896MB | Heavy graphics | Overkill, wastes RAM |
| 256MB | 768MB | 4K displays | Not recommended for 3B+ |

**Recommendation:** 64MB for React + Framer Motion at 1920×480 resolution.

### 1.4 Display Orientation

For standard landscape orientation (1920×480), no rotation needed. If side-mounting requires rotation:

```
# Add to config.txt for 180° rotation
display_lcd_rotate=2
```

---

## 2. Operating System Setup

### 2.1 Raspberry Pi OS Lite Installation

**Why Lite Edition:**
- No desktop environment (saves ~300-400MB RAM)
- Minimal running services
- Faster boot time (~15-20 seconds)
- Full control over graphics stack

**Installation Steps:**

1. **Download Raspberry Pi OS Lite (64-bit, Bookworm):**
    - Use Raspberry Pi Imager
    - Enable SSH in advanced settings
    - Configure WiFi credentials (if needed)
    - Set hostname: `kiosk-dashboard`
2. **First Boot Configuration:**

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Set timezone
sudo timedatectl set-timezone America/New_York

# Set locale
sudo raspi-config nonint do_change_locale en_US.UTF-8

# Disable unnecessary services
sudo systemctl disable bluetooth.service
sudo systemctl disable hciuart.service
sudo systemctl disable avahi-daemon.service
sudo systemctl disable triggerhappy.service
```

### 2.2 Minimal Package Requirements

**Essential Packages:**

```bash
# Window manager (choose one - see Section 3)
# Option A: Openbox (X11)
sudo apt install -y xserver-xorg-core xserver-xorg-video-fbdev \
  xinit openbox

# Option B: labwc (Wayland)
sudo apt install -y labwc wayland

# Chromium browser
sudo apt install -y chromium-browser

# Development tools (for building React app on Pi if needed)
sudo apt install -y git curl

# Optional: Node.js (if building on Pi)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

**Package Size Comparison:**

| Package Stack | Disk Space | RAM Usage (Idle) |
| --- | --- | --- |
| Openbox + X11 | ~180MB | ~45MB |
| labwc + Wayland | ~120MB | ~35MB |
| Chromium | ~250MB | ~150MB (kiosk) |

### 2.3 Auto-Login Configuration

**Create Kiosk User:**

```bash
# Create dedicated kiosk user
sudo useradd -m -G video,audio,input kiosk
sudo passwd kiosk  # Set password

# Enable auto-login for kiosk user
sudo systemctl edit getty@tty1

# Add these lines:
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin kiosk --noclear %I $TERM
```

### 2.4 Boot Optimization

**Disable Unnecessary Services:**

```bash
# List all services
systemctl list-unit-files --type=service --state=enabled

# Disable unused services (safe to disable for kiosk)
sudo systemctl disable apt-daily.timer apt-daily-upgrade.timer
sudo systemctl disable man-db.timer
sudo systemctl disable systemd-timesyncd.service  # Use simpler NTP client
sudo systemctl disable rsync.service
sudo systemctl disable keyboard-setup.service  # Not needed for kiosk
```

**Expected Boot Time:** ~12-15 seconds to Chromium launch

---

## 3. Window Manager Comparison

### 3.1 Option A: Openbox (X11) - **RECOMMENDED for Stability**

**Architecture:**

```
[Hardware] → [DRM/KMS] → [X Server] → [Openbox] → [Chromium]
```

**Pros:**
- ✅ **Battle-tested:** Proven on low-RAM systems
- ✅ **Stable:** Mature codebase, extensive documentation
- ✅ **Chromium compatibility:** Excellent support
- ✅ **Resource monitoring:** Good tools (htop, xrestop)
- ✅ **Fallback support:** Easy debugging

**Cons:**
- ❌ **X11 overhead:** ~20MB extra RAM vs Wayland
- ❌ **Older architecture:** Not leveraging modern GPU features
- ❌ **Tearing possible:** May need compositor tweaks

**Memory Footprint:**
- X Server: ~25MB
- Openbox: ~18MB
- **Total:** ~43MB (before Chromium)

**Configuration:**

Create `/home/kiosk/.config/openbox/autostart`:

```bash
#!/bin/bash

# Disable screen blanking
xset s off
xset -dpms
xset s noblank

# Hide cursor after 1 second of inactivity
unclutter -idle 1 -root &

# Launch Chromium in kiosk mode
chromium-browser \
  --kiosk \
  --noerrdialogs \
  --disable-infobars \
  --no-first-run \
  --enable-features=OverlayScrollbar \
  --disable-session-crashed-bubble \
  --disable-component-update \
  http://localhost:3000
```

**Start Command:**

```bash
# Add to /home/kiosk/.xinitrc
exec openbox-session
```

### 3.2 Option B: labwc (Wayland) - **RECOMMENDED for Performance**

**Architecture:**

```
[Hardware] → [DRM/KMS] → [labwc compositor] → [Chromium (Ozone/Wayland)]
```

**Pros:**
- ✅ **Lower memory:** ~15MB less than X11 stack
- ✅ **Better GPU utilization:** Direct rendering
- ✅ **No screen tearing:** Built-in compositing
- ✅ **Modern:** Wayland protocol advantages
- ✅ **Simpler stack:** Fewer moving parts

**Cons:**
- ❌ **Newer:** Less battle-tested (first release 2021)
- ❌ **Limited tooling:** Fewer debugging tools
- ❌ **Chromium Ozone:** Requires –ozone-platform=wayland flag
- ❌ **Documentation:** Still growing

**Memory Footprint:**
- labwc compositor: ~28MB
- **Total:** ~28MB (before Chromium)

**Configuration:**

Create `/home/kiosk/.config/labwc/autostart`:

```bash
#!/bin/bash

# Launch Chromium with Wayland backend
chromium-browser \
  --kiosk \
  --ozone-platform=wayland \
  --enable-features=UseOzonePlatform \
  --noerrdialogs \
  --disable-infobars \
  --no-first-run \
  --disable-session-crashed-bubble \
  http://localhost:3000 &
```

**Start Command:**

```bash
# Add to /home/kiosk/.bash_profile
if [ -z "$WAYLAND_DISPLAY" ] && [ "$XDG_VTNR" -eq 1 ]; then
  exec labwc
fi
```

### 3.3 Decision Matrix

| Criteria | Openbox (X11) | labwc (Wayland) | Winner |
| --- | --- | --- | --- |
| **Stability** | 5/5 | 3/5 | Openbox |
| **RAM Usage** | 3/5 (43MB) | 5/5 (28MB) | labwc |
| **GPU Efficiency** | 3/5 | 5/5 | labwc |
| **Chromium Support** | 5/5 | 4/5 | Openbox |
| **Debugging Tools** | 5/5 | 3/5 | Openbox |
| **Screen Tearing** | 3/5 | 5/5 | labwc |
| **Documentation** | 5/5 | 3/5 | Openbox |

**Recommendation:**
- **Production/Critical Use:** Openbox (proven reliability)
- **Prototype/Personal Use:** labwc (better performance)
- **Your Use Case:** Start with **Openbox**, migrate to labwc if comfortable

---

## 4. Chromium Kiosk Configuration

### 4.1 Essential Flags for 1GB RAM

**Complete Flag List with Justification:**

```bash
chromium-browser \
  # Kiosk mode
  --kiosk \
  --start-fullscreen \

  # Disable UI elements
  --noerrdialogs \
  --disable-infobars \
  --disable-session-crashed-bubble \
  --disable-component-update \

  # Memory optimization
  --incognito \                         # No cache/history = -30MB
  --disable-dev-shm-usage \             # Use /tmp instead of /dev/shm
  --disable-background-timer-throttling \
  --disable-backgrounding-occluded-windows \
  --disable-breakpad \                  # No crash reporting
  --disable-sync \                      # No Google sync = -15MB
  --disable-translate \                 # No translation engine = -10MB
  --disable-features=TranslateUI \

  # GPU optimization (critical for Pi)
  --disable-gpu-rasterization \         # CPU raster faster on Pi
  --enable-gpu-memory-buffer-video-frames \
  --num-raster-threads=2 \              # Match CPU cores available

  # Network optimization
  --disable-background-networking \
  --disable-default-apps \
  --disable-extensions \

  # Performance
  --no-first-run \
  --no-pings \
  --prerender-from-omnibox=disabled \
  --dns-prefetch-disable \

  # Application URL
  http://localhost:3000
```

### 4.2 Flag Impact Analysis

| Flag Category | Flags | RAM Saved | Performance Impact |
| --- | --- | --- | --- |
| **Incognito Mode** | –incognito | ~30MB | None (actually faster) |
| **Disable Sync** | –disable-sync | ~15MB | None for kiosk |
| **Disable Translate** | –disable-translate | ~10MB | None for English-only |
| **GPU Optimization** | –disable-gpu-rasterization | ~20MB | Better on Pi3B+ |
| **Background Features** | –disable-background-* | ~15MB | None for kiosk |
| **Dev/Shm Fix** | –disable-dev-shm-usage | ~0MB | Prevents crashes |

**Total RAM Savings:** ~90MB compared to default Chromium

### 4.3 GPU Rasterization Decision

**Why `--disable-gpu-rasterization` on Pi 3B+:**

The VideoCore IV GPU in Pi 3B+ has limited rasterization performance. CPU-based rasterization is actually faster for:
- Text rendering
- CSS shadows and borders
- Simple geometric shapes

**When to Enable GPU Rasterization:**
- Pi 4 or CM4 (better GPU)
- Heavy WebGL content
- Video decoding (use hardware decode instead)

### 4.4 Testing Memory Usage

```bash
# Monitor Chromium memory in real-time
ps aux | grep chromium | awk '{sum+=$6} END {print sum/1024 " MB"}'

# Detailed breakdown
pmap -x $(pgrep chromium | head -1) | tail -1
```

**Expected Memory Usage:**
- Chromium (minimal flags): ~250-280MB
- Chromium (optimized flags): ~160-180MB
- React app bundle: ~15-25MB
- **Total:** ~200MB for entire browser + app

---

## 5. System Memory Optimization

### 5.1 Swap Configuration

**Zram vs File-Based Swap:**

**Option A: Zram (Recommended)**
- Compressed RAM swap
- 2-3x compression ratio
- Faster than SD card
- No SD wear

```bash
# Install zram
sudo apt install -y zram-tools

# Configure zram
sudo nano /etc/default/zramswap

# Set to 25% of RAM (256MB)
ALGO=lz4
PERCENT=25
PRIORITY=100

# Enable
sudo systemctl enable zramswap
sudo systemctl start zramswap

# Verify
sudo zramctl
```

**Option B: File-Based Swap (Fallback)**

```bash
# Create 512MB swap file
sudo fallocate -l 512M /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Optimize swappiness for kiosk
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```

**Swap Size Recommendations:**

| System RAM | Zram Size | File Swap | Total Available |
| --- | --- | --- | --- |
| 960MB (64MB GPU) | 256MB | 256MB | ~1,450MB effective |

### 5.2 Kernel Parameters

**Memory Optimization:**

```bash
# Edit /etc/sysctl.conf
sudo nano /etc/sysctl.conf

# Add these lines:
vm.swappiness=10                    # Prefer RAM over swap
vm.vfs_cache_pressure=50            # Keep inodes cached
vm.dirty_ratio=10                   # Write dirty pages at 10%
vm.dirty_background_ratio=5         # Background write at 5%
vm.min_free_kbytes=16384           # Keep 16MB free
```

### 5.3 OOM (Out-Of-Memory) Prevention

**OOM Score Adjustment:**

Prevent critical services from being killed:

```bash
# Create script: /usr/local/bin/oom-adjust.sh
#!/bin/bash

# Find Chromium PID
CHROMIUM_PID=$(pgrep -f "chromium.*kiosk" | head -1)

if [ -n "$CHROMIUM_PID" ]; then
  # Lower OOM score = less likely to be killed
  # Range: -1000 (never kill) to 1000 (kill first)
  echo -500 > /proc/$CHROMIUM_PID/oom_score_adj
  echo "Chromium OOM score adjusted to -500"
fi
```

### 5.4 Service Memory Limits

**Limit Background Services:**

```bash
# Create systemd drop-in for memory limits
sudo mkdir -p /etc/systemd/system/chromium-kiosk.service.d
sudo nano /etc/systemd/system/chromium-kiosk.service.d/memory.conf

[Service]
MemoryHigh=600M
MemoryMax=700M
MemorySwapMax=200M
```

### 5.5 Real-Time Monitoring

**Memory Monitoring Script:**

```bash
# Create /home/kiosk/monitor-memory.sh
#!/bin/bash

while true; do
  TOTAL_MEM=$(free -m | awk 'NR==2{print $2}')
  USED_MEM=$(free -m | awk 'NR==2{print $3}')
  FREE_MEM=$(free -m | awk 'NR==2{print $4}')
  CHROME_MEM=$(ps aux | grep chromium | awk '{sum+=$6} END {print int(sum/1024)}')

  echo "$(date '+%Y-%m-%d %H:%M:%S') | Total:${TOTAL_MEM}MB | Used:${USED_MEM}MB | Free:${FREE_MEM}MB | Chrome:${CHROME_MEM}MB"

  # Alert if memory usage > 85%
  PERCENT=$(echo "scale=0;$USED_MEM * 100 /$TOTAL_MEM" | bc)
  if [ $PERCENT -gt 85 ]; then
    logger "WARNING: Memory usage at${PERCENT}%"
  fi

  sleep 30
done
```

---

## 6. React Dashboard Implementation

### 6.1 Why React for This Project

**Advantages:**
- ✅ **Component reusability:** Modular panel design
- ✅ **Framer Motion:** Best-in-class animation library
- ✅ **Hooks:** Efficient state management (useState, useMemo, useCallback)
- ✅ **Developer experience:** Fast iteration, familiar tooling
- ✅ **Bundle optimization:** Tree-shaking, code splitting
- ✅ **Production builds:** Highly optimized

**Addressing RAM Concerns:**

React’s runtime is lightweight (~45KB gzipped). The real consideration is:
- **Build tools:** Use Vite (faster, smaller than CRA)
- **Bundle size:** Target < 200KB total (achievable)
- **Lazy loading:** Split code by route/panel
- **Memoization:** Prevent unnecessary re-renders

### 6.2 Build Tool: Vite vs Create React App

**Comparison:**

| Feature | Vite | Create React App |
| --- | --- | --- |
| **Build time** | ~3s | ~30s |
| **Bundle size** | Smaller (better tree-shaking) | Larger |
| **Dev server** | Instant HMR | Slower |
| **Configuration** | Simple | Complex eject required |
| **Recommendation** | ✅ **Use Vite** | ❌ Too heavy |

### 6.3 Project Initialization

```bash
# On development machine (not Pi)
npm create vite@latest pi-kiosk-dashboard -- --template react
cd pi-kiosk-dashboard

# Install dependencies
npm install

# Install Framer Motion
npm install framer-motion

# Install additional utilities
npm install date-fns  # Lightweight date library
```

### 6.4 Vite Configuration for Production

**`vite.config.js` - Optimized for Pi 3B+:**

```jsx
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    react({
      // Use automatic JSX runtime
      jsxRuntime: 'automatic',
      // Disable Fast Refresh in production
      fastRefresh: false,
    })
  ],

  build: {
    // Target modern browsers (Chromium 120+)
    target: 'es2020',

    // Optimize bundle size
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,      // Remove console.logs
        drop_debugger: true,
        pure_funcs: ['console.log'],
      },
    },

    // Chunk splitting strategy
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['react', 'react-dom'],
          'motion': ['framer-motion'],
        },
      },
    },

    // Reduce chunk size warning limit
    chunkSizeWarningLimit: 500,

    // Optimize CSS
    cssCodeSplit: true,

    // Source maps (disable for production)
    sourcemap: false,
  },

  // Preview server (for testing on Pi)
  preview: {
    port: 3000,
    host: '0.0.0.0',  // Allow external connections
  },

  // Development server
  server: {
    port: 3000,
    host: '0.0.0.0',
  },
})
```

### 6.5 Package.json Scripts

```json
{
  "name": "pi-kiosk-dashboard",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "build:analyze": "vite build --mode analyze",
    "serve": "vite preview --port 3000 --host"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "framer-motion": "^11.0.0",
    "date-fns": "^3.0.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@vitejs/plugin-react": "^4.2.0",
    "vite": "^5.0.0"
  }
}
```

### 6.6 Component Architecture

**Layout Strategy for 1920×480:**

```
┌─────────────┬─────────────┬─────────────┬─────────────┐
│             │             │             │             │
│   Panel 1   │   Panel 2   │   Panel 3   │   Panel 4   │
│  (480×480)  │  (480×480)  │  (480×480)  │  (480×480)  │
│             │             │             │             │
│ Now Playing │   Weather   │    Clock    │   Standup   │
│             │             │             │             │
└─────────────┴─────────────┴─────────────┴─────────────┘
```

**Component Structure:**

```
src/
├── App.jsx                 # Main container
├── components/
│   ├── NowPlaying.jsx      # Spotify/Music widget
│   ├── Weather.jsx         # Weather widget
│   ├── Clock.jsx           # Date/Time display
│   └── Standup.jsx         # Calendar/Standup info
├── hooks/
│   ├── useTime.js          # Custom hook for time
│   ├── useWeather.js       # Weather data hook
│   └── useSpotify.js       # Spotify API hook
├── services/
│   ├── spotify.js          # Spotify API client
│   ├── weather.js          # Weather API client
│   └── calendar.js         # Calendar API client
├── styles/
│   └── globals.css         # Global styles
└── utils/
    └── cache.js            # localStorage caching
```

### 6.7 Example Component: Clock Panel

```jsx
// src/components/Clock.jsx
import { motion } from 'framer-motion'
import { useTime } from '../hooks/useTime'

export function Clock() {
  const { time, date } = useTime()

  return (
    <motion.div
      className="panel clock-panel"
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      transition={{ duration: 0.5 }}
    >
      <div className="time-display">
        <motion.h1
          className="time"
          key={time} // Re-animate on time change
          initial={{ y: 20, opacity: 0 }}
          animate={{ y: 0, opacity: 1 }}
          transition={{ duration: 0.3 }}
        >
          {time}
        </motion.h1>
        <p className="date">{date}</p>
      </div>
    </motion.div>
  )
}
```

### 6.8 Custom Hook: useTime

```jsx
// src/hooks/useTime.js
import { useState, useEffect } from 'react'
import { format } from 'date-fns'

export function useTime() {
  const [time, setTime] = useState('')
  const [date, setDate] = useState('')

  useEffect(() => {
    const updateTime = () => {
      const now = new Date()
      setTime(format(now, 'HH:mm'))
      setDate(format(now, 'EEEE, MMMM d'))
    }

    updateTime() // Initial update

    // Update every minute at :00 seconds
    const now = new Date()
    const msUntilNextMinute = (60 - now.getSeconds()) * 1000

    const timeout = setTimeout(() => {
      updateTime()
      const interval = setInterval(updateTime, 60000)
      return () => clearInterval(interval)
    }, msUntilNextMinute)

    return () => clearTimeout(timeout)
  }, [])

  return { time, date }
}
```

### 6.9 Main App Component

```jsx
// src/App.jsx
import { NowPlaying } from './components/NowPlaying'
import { Weather } from './components/Weather'
import { Clock } from './components/Clock'
import { Standup } from './components/Standup'
import './styles/globals.css'

function App() {
  return (
    <div className="dashboard">
      <NowPlaying />
      <Weather />
      <Clock />
      <Standup />
    </div>
  )
}

export default App
```

---

## 7. Framer Motion Performance Optimization

### 7.1 Configuration for Pi 3B+ GPU

**Framer Motion reduced motion config:**

```jsx
// src/main.jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import { MotionConfig } from 'framer-motion'
import App from './App'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <MotionConfig
      reducedMotion="user"  // Respect system preference
      transition={{
        duration: 0.3,       // Shorter = less GPU work
        ease: "easeInOut"    // Simpler easing
      }}
    >
      <App />
    </MotionConfig>
  </React.StrictMode>
)
```

### 7.2 GPU-Friendly Animation Properties

**DO Use (GPU-accelerated):**
- ✅ `opacity`
- ✅ `transform` (translate, scale, rotate)
- ✅ `filter` (with caution)

**DON’T Use (CPU-intensive):**
- ❌ `width` / `height`
- ❌ `top` / `left` / `right` / `bottom`
- ❌ `margin` / `padding`
- ❌ `background-position`

**Example - Efficient Animation:**

```jsx
// ✅ GOOD: GPU-accelerated
<motion.div
  initial={{ opacity: 0, x: -20 }}
  animate={{ opacity: 1, x: 0 }}
  transition={{ duration: 0.3 }}
/>

// ❌ BAD: Forces layout recalculation
<motion.div
  initial={{ opacity: 0, marginLeft: -20 }}
  animate={{ opacity: 1, marginLeft: 0 }}
/>
```

### 7.3 Layout Animations - Use Sparingly

Framer Motion’s layout animations are powerful but expensive:

```jsx
// Use only when necessary
<motion.div layout layoutId="unique-id">
  {content}
</motion.div>

// Prefer explicit animations
<motion.div
  animate={{ x: isExpanded ? 0 : 100 }}
/>
```

### 7.4 Animation Performance Monitoring

```jsx
// src/utils/performance.js
export function monitorFrameRate() {
  let lastTime = performance.now()
  let frames = 0

  function checkFrameRate() {
    frames++
    const currentTime = performance.now()

    if (currentTime >= lastTime + 1000) {
      const fps = Math.round((frames * 1000) / (currentTime - lastTime))
      console.log(`FPS:${fps}`)

      if (fps < 50) {
        console.warn('Performance degradation detected')
      }

      frames = 0
      lastTime = currentTime
    }

    requestAnimationFrame(checkFrameRate)
  }

  requestAnimationFrame(checkFrameRate)
}
```

### 7.5 Conditional Animation Strategy

Disable animations if performance drops:

```jsx
// src/hooks/usePerformanceMode.js
import { useState, useEffect } from 'react'

export function usePerformanceMode() {
  const [reducedMotion, setReducedMotion] = useState(false)

  useEffect(() => {
    // Check for reduced motion preference
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)')
    setReducedMotion(mediaQuery.matches)

    // Monitor FPS (simplified)
    let frameCount = 0
    let lastTime = performance.now()

    function checkPerformance() {
      frameCount++
      const currentTime = performance.now()

      if (currentTime >= lastTime + 1000) {
        const fps = Math.round((frameCount * 1000) / (currentTime - lastTime))

        // Disable animations if FPS drops below 45
        if (fps < 45 && !reducedMotion) {
          setReducedMotion(true)
          console.warn('Performance mode activated')
        }

        frameCount = 0
        lastTime = currentTime
      }

      requestAnimationFrame(checkPerformance)
    }

    const rafId = requestAnimationFrame(checkPerformance)
    return () => cancelAnimationFrame(rafId)
  }, [])

  return reducedMotion
}

// Usage in components:
const reducedMotion = usePerformanceMode()

<motion.div
  animate={reducedMotion ? {} : { opacity: 1, x: 0 }}
/>
```

### 7.6 Framer Motion Bundle Size Optimization

**Tree-shaking - Import only what you need:**

```jsx
// ❌ BAD: Imports entire library
import { motion, AnimatePresence, useAnimation } from 'framer-motion'

// ✅ GOOD: Import specific features (if available)
// Note: Framer Motion doesn't support deep imports, but minification handles this
import { motion } from 'framer-motion'

// For minimal builds, consider alternatives:
// - react-spring (smaller footprint)
// - CSS animations (no JS overhead)
```

**Bundle Size Comparison:**
- Framer Motion: ~60KB gzipped
- React Spring: ~30KB gzipped
- CSS Animations: 0KB (native browser)

**Recommendation:** Framer Motion is worth the size for the DX and features.

---

## 8. UI/UX Performance Optimization

### 8.1 Deep Black Theme CSS

**Global Styles (`src/styles/globals.css`):**

```css
:root {
  /* Color Palette */
  --black: #000000;
  --gray-900: #0a0a0a;
  --gray-800: #1a1a1a;
  --white: #ffffff;
  --gray-300: #d1d5db;
  --accent: #3b82f6;

  /* Typography */
  --font-display: 'SF Pro Display', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --font-mono: 'SF Mono', 'Cascadia Code', 'Fira Code', monospace;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: var(--font-display);
  background: var(--black);
  color: var(--white);
  overflow: hidden;  /* Hide scrollbars in kiosk */
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  cursor: none;  /* Hide cursor in kiosk mode */
}

#root {
  width: 100vw;
  height: 100vh;
}

/* Dashboard Grid */
.dashboard {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-template-rows: 1fr;
  width: 1920px;
  height: 480px;
  gap: 0;
}

/* Panel Base Styles */
.panel {
  width: 480px;
  height: 480px;
  padding: 32px;
  background: var(--black);
  border-right: 1px solid var(--gray-800);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;

  /* Hardware acceleration */
  will-change: transform;
  transform: translateZ(0);
  backface-visibility: hidden;

  /* Prevent text selection */
  user-select: none;
  -webkit-user-select: none;
}

.panel:last-child {
  border-right: none;
}

/* Typography Performance */
.time {
  font-size: 96px;
  font-weight: 600;
  letter-spacing: -0.02em;
  font-variant-numeric: tabular-nums;  /* Monospace numbers */

  /* Text rendering optimization */
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
}

/* Animation Performance */
@media (prefers-reduced-motion: no-preference) {
  .panel {
    transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }
}

/* GPU-accelerated blur (use sparingly) */
.blur-panel {
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
}
```

### 8.2 CSS Containment for Performance

**Layout Containment:**

```css
/* Tell browser this panel's layout doesn't affect siblings */
.panel {
  contain: layout style paint;
}

/* For animated panels */
.animated-panel {
  contain: layout style;  /* Exclude paint for animations */
}

/* For static content */
.static-panel {
  contain: strict;  /* Maximum containment */
}
```

**Containment Impact:**
- Prevents layout recalculation of sibling panels
- ~20-30% faster paint times
- Essential for 4-panel grid layout

### 8.3 Font Loading Strategy

**Optimal Font Loading:**

```html
<!-- index.html -->
<head>
  <!-- Preload critical fonts -->
  <link
    rel="preload"
    href="/fonts/SFProDisplay-Regular.woff2"
    as="font"
    type="font/woff2"
    crossorigin
/>

  <!-- Font face with optimal display -->
  <style>
    @font-face {
      font-family: 'SF Pro Display';
      src: url('/fonts/SFProDisplay-Regular.woff2') format('woff2');
      font-weight: 400;
      font-style: normal;
      font-display: swap;  /* Show fallback immediately */
    }
  </style>
</head>
```

**Font Subsetting:**

```bash
# Use pyftsubset to reduce font file size
pip install fonttools brotli

pyftsubset SFProDisplay-Regular.ttf \
  --output-file=SFProDisplay-Regular-subset.woff2 \
  --flavor=woff2 \
  --unicodes="U+0020-007E,U+00A0-00FF"  # Basic Latin + extended
```

### 8.4 Image Optimization

**For Weather Icons, Album Art:**

```css
/* Lazy loading images */
img {
  loading: lazy;
  decoding: async;
}

/* Optimize rendering */
.album-art {
  width: 200px;
  height: 200px;
  object-fit: cover;
  will-change: transform;
  transform: translateZ(0);
}

/* Prevent layout shift */
.image-container {
  aspect-ratio: 1 / 1;
  background: var(--gray-900);
}
```

**Image Format Recommendations:**
- **Album Art:** WebP (70% quality) ~15KB
- **Icons:** SVG (inlined in React components)
- **Backgrounds:** CSS gradients (0KB)

### 8.5 Avoiding Reflows and Repaints

**Reflow Triggers (AVOID):**

```jsx
// ❌ BAD: Causes layout recalculation
element.style.width = '200px'
element.offsetHeight  // Forces sync layout
element.classList.add('wide')

// ✅ GOOD: Use transforms
element.style.transform = 'scaleX(2)'
```

**Batching DOM Reads/Writes:**

```jsx
// ❌ BAD: Interleaved reads/writes
element1.style.height = element1.offsetHeight + 'px'
element2.style.height = element2.offsetHeight + 'px'

// ✅ GOOD: Batch reads, then writes
const height1 = element1.offsetHeight
const height2 = element2.offsetHeight
element1.style.height = height1 + 'px'
element2.style.height = height2 + 'px'
```

### 8.6 React Performance Optimizations

**Memoization Strategy:**

```jsx
import { memo, useMemo, useCallback } from 'react'

// Memoize expensive components
export const Weather = memo(function Weather({ data }) {
  // Only re-render if data changes
  const formattedTemp = useMemo(
    () => `${Math.round(data.temp)}°C`,
    [data.temp]
  )

  return<div>{formattedTemp}</div>
})

// Memoize callbacks passed to children
function Parent() {
  const handleUpdate = useCallback(() => {
    // Update logic
  }, [/* dependencies */])

  return<Child onUpdate={handleUpdate} />
}
```

### 8.7 Virtual Scrolling (If Needed)

For calendar/event lists:

```jsx
// Use react-window for long lists
import { FixedSizeList } from 'react-window'

function EventList({ events }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      {events[index].title}
    </div>
  )

  return (
    <FixedSizeList
      height={400}
      itemCount={events.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  )
}
```

### 8.8 CSS Grid vs Flexbox Performance

**For 1920×480 Layout:**

```css
/* ✅ GOOD: CSS Grid (fewer calculations) */
.dashboard {
  display: grid;
  grid-template-columns: repeat(4, 480px);
  grid-template-rows: 480px;
}

/* ❌ AVOID: Flexbox (more calculations for equal widths) */
.dashboard {
  display: flex;
  justify-content: space-between;
}
.panel {
  flex: 1;
}
```

**Performance Impact:**
- CSS Grid: Single layout calculation
- Flexbox: Iterative width calculations

---

## 9. Project Structure

### 9.1 Complete Directory Layout

```
pi-kiosk-dashboard/
├── public/
│   ├── fonts/
│   │   ├── SFProDisplay-Regular.woff2
│   │   └── SFProDisplay-Bold.woff2
│   └── favicon.ico
│
├── src/
│   ├── components/
│   │   ├── NowPlaying.jsx
│   │   ├── Weather.jsx
│   │   ├── Clock.jsx
│   │   └── Standup.jsx
│   │
│   ├── hooks/
│   │   ├── useTime.js
│   │   ├── useWeather.js
│   │   ├── useSpotify.js
│   │   └── usePerformanceMode.js
│   │
│   ├── services/
│   │   ├── spotify.js
│   │   ├── weather.js
│   │   ├── calendar.js
│   │   └── cache.js
│   │
│   ├── styles/
│   │   ├── globals.css
│   │   └── components/
│   │       ├── nowplaying.css
│   │       ├── weather.css
│   │       ├── clock.css
│   │       └── standup.css
│   │
│   ├── utils/
│   │   ├── performance.js
│   │   └── constants.js
│   │
│   ├── App.jsx
│   ├── main.jsx
│   └── index.html
│
├── .env.example
├── .gitignore
├── package.json
├── vite.config.js
└── README.md
```

### 9.2 Environment Configuration

**`.env.example`:**

```
# API Keys (replace with your own)
VITE_SPOTIFY_CLIENT_ID=your_spotify_client_id
VITE_SPOTIFY_CLIENT_SECRET=your_spotify_client_secret
VITE_WEATHER_API_KEY=your_openweather_api_key
VITE_CALENDAR_API_KEY=your_google_calendar_api_key

# Configuration
VITE_LOCATION_LAT=40.7128
VITE_LOCATION_LON=-74.0060
VITE_TEMP_UNIT=metric
```

### 9.3 Build Process

**Development Build:**

```bash
npm run dev
# Accessible at http://localhost:3000
```

**Production Build:**

```bash
npm run build
# Output: dist/ directory

# Analyze bundle size
npm run build:analyze
```

**Expected Bundle Sizes:**
- `index.html`: ~2KB
- `vendor.js`: ~140KB (React + React-DOM)
- `motion.js`: ~60KB (Framer Motion)
- `index.js`: ~20KB (App code)
- `index.css`: ~5KB
- **Total:** ~230KB gzipped (~80KB)

### 9.4 Deployment to Raspberry Pi

**Option A: Build on Development Machine, Deploy to Pi**

```bash
# On dev machine
npm run build

# Copy to Pi
rsync -avz --delete dist/ kiosk@raspberrypi.local:/home/kiosk/dashboard/

# On Pi, serve with built-in preview server
cd /home/kiosk/dashboard
npm install -g serve
serve -s dist -l 3000
```

**Option B: Build on Pi (Slower)**

```bash
# On Pi
cd /home/kiosk
git clone https://github.com/yourusername/pi-kiosk-dashboard.git dashboard
cd dashboard
npm install
npm run build
npm run preview
```

---

## 10. Autostart Configuration

### 10.1 Systemd Service (Recommended)

**Create `/etc/systemd/system/kiosk.service`:**

```
[Unit]
Description=Kiosk Mode Dashboard
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=kiosk
Group=kiosk
Environment=DISPLAY=:0
Environment=XAUTHORITY=/home/kiosk/.Xauthority

# Start X server and Openbox
ExecStartPre=/usr/bin/sleep 5
ExecStart=/usr/bin/startx /usr/bin/openbox-session -- :0 vt1

# Restart on failure
Restart=always
RestartSec=10

# Memory limits
MemoryHigh=700M
MemoryMax=800M

# Logging
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Enable and Start:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable kiosk.service
sudo systemctl start kiosk.service

# Check status
sudo systemctl status kiosk.service

# View logs
sudo journalctl -u kiosk.service -f
```

### 10.2 Openbox Autostart Configuration

**Create `/home/kiosk/.config/openbox/autostart`:**

```bash
#!/bin/bash

# Wait for X server to fully initialize
sleep 2

# Disable screen blanking
xset s off
xset -dpms
xset s noblank

# Hide cursor after 1 second
unclutter -idle 1 -root &

# Start dashboard server (if using separate server)
cd /home/kiosk/dashboard
serve -s dist -l 3000 &

# Wait for server to start
sleep 3

# Launch Chromium in kiosk mode
chromium-browser \
  --kiosk \
  --noerrdialogs \
  --disable-infobars \
  --no-first-run \
  --disable-session-crashed-bubble \
  --disable-component-update \
  --incognito \
  --disable-dev-shm-usage \
  --disable-background-timer-throttling \
  --disable-backgrounding-occluded-windows \
  --disable-breakpad \
  --disable-sync \
  --disable-translate \
  --disable-features=TranslateUI \
  --disable-gpu-rasterization \
  --enable-gpu-memory-buffer-video-frames \
  --num-raster-threads=2 \
  --disable-background-networking \
  --disable-default-apps \
  --disable-extensions \
  --prerender-from-omnibox=disabled \
  --dns-prefetch-disable \
  http://localhost:3000
```

**Make executable:**

```bash
chmod +x /home/kiosk/.config/openbox/autostart
```

### 10.3 Alternative: labwc Autostart

**Create `/home/kiosk/.config/labwc/autostart`:**

```bash
#!/bin/bash

# Start dashboard server
cd /home/kiosk/dashboard
serve -s dist -l 3000 &

sleep 3

# Launch Chromium with Wayland backend
chromium-browser \
  --kiosk \
  --ozone-platform=wayland \
  --enable-features=UseOzonePlatform \
  --noerrdialogs \
  --disable-infobars \
  --no-first-run \
  --incognito \
  --disable-dev-shm-usage \
  --disable-gpu-rasterization \
  http://localhost:3000 &
```

### 10.4 Crash Recovery Script

**Create `/home/kiosk/watchdog.sh`:**

```bash
#!/bin/bash

while true; do
  # Check if Chromium is running
  if !pgrep -x "chromium-browse" > /dev/null; then
    echo "$(date): Chromium crashed, restarting..."
    logger "Kiosk watchdog: Chromium crashed, restarting"

    # Kill any stale processes
    pkill -9 chromium

    # Wait a moment
    sleep 2

    # Restart via systemctl
    systemctl restart kiosk.service
  fi

  sleep 30
done
```

**Add to systemd service:**

```
[Service]
ExecStartPost=/home/kiosk/watchdog.sh
```

---

## 11. CM4 Migration Planning (Phase 2)

### 11.1 Compute Module 4 Overview

**CM4 Advantages Over Pi 3B+:**

| Feature | Pi 3B+ | CM4 |
| --- | --- | --- |
| **RAM Options** | 1GB | 1GB / 2GB / 4GB / 8GB |
| **Storage** | SD Card | eMMC / SD / NVMe |
| **Form Factor** | Standard | Compact (55×40mm) |
| **PCIe** | No | Yes (Gen 2 x1) |
| **Dual Display** | No | Yes (2× DSI/HDMI) |
| **Power** | 5V/2.5A | 5V/3A (under load) |

**Recommendation:** CM4 with 2GB RAM and 16GB eMMC (~$50)

### 11.2 Waveshare Nano Carrier Board

**Specifications:**
- Minimal carrier for CM4
- DSI connector (compatible with 8.8” display)
- USB-C power input
- GPIO breakout
- No Pogo pins (relevant for side-mounting)

**Pinout Diagram:**

```
┌─────────────────────────────────┐
│     Waveshare CM4 Nano          │
│                                 │
│  ┌─────────────────────────┐   │
│  │       CM4 Module        │   │
│  │      (55×40mm)          │   │
│  └─────────────────────────┘   │
│                                 │
│  ○ 5V IN  ○ GND  ○ USB-C       │
│                                 │
│  DSI Connector ───────────────► │
│                                 │
└─────────────────────────────────┘
```

### 11.3 Wiring Changes for Side-Mount

**Challenge:** Pogo pins disconnected when side-mounting CM4

**Power Delivery Options:**

### Option A: Direct Solder to 5V Pads (Recommended)

```
CM4 Module 5V Pads → Wire → Carrier Board 5V Input
```

**Steps:**
1. Identify 5V pads on CM4 underside (J8 header area)
2. Solder 22 AWG silicone wire to pads
3. Route wire to carrier board 5V input terminals
4. Use strain relief (hot glue or cable ties)

**Wire Specifications:**
- Gauge: 22 AWG (0.64mm²) minimum for 3A
- Type: Silicone insulated (flexible)
- Length: Keep < 10cm to minimize voltage drop
- Connector: XT30 or JST-XH for easy disconnect

**Voltage Drop Calculation:**

```
V_drop = I × R
R = ρ × L / A

For 22 AWG, 10cm wire:
R = 0.053 Ω
V_drop @ 3A = 3A × 0.053Ω = 0.16V ✓ (acceptable)
```

### Option B: USB-C PD Trigger

**Use dedicated USB-C PD trigger board:**
- Boards like ZY12PDN trigger 5V/3A from USB-C PD
- Connect to carrier board USB-C input
- Cleaner than direct solder
- Cost: ~$5

**Wiring:**

```
USB-C PD Wall Adapter → PD Trigger Board → Carrier USB-C Input
```

### 11.4 DSI Cable Routing

**Considerations for Side-Mount:**
- Use 15-pin DSI FFC (same as Pi 3B+)
- Cable length: 150mm recommended
- Route cable along side of enclosure
- Avoid sharp bends (>90°)
- Secure with Kapton tape

**Cable Spec:**
- Type: 15-pin, 1.0mm pitch FFC
- Length: 150mm (allows flexibility)
- Orientation: Contacts same side (Type A)

### 11.5 Thermal Management

**CM4 Heat Dissipation:**
- TDP: ~5W under load
- Side-mounting affects natural convection

**Cooling Solutions:**

1. **Passive Heatsink (Recommended):**
    - Low-profile aluminum heatsink
    - Size: 40×40×5mm
    - Thermal pad: 1mm thickness
    - Mounting: Thermal adhesive
2. **Active Cooling (If Needed):**
    - 30mm×30mm×7mm fan
    - Voltage: 5V
    - CFM: 3-5 (quiet operation)
    - Mount on side opposite display

**Temperature Monitoring:**

```bash
# Check CM4 temperature
vcgencmd measure_temp

# Log to file
while true; do
  echo "$(date):$(vcgencmd measure_temp)" >> /home/kiosk/temp.log
  sleep 60
done
```

### 11.6 Software Migration Checklist

**Compatibility:**
- ✅ Same OS: Raspberry Pi OS Bookworm (64-bit)
- ✅ Same display driver: `dtoverlay=vc4-kms-dsi-waveshare-panel,8_8_inch`
- ✅ Same Chromium configuration
- ✅ Same React dashboard

**Configuration Changes:**

```
# Update /boot/firmware/config.txt for CM4

# GPU memory can be reduced with more RAM
gpu_mem=64  # Keep at 64MB for consistency

# CM4-specific optimizations
# Enable PCIe Gen 2 (if using NVMe)
dtparam=pciex1_gen=2

# Faster boot with eMMC
dtparam=sd_poll_once

# Rest remains the same
```

**Performance Improvements:**

| Metric | Pi 3B+ | CM4 (2GB) | Improvement |
| --- | --- | --- | --- |
| Available RAM | 960MB | 1984MB | +106% |
| Boot time (eMMC) | 15s | 8s | -47% |
| React build time | 120s | 45s | -62% |
| Bundle load time | 2.5s | 1.2s | -52% |

### 11.7 Migration Steps

**Hardware Migration:**

1. Power down Pi 3B+ setup
2. Disconnect DSI cable from Pi 3B+
3. Mount CM4 to Nano carrier board
4. Solder 5V power wires (if side-mounting)
5. Connect DSI cable to CM4 carrier
6. Connect power supply (5V/3A minimum)
7. Test boot with existing SD card

**Software Migration:**

1. Clone existing SD card to new eMMC:
    
    ```bash
    # Use rpiboot tool to mount CM4 eMMC as USB drive
    sudo rpiboot
    
    # Clone with dd
    sudo dd if=/dev/sda of=/dev/sdb bs=4M status=progress
    ```
    
2. Update config.txt (if needed)
3. Test boot and verify display
4. Run memory stress test:
    
    ```bash
    # Verify memory improvements
    free -h
    stress-ng --vm 1 --vm-bytes 1.5G --timeout 60s
    ```
    

### 11.8 Power Budget

**Pi 3B+ Power:**
- Idle: ~1.5W (300mA @ 5V)
- Load: ~5W (1A @ 5V)
- Peak: ~8W (1.6A @ 5V)

**CM4 Power:**
- Idle: ~2W (400mA @ 5V)
- Load: ~6W (1.2A @ 5V)
- Peak: ~12W (2.4A @ 5V)

**Display Power:**
- Waveshare 8.8” DSI: ~2.5W (500mA @ 5V)

**Total System:**
- Pi 3B+ system: ~10W peak
- CM4 system: ~14W peak

**Power Supply Recommendation:**
- Voltage: 5V regulated
- Current: 3A minimum (4A recommended)
- Type: USB-C PD or barrel jack
- Quality: Use official Raspberry Pi power supply

---

## 12. Data Integration Options

### 12.1 Spotify Web API Integration

**API Setup:**

1. Create Spotify Developer Account
2. Register application: https://developer.spotify.com/dashboard
3. Get Client ID and Client Secret
4. Set redirect URI: `http://localhost:3000/callback`

**Authentication Flow (PKCE):**

```jsx
// src/services/spotify.js
const CLIENT_ID = import.meta.env.VITE_SPOTIFY_CLIENT_ID
const REDIRECT_URI = 'http://localhost:3000/callback'
const SCOPES = 'user-read-currently-playing user-read-playback-state'

export async function authenticateSpotify() {
  // Generate code verifier and challenge
  const codeVerifier = generateRandomString(64)
  const codeChallenge = await generateCodeChallenge(codeVerifier)

  localStorage.setItem('code_verifier', codeVerifier)

  const params = new URLSearchParams({
    client_id: CLIENT_ID,
    response_type: 'code',
    redirect_uri: REDIRECT_URI,
    scope: SCOPES,
    code_challenge_method: 'S256',
    code_challenge: codeChallenge,
  })

  window.location.href = `https://accounts.spotify.com/authorize?${params}`
}

export async function getCurrentlyPlaying(accessToken) {
  const response = await fetch(
    'https://api.spotify.com/v1/me/player/currently-playing',
    {
      headers: {
        Authorization: `Bearer${accessToken}`,
      },
    }
  )

  if (response.status === 204) {
    return null // Nothing playing
  }

  return response.json()
}

// Helper functions
function generateRandomString(length) {
  const possible = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
  const values = crypto.getRandomValues(new Uint8Array(length))
  return values.reduce((acc, x) => acc + possible[x % possible.length], '')
}

async function generateCodeChallenge(codeVerifier) {
  const digest = await crypto.subtle.digest(
    'SHA-256',
    new TextEncoder().encode(codeVerifier)
  )
  return btoa(String.fromCharCode(...new Uint8Array(digest)))
    .replace(/=/g, '')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
}
```

**Polling Strategy:**

```jsx
// src/hooks/useSpotify.js
import { useState, useEffect } from 'react'
import { getCurrentlyPlaying } from '../services/spotify'

export function useSpotify(accessToken) {
  const [track, setTrack] = useState(null)
  const [isPlaying, setIsPlaying] = useState(false)

  useEffect(() => {
    if (!accessToken) return

    async function fetchNowPlaying() {
      try {
        const data = await getCurrentlyPlaying(accessToken)

        if (data && data.item) {
          setTrack({
            name: data.item.name,
            artist: data.item.artists[0].name,
            album: data.item.album.name,
            albumArt: data.item.album.images[0].url,
            progress: data.progress_ms,
            duration: data.item.duration_ms,
          })
          setIsPlaying(data.is_playing)
        } else {
          setTrack(null)
          setIsPlaying(false)
        }
      } catch (error) {
        console.error('Spotify API error:', error)
      }
    }

    fetchNowPlaying()

    // Poll every 5 seconds
    const interval = setInterval(fetchNowPlaying, 5000)

    return () => clearInterval(interval)
  }, [accessToken])

  return { track, isPlaying }
}
```

**Caching Strategy:**

```jsx
// Cache album art to reduce API calls
const CACHE_DURATION = 30 * 60 * 1000 // 30 minutes

function cacheImage(url) {
  const cached = localStorage.getItem(`album_${url}`)

  if (cached) {
    const { data, timestamp } = JSON.parse(cached)
    if (Date.now() - timestamp < CACHE_DURATION) {
      return data
    }
  }

  // Fetch and cache
  fetch(url)
    .then(r => r.blob())
    .then(blob => {
      const reader = new FileReader()
      reader.onload = () => {
        localStorage.setItem(
          `album_${url}`,
          JSON.stringify({
            data: reader.result,
            timestamp: Date.now(),
          })
        )
      }
      reader.readAsDataURL(blob)
    })
}
```

### 12.2 Weather API Integration

**OpenWeather API Setup:**

1. Sign up: https://openweathermap.org/api
2. Get API key (free tier: 1000 calls/day)
3. Choose endpoint: Current Weather or One Call API

**API Client:**

```jsx
// src/services/weather.js
const API_KEY = import.meta.env.VITE_WEATHER_API_KEY
const LAT = import.meta.env.VITE_LOCATION_LAT
const LON = import.meta.env.VITE_LOCATION_LON

export async function getWeather() {
  const url = `https://api.openweathermap.org/data/2.5/weather?lat=${LAT}&lon=${LON}&appid=${API_KEY}&units=metric`

  const response = await fetch(url)
  const data = await response.json()

  return {
    temp: Math.round(data.main.temp),
    feelsLike: Math.round(data.main.feels_like),
    condition: data.weather[0].main,
    description: data.weather[0].description,
    icon: data.weather[0].icon,
    humidity: data.main.humidity,
    windSpeed: data.wind.speed,
  }
}
```

**Caching with React Query (Optional):**

```jsx
// Install: npm install @tanstack/react-query

import { useQuery } from '@tanstack/react-query'
import { getWeather } from '../services/weather'

export function useWeather() {
  return useQuery({
    queryKey: ['weather'],
    queryFn: getWeather,
    staleTime: 10 * 60 * 1000,  // 10 minutes
    cacheTime: 30 * 60 * 1000,   // 30 minutes
    refetchInterval: 10 * 60 * 1000,  // Refresh every 10 min
  })
}
```

### 12.3 Google Calendar API

**Setup:**

1. Enable Google Calendar API in Cloud Console
2. Create OAuth 2.0 credentials
3. Download credentials.json

**API Client:**

```jsx
// src/services/calendar.js
const API_KEY = import.meta.env.VITE_CALENDAR_API_KEY
const CALENDAR_ID = 'primary'

export async function getUpcomingEvents(accessToken) {
  const now = new Date().toISOString()
  const url = `https://www.googleapis.com/calendar/v3/calendars/${CALENDAR_ID}/events?timeMin=${now}&maxResults=5&singleEvents=true&orderBy=startTime`

  const response = await fetch(url, {
    headers: {
      Authorization: `Bearer${accessToken}`,
    },
  })

  const data = await response.json()

  return data.items.map(event => ({
    id: event.id,
    title: event.summary,
    start: new Date(event.start.dateTime || event.start.date),
    end: new Date(event.end.dateTime || event.end.date),
    location: event.location,
  }))
}
```

### 12.4 Rate Limiting & Error Handling

**Rate Limiter:**

```jsx
// src/utils/rateLimiter.js
export class RateLimiter {
  constructor(maxCalls, perMilliseconds) {
    this.maxCalls = maxCalls
    this.perMilliseconds = perMilliseconds
    this.calls = []
  }

  async execute(fn) {
    const now = Date.now()

    // Remove old calls
    this.calls = this.calls.filter(
      time => now - time < this.perMilliseconds
    )

    // Check if we've hit the limit
    if (this.calls.length >= this.maxCalls) {
      const oldestCall = this.calls[0]
      const waitTime = this.perMilliseconds - (now - oldestCall)
      await new Promise(resolve => setTimeout(resolve, waitTime))
      return this.execute(fn)
    }

    // Execute and record
    this.calls.push(now)
    return fn()
  }
}

// Usage
const spotifyLimiter = new RateLimiter(100, 60000) // 100 calls per minute

async function fetchWithLimit() {
  return spotifyLimiter.execute(() => getCurrentlyPlaying(token))
}
```

**Fallback Behavior:**

```jsx
// src/hooks/useWeather.js
export function useWeather() {
  const [weather, setWeather] = useState(null)
  const [error, setError] = useState(null)

  useEffect(() => {
    async function fetch() {
      try {
        const data = await getWeather()
        setWeather(data)
        setError(null)

        // Cache successful response
        localStorage.setItem('weather_cache', JSON.stringify({
          data,
          timestamp: Date.now(),
        }))
      } catch (err) {
        console.error('Weather API error:', err)
        setError(err)

        // Use cached data on error
        const cached = localStorage.getItem('weather_cache')
        if (cached) {
          const { data } = JSON.parse(cached)
          setWeather(data)
        }
      }
    }

    fetch()
    const interval = setInterval(fetch, 10 * 60 * 1000)
    return () => clearInterval(interval)
  }, [])

  return { weather, error }
}
```

---

## 13. Monitoring & Maintenance

### 13.1 System Health Monitoring

**Create `/home/kiosk/monitor.sh`:**

```bash
#!/bin/bash

LOG_FILE="/var/log/kiosk-monitor.log"

while true; do
  TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

  # Memory usage
  MEM_TOTAL=$(free -m | awk 'NR==2{print $2}')
  MEM_USED=$(free -m | awk 'NR==2{print $3}')
  MEM_FREE=$(free -m | awk 'NR==2{print $4}')
  MEM_PERCENT=$(echo "scale=1;$MEM_USED * 100 /$MEM_TOTAL" | bc)

  # CPU temperature
  TEMP=$(vcgencmd measure_temp | cut -d'=' -f2 | cut -d"'" -f1)

  # Chromium memory
  CHROME_PID=$(pgrep -f "chromium.*kiosk" | head -1)
  if [ -n "$CHROME_PID" ]; then
    CHROME_MEM=$(ps -p $CHROME_PID -o rss= | awk '{print int($1/1024)}')
  else
    CHROME_MEM=0
  fi

  # Disk usage
  DISK_USAGE=$(df -h / | awk 'NR==2{print $5}' | tr -d '%')

  # Log metrics
  echo "$TIMESTAMP | CPU:${TEMP}°C | RAM:${MEM_USED}/${MEM_TOTAL}MB (${MEM_PERCENT}%) | Chrome:${CHROME_MEM}MB | Disk:${DISK_USAGE}%" >> $LOG_FILE

  # Alerts
  if (( $(echo "$MEM_PERCENT > 90" | bc -l) )); then
    logger "ALERT: High memory usage:${MEM_PERCENT}%"
  fi

  if (( $(echo "$TEMP > 75" | bc -l) )); then
    logger "ALERT: High CPU temperature:${TEMP}°C"
  fi

  sleep 60
done
```

**Run as systemd service:**

```
# /etc/systemd/system/kiosk-monitor.service
[Unit]
Description=Kiosk System Monitor
After=kiosk.service

[Service]
Type=simple
User=kiosk
ExecStart=/home/kiosk/monitor.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

### 13.2 Chromium Crash Recovery

**Automatic Restart on Crash:**

```bash
# /home/kiosk/chromium-watchdog.sh
#!/bin/bash

COUNTER=0
MAX_RESTARTS=5

while true; do
  if !pgrep -f "chromium.*kiosk" > /dev/null; then
    COUNTER=$((COUNTER + 1))

    if [ $COUNTER -le $MAX_RESTARTS ]; then
      echo "$(date): Chromium crash detected (restart$COUNTER/$MAX_RESTARTS)"
      logger "Kiosk watchdog: Chromium crashed, restarting ($COUNTER/$MAX_RESTARTS)"

      # Clean up stale processes
      pkill -9 chromium

      # Clear cache to free memory
      rm -rf /home/kiosk/.config/chromium/Default/Cache/*

      # Wait before restart
      sleep 5

      # Restart kiosk service
      systemctl --user restart kiosk
    else
      echo "$(date): Max restart attempts reached, giving up"
      logger "Kiosk watchdog: Max restart attempts reached"
      break
    fi
  else
    # Reset counter if running successfully
    COUNTER=0
  fi

  sleep 30
done
```

### 13.3 Log Rotation

**Configure `/etc/logrotate.d/kiosk`:**

```
/var/log/kiosk-monitor.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    create 0644 kiosk kiosk
}
```

### 13.4 Remote Access Setup

**SSH Configuration:**

```bash
# Enable SSH
sudo systemctl enable ssh
sudo systemctl start ssh

# Security: Key-based authentication only
sudo nano /etc/ssh/sshd_config

# Set these values:
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no

# Restart SSH
sudo systemctl restart ssh
```

**VNC for Remote Display (Debugging Only):**

```bash
# Install x11vnc (for Openbox)
sudo apt install -y x11vnc

# Set password
x11vnc -storepasswd /home/kiosk/.vnc/passwd

# Start VNC server
x11vnc -forever -display :0 -rfbauth /home/kiosk/.vnc/passwd &
```

### 13.5 Automated Updates Strategy

**Selective Updates:**

```bash
# Create /home/kiosk/update-dashboard.sh
#!/bin/bash

cd /home/kiosk/dashboard

# Fetch latest code
git fetch origin main

# Check if updates available
LOCAL=$(git rev-parse @)
REMOTE=$(git rev-parse @{u})

if [ $LOCAL != $REMOTE ]; then
  echo "Updates available, pulling..."
  git pull origin main

  # Rebuild
  npm install
  npm run build

  # Restart kiosk
  systemctl --user restart kiosk

  logger "Dashboard updated to$(git rev-parse --short HEAD)"
else
  echo "Already up to date"
fi
```

**Schedule with cron:**

```bash
# Edit crontab
crontab -e

# Add line (update daily at 3 AM)
0 3 * * * /home/kiosk/update-dashboard.sh >> /var/log/dashboard-update.log 2>&1
```

**OS Updates (Manual Recommended):**

```bash
# Monthly maintenance script
#!/bin/bash

# Update package list
sudo apt update

# Security updates only (safer)
sudo apt upgrade -y --only-upgrade $(apt list --upgradable 2>/dev/null | grep -i security | cut -d'/' -f1)

# Clean up
sudo apt autoremove -y
sudo apt clean

# Reboot if kernel updated
if [ -f /var/run/reboot-required ]; then
  sudo reboot
fi
```

---

## 14. Production Deployment Checklist

### 14.1 Pre-Deployment Verification

**Hardware:**
- [ ] Raspberry Pi 3B+ with 5V/2.5A power supply
- [ ] Waveshare 8.8” DSI LCD connected and tested
- [ ] microSD card (16GB minimum, Class 10)
- [ ] Network connectivity (WiFi or Ethernet)
- [ ] Adequate ventilation for Pi

**Software:**
- [ ] Raspberry Pi OS Lite (64-bit, Bookworm) flashed
- [ ] SSH enabled and tested
- [ ] System updated (`sudo apt update && upgrade`)
- [ ] Timezone and locale configured

### 14.2 Dashboard Deployment Steps

**1. Build Dashboard:**

```bash
# On development machine
cd pi-kiosk-dashboard
npm install
npm run build

# Verify bundle size
du -sh dist/
# Should be < 2MB total
```

**2. Deploy to Pi:**

```bash
# Copy via rsync
rsync -avz --delete dist/ kiosk@raspberrypi.local:/home/kiosk/dashboard/

# Or via SCP
scp -r dist/* kiosk@raspberrypi.local:/home/kiosk/dashboard/
```

**3. Install Serve:**

```bash
# On Pi
sudo npm install -g serve

# Test manually
cd /home/kiosk/dashboard
serve -s . -l 3000

# Verify accessible: curl http://localhost:3000
```

**4. Configure Boot:**

```bash
# Copy config.txt
sudo nano /boot/firmware/config.txt
# Paste production config (Section 1.3)

# Copy Openbox autostart
mkdir -p /home/kiosk/.config/openbox
nano /home/kiosk/.config/openbox/autostart
# Paste autostart script (Section 10.2)
chmod +x /home/kiosk/.config/openbox/autostart
```

**5. Create Systemd Service:**

```bash
# Copy kiosk.service
sudo nano /etc/systemd/system/kiosk.service
# Paste service configuration (Section 10.1)

# Enable service
sudo systemctl daemon-reload
sudo systemctl enable kiosk.service
```

**6. Reboot and Test:**

```bash
sudo reboot
```

### 14.3 Post-Deployment Verification

**Checklist:**

- [ ]  Display shows dashboard within 30 seconds of boot
- [ ]  All 4 panels rendering correctly at 1920×480
- [ ]  Animations smooth (no stuttering)
- [ ]  API integrations working (weather, Spotify, etc.)
- [ ]  Memory usage stable (< 800MB)
- [ ]  CPU temperature normal (< 70°C)
- [ ]  No cursor visible on screen
- [ ]  Screen saver disabled (screen stays on)

**Verification Commands:**

```bash
# SSH into Pi and run:

# Check memory
free -h

# Check temperature
vcgencmd measure_temp

# Check Chromium running
pgrep -af chromium

# Check service status
sudo systemctl status kiosk.service

# Check logs
sudo journalctl -u kiosk.service -n 50

# Monitor in real-time
htop
```

### 14.4 Performance Benchmarks

**Expected Metrics:**

| Metric | Target | Acceptable | Critical |
| --- | --- | --- | --- |
| **Boot Time** | < 15s | < 20s | > 30s |
| **RAM Usage** | < 650MB | < 750MB | > 850MB |
| **CPU Temp (Idle)** | < 55°C | < 65°C | > 75°C |
| **CPU Temp (Load)** | < 65°C | < 75°C | > 85°C |
| **Frame Rate** | 60 FPS | > 50 FPS | < 45 FPS |
| **API Response** | < 1s | < 2s | > 3s |

### 14.5 Troubleshooting Guide

**Issue: Display Not Working**

```bash
# Check DSI connection physically
# Verify config.txt
cat /boot/firmware/config.txt | grep dsi

# Check kernel messages
dmesg | grep -i dsi

# Try safe mode (remove overclock)
sudo nano /boot/firmware/config.txt
# Comment out over_voltage=2
```

**Issue: Out of Memory**

```bash
# Check what's using memory
ps aux --sort=-%mem | head -10

# Kill Chromium and restart
pkill chromium
sudo systemctl restart kiosk

# Increase swap
sudo dphys-swapfile swapoff
sudo nano /etc/dphys-swapfile
# Set CONF_SWAPSIZE=1024
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```

**Issue: Chromium Won’t Start**

```bash
# Check logs
sudo journalctl -u kiosk.service -f

# Test manually
DISPLAY=:0 chromium-browser --version

# Clear cache
rm -rf /home/kiosk/.config/chromium/Default/Cache/*

# Reinstall Chromium
sudo apt remove --purge chromium-browser
sudo apt install chromium-browser
```

**Issue: API Calls Failing**

```bash
# Test network
ping -c 4 8.8.8.8

# Test API manually
curl -I https://api.openweathermap.org

# Check environment variables
env | grep VITE

# Verify API keys in browser console
```

### 14.6 Emergency Recovery

**Recovery Mode Boot:**

1. Power off Pi
2. Remove SD card
3. Mount on another computer
4. Edit `/boot/firmware/config.txt`:
    - Comment out `dtoverlay=vc4-kms-dsi...`
    - Add `hdmi_force_hotplug=1`
5. Connect HDMI monitor
6. Boot and troubleshoot

**Factory Reset:**

```bash
# Backup important data
cd /home/kiosk
tar -czf dashboard-backup.tar.gz dashboard/

# Reinstall OS
# Use Raspberry Pi Imager to flash new OS

# Restore dashboard
tar -xzf dashboard-backup.tar.gz
```

---

## Appendix A: Complete Example Component

**NowPlaying.jsx - Full Implementation:**

```jsx
import { motion, AnimatePresence } from 'framer-motion'
import { useSpotify } from '../hooks/useSpotify'
import '../styles/components/nowplaying.css'

export function NowPlaying() {
  const { track, isPlaying } = useSpotify()

  if (!track) {
    return (
      <div className="panel nowplaying-panel">
        <div className="empty-state">
          <p>Nothing playing</p>
        </div>
      </div>
    )
  }

  const progress = (track.progress / track.duration) * 100

  return (
    <motion.div
      className="panel nowplaying-panel"
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      transition={{ duration: 0.5 }}
    >
      <div className="content">
        <p className="label">NOW PLAYING</p>

        <AnimatePresence mode="wait">
          <motion.div
            key={track.name}
            className="track-info"
            initial={{ y: 20, opacity: 0 }}
            animate={{ y: 0, opacity: 1 }}
            exit={{ y: -20, opacity: 0 }}
            transition={{ duration: 0.3 }}
          >
            <motion.img
              src={track.albumArt}
              alt={track.album}
              className="album-art"
              initial={{ scale: 0.8, opacity: 0 }}
              animate={{ scale: 1, opacity: 1 }}
              transition={{ duration: 0.4 }}
            />

            <h2 className="track-name">{track.name}</h2>
            <p className="artist-name">{track.artist}</p>

            <div className="progress-bar">
              <motion.div
                className="progress-fill"
                initial={{ width: 0 }}
                animate={{ width: `${progress}%` }}
                transition={{ duration: 0.5 }}
              />
            </div>
          </motion.div>
        </AnimatePresence>
      </div>
    </motion.div>
  )
}
```

**nowplaying.css:**

```css
.nowplaying-panel {
  background: linear-gradient(135deg, var(--black) 0%, var(--gray-900) 100%);
}

.label {
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.1em;
  color: var(--gray-300);
  margin-bottom: 24px;
}

.track-info {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.album-art {
  width: 200px;
  height: 200px;
  border-radius: 8px;
  object-fit: cover;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5);
  will-change: transform;
  transform: translateZ(0);
}

.track-name {
  font-size: 28px;
  font-weight: 700;
  text-align: center;
  max-width: 400px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.artist-name {
  font-size: 18px;
  color: var(--gray-300);
  text-align: center;
}

.progress-bar {
  width: 300px;
  height: 4px;
  background: var(--gray-800);
  border-radius: 2px;
  overflow: hidden;
  margin-top: 8px;
}

.progress-fill {
  height: 100%;
  background: var(--accent);
  border-radius: 2px;
  will-change: width;
}

.empty-state {
  color: var(--gray-300);
  font-size: 18px;
}
```

---

## Appendix B: Useful Commands Reference

**System Monitoring:**

```bash
# Memory
free -h
cat /proc/meminfo
vmstat 1

# CPU
htop
top
mpstat 1

# Temperature
vcgencmd measure_temp
watch -n 1 vcgencmd measure_temp

# GPU Memory
vcgencmd get_mem arm
vcgencmd get_mem gpu

# Processes
ps aux --sort=-%mem | head -10
pstree -p

# Disk
df -h
du -sh /home/kiosk/dashboard

# Network
ifconfig
ping google.com
curl -I example.com
```

**Service Management:**

```bash
# Start/Stop/Restart
sudo systemctl start kiosk
sudo systemctl stop kiosk
sudo systemctl restart kiosk

# Enable/Disable
sudo systemctl enable kiosk
sudo systemctl disable kiosk

# Status
sudo systemctl status kiosk
sudo systemctl is-active kiosk

# Logs
sudo journalctl -u kiosk.service
sudo journalctl -u kiosk.service -f  # Follow
sudo journalctl -u kiosk.service --since today
```

**Chromium Control:**

```bash
# Find Chromium process
pgrep -af chromium

# Kill Chromium
pkill chromium
pkill -9 chromium  # Force kill

# Memory usage
ps -p $(pgrep chromium | head -1) -o pid,vsz,rss,comm

# Clear cache
rm -rf ~/.config/chromium/Default/Cache/*
```

---

## Appendix C: Additional Resources

**Official Documentation:**
- Raspberry Pi OS: https://www.raspberrypi.com/documentation/
- Chromium Flags: https://peter.sh/experiments/chromium-command-line-switches/
- React Performance: https://react.dev/learn/render-and-commit
- Framer Motion: https://www.framer.com/motion/

**Community Resources:**
- Raspberry Pi Forums: https://forums.raspberrypi.com/
- r/raspberry_pi: https://reddit.com/r/raspberry_pi
- Waveshare Wiki: https://www.waveshare.com/wiki/

**Tools:**
- Raspberry Pi Imager: https://www.raspberrypi.com/software/
- rpiboot (CM4): https://github.com/raspberrypi/usbboot
- React DevTools: https://react.dev/learn/react-developer-tools

---

## Document Revision History

| Version | Date | Changes |
| --- | --- | --- |
| 1.0 | January 2026 | Initial release |

---

**End of Technical Specification**

For questions or issues, consult the troubleshooting guide (Section 14.5) or check the community resources in Appendix C.
