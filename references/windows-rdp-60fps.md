---
name: windows-rdp-tuning
description: >-
  Comprehensive guide and automated tuning script for Windows Remote Desktop
  Protocol (RDP) / Windows App on Android tablets over Tailscale. Use when RDP
  feels laggy, has scrolling stutter in VS Code, needs 60 FPS GPU acceleration,
  or needs low-latency mic/storage/webcam/dynamic rotation setup.
---

# Windows RDP Ultra-Responsive Tuning Skill

## Overview

This skill packages the complete low-latency tuning workflow for:
- **Host**: Windows 10/11 with Dedicated GPU (NVIDIA RTX 3050 Laptop GPU)
- **Client**: Android Tablet (Android Tablet Client) using Windows App / Microsoft Remote Desktop Client
- **Tunnel**: Tailscale WireGuard Direct P2P (Port 3389)
- **Key Features**: Native Auto-Rotate, Native Microphone, Native Storage (`\\tsclient`), Native Clipboard, 60 FPS GPU Accelerated Pipeline

---

## Quick Root Cause & Fix Matrix

| Symptom | Root Cause | Fix |
|---|---|---|
| **Layar patah-patah saat scroll di VS Code** | 32-bit bitmap flood & TCP delayed ACK | Set `VisualQuality = 0` (Dynamic), `fEnableProgressiveEncoding = 1`, `TcpDelAckTicks = 0` |
| **Kursor mouse terasa licin / melayang** | RDP input update batching delay | Set `MinSendInterval = 1`, `InteractiveDelay = 0`, `MaxUnacknowledgedUpdates = 1` |
| **Stutter parah di tablet Android** | AVC 4:4:4 enforced (Android lacks 4:4:4 HW decoder) | Set `bEnforceAVC444 = 0` to allow standard H.264 4:2:0 GPU decode |
| **FPS kaku / terkunci 30 FPS** | Windows DWM Terminal Server default cap | Set `DWMFrameRate = 60` in Terminal Server WinStations |
| **Koneksi RDP freeze saat sinyal goyang** | Pure TCP transmission packet drop stall | Enable RDP-UDP transport (`SelectTransport = 2`, `SelectNetworkDetectMode = 1`) |

---

## Full Automated PowerShell Tuning Script

Run in PowerShell as Administrator on Windows Host:

```powershell
# ── 1. Unlock 60 FPS & GPU Hardware Acceleration ────────────────────────────
$tsPolicy = "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services"
$tsWinStation = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations"

Set-ItemProperty -Path $tsWinStation -Name "DWMFrameRate" -Value 60 -Type DWord -Force
Set-ItemProperty -Path $tsPolicy -Name "fEnableVirtualizedGraphics" -Value 1 -Type DWord -Force
Set-ItemProperty -Path $tsPolicy -Name "fEnableWddmDriver" -Value 1 -Type DWord -Force

# ── 2. Dynamic RemoteFX Compression for Butter-Smooth 60 FPS Scrolling ──────
Set-ItemProperty -Path $tsPolicy -Name "bEnforceAVC444" -Value 0 -Type DWord -Force # AVC 4:2:0 GPU Decode
Set-ItemProperty -Path $tsPolicy -Name "VisualQuality" -Value 0 -Type DWord -Force   # Dynamic Adaptive Quality
Set-ItemProperty -Path $tsPolicy -Name "fEnableProgressiveEncoding" -Value 1 -Type DWord -Force
Set-ItemProperty -Path $tsPolicy -Name "fEnableDynamicFrameRate" -Value 1 -Type DWord -Force
Set-ItemProperty -Path $tsPolicy -Name "RemoteFXAdaptiveGraphics" -Value 1 -Type DWord -Force
Set-ItemProperty -Path $tsPolicy -Name "MaxUnacknowledgedUpdates" -Value 1 -Type DWord -Force # Immediate Flush

# ── 3. Zero Input & Scrolling Delay (Eliminate TCP Delayed ACK) ──────────────
$interfaces = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces"
Get-ChildItem $interfaces | ForEach-Object {
    Set-ItemProperty -Path $_.PSPath -Name "TcpAckFrequency" -Value 1 -Type DWord -Force -ErrorAction SilentlyContinue
    Set-ItemProperty -Path $_.PSPath -Name "TCPNoDelay" -Value 1 -Type DWord -Force -ErrorAction SilentlyContinue
    Set-ItemProperty -Path $_.PSPath -Name "TcpDelAckTicks" -Value 0 -Type DWord -Force -ErrorAction SilentlyContinue
}

# ── 4. Persistent Bitmap Caching & Priority QoS (Port 3389) ──────────────────
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Terminal Server Client" -Name "BitmapPersistence" -Value 1 -Type DWord -Force
New-NetQosPolicy -Name "RDP_TCP_QoS" -IPPortMatchCondition 3389 -IPProtocolMatchCondition TCP -DSCPAction 46 -NetworkProfile All -ErrorAction SilentlyContinue
New-NetQosPolicy -Name "RDP_UDP_QoS" -IPPortMatchCondition 3389 -IPProtocolMatchCondition UDP -DSCPAction 46 -NetworkProfile All -ErrorAction SilentlyContinue

# ── 5. IFEO High Priority for RDP Core Processes ────────────────────────────
$rdpExes = @("rdpclip.exe", "rdpinit.exe", "rdpinput.exe")
foreach ($exe in $rdpExes) {
    $p = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\$exe\PerfOptions"
    if (-not (Test-Path $p)) { New-Item -Path $p -Force | Out-Null }
    Set-ItemProperty -Path $p -Name "CpuPriorityClass" -Value 3 -Type DWord -Force
}

# ── 6. Enable Native Hardware Redirection (Mic, Webcam, Storage) ────────────
Set-ItemProperty -Path $tsPolicy -Name "fEnableAudioCapture" -Value 1 -Type DWord -Force
Set-ItemProperty -Path $tsPolicy -Name "fDisableCam" -Value 0 -Type DWord -Force
Set-ItemProperty -Path $tsPolicy -Name "AudioQuality" -Value 0 -Type DWord -Force
Enable-NetFirewallRule -DisplayGroup "Remote Desktop" -ErrorAction SilentlyContinue

Write-Host "✅ ALL WINDOWS RDP ULTRA-RESPONSIVE TWEAKS APPLIED!" -ForegroundColor Green
```

---

## Recommended Android Client Settings (Windows App / RD Client)

- **PC Name**: `100.89.122.63` (Tailscale IP)
- **Display Resolution**: `1920x1200` (Sweet spot for smooth scrolling & battery) or `Match this device`
- **Sound**: Play on this device
- **Microphone**: Record audio on this PC (ON)
- **Storage**: Redirect local storage (ON)
- **Camera**: Redirect camera (ON)
