---
name: moonlight-vibeshine-tuning
description: >-
  Comprehensive guide and automated tuning script for Moonlight game streaming
  with Sunshine/VibeShine on Windows. Use when streaming feels laggy, stuttery,
  low-resolution, or drops FPS. Covers host config (sunshine.conf), Windows
  kernel network/GPU tuning, Wi-Fi adapter optimization, and Tailscale P2P
  verification for Windows Host GPU → Android Tablet Client streaming setup.
  Also use when setting up a fresh Windows install or after driver updates.
---

# Moonlight + VibeShine Ultra-Low Latency Tuning Skill

## Overview

This skill packages the complete low-latency tuning workflow for:
- **Host**: Windows Host (Intel i5-12500H, NVIDIA RTX 3050 Laptop GPU, Windows 11)
- **Client**: Android Tablet Client running Moonlight / Artemis Android app
- **Tunnel**: Tailscale WireGuard Direct P2P (MTU 1280)
- **Software**: VibeShine (Sunshine fork) running as `SunshineService`

Use this skill whenever the user reports:
- Stream suddenly goes blurry/low-res then recovers
- Frame stuttering, micro-freezes, or "patah-patah"
- Virtual display stuck at 240Hz on every reconnect
- Keyboard/mouse input drops or double-registers
- Stream generally feels sluggish vs. local display

---

## Quick Diagnosis Checklist

Before tuning, always verify these first:

```powershell
# 1. Confirm Tailscale is in DIRECT mode (not relay)
& "C:\Program Files\Tailscale\tailscale.exe" status | Select-String "direct|relay|redmi"
# Expected: "active; direct <IP>:<port>"

# 2. Check current sunshine.conf
Get-Content "C:\Program Files\Sunshine\config\sunshine.conf"

# 3. Check SunshineService is running
Get-Service SunshineService

# 4. Check Roaming Aggressiveness (must be 1. Lowest)
Get-NetAdapterAdvancedProperty -Name "*Wi-Fi*" | Select-Object DisplayName, DisplayValue
```

---

## Root Cause Map: Symptom → Fix

| Symptom | Root Cause | Fix |
|---|---|---|
| **Layar virtual stuck di 240Hz tiap reconnect** | VibeShine default framegen multiplier 4x (`60x4=240Hz`) | Set `dd_refresh_rate_option = manual`, `dd_manual_refresh_rate = 60`, `framegen = disabled` |
| **Patah-patah parah / stutter judder** | 240Hz Virtual Display vs 60Hz stream mismatch + zombie virtual displays | Disable old VDD devices (`ROOT\DISPLAY\0000`, `0001`) & lock 60Hz |
| **Tiba-tiba blur/low-res lalu jernih lagi** | Moonlight adaptive bitrate drop + IDR keyframe spike | Matikan "Reduce bitrate on packet loss" di Moonlight app |
| **Stream patah-patah periodik (~60 detik)** | Windows Wi-Fi background scanning (roaming) | Set Roaming Aggressiveness = `1. Lowest` |
| **FPS drop saat layar diam, lalu hitch saat gerak** | VibeShine idle FPS floor turun ke 30 FPS | Set `minimum_fps_target = 60`, `fps = [60]`, `nvenc_preset = p1` |
| **Stream terasa lebih berat dari Sunshine bawaan** | VibeShine display_helper & WGC engine aktif | Set `display_helper_engine: none` di vibeshine_state.json |
| **Win+Shift+S tidak bisa (screenshot)** | `key_rightalt_to_key_win = enabled` memblokir modifier key | Set `key_rightalt_to_key_win = disabled` di sunshine.conf |

---

## Optimized `sunshine.conf` (Canonical Template v2 - Master 60Hz Lock)

```ini
# ==============================================================================
# TRI WAHYU HANDOYO - VIBESHINE ZERO-FRAMEGEN STRICT 60HZ CONFIG
# Hardware: Windows Host (Intel Core i5-12500H, NVIDIA RTX 3050 Laptop GPU)
# Client: Android Tablet Client (Moonlight over Tailscale Direct P2P)
# ==============================================================================

# Virtual Display Lock - Strict 60Hz (Disable 4x Multiplier / 240Hz FrameGen)
dd_refresh_rate_option = manual
dd_manual_refresh_rate = 60
frame_limiter_auto_virtual_framegen = disabled
framegen = disabled
auto_virtual_framegen_limiter = disabled

# Stream FPS Lock
fps = [60]
minimum_fps_target = 60
min_threads = 4

# Dedicated GPU Capture
adapter_name = NVIDIA GeForce RTX 3050 Laptop GPU

# NVENC Single-Pass Ultra Low Latency
nvenc_preset = p1
nvenc_tune = ull
nvenc_rate_control = cbr
nvenc_spatial_aq = disabled
nvenc_temporal_aq = disabled

# FEC - Packet loss recovery over Tailscale
fec_percentage = 20

# Keyboard / Shortcuts
key_rightalt_to_key_win = disabled
```

---

## Full Windows System Tuning Script

Use this script to apply all kernel/registry/driver tweaks:

```powershell
# Run as Administrator in PowerShell

# ── 1. Disable Zombie Virtual Displays (Prevent DWM Conflicts) ───────────────
Disable-PnpDevice -InstanceId "ROOT\DISPLAY\0000" -Confirm:$false -ErrorAction SilentlyContinue
Disable-PnpDevice -InstanceId "ROOT\DISPLAY\0001" -Confirm:$false -ErrorAction SilentlyContinue

# ── 2. Force VibeShine (Sunshine.exe) onto Dedicated NVIDIA GPU ──────────────
$apps = @(
  "C:\Program Files\Sunshine\sunshine.exe",
  "C:\Program Files\Sunshine\tools\sunshinesvc.exe",
  "C:\Program Files\Sunshine\tools\sunshine_display_helper.exe"
)
foreach ($app in $apps) {
  Set-ItemProperty -Path "HKCU:\Software\Microsoft\DirectX\UserGpuPreferences" `
    -Name $app -Value "GpuPreference=2;" -Force
}

# ── 3. Disable Windows Network Throttling ────────────────────────────────────
$mmPath = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile"
Set-ItemProperty $mmPath -Name "NetworkThrottlingIndex" -Value 0xFFFFFFFF -Type DWord
Set-ItemProperty $mmPath -Name "SystemResponsiveness"   -Value 0          -Type DWord

# ── 4. Wi-Fi Adapter: Disable Power Save + Roaming Aggression ────────────────
Set-NetAdapterAdvancedProperty -Name "*Wi-Fi*" -RegistryKeyword "RoamAggressiveness"      -RegistryValue "1"
Set-NetAdapterAdvancedProperty -Name "*Wi-Fi*" -RegistryKeyword "MIMOPowerSaveMode"        -RegistryValue "3"  # No SMPS
Set-NetAdapterAdvancedProperty -Name "*Wi-Fi*" -RegistryKeyword "ThroughputBoosterEnabled" -RegistryValue "0"  # Disable
Set-NetAdapterAdvancedProperty -Name "*Wi-Fi*" -RegistryKeyword "*PacketCoalescing"         -RegistryValue "0"  # Disable

# ── 5. Tailscale Adapter: Disable Power Management ───────────────────────────
Disable-NetAdapterPowerManagement -Name "Tailscale" -ErrorAction SilentlyContinue

# ── 6. UDP Buffer Tuning (64KB optimal for low-latency streaming) ────────────
$afd = "HKLM:\SYSTEM\CurrentControlSet\Services\AFD\Parameters"
Set-ItemProperty $afd -Name "DefaultReceiveWindow"      -Value 65536 -Type DWord
Set-ItemProperty $afd -Name "DefaultSendWindow"         -Value 65536 -Type DWord
Set-ItemProperty $afd -Name "FastSendDatagramThreshold" -Value 1500  -Type DWord

# ── 7. Disable Nagle Algorithm (TCP Instant ACK) ─────────────────────────────
$tcp = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters"
Set-ItemProperty $tcp -Name "TcpNoDelay"      -Value 1 -Type DWord
Set-ItemProperty $tcp -Name "TcpAckFrequency" -Value 1 -Type DWord

# ── 8. Restart VibeShine Service ─────────────────────────────────────────────
Restart-Service -Name "SunshineService" -Force
Get-Service SunshineService

Write-Host ""
Write-Host "✅ ALL TWEAKS APPLIED! VibeShine restarted successfully." -ForegroundColor Green
```

---

## Moonlight Client Settings (Android Tablet Client)

| Setting | Recommended Value | Reason |
|---|---|---|
| **Bitrate** | `10–12 Mbps` (remote/seluler) / `20–25 Mbps` (Wi-Fi rumah) | Anti-buffer bloat |
| **Resolution** | Native (2560×1600 or match display) | Kualitas teks maksimal |
| **Frame Rate** | `60 FPS` | Match host 60Hz locked refresh rate |
| **Reduce bitrate on packet loss** | **OFF** | Mencegah sudden low-res + IDR spike |
| **Capture system keys** | **ON** | Agar Win+Shift+S, Win+Tab, dll berfungsi |
| **Codec** | `HEVC (H.265)` | Kompresi lebih efisien di RTX 3050 |
