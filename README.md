# Lolo's Tweaks

A reversible Windows gaming optimizer with a benchmark, a game session recorder and a "FPS boost" tab for steadier 1% lows. Mainly made for laptops, with a desktop PC mode.

**Use at your own risk.** Every change is backed up and can be undone with the Undo buttons or Undo all, but you are changing Windows settings. Read the script first, it is plain text.

## Run it
Open PowerShell and paste (replace the link with the raw link of lolos-tweaks.ps1 in this repo):

```powershell
irm https://raw.githubusercontent.com/lolotweaker/lolos-tweaks/main/lolos-tweaks-v2.txt -OutFile $env:TEMP\lolos-tweaks.ps1; powershell -STA -ExecutionPolicy Bypass -File $env:TEMP\lolos-tweaks.ps1
```

It asks for administrator permission, then asks whether you are on a laptop or a desktop PC.

## What works where
- Windows 10 / 11, laptops and desktops.
- Most tweaks are generic Windows settings.
- Fan control (Turbo only while a game runs) is ASUS laptops only.
- The recorder shows temperatures, clocks and throttling on NVIDIA cards; on AMD and Intel it shows GPU usage, CPU and RAM.
- Built-in game support is Valorant and Fortnite; use "Add a game..." for others.
- Windows may show SmartScreen or Defender warnings because the script is unsigned and asks for admin.
