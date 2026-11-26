# warp — Ultra-Low Latency Windows RDP & Moonlight Streaming

AI Agent Skill for **Antigravity**, **Claude Code**, and **OpenAI Codex**. Tuning playbook for 60 FPS GPU-accelerated Windows RDP over Tailscale and Moonlight + Sunshine/VibeShine host/Wi-Fi/kernel tuning for Android tablets and remote clients.

---

## Quick Install (Claude Code / Codex / Antigravity)

Clone this skill into your agent's global or project skills directory:

| Agent | Global Install Path | Project-Local Path |
| :--- | :--- | :--- |
| **Antigravity** | `~/.gemini/config/skills/warp` | `.agents/skills/warp` |
| **Claude Code** | `~/.claude/skills/warp` | `.claude/skills/warp` |
| **OpenAI Codex** | `~/.codex/skills/warp` | `.agents/skills/warp` |

```bash
# Example (Antigravity / Claude Code / Codex — pick your path):
git clone https://github.com/triwahyu45/warp.git ~/.claude/skills/warp
```

---

## Requirements & Dependencies (For Agents)

- **Runtime / Tools**: **Windows 10/11 Host** (Admin PowerShell for registry/network tuning), **Tailscale**, and **Sunshine / VibeShine** + **Moonlight**
- **MCP Servers**: None required
- **Related Skills**: None required
- **API Keys / Secrets**: None

---

## Included Files

- `SKILL.md` — Quick router for RDP vs Moonlight tuning
- `references/windows-rdp-60fps.md` — GPU encoding, AVC 4:4:4, scroll stutter fix, & Tailscale RDP
- `references/moonlight-sunshine.md` — Host encoder, Wi-Fi/QoS, & kernel latency tuning

## When to Trigger / Search Keywords

`warp`, `rdp 60fps`, `windows rdp lag`, `tailscale rdp`, `moonlight`, `sunshine`, `vibeshine`, `remote desktop tuning`
