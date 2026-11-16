# Lolo's Tweaks - reversible gaming optimizer + FPS boost + benchmark + game session recorder
# Made on an ASUS TUF A15 for Valorant + Fortnite, works on most Windows 10/11 laptops and desktops.
# Fan control is ASUS-only. Use at your own risk: every change is backed up and can be undone from the app.
# Every change is backed up and can be undone from the app.

Add-Type -AssemblyName PresentationFramework, PresentationCore, WindowsBase, System.Windows.Forms

$script:SelfPath   = $PSCommandPath
$script:AppName    = "Lolo's Tweaks"
$script:Mode       = ''
$script:FanOk      = $false
$script:CustomGames = @()
$script:BackupDir  = Join-Path $env:APPDATA 'LoloTune'
$script:BackupFile = Join-Path $script:BackupDir 'backup.json'
$script:BenchFile  = Join-Path $script:BackupDir 'bench.json'
New-Item -ItemType Directory -Force -Path $script:BackupDir | Out-Null

function Test-Admin {
    ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}

# ---- relaunch elevated + STA (needed for the window) ----
$isSta = ([Threading.Thread]::CurrentThread.ApartmentState -eq 'STA')
if ($script:SelfPath -and ((-not (Test-Admin)) -or (-not $isSta))) {
    try {
        $argList = "-NoProfile -STA -ExecutionPolicy Bypass -WindowStyle Hidden -File `"$($script:SelfPath)`""
        if (Test-Admin) { Start-Process powershell.exe -ArgumentList $argList }
        else            { Start-Process powershell.exe -ArgumentList $argList -Verb RunAs }
        exit
    } catch { }
}

# =============== native helpers (mouse + benchmark engine) ===============
if (-not ('NativeMouse' -as [type])) {
    Add-Type -TypeDefinition @'
using System;
using System.Runtime.InteropServices;
public class NativeMouse {
    [DllImport("user32.dll", SetLastError=true)]
    public static extern bool SystemParametersInfo(uint uiAction, uint uiParam, int[] pvParam, uint fWinIni);
}
'@
}

# ASUS fan/thermal mode (same interface Armoury Crate uses) + high-precision timer
if (-not ('AsusAcpi' -as [type])) {
    Add-Type -TypeDefinition @'
using System;
using System.Runtime.InteropServices;

public static class AsusAcpi
{
    [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
    static extern IntPtr CreateFile(string fileName, uint access, uint share, IntPtr security, uint disposition, uint flags, IntPtr template);
    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool DeviceIoControl(IntPtr handle, uint code, byte[] inBuf, uint inSize, byte[] outBuf, uint outSize, ref uint returned, IntPtr overlapped);
    [DllImport("kernel32.dll")]
    static extern bool CloseHandle(IntPtr handle);

    const uint Control = 0x0022240C;
    const uint DSTS = 0x53545344;
    const uint DEVS = 0x53564544;

    static byte[] Call(uint method, uint id, uint value)
    {
        IntPtr h = CreateFile("\\\\.\\ATKACPI", 0xC0000000, 3, IntPtr.Zero, 3, 0x80, IntPtr.Zero);
        if (h == new IntPtr(-1)) { throw new InvalidOperationException("ATKACPI not available"); }
        try
        {
            byte[] args = new byte[8];
            BitConverter.GetBytes(id).CopyTo(args, 0);
            BitConverter.GetBytes(value).CopyTo(args, 4);
            byte[] inBuf = new byte[8 + args.Length];
            BitConverter.GetBytes(method).CopyTo(inBuf, 0);
            BitConverter.GetBytes((uint)args.Length).CopyTo(inBuf, 4);
            Array.Copy(args, 0, inBuf, 8, args.Length);
            byte[] outBuf = new byte[16];
            uint ret = 0;
            if (!DeviceIoControl(h, Control, inBuf, (uint)inBuf.Length, outBuf, (uint)outBuf.Length, ref ret, IntPtr.Zero))
            {
                throw new InvalidOperationException("ATKACPI call failed");
            }
            return outBuf;
        }
        finally { CloseHandle(h); }
    }

    public static int Get(uint id)
    {
        byte[] o = Call(DSTS, id, 0);
        return BitConverter.ToInt32(o, 0) - 65536;
    }

    public static bool Set(uint id, int value)
    {
        byte[] o = Call(DEVS, id, (uint)value);
        return BitConverter.ToInt32(o, 0) == 1;
    }
}

public static class WinTimer
{
    [DllImport("ntdll.dll")]
    static extern int NtSetTimerResolution(uint desired, bool set, out uint current);

    public static void Hold() { uint c; NtSetTimerResolution(5000, true, out c); }
    public static void Release() { uint c; NtSetTimerResolution(5000, false, out c); }
}
'@
}

if (-not ('LoloBench' -as [type])) {
    Add-Type -TypeDefinition @'
using System;
using System.IO;
using System.Diagnostics;
using System.Threading;

public static class LoloBench
{
    public static volatile string Status = "Idle";
    public static volatile bool Running = false;
    public static volatile bool Done = false;
    public static volatile int Percent = 0;
    public static volatile string Error = "";
    public static double CpuSingle = 0;
    public static double CpuMulti = 0;
    public static double CpuSustained = 0;
    public static double MemGBs = 0;
    public static double DiskWriteMBs = 0;
    public static int Threads = 0;
    public static long Sink = 0;

    const int SingleIter = 10000000;
    const int MultiIter = 4000000;

    static long Work(int n)
    {
        long acc = 0;
        double x = 0.5;
        for (int i = 1; i <= n; i++)
        {
            x = Math.Sin(x) * Math.Cos(x) + Math.Sqrt(i);
            acc += ((long)(x * 1000.0)) ^ i;
            int m = i | 1;
            bool p = m > 2;
            for (int d = 3; d * d <= m && d < 150; d += 2)
            {
                if (m % d == 0) { p = false; break; }
            }
            if (p) acc++;
        }
        return acc;
    }

    static double RunMulti(int threads, int n)
    {
        Thread[] ts = new Thread[threads];
        long[] res = new long[threads];
        Stopwatch sw = Stopwatch.StartNew();
        for (int t = 0; t < threads; t++)
        {
            int idx = t;
            ts[t] = new Thread(delegate() { res[idx] = Work(n); });
            ts[t].Start();
        }
        for (int t = 0; t < threads; t++) { ts[t].Join(); }
        sw.Stop();
        long s = 0;
        for (int t = 0; t < threads; t++) { s += res[t]; }
        Sink += s;
        return ((double)threads * n) / sw.Elapsed.TotalSeconds / 1000000.0;
    }

    static double MemTest()
    {
        int size = 128 * 1024 * 1024;
        byte[] a = new byte[size];
        byte[] b = new byte[size];
        new Random(1).NextBytes(a);
        Buffer.BlockCopy(a, 0, b, 0, size);
        int reps = 12;
        Stopwatch sw = Stopwatch.StartNew();
        for (int r = 0; r < reps; r++) { Buffer.BlockCopy(a, 0, b, 0, size); }
        sw.Stop();
        Sink += b[size - 1];
        double gb = (double)size * reps / (1024.0 * 1024.0 * 1024.0);
        return gb / sw.Elapsed.TotalSeconds;
    }

    static double DiskTest()
    {
        string path = Path.Combine(Path.GetTempPath(), "lolotune_bench.tmp");
        int chunk = 1024 * 1024;
        int count = 512;
        try
        {
            DriveInfo di = new DriveInfo(Path.GetPathRoot(path));
            if (di.AvailableFreeSpace < 1500L * 1024 * 1024) { return -1; }
        }
        catch (Exception) { }
        byte[] buf = new byte[chunk];
        new Random(2).NextBytes(buf);
        Stopwatch sw = new Stopwatch();
        try
        {
            using (FileStream fs = new FileStream(path, FileMode.Create, FileAccess.Write, FileShare.None, chunk, FileOptions.WriteThrough))
            {
                sw.Start();
                for (int i = 0; i < count; i++) { fs.Write(buf, 0, chunk); }
                fs.Flush(true);
                sw.Stop();
            }
        }
        finally
        {
            try { if (File.Exists(path)) { File.Delete(path); } } catch (Exception) { }
        }
        return count / sw.Elapsed.TotalSeconds;
    }

    public static void Start()
    {
        if (Running) { return; }
        Running = true;
        Done = false;
        Percent = 0;
        Error = "";
        Status = "Starting...";
        CpuSingle = 0; CpuMulti = 0; CpuSustained = 0; MemGBs = 0; DiskWriteMBs = 0;
        Thread t = new Thread(Run);
        t.IsBackground = true;
        t.Start();
    }

    static void Run()
    {
        try
        {
            Threads = Environment.ProcessorCount;
            Status = "Warming up...";
            Sink += Work(300000);
            Percent = 3;

            Status = "CPU single-core test...";
            Stopwatch sw = Stopwatch.StartNew();
            Sink += Work(SingleIter);
            sw.Stop();
            CpuSingle = SingleIter / sw.Elapsed.TotalSeconds / 1000000.0;
            Percent = 25;

            double[] r = new double[5];
            for (int i = 0; i < 5; i++)
            {
                Status = "CPU all-core test, round " + (i + 1) + " of 5...";
                r[i] = RunMulti(Threads, MultiIter);
                Percent = 25 + (i + 1) * 8;
            }
            CpuMulti = r[0];
            CpuSustained = r[4];

            Status = "Memory speed test...";
            try { MemGBs = MemTest(); } catch (Exception e1) { Error += "Memory: " + e1.Message + " "; }
            Percent = 80;

            Status = "Disk write test...";
            try { DiskWriteMBs = DiskTest(); } catch (Exception e2) { Error += "Disk: " + e2.Message + " "; }
            Percent = 98;
            Status = "Done";
        }
        catch (Exception ex)
        {
            Error += ex.Message;
            Status = "Failed";
        }
        finally
        {
            Percent = 100;
            Running = false;
            Done = true;
        }
    }
}
'@
}

# =============== backup helpers ===============
function Get-Bk {
    $h = @{}
    if (Test-Path $script:BackupFile) {
        try {
            $o = Get-Content $script:BackupFile -Raw | ConvertFrom-Json
            if ($o) { $o.PSObject.Properties | ForEach-Object { $h[$_.Name] = $_.Value } }
        } catch { }
    }
    return $h
}
function Save-Bk($h) { $h | ConvertTo-Json | Set-Content -Path $script:BackupFile -Encoding UTF8 }

function Test-Has($prefixes) {
    $bk = Get-Bk
    foreach ($k in $bk.Keys) { foreach ($p in $prefixes) { if ($k.StartsWith($p)) { return $true } } }
    return $false
}

function Set-Reg {
    param($Path, $Name, $Value, $Type = 'DWord')
    if ([string]$Path -notmatch '^HK(LM|CU):') { throw "Bad registry path: $Path" }
    $bk  = Get-Bk
    $key = "$Path|$Name"
    $needSave = $false
    if (-not $bk.ContainsKey($key)) {
        $old = $null
        try { $old = (Get-ItemProperty -Path $Path -Name $Name -ErrorAction Stop).$Name } catch { }
        if ($null -eq $old) { $bk[$key] = '<absent>' } else { $bk[$key] = $old }
        $needSave = $true
    }
    if (-not (Test-Path $Path)) { New-Item -Path $Path -Force -ErrorAction Stop | Out-Null }
    New-ItemProperty -Path $Path -Name $Name -Value $Value -PropertyType $Type -Force -ErrorAction Stop | Out-Null
    if ($needSave) { Save-Bk $bk }
}

function Restore-Key($k) {
    $bk = Get-Bk
    if (-not $bk.ContainsKey($k)) { return }
    $old = $bk[$k]
    if ($k -eq 'powerplan') {
        $null = powercfg /setactive $old 2>&1
    } elseif ($k -eq 'createdplan') {
        $null = powercfg /delete $old 2>&1
    } elseif ($k.StartsWith('defender|')) {
        try { Remove-MpPreference -ExclusionPath $k.Substring(9) -ErrorAction Stop } catch { }
    } else {
        $parts = $k -split '\|', 2
        $path = $parts[0]; $name = $parts[1]
        if ($old -eq '<absent>') {
            Remove-ItemProperty -Path $path -Name $name -ErrorAction SilentlyContinue
        } elseif ($old -is [string]) {
            if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
            New-ItemProperty -Path $path -Name $name -Value $old -PropertyType String -Force | Out-Null
        } else {
            if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
            New-ItemProperty -Path $path -Name $name -Value ([int]$old) -PropertyType DWord -Force | Out-Null
        }
    }
    $bk = Get-Bk
    $bk.Remove($k)
    Save-Bk $bk
}

function Restore-Prefix($prefixes) {
    foreach ($k in @((Get-Bk).Keys)) {
        foreach ($p in $prefixes) { if ($k.StartsWith($p)) { Restore-Key $k; break } }
    }
}

function Log($m) {
    if ($script:ui -and $script:ui.txtLog) {
        $script:ui.txtLog.AppendText(("[{0}] {1}`r`n" -f (Get-Date -Format 'HH:mm:ss'), $m))
        $script:ui.txtLog.ScrollToEnd()
    }
}

# =============== custom tweaks ===============

# ---- power plan ----
function Tune-Plan($guid) {
    $sub = '54533251-82be-4824-96c1-47b60b740d00'
    $null = powercfg /setacvalueindex $guid $sub 893dee8e-2bef-41e0-89c6-b55d0929964c 100 2>&1   # min CPU 100%
    $null = powercfg /setacvalueindex $guid $sub be337238-0d82-4146-a960-4f3749d470c7 2 2>&1     # boost: aggressive
    $null = powercfg /setacvalueindex $guid 2a737441-1930-4402-8d77-b2bebba308a3 48e6b7a6-50f5-4782-a5d4-53bb8f07e226 0 2>&1  # USB selective suspend off
    $null = powercfg /setacvalueindex $guid 501a4d13-42af-4429-9fd1-a8218c268e20 ee12f906-d277-404b-b6da-e5fa1a576df5 0 2>&1  # PCIe link power saving off
    $null = powercfg /setactive $guid 2>&1
}

function Apply-Power {
    $bk = Get-Bk
    if (-not $bk.ContainsKey('powerplan')) {
        $line = powercfg /getactivescheme | Out-String
        if ($line -match '([0-9a-fA-F\-]{36})') { $bk['powerplan'] = $Matches[1]; Save-Bk $bk }
    }
    $bk = Get-Bk
    if ($bk.ContainsKey('createdplan')) {
        $listing = powercfg /list | Out-String
        if ($listing -match [regex]::Escape([string]$bk['createdplan'])) {
            Tune-Plan $bk['createdplan']
            Log 'Performance power plan active.'
            return
        }
    }
    $new = $null
    foreach ($src in @('e9a42b02-d5df-448d-aa00-03f14749eb61', '8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c')) {
        $out = powercfg -duplicatescheme $src 2>&1 | Out-String
        if ($out -match '([0-9a-fA-F\-]{36})') { $new = $Matches[1]; break }
    }
    if ($new) {
        $bk = Get-Bk; $bk['createdplan'] = $new; Save-Bk $bk
        Tune-Plan $new
        Log 'Performance power plan active: CPU at full speed, aggressive boost, USB and PCIe power saving off (when plugged in).'
    } else {
        $bk = Get-Bk
        if (-not $bk.ContainsKey('createdplan')) { $bk.Remove('powerplan'); Save-Bk $bk }
        Log 'Could not create a performance plan. Set it in Settings > System > Power.'
    }
}
function Revert-Power {
    Restore-Key 'powerplan'
    Restore-Key 'createdplan'
    Log 'Power plan restored.'
}

# ---- find games ----
$script:GpuPath = 'HKCU:\Software\Microsoft\DirectX\UserGpuPreferences'
$script:ExeName = @{ Valorant = 'VALORANT-Win64-Shipping.exe'; Fortnite = 'FortniteClient-Win64-Shipping.exe' }

function Find-Game($game) {
    $rels = @()
    if ($game -eq 'Fortnite') {
        $dir = Join-Path $env:ProgramData 'Epic\EpicGamesLauncher\Data\Manifests'
        if (Test-Path $dir) {
            foreach ($f in (Get-ChildItem $dir -Filter *.item -ErrorAction SilentlyContinue)) {
                try { $m = Get-Content $f.FullName -Raw | ConvertFrom-Json } catch { continue }
                if ($m.AppName -eq 'Fortnite' -or $m.DisplayName -eq 'Fortnite') {
                    $p = Join-Path $m.InstallLocation 'FortniteGame\Binaries\Win64\FortniteClient-Win64-Shipping.exe'
                    if (Test-Path $p) { return $p }
                }
            }
        }
        $rels = @(
            'Program Files\Epic Games\Fortnite\FortniteGame\Binaries\Win64\FortniteClient-Win64-Shipping.exe',
            'Epic Games\Fortnite\FortniteGame\Binaries\Win64\FortniteClient-Win64-Shipping.exe',
            'Games\Epic Games\Fortnite\FortniteGame\Binaries\Win64\FortniteClient-Win64-Shipping.exe')
    } else {
        $j = Join-Path $env:ProgramData 'Riot Games\RiotClientInstalls.json'
        if (Test-Path $j) {
            try {
                $o = Get-Content $j -Raw | ConvertFrom-Json
                $rc = $o.rc_live; if (-not $rc) { $rc = $o.rc_default }
                if ($rc) {
                    $root = Split-Path (Split-Path ($rc -replace '/', '\'))
                    $p = Join-Path $root 'VALORANT\live\ShooterGame\Binaries\Win64\VALORANT-Win64-Shipping.exe'
                    if (Test-Path $p) { return $p }
                }
            } catch { }
        }
        $rels = @(
            'Riot Games\VALORANT\live\ShooterGame\Binaries\Win64\VALORANT-Win64-Shipping.exe',
            'Program Files\Riot Games\VALORANT\live\ShooterGame\Binaries\Win64\VALORANT-Win64-Shipping.exe')
    }
    foreach ($d in (Get-PSDrive -PSProvider FileSystem | Where-Object { $_.Root -match '^[A-Za-z]:\\$' })) {
        foreach ($r in $rels) {
            $p = Join-Path $d.Root $r
            if (Test-Path $p) { return $p }
        }
    }
    return $null
}

# ---- force games onto the GPU ----
function Apply-Gpu {
    foreach ($g in @('Valorant', 'Fortnite')) {
        $exe = Find-Game $g
        if (-not $exe) {
            $name = $script:ExeName[$g]
            $r = [System.Windows.MessageBox]::Show("Couldn't find $g automatically.`n`nDo you want to locate it yourself? Look for:`n$name", $script:AppName, 'YesNo', 'Question')
            if ($r -eq 'Yes') {
                $dlg = New-Object System.Windows.Forms.OpenFileDialog
                $dlg.Title  = "Locate $name"
                $dlg.Filter = "$name|$name|All programs|*.exe"
                if ($dlg.ShowDialog() -eq 'OK') { $exe = $dlg.FileName }
            }
        }
        if ($exe) {
            Set-Reg -Path $script:GpuPath -Name $exe -Value 'GpuPreference=2;' -Type 'String'
            Log "$g set to run on the dedicated GPU: $exe"
        } else {
            Log "$g skipped (not found)."
        }
    }
    foreach ($p in @($script:CustomGames)) {
        if ($p -and (Test-Path $p)) {
            Set-Reg -Path $script:GpuPath -Name $p -Value 'GpuPreference=2;' -Type 'String'
            Log "Added game set to run on the dedicated GPU: $p"
        }
    }
    Log 'Restart the games for this to take effect.'
}
function Revert-Gpu {
    Restore-Prefix @("$($script:GpuPath)|")
    Log 'Game GPU preferences restored.'
}

# ---- Defender exclusions for game folders ----
function Apply-Defender {
    if (-not (Get-Command Add-MpPreference -ErrorAction SilentlyContinue)) { Log 'Windows Defender commands are not available (another antivirus may be active).'; return }
    $added = 0
    foreach ($g in @('Valorant', 'Fortnite')) {
        $exe = Find-Game $g
        if (-not $exe) { continue }
        if ($g -eq 'Valorant') { $dir = $exe -replace '\\ShooterGame\\.*$', '' } else { $dir = $exe -replace '\\FortniteGame\\.*$', '' }
        $existing = @((Get-MpPreference).ExclusionPath)
        if ($existing -contains $dir) { Log "Already excluded: $dir"; continue }
        Add-MpPreference -ExclusionPath $dir -ErrorAction Stop
        $bk = Get-Bk; $bk["defender|$dir"] = 'excluded'; Save-Bk $bk
        Log "Defender will skip real-time scanning of: $dir"
        $added++
    }
    if ($added -eq 0) { Log 'No game folders found (or already excluded).' }
}
function Revert-Defender {
    Restore-Prefix @('defender|')
    Log 'Defender exclusions removed.'
}

# ---- sync mouse after change ----
function Sync-Mouse {
    try {
        $m = Get-ItemProperty 'HKCU:\Control Panel\Mouse'
        $vals = [int[]]@([int]$m.MouseThreshold1, [int]$m.MouseThreshold2, [int]$m.MouseSpeed)
        [void][NativeMouse]::SystemParametersInfo(4, 0, $vals, 3)
    } catch { }
}

# ---- actions ----
function Apply-Cleanup {
    $dirs = @($env:TEMP, "$env:WINDIR\Temp") | Select-Object -Unique
    $cutoff = (Get-Date).AddDays(-1)
    $total = 0
    foreach ($d in $dirs) {
        if (Test-Path $d) {
            $s = (Get-ChildItem $d -Recurse -Force -File -ErrorAction SilentlyContinue | Where-Object { $_.LastWriteTime -lt $cutoff } | Measure-Object Length -Sum).Sum
            if ($s) { $total += $s }
        }
    }
    $mb = [math]::Round($total / 1MB)
    $r = [System.Windows.MessageBox]::Show("Delete about $mb MB of temp files older than 1 day?`n`nThis cannot be undone.", $script:AppName, 'YesNo', 'Question')
    if ($r -ne 'Yes') { Log 'Cleanup cancelled.'; return }
    foreach ($d in $dirs) {
        Get-ChildItem $d -Recurse -Force -File -ErrorAction SilentlyContinue | Where-Object { $_.LastWriteTime -lt $cutoff } |
            Remove-Item -Force -ErrorAction SilentlyContinue
    }
    Log "Cleaned about $mb MB of temp files (files in use were skipped)."
}

function Apply-Startup {
    Start-Process taskmgr.exe
    Log 'Task Manager opened: go to the Startup apps tab, right-click launchers/chat apps you do not need at boot, and choose Disable.'
}

function Apply-Dns {
    $null = ipconfig /flushdns 2>&1
    Log 'DNS cache flushed.'
}

function Get-GpuVendors {
    $names = @()
    try { $names = @(Get-CimInstance Win32_VideoController | ForEach-Object { $_.Name }) } catch { }
    $v = @()
    if ($names -match 'NVIDIA|GeForce') { $v += 'NVIDIA' }
    if ($names -match 'AMD|Radeon') { $v += 'AMD' }
    if ($names -match 'Intel') { $v += 'Intel' }
    return $v
}

function Apply-Guide {
    $v = @(Get-GpuVendors)
    $sb = New-Object System.Text.StringBuilder
    if ($script:Mode -ne 'PC') {
        [void]$sb.AppendLine('LAPTOP PERFORMANCE MODE')
        [void]$sb.AppendLine('- Use your laptop performance/Turbo profile while plugged in (ASUS: Armoury Crate > Turbo).')
        [void]$sb.AppendLine('')
    }
    if ($v -contains 'NVIDIA') {
        [void]$sb.AppendLine('NVIDIA CONTROL PANEL > Manage 3D settings > Global')
        [void]$sb.AppendLine('- Power management mode: Prefer maximum performance')
        [void]$sb.AppendLine('- Low Latency Mode: On (Ultra if the game has no Reflex)')
        [void]$sb.AppendLine('- Texture filtering - Quality: High performance')
        [void]$sb.AppendLine('- Vertical sync: Off')
        [void]$sb.AppendLine('')
    }
    if ($v -contains 'AMD') {
        [void]$sb.AppendLine('AMD ADRENALIN > Gaming > Graphics')
        [void]$sb.AppendLine('- Radeon Anti-Lag: On')
        [void]$sb.AppendLine('- Radeon Chill: Off, Enhanced Sync: Off')
        [void]$sb.AppendLine('- Wait for Vertical Refresh: Off')
        [void]$sb.AppendLine('')
    }
    if ($v -contains 'Intel') {
        [void]$sb.AppendLine('INTEL GRAPHICS COMMAND CENTER')
        [void]$sb.AppendLine('- Set power to Maximum performance and turn V-Sync off for games.')
        [void]$sb.AppendLine('')
    }
    [void]$sb.AppendLine('VALORANT (Settings > Video)')
    [void]$sb.AppendLine('- Display mode: Fullscreen, Multithreaded Rendering: On')
    [void]$sb.AppendLine('- NVIDIA Reflex Low Latency: On + Boost (NVIDIA cards only)')
    [void]$sb.AppendLine('- V-Sync: Off, Limit FPS always: off (or above your refresh rate)')
    [void]$sb.AppendLine('- Material / Texture / Detail quality: Low to Medium')
    [void]$sb.AppendLine('- Vignette, Bloom, Distortion, Cast Shadows: Off')
    [void]$sb.AppendLine('')
    [void]$sb.AppendLine('FORTNITE (Settings > Video)')
    [void]$sb.AppendLine('- Rendering mode: Performance (best FPS) or DirectX 12 for visuals')
    [void]$sb.AppendLine('- NVIDIA Reflex: On + Boost (NVIDIA cards only)')
    [void]$sb.AppendLine('- Shadows, Global Illumination, Reflections: Off; Effects, Post Processing: Low')
    [void]$sb.AppendLine('- If you use upscaling (DLSS / FSR): Quality or Balanced')
    [void]$sb.AppendLine('')
    if ($script:Mode -ne 'PC') { [void]$sb.AppendLine('Keep the laptop on a hard, ventilated surface and plugged in.') }
    else { [void]$sb.AppendLine('Make sure the case has good airflow and the monitor is plugged into the graphics card, not the motherboard.') }
    [System.Windows.MessageBox]::Show($sb.ToString(), 'Recommended game settings', 'OK', 'Information') | Out-Null
    Log 'Opened recommended game settings.'
}

function Apply-CloseApps {
    $protect = @('explorer','dwm','powershell','pwsh','WindowsTerminal','conhost','ShellExperienceHost','SearchHost','StartMenuExperienceHost','TextInputHost','VALORANT-Win64-Shipping','FortniteClient-Win64-Shipping','RiotClientServices','EpicGamesLauncher','LockApp','ApplicationFrameHost','SystemSettings')
    $procs = Get-Process | Where-Object { $_.MainWindowHandle -ne 0 -and $protect -notcontains $_.ProcessName -and $_.Id -ne $PID } |
        Sort-Object WorkingSet64 -Descending | Select-Object -First 20
    if (-not $procs) { Log 'No closable apps found.'; return }
    [xml]$dx = @'
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Close background apps" Width="560" Height="520" WindowStartupLocation="CenterOwner"
        Background="#0B0B0D" Foreground="#F2F2F5" FontFamily="Segoe UI">
  <Grid Margin="18">
    <Grid.RowDefinitions>
      <RowDefinition Height="Auto"/>
      <RowDefinition Height="*"/>
      <RowDefinition Height="Auto"/>
    </Grid.RowDefinitions>
    <TextBlock Text="Tick the apps to close (biggest RAM users first). They may ask to save first." TextWrapping="Wrap" Foreground="#8E8E99" Margin="0,0,0,10"/>
    <ScrollViewer Grid.Row="1" VerticalScrollBarVisibility="Auto"><StackPanel x:Name="panel"/></ScrollViewer>
    <Button Grid.Row="2" x:Name="btnClose" Content="Close selected" Margin="0,12,0,0" HorizontalAlignment="Right"/>
  </Grid>
</Window>
'@
    $script:dlg = [Windows.Markup.XamlReader]::Load((New-Object System.Xml.XmlNodeReader $dx))
    $script:dlg.Owner = $script:Window
    $script:dlgPanel = $script:dlg.FindName('panel')
    $btn = $script:dlg.FindName('btnClose')
    $btn.Style = $script:Window.FindResource('BtnAccent')
    foreach ($p in $procs) {
        $title = $p.MainWindowTitle
        if ($title.Length -gt 40) { $title = $title.Substring(0, 40) }
        $cb = New-Object System.Windows.Controls.CheckBox
        $cb.Content = ("{0}  -  {1} MB  -  {2}" -f $p.ProcessName, [math]::Round($p.WorkingSet64 / 1MB), $title)
        $cb.Tag = $p.Id
        $cb.Foreground = [System.Windows.Media.Brushes]::White
        $cb.Margin = New-Object System.Windows.Thickness(0, 5, 0, 5)
        [void]$script:dlgPanel.Children.Add($cb)
    }
    $btn.Add_Click({
        foreach ($c in $script:dlgPanel.Children) {
            if ($c.IsChecked) {
                try {
                    $pr = Get-Process -Id $c.Tag -ErrorAction Stop
                    [void]$pr.CloseMainWindow()
                    Log "Asked $($pr.ProcessName) to close."
                } catch { }
            }
        }
        $script:dlg.Close()
    })
    [void]$script:dlg.ShowDialog()
}

# =============== game boost (fans, priority, timer) ===============
$script:FanFile    = Join-Path $script:BackupDir 'fan.json'
$script:SetFile    = Join-Path $script:BackupDir 'settings.json'
$script:ThermalId  = 0x00120075
$script:FanNames   = @{ 0 = 'Balanced'; 1 = 'Turbo'; 2 = 'Silent' }
$script:GameActive = $false
$script:GameMiss   = 0
$script:TimerHeld  = $false
$script:PrioDone   = @{}
$script:LayersPath = 'HKCU:\Software\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Layers'

function Get-FanMode {
    try { $v = [AsusAcpi]::Get([uint32]$script:ThermalId) } catch { return $null }
    if ($v -lt 0 -or $v -gt 2) { return $null }
    return [int]$v
}

function Set-FanMode($m) {
    try { [void][AsusAcpi]::Set([uint32]$script:ThermalId, [int]$m) } catch { return $false }
    Start-Sleep -Milliseconds 400
    return ((Get-FanMode) -eq [int]$m)
}

function Save-FanState($prev) { @{ prev = [int]$prev } | ConvertTo-Json | Set-Content -Path $script:FanFile -Encoding UTF8 }
function Get-FanState {
    if (-not (Test-Path $script:FanFile)) { return $null }
    try { return (Get-Content $script:FanFile -Raw | ConvertFrom-Json).prev } catch { return $null }
}
function Clear-FanState { Remove-Item $script:FanFile -Force -ErrorAction SilentlyContinue }

function Restore-Fan {
    $prev = Get-FanState
    if ($null -eq $prev) { return $false }
    if (Set-FanMode ([int]$prev)) {
        Clear-FanState
        Log ("Fans: back to {0} mode." -f $script:FanNames[[int]$prev])
        return $true
    }
    Log 'Fans: could not restore the previous mode automatically. Set it in Armoury Crate.'
    return $false
}

function Get-GameNames {
    $raw = ''
    if ($script:ui -and $script:ui.txtGames) { $raw = [string]$script:ui.txtGames.Text }
    return @($raw -split '[,;\s]+' | ForEach-Object { ($_.Trim() -replace '(?i)\.exe$', '') } | Where-Object { $_ })
}

function Save-GameSettings {
    try {
        [pscustomobject]@{
            fan   = [bool]$script:ui.chkFan.IsChecked
            prio  = [bool]$script:ui.chkPrio.IsChecked
            timer = [bool]$script:ui.chkTimer.IsChecked
            mode  = [string]$script:Mode
            custom = @($script:CustomGames)
            games = [string]$script:ui.txtGames.Text
        } | ConvertTo-Json | Set-Content -Path $script:SetFile -Encoding UTF8
    } catch { }
}

function Load-GameSettings {
    $script:ui.txtGames.Text = 'VALORANT-Win64-Shipping, FortniteClient-Win64-Shipping'
    if (Test-Path $script:SetFile) {
        try {
            $o = Get-Content $script:SetFile -Raw | ConvertFrom-Json
            if ($o.mode) { $script:Mode = [string]$o.mode }
            if ($o.custom) { $script:CustomGames = @($o.custom) }
            $script:ui.chkFan.IsChecked   = [bool]$o.fan
            $script:ui.chkPrio.IsChecked  = [bool]$o.prio
            $script:ui.chkTimer.IsChecked = [bool]$o.timer
            if ($o.games) { $script:ui.txtGames.Text = [string]$o.games }
        } catch { }
    }
}

function Set-GamePriority($procs) {
    foreach ($p in $procs) {
        if ($script:PrioDone.ContainsKey($p.Id)) { continue }
        $script:PrioDone[$p.Id] = $true
        try {
            $p.PriorityClass = [System.Diagnostics.ProcessPriorityClass]::High
            Log ("{0}: CPU priority set to High." -f $p.ProcessName)
        } catch {
            Log ("{0}: could not change priority (the anti-cheat may block it). Skipped." -f $p.ProcessName)
        }
    }
}

function Start-GameBoost($procs) {
    $script:GameActive = $true
    $script:GameMiss = 0
    Log ("Game detected: {0}" -f ((@($procs | ForEach-Object { $_.ProcessName } | Select-Object -Unique)) -join ', '))
    if ($script:ui.chkFan.IsChecked) {
        $cur = Get-FanMode
        if ($null -eq $cur) {
            Log 'Fans: could not reach the ASUS fan interface (it needs the ASUS system driver that comes with Armoury Crate). Fans left alone.'
        } else {
            $saved = Get-FanState
            if ($null -eq $saved -and $cur -ne 1) { Save-FanState $cur }
            if ($cur -ne 1) {
                if (Set-FanMode 1) { Log 'Fans: Turbo mode on for this game session.' }
                else {
                    Log 'Fans: the laptop did not accept Turbo mode. Set Turbo in Armoury Crate instead.'
                    if ($null -eq $saved) { Clear-FanState }
                }
            } else { Log 'Fans: already on Turbo.' }
        }
    }
    if ($script:ui.chkTimer.IsChecked) {
        try { [WinTimer]::Hold(); $script:TimerHeld = $true; Log 'Timer: 0.5 ms system timer requested.' } catch { }
    }
    if ($script:ui.chkPrio.IsChecked) { Set-GamePriority $procs }
}

function Stop-GameBoost {
    if ($script:TimerHeld) { try { [WinTimer]::Release() } catch { }; $script:TimerHeld = $false }
    [void](Restore-Fan)
    $script:PrioDone = @{}
    if ($script:GameActive) { Log 'Game closed: boosts removed.' }
    $script:GameActive = $false
    $script:GameMiss = 0
}

function Update-GameBoost {
    $names = @(Get-GameNames)
    $procs = @()
    if ($names.Count -gt 0 -and ($script:ui.chkFan.IsChecked -or $script:ui.chkPrio.IsChecked -or $script:ui.chkTimer.IsChecked)) {
        $procs = @(Get-Process -Name $names -ErrorAction SilentlyContinue)
    }
    if ($procs.Count -gt 0) {
        $script:GameMiss = 0
        if (-not $script:GameActive) { Start-GameBoost $procs }
        elseif ($script:ui.chkPrio.IsChecked) { Set-GamePriority $procs }
    } elseif ($script:GameActive) {
        $script:GameMiss++
        if ($script:GameMiss -ge 2) { Stop-GameBoost }
    }
    $fm = Get-FanMode
    $fmTxt = 'unavailable'
    if ($null -ne $fm) { $fmTxt = $script:FanNames[$fm] }
    $state = 'waiting for a game'
    if ($script:GameActive) { $state = 'GAME RUNNING - boosts on' }
    if (-not $script:FanOk) { $script:ui.gameStatus.Text = ("Status: {0}" -f $state) }
    else { $script:ui.gameStatus.Text = ("Status: {0}   |   Laptop fan mode: {1}" -f $state, $fmTxt) }
}

# ---- FPS tab tweaks ----
function Apply-Fso {
    foreach ($g in @('Valorant', 'Fortnite')) {
        $exe = Find-Game $g
        if (-not $exe) { Log "$g skipped (not found)."; continue }
        $cur = ''
        try { $cur = [string](Get-ItemProperty -Path $script:LayersPath -Name $exe -ErrorAction Stop).$exe } catch { }
        if ($cur -match 'DISABLEDXMAXIMIZEDWINDOWEDMODE') { Log "$g already has fullscreen optimizations off."; continue }
        $new = ($cur.Trim() + ' DISABLEDXMAXIMIZEDWINDOWEDMODE').Trim()
        if (-not $new.StartsWith('~')) { $new = '~ ' + $new }
        Set-Reg -Path $script:LayersPath -Name $exe -Value $new -Type 'String'
        Log "Fullscreen optimizations off for $g."
    }
    foreach ($p in @($script:CustomGames)) {
        if (-not ($p -and (Test-Path $p))) { continue }
        $cur = ''
        try { $cur = [string](Get-ItemProperty -Path $script:LayersPath -Name $p -ErrorAction Stop).$p } catch { }
        if ($cur -match 'DISABLEDXMAXIMIZEDWINDOWEDMODE') { continue }
        $new = ($cur.Trim() + ' DISABLEDXMAXIMIZEDWINDOWEDMODE').Trim()
        if (-not $new.StartsWith('~')) { $new = '~ ' + $new }
        Set-Reg -Path $script:LayersPath -Name $p -Value $new -Type 'String'
        Log "Fullscreen optimizations off for: $p"
    }
    Log 'Restart the games for this to take effect. If a game feels worse, press Undo.'
}
function Revert-Fso {
    Restore-Prefix @("$($script:LayersPath)|")
    Log 'Fullscreen optimization settings restored.'
}

function Apply-FpsGuide {
    $txt = @'
GRAPHICS DRIVER SETTINGS
- NVIDIA (Control Panel > Manage 3D settings): Shader Cache Size Unlimited, Power management mode Prefer maximum performance.
- AMD (Adrenalin > Gaming > Graphics): Radeon Anti-Lag on, Radeon Chill off, Enhanced Sync off.
- Intel (Graphics Command Center): set power to Maximum performance.
- Max frame rate: leave uncapped in Valorant. In Fortnite, cap it a few FPS under what you can hold steadily: a steady FPS gives better 1% lows than a swinging one.

VRAM
- If texture quality is higher than your graphics card has memory for (many laptop cards have only 4 GB), the game spills into system RAM and 1% lows collapse. Keep textures Low to Medium on 4 GB cards.

OVERLAYS AND BACKGROUND
- Turn off the Discord overlay, GeForce/Radeon overlay, Steam overlay and RGB sync software while gaming. Overlays are a common cause of stutter.
- Close Chrome and other heavy apps (use "Close background apps" on the Optimizations tab).

FORTNITE
- Rendering mode: Performance. After each update, let the game finish compiling shaders before judging stutter.
- View distance: Medium or Far. Shadows, Global Illumination, Reflections: Off.

VALORANT
- Multithreaded Rendering: On. Limit FPS always: Off. NVIDIA Reflex: On + Boost (NVIDIA cards only).

HEAT
- Above about 85 C the GPU slows down and 1% lows drop. On a laptop use a flat hard surface or a cooling stand; on a desktop check case airflow and fan curves. Keep dust out of the vents.

MEASURING
- The Game session recorder shows GPU usage, CPU and RAM (and temperatures and throttling on NVIDIA cards). For actual average FPS and 1% lows, the free tool CapFrameX shows both.
'@
    [System.Windows.MessageBox]::Show($txt, 'FPS and 1% lows tips', 'OK', 'Information') | Out-Null
    Log 'Opened FPS and 1% lows tips.'
}

# =============== tweak table ===============
$D = 'DWord'; $S = 'String'
$script:Tweaks = @(
    @{ Id='power'; Group='Recommended'; Title='Performance power plan'; Label='Apply'; CanRevert=$true; Recommended=$true; Keys=@('powerplan');
       Desc='Ultimate/High Performance plan, CPU always at full speed with aggressive boost, USB and PCIe power saving off. Keep the charger plugged in.';
       Apply={ Apply-Power }; Revert={ Revert-Power } },
    @{ Id='gpu'; LaptopOnly=$true; Group='Recommended'; Title='Force games onto the dedicated GPU (laptops)'; Label='Apply'; CanRevert=$true; Recommended=$true; Keys=@("$($script:GpuPath)|");
       Desc='Makes Valorant, Fortnite and any game you add use the dedicated graphics card instead of the weak built-in graphics. Often the biggest free FPS gain on laptops.';
       Apply={ Apply-Gpu }; Revert={ Revert-Gpu } },
    @{ Id='gamemode'; Group='Recommended'; Title='Game Mode on, background capture and Game Bar off'; Label='Apply'; CanRevert=$true; Recommended=$true;
       Desc='Turns Windows Game Mode on and stops Xbox Game Bar recording and its overlay.';
       Msg='Game Mode on, Game Bar capture and overlay off.';
       Regs=@( @('HKCU:\Software\Microsoft\GameBar','AutoGameModeEnabled',1,$D), @('HKCU:\Software\Microsoft\GameBar','AllowAutoGameMode',1,$D),
               @('HKCU:\Software\Microsoft\GameBar','ShowStartupPanel',0,$D), @('HKCU:\Software\Microsoft\GameBar','UseNexusForGameBarEnabled',0,$D),
               @('HKCU:\System\GameConfigStore','GameDVR_Enabled',0,$D), @('HKCU:\Software\Microsoft\Windows\CurrentVersion\GameDVR','AppCaptureEnabled',0,$D) ) },
    @{ Id='mouse'; Group='Recommended'; Title='Mouse acceleration off'; Label='Apply'; CanRevert=$true; Recommended=$true;
       Desc='Disables Enhance pointer precision so your aim in Valorant is consistent 1:1. Applies instantly.';
       Msg='Mouse acceleration off (raw 1:1 aim).';
       Regs=@( @('HKCU:\Control Panel\Mouse','MouseSpeed','0',$S), @('HKCU:\Control Panel\Mouse','MouseThreshold1','0',$S), @('HKCU:\Control Panel\Mouse','MouseThreshold2','0',$S) );
       After={ Sync-Mouse } },
    @{ Id='mmcss'; Group='Recommended'; Title='Game CPU and GPU scheduling priority'; Label='Apply'; CanRevert=$true; Recommended=$true; Restart=$true;
       Desc='Tells Windows to give games top priority for CPU time and disk I/O, and stops network throttling of multimedia/game traffic.';
       Msg='Game scheduling priority set to High.';
       Regs=@( @('HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games','GPU Priority',8,$D),
               @('HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games','Priority',6,$D),
               @('HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games','Scheduling Category','High',$S),
               @('HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games','SFIO Priority','High',$S),
               @('HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile','SystemResponsiveness',10,$D),
               @('HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile','NetworkThrottlingIndex',-1,$D) ) },
    @{ Id='fgboost'; Group='Recommended'; Title='Foreground app priority boost'; Label='Apply'; CanRevert=$true; Recommended=$true;
       Desc='Gives the app you are using (your game) a bigger share of CPU time slices over background tasks.';
       Msg='Foreground priority boost set.';
       Regs=@( @('HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl','Win32PrioritySeparation',38,$D) ) },
    @{ Id='dopt'; Group='Recommended'; Title='Stop Windows Update peer sharing'; Label='Apply'; CanRevert=$true; Recommended=$true;
       Desc='Stops Windows uploading updates to other PCs in the background, which can eat bandwidth and cause ping spikes.';
       Msg='Update peer sharing off (downloads from Microsoft only).';
       Regs=@( @('HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization','DODownloadMode',0,$D) ) },

    @{ Id='hags'; Group='Advanced'; Title='Hardware-accelerated GPU scheduling'; Label='Apply'; CanRevert=$true; Recommended=$false; Restart=$true;
       Desc='Lets the GPU manage its own memory scheduling. Helps some games and DLSS frame generation, neutral in others. Test with the benchmark/recorder.';
       Msg='GPU scheduling enabled.';
       Regs=@( @('HKLM:\SYSTEM\CurrentControlSet\Control\GraphicsDrivers','HwSchMode',2,$D) ) },
    @{ Id='pthrottle'; Group='Advanced'; Title='Power throttling off'; Label='Apply'; CanRevert=$true; Recommended=$false;
       Desc='Stops Windows slowing down apps it thinks are in the background. Uses more battery, so only worth it when plugged in.';
       Msg='Power throttling off.';
       Regs=@( @('HKLM:\SYSTEM\CurrentControlSet\Control\Power\PowerThrottling','PowerThrottlingOff',1,$D) ) },
    @{ Id='faststart'; Group='Advanced'; Title='Fast Startup off'; Label='Apply'; CanRevert=$true; Recommended=$false; Restart=$true;
       Desc='Makes Shut down a real clean boot, which clears stale driver and RAM state that can cause stutter after days of uptime.';
       Msg='Fast Startup off.';
       Regs=@( @('HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power','HiberbootEnabled',0,$D) ) },
    @{ Id='bgapps'; Group='Advanced'; Title='Background apps off'; Label='Apply'; CanRevert=$true; Recommended=$false;
       Desc='Stops Store apps running in the background. Small gain, frees some RAM and CPU.';
       Msg='Background apps off.';
       Regs=@( @('HKCU:\Software\Microsoft\Windows\CurrentVersion\BackgroundAccessApplications','GlobalUserDisabled',1,$D),
               @('HKCU:\Software\Microsoft\Windows\CurrentVersion\Search','BackgroundAppGlobalToggle',0,$D) ) },
    @{ Id='telemetry'; Group='Advanced'; Title='Telemetry service off'; Label='Apply'; CanRevert=$true; Recommended=$false; Restart=$true;
       Desc='Disables the Connected User Experiences and Telemetry service (DiagTrack). Less background disk and CPU use. Small gain.';
       Msg='Telemetry service set to disabled.';
       Regs=@( @('HKLM:\SYSTEM\CurrentControlSet\Services\DiagTrack','Start',4,$D) );
       After={ if ((Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\DiagTrack' -ErrorAction SilentlyContinue).Start -eq 4) { Stop-Service -Name DiagTrack -Force -ErrorAction SilentlyContinue } else { Start-Service -Name DiagTrack -ErrorAction SilentlyContinue } } },
    @{ Id='ui'; Group='Advanced'; Title='Snappier Windows UI'; Label='Apply'; CanRevert=$true; Recommended=$false;
       Desc='Removes menu delay and window/taskbar animations. Fully applies after you sign out and back in.';
       Msg='UI animations off.';
       Regs=@( @('HKCU:\Control Panel\Desktop','MenuShowDelay','0',$S), @('HKCU:\Control Panel\Desktop\WindowMetrics','MinAnimate','0',$S),
               @('HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced','TaskbarAnimations',0,$D) ) },
    @{ Id='trans'; Group='Advanced'; Title='Windows transparency effects off'; Label='Apply'; CanRevert=$true; Recommended=$false;
       Desc='Saves a little GPU and RAM from the Windows blur effects.';
       Msg='Transparency effects off.';
       Regs=@( @('HKCU:\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize','EnableTransparency',0,$D) ) },
    @{ Id='defender'; Group='Advanced'; Title='Skip Defender scanning of game folders'; Label='Apply'; CanRevert=$true; Recommended=$false; Keys=@('defender|');
       Desc='Excludes the Valorant and Fortnite install folders from real-time scanning to cut load-time stutter. Small security tradeoff, so it is optional.';
       Apply={ Apply-Defender }; Revert={ Revert-Defender } },

    @{ Id='gtimer'; Group='FPS'; Title='Global timer resolution (smoother frame pacing)'; Label='Apply'; CanRevert=$true; Recommended=$false; Restart=$true;
       Desc='Lets the 0.5 ms game timer apply system-wide on Windows 11. Works together with the high-precision timer option above. Can improve 1% lows; costs a little battery.';
       Msg='Global timer resolution enabled.';
       Regs=@( @('HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\kernel','GlobalTimerResolutionRequests',1,$D) ) },
    @{ Id='mpo'; Group='FPS'; Title='Multiplane overlay (MPO) off'; Label='Apply'; CanRevert=$true; Recommended=$false; Restart=$true;
       Desc='Turns off a Windows display feature that causes micro-stutter and flicker on some NVIDIA setups. Worth testing if your 1% lows are poor.';
       Msg='MPO off.';
       Regs=@( @('HKLM:\SOFTWARE\Microsoft\Windows\Dwm','OverlayTestMode',5,$D) ) },
    @{ Id='fso'; Group='FPS'; Title='Fullscreen optimizations off for your games'; Label='Apply'; CanRevert=$true; Recommended=$false; Keys=@("$($script:LayersPath)|");
       Desc='Sets the "Disable fullscreen optimizations" flag on Valorant, Fortnite and any game you add. Helps some setups and hurts others, so test it and press Undo if it feels worse.';
       Apply={ Apply-Fso }; Revert={ Revert-Fso } },
    @{ Id='fpsguide'; Group='FPS'; Title='FPS and 1% lows tips'; Label='Show'; CanRevert=$false; Recommended=$false; Keys=@();
       Desc='Shader cache, VRAM, overlays, heat and in-game settings that matter most for steady frame times.';
       Apply={ Apply-FpsGuide } },

    @{ Id='apps'; Group='Tools'; Title='Close background apps'; Label='Choose apps'; CanRevert=$false; Recommended=$false; Keys=@();
       Desc='Shows your biggest open apps so you can free RAM before gaming.';
       Apply={ Apply-CloseApps } },
    @{ Id='clean'; Group='Tools'; Title='Clean temp files'; Label='Clean'; CanRevert=$false; Recommended=$false; Keys=@();
       Desc='Deletes temp files older than a day. Asks before deleting. Cannot be undone.';
       Apply={ Apply-Cleanup } },
    @{ Id='dns'; Group='Tools'; Title='Flush DNS cache'; Label='Flush'; CanRevert=$false; Recommended=$false; Keys=@();
       Desc='Clears the DNS cache. Handy if matchmaking or the launcher has connection trouble.';
       Apply={ Apply-Dns } },
    @{ Id='startup'; Group='Tools'; Title='Trim startup programs'; Label='Open'; CanRevert=$false; Recommended=$false; Keys=@();
       Desc='Opens Task Manager on startup apps so you can disable launchers and chat apps you do not need at boot.';
       Apply={ Apply-Startup } },
    @{ Id='guide'; Group='Tools'; Title='Recommended in-game and NVIDIA settings'; Label='Show'; CanRevert=$false; Recommended=$false; Keys=@();
       Desc='Settings that cannot be changed safely from outside the games: Armoury Crate, NVIDIA Control Panel, Valorant and Fortnite video options.';
       Apply={ Apply-Guide } }
)

# derive Keys for generic registry tweaks
foreach ($t in $script:Tweaks) {
    if ($t.Regs) {
        # a tweak with a single registry entry gets flattened by PowerShell; wrap it back into a list of entries
        if ($t.Regs[0] -isnot [array]) { $t.Regs = @(, $t.Regs) }
        $t.Keys = @($t.Regs | ForEach-Object { "{0}|{1}" -f $_[0], $_[1] })
    }
}

# =============== window ===============
[xml]$xaml = @'
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Lolo's Tweaks" Width="1120" Height="820" WindowStartupLocation="CenterScreen"
        Background="#0B0B0D" Foreground="#F2F2F5" FontFamily="Segoe UI">
  <Window.Resources>
    <Style x:Key="Btn" TargetType="Button">
      <Setter Property="Foreground" Value="White"/>
      <Setter Property="Background" Value="#2A2A32"/>
      <Setter Property="Padding" Value="14,7"/>
      <Setter Property="Cursor" Value="Hand"/>
      <Setter Property="BorderThickness" Value="0"/>
      <Setter Property="Template">
        <Setter.Value>
          <ControlTemplate TargetType="Button">
            <Border x:Name="bd" Background="{TemplateBinding Background}" CornerRadius="8" Padding="{TemplateBinding Padding}">
              <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
            </Border>
            <ControlTemplate.Triggers>
              <Trigger Property="IsMouseOver" Value="True"><Setter TargetName="bd" Property="Opacity" Value="0.85"/></Trigger>
              <Trigger Property="IsEnabled" Value="False"><Setter TargetName="bd" Property="Opacity" Value="0.4"/></Trigger>
            </ControlTemplate.Triggers>
          </ControlTemplate>
        </Setter.Value>
      </Setter>
    </Style>
    <Style x:Key="BtnAccent" TargetType="Button" BasedOn="{StaticResource Btn}">
      <Setter Property="Background" Value="#8B5CF6"/>
    </Style>
    <Style x:Key="BtnWhite" TargetType="Button" BasedOn="{StaticResource Btn}">
      <Setter Property="Background" Value="#F2F2F5"/>
      <Setter Property="Foreground" Value="#0B0B0D"/>
      <Setter Property="FontWeight" Value="SemiBold"/>
    </Style>
  </Window.Resources>
  <Grid Margin="22">
    <Grid.RowDefinitions>
      <RowDefinition Height="Auto"/>
      <RowDefinition Height="Auto"/>
      <RowDefinition Height="Auto"/>
      <RowDefinition Height="*"/>
      <RowDefinition Height="140"/>
    </Grid.RowDefinitions>
    <Grid Grid.Row="0" Margin="0,0,0,14">
      <Grid.ColumnDefinitions>
        <ColumnDefinition Width="*"/>
        <ColumnDefinition Width="Auto"/>
      </Grid.ColumnDefinitions>
      <StackPanel>
        <StackPanel Orientation="Horizontal">
          <TextBlock Text="Lolo's Tweaks" FontSize="28" FontWeight="Bold"/>
          <Border Background="#2A1F4D" BorderBrush="#8B5CF6" BorderThickness="1" CornerRadius="10" Padding="9,2" Margin="12,0,0,0" VerticalAlignment="Center">
            <TextBlock Text="mainly made for laptops" FontSize="11" Foreground="#C9B6FF"/>
          </Border>
          <Button x:Name="btnMode" Content="Laptop mode" Style="{StaticResource Btn}" Padding="10,3" FontSize="11" Margin="10,0,0,0" VerticalAlignment="Center"/>
        </StackPanel>
        <TextBlock x:Name="subtitle" Foreground="#8E8E99" FontSize="12" Margin="0,2,0,0"/>
      </StackPanel>
      <StackPanel Grid.Column="1" Orientation="Horizontal" VerticalAlignment="Center">
        <Button x:Name="btnReport" Content="Report" Style="{StaticResource Btn}" Margin="0,0,8,0"/>
        <Button x:Name="btnShortcut" Content="Shortcut" Style="{StaticResource Btn}" Margin="0,0,8,0"/>
        <Button x:Name="btnUndoAll" Content="Undo all" Style="{StaticResource Btn}" Margin="0,0,8,0"/>
        <Button x:Name="btnAll" Content="Apply recommended" Style="{StaticResource BtnAccent}" Margin="0,0,8,0"/>
        <Button x:Name="btnEverything" Content="Apply ALL" Style="{StaticResource BtnWhite}"/>
      </StackPanel>
    </Grid>
    <Border Grid.Row="1" Background="#17171B" CornerRadius="10" Padding="14,10" Margin="0,0,0,12">
      <TextBlock x:Name="stats" FontFamily="Consolas" FontSize="13" Text="Loading stats..."/>
    </Border>
    <StackPanel Grid.Row="2" Orientation="Horizontal" Margin="0,0,0,12">
      <Button x:Name="btnNavOpt" Content="Optimizations" Style="{StaticResource BtnAccent}" Margin="0,0,8,0"/>
      <Button x:Name="btnNavFps" Content="FPS boost" Style="{StaticResource Btn}" Margin="0,0,8,0"/>
      <Button x:Name="btnNavBench" Content="Benchmark" Style="{StaticResource Btn}" Margin="0,0,8,0"/>
      <Button x:Name="btnNavRec" Content="Game session recorder" Style="{StaticResource Btn}"/>
    </StackPanel>
    <Grid Grid.Row="3">
      <ScrollViewer x:Name="viewOpt" VerticalScrollBarVisibility="Auto">
        <StackPanel x:Name="tweakPanel"/>
      </ScrollViewer>

      <ScrollViewer x:Name="viewFps" Visibility="Collapsed" VerticalScrollBarVisibility="Auto">
        <StackPanel>
          <Border Background="#17171B" CornerRadius="12" Padding="16" Margin="0,0,0,10">
            <StackPanel>
              <TextBlock Text="Game boost (only while a game is running)" FontSize="15" FontWeight="SemiBold"/>
              <TextBlock Text="Lolo's Tweaks watches for your games. When one starts it applies the options you tick below, and when the game closes everything goes back to normal, so nothing stays on at the desktop. Keep Lolo's Tweaks open (minimized is fine). Turbo is the same mode as Armoury Crate Turbo: the strongest fan curve and highest power limits."
                         FontSize="12" Foreground="#8E8E99" TextWrapping="Wrap" Margin="0,4,0,10"/>
              <CheckBox x:Name="chkFan" Content="Fans on max (ASUS laptops only: Turbo mode) while a game is running" Foreground="#F2F2F5" Margin="0,4,0,4"/>
              <CheckBox x:Name="chkPrio" Content="Give the game High CPU priority (helps 1% lows)" Foreground="#F2F2F5" Margin="0,4,0,4"/>
              <CheckBox x:Name="chkTimer" Content="High-precision 0.5 ms system timer (smoother frame pacing)" Foreground="#F2F2F5" Margin="0,4,0,4"/>
              <TextBlock Text="Games to watch (process names, comma separated):" FontSize="12" Foreground="#8E8E99" Margin="0,10,0,4"/>
              <TextBox x:Name="txtGames" Background="#070709" Foreground="#F2F2F5" BorderThickness="0" Padding="8" FontFamily="Consolas" FontSize="12"/>
              <StackPanel Orientation="Horizontal" Margin="0,12,0,0">
                <Button x:Name="btnAddGame" Content="Add a game..." Style="{StaticResource BtnAccent}" Margin="0,0,8,0"/>
                <Button x:Name="btnFanOn" Content="Test: Turbo now" Style="{StaticResource Btn}" Margin="0,0,8,0"/>
                <Button x:Name="btnFanOff" Content="Restore fan mode" Style="{StaticResource Btn}"/>
              </StackPanel>
              <TextBlock x:Name="gameStatus" Text="" FontFamily="Consolas" FontSize="12" Foreground="#8E8E99" Margin="0,10,0,0" TextWrapping="Wrap"/>
            </StackPanel>
          </Border>
          <StackPanel x:Name="fpsPanel"/>
        </StackPanel>
      </ScrollViewer>

      <ScrollViewer x:Name="viewBench" Visibility="Collapsed" VerticalScrollBarVisibility="Auto">
        <StackPanel>
          <Border Background="#17171B" CornerRadius="12" Padding="16" Margin="0,0,0,10">
            <StackPanel>
              <TextBlock Text="Benchmark" FontSize="15" FontWeight="SemiBold"/>
              <TextBlock Text="Tests CPU single-core, CPU all-core (burst and sustained), memory speed and disk write. Close other apps first (on a laptop, plug in the charger). Takes about 30 to 60 seconds. Run it before and after applying tweaks and the results are compared with your previous run."
                         FontSize="12" Foreground="#8E8E99" TextWrapping="Wrap" Margin="0,4,0,0"/>
              <StackPanel Orientation="Horizontal" Margin="0,12,0,0">
                <Button x:Name="btnBench" Content="Run benchmark" Style="{StaticResource BtnAccent}" Margin="0,0,8,0"/>
                <Button x:Name="btnBenchClear" Content="Clear history" Style="{StaticResource Btn}"/>
              </StackPanel>
              <ProgressBar x:Name="benchBar" Height="8" Minimum="0" Maximum="100" Value="0" Margin="0,14,0,0" Foreground="#8B5CF6" Background="#2A2A32" BorderThickness="0"/>
              <TextBlock x:Name="benchStatus" Text="Ready." FontSize="12" Foreground="#8E8E99" Margin="0,6,0,0"/>
            </StackPanel>
          </Border>
          <Border Background="#17171B" CornerRadius="12" Padding="16">
            <TextBlock x:Name="benchResult" FontFamily="Consolas" FontSize="13" TextWrapping="Wrap"/>
          </Border>
        </StackPanel>
      </ScrollViewer>

      <ScrollViewer x:Name="viewRec" Visibility="Collapsed" VerticalScrollBarVisibility="Auto">
        <StackPanel>
          <Border Background="#17171B" CornerRadius="12" Padding="16" Margin="0,0,0,10">
            <StackPanel>
              <TextBlock Text="Game session recorder" FontSize="15" FontWeight="SemiBold"/>
              <TextBlock Text="Press Start, then play your game for a few minutes, alt-tab back and press Stop. The report shows GPU usage, CPU and RAM use and, on NVIDIA cards, temperature, clocks, power and thermal throttling. It also tells you if the game was really running on the dedicated GPU. This is the best check for overheating and stutter."
                         FontSize="12" Foreground="#8E8E99" TextWrapping="Wrap" Margin="0,4,0,0"/>
              <StackPanel Orientation="Horizontal" Margin="0,12,0,0">
                <Button x:Name="btnRec" Content="Start recording" Style="{StaticResource BtnAccent}"/>
              </StackPanel>
              <TextBlock x:Name="recStatus" Text="Not recording." FontFamily="Consolas" FontSize="12" Foreground="#8E8E99" Margin="0,10,0,0"/>
            </StackPanel>
          </Border>
          <Border Background="#17171B" CornerRadius="12" Padding="16">
            <TextBlock x:Name="recResult" FontFamily="Consolas" FontSize="13" TextWrapping="Wrap"/>
          </Border>
        </StackPanel>
      </ScrollViewer>
    </Grid>
    <TextBox Grid.Row="4" x:Name="txtLog" Margin="0,10,0,0" IsReadOnly="True" Background="#070709" Foreground="#B4B4BE"
             BorderThickness="0" FontFamily="Consolas" FontSize="12" Padding="10" TextWrapping="Wrap" VerticalScrollBarVisibility="Auto"/>
  </Grid>
</Window>
'@

try {
    $script:Window = [Windows.Markup.XamlReader]::Load((New-Object System.Xml.XmlNodeReader $xaml))
} catch {
    [System.Windows.MessageBox]::Show("$($script:AppName) could not start: $($_.Exception.Message)", $script:AppName) | Out-Null
    exit
}

$script:ui = @{}
foreach ($n in @('subtitle','btnReport','btnShortcut','btnUndoAll','btnAll','stats','btnNavOpt','btnNavBench','btnNavRec',
                 'viewOpt','viewBench','viewRec','tweakPanel','btnBench','btnBenchClear','benchBar','benchStatus','benchResult',
                 'btnRec','recStatus','recResult','txtLog',
                 'btnNavFps','viewFps','chkFan','chkPrio','chkTimer','txtGames','btnFanOn','btnFanOff','gameStatus','fpsPanel','btnMode','btnEverything','btnAddGame')) {
    $script:ui[$n] = $script:Window.FindName($n)
}

# =============== cards ===============
$script:Cards = @()

function New-Card($t) {
    $safeTitle = [Security.SecurityElement]::Escape($t.Title)
    $safeDesc  = [Security.SecurityElement]::Escape($t.Desc)
    $restartBadge = ''
    $cardXaml = @"
<Border xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" Background="#17171B" CornerRadius="12" Padding="16" Margin="0,0,0,10">
  <Grid>
    <Grid.ColumnDefinitions>
      <ColumnDefinition Width="*"/>
      <ColumnDefinition Width="Auto"/>
    </Grid.ColumnDefinitions>
    <StackPanel Margin="0,0,16,0">
      <StackPanel Orientation="Horizontal">
        <TextBlock Text="$safeTitle" FontSize="15" FontWeight="SemiBold" Foreground="#F2F2F5"/>
        <Border x:Name="badge" Visibility="Collapsed" Background="#8B5CF6" CornerRadius="6" Padding="7,1" Margin="10,1,0,0">
          <TextBlock Text="ON" FontSize="11" FontWeight="Bold" Foreground="#FFFFFF"/>
        </Border>
      </StackPanel>
      <TextBlock Text="$safeDesc" FontSize="12" Foreground="#8E8E99" TextWrapping="Wrap" Margin="0,4,0,0"/>
    </StackPanel>
    <StackPanel Grid.Column="1" Orientation="Horizontal" VerticalAlignment="Center">
      <Button x:Name="btnUndo" Content="Undo" Margin="0,0,8,0"/>
      <Button x:Name="btnApply" Content="Apply"/>
    </StackPanel>
  </Grid>
</Border>
"@
    $card  = [Windows.Markup.XamlReader]::Parse($cardXaml)
    $apply = $card.FindName('btnApply')
    $undo  = $card.FindName('btnUndo')
    $apply.Content = $t.Label
    $apply.Style = $script:Window.FindResource('BtnAccent')
    $undo.Style  = $script:Window.FindResource('Btn')
    $apply.Tag = $t
    $undo.Tag  = $t
    if (-not $t.CanRevert) { $undo.Visibility = 'Collapsed' }
    $apply.Add_Click({ param($s, $e) Invoke-Tweak $s.Tag 'Apply' })
    $undo.Add_Click({ param($s, $e) Invoke-Tweak $s.Tag 'Revert' })
    $script:Cards += @{ Tweak = $t; Badge = $card.FindName('badge'); Card = $card }
    return $card
}

function New-Heading($text) {
    $tb = New-Object System.Windows.Controls.TextBlock
    $tb.Text = $text
    $tb.FontSize = 13
    $tb.FontWeight = [System.Windows.FontWeights]::Bold
    $conv = New-Object System.Windows.Media.BrushConverter
    $tb.Foreground = $conv.ConvertFromString('#8E8E99')
    $tb.Margin = New-Object System.Windows.Thickness(2, 8, 0, 8)
    return $tb
}

function Update-Badges {
    foreach ($c in $script:Cards) {
        $t = $c.Tweak
        if ($t.Keys -and $t.Keys.Count -gt 0 -and (Test-Has $t.Keys)) { $c.Badge.Visibility = 'Visible' }
        else { $c.Badge.Visibility = 'Collapsed' }
    }
}

function Invoke-Tweak($t, $mode) {
    $script:Window.Cursor = [System.Windows.Input.Cursors]::Wait
    try {
        if ($t.Regs) {
            if ($mode -eq 'Apply') {
                foreach ($r in $t.Regs) { Set-Reg $r[0] $r[1] $r[2] $r[3] }
                if ($t.After) { & $t.After }
                Log $t.Msg
                if ($t.Restart) { Log '(Restart your PC for this to fully apply.)' }
            } else {
                foreach ($r in $t.Regs) { Restore-Key ("{0}|{1}" -f $r[0], $r[1]) }
                if ($t.After) { & $t.After }
                Log "$($t.Title): restored."
            }
        } else {
            if ($mode -eq 'Apply') { & $t.Apply } else { & $t.Revert }
        }
    } catch {
        Log "ERROR in '$($t.Title)': $($_.Exception.Message)"
    } finally {
        $script:Window.Cursor = $null
    }
    Update-Badges
}

$lastGroup = ''
foreach ($t in $script:Tweaks) {
    if ($t.Group -ne $lastGroup) {
        $label = switch ($t.Group) { 'Recommended' { 'RECOMMENDED - safe for everyone' } 'Advanced' { 'ADVANCED / OPTIONAL - test and keep what helps' } 'FPS' { 'FPS AND 1% LOWS - test each one and keep what helps' } default { 'TOOLS' } }
        $tp = $script:ui.tweakPanel; if ($t.Group -eq 'FPS') { $tp = $script:ui.fpsPanel }
        [void]$tp.Children.Add((New-Heading $label))
        $lastGroup = $t.Group
    }
    $tp = $script:ui.tweakPanel; if ($t.Group -eq 'FPS') { $tp = $script:ui.fpsPanel }
    [void]$tp.Children.Add((New-Card $t))
}

# =============== top buttons ===============
function Apply-Recommended {
    foreach ($t in $script:Tweaks) { if ($t.Recommended -and -not ($t.LaptopOnly -and $script:Mode -eq 'PC')) { Invoke-Tweak $t 'Apply' } }
    Log 'Recommended set applied. Restart your PC, then run the benchmark to see the difference.'
}

function Undo-All {
    $r = [System.Windows.MessageBox]::Show('Restore everything Lolo''s Tweaks changed to how it was before?', $script:AppName, 'YesNo', 'Question')
    if ($r -ne 'Yes') { return }
    Restore-Key 'powerplan'
    foreach ($k in @((Get-Bk).Keys)) { Restore-Key $k }
    Sync-Mouse
    Update-Badges
    foreach ($n in @('chkFan','chkPrio','chkTimer')) { $script:ui[$n].IsChecked = $false }
    Save-GameSettings
    Stop-GameBoost
    Log 'All changes restored (game boost options switched off too).'
}

function New-DesktopShortcut {
    if (-not $script:SelfPath) { Log 'Run the app from a saved file first, then try again.'; return }
    $dir = Join-Path $env:LOCALAPPDATA 'LolosTweaks'
    New-Item -ItemType Directory -Force -Path $dir | Out-Null
    $dest = Join-Path $dir 'lolos-tweaks.ps1'
    if ($script:SelfPath -ne $dest) { Copy-Item $script:SelfPath $dest -Force }
    $ws  = New-Object -ComObject WScript.Shell
    $lnk = $ws.CreateShortcut((Join-Path ([Environment]::GetFolderPath('Desktop')) 'Lolos Tweaks.lnk'))
    $lnk.TargetPath = "$env:WINDIR\System32\WindowsPowerShell\v1.0\powershell.exe"
    $lnk.Arguments  = "-NoProfile -STA -ExecutionPolicy Bypass -WindowStyle Hidden -File `"$dest`""
    $lnk.WorkingDirectory = $dir
    $lnk.Save()
    Log 'Desktop shortcut created. Double-click Lolos Tweaks on your desktop to open the app (it asks for admin).'
}

function Show-Report {
    Log '--- System report ---'
    $cpu  = Get-CimInstance Win32_Processor | Select-Object -First 1
    $os   = Get-CimInstance Win32_OperatingSystem
    $bios = Get-CimInstance Win32_BIOS
    $totalGB = [math]::Round($os.TotalVisibleMemorySize / 1MB, 1)
    $freeGB  = [math]::Round($os.FreePhysicalMemory / 1MB, 1)
    Log ("CPU: {0} ({1} cores / {2} threads)" -f $cpu.Name.Trim(), $cpu.NumberOfCores, $cpu.NumberOfLogicalProcessors)
    Log ("RAM: {0} GB total, {1} GB free" -f $totalGB, $freeGB)
    foreach ($g in (Get-CimInstance Win32_VideoController)) {
        $d = 'unknown'; if ($g.DriverDate) { $d = $g.DriverDate.ToString('yyyy-MM-dd') }
        Log ("GPU: {0} | driver {1} ({2})" -f $g.Name, $g.DriverVersion, $d)
        if ($g.DriverDate -and $g.DriverDate -lt (Get-Date).AddYears(-1)) {
            Log ("  [!] {0} driver is over a year old. Update from ASUS support or AMD/NVIDIA." -f $g.Name)
        }
    }
    Log ("BIOS: {0} ({1})" -f $bios.SMBIOSBIOSVersion, $bios.ReleaseDate.ToString('yyyy-MM-dd'))
    if ($totalGB -gt 0 -and ($freeGB / $totalGB) -lt 0.45) { Log '[!] Less than 45% RAM free. Use "Close background apps".' }
    $batt = Get-CimInstance Win32_Battery -ErrorAction SilentlyContinue
    if ($batt -and $batt.BatteryStatus -ne 2 -and $script:Mode -ne 'PC') { Log '[!] Running on battery. Plug in for full CPU/GPU power.' }
    if ($script:Mode -ne 'PC') { Log 'Tip: set Armoury Crate to Turbo while plugged in.' }
}

# =============== add a game ===============
function Add-CustomGame {
    $dlg = New-Object System.Windows.Forms.OpenFileDialog
    $dlg.Title  = 'Pick the game program (the .exe that runs while you play)'
    $dlg.Filter = 'Programs|*.exe'
    if ($dlg.ShowDialog() -ne 'OK') { return }
    $exe  = $dlg.FileName
    $name = [IO.Path]::GetFileNameWithoutExtension($exe)
    $names = @(Get-GameNames)
    if ($names -notcontains $name) { $script:ui.txtGames.Text = ((@($names) + $name) -join ', ') }
    if (@($script:CustomGames) -notcontains $exe) { $script:CustomGames = @($script:CustomGames) + $exe }
    Save-GameSettings
    Log "Added game: $name. Game boost now watches it, and Apply on the GPU and fullscreen tweaks will include it."
}

# =============== mode (laptop / desktop PC) and apply-all ===============
function Show-ModeDialog {
    [xml]$mx = @'
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Welcome" Width="470" Height="290" WindowStartupLocation="CenterScreen" ResizeMode="NoResize" Topmost="True"
        Background="#0B0B0D" Foreground="#F2F2F5" FontFamily="Segoe UI">
  <Grid Margin="26">
    <Grid.RowDefinitions>
      <RowDefinition Height="Auto"/>
      <RowDefinition Height="Auto"/>
      <RowDefinition Height="*"/>
    </Grid.RowDefinitions>
    <TextBlock Text="Welcome to Lolo's Tweaks" FontSize="20" FontWeight="Bold"/>
    <TextBlock Grid.Row="1" TextWrapping="Wrap" Foreground="#8E8E99" FontSize="12" Margin="0,8,0,0"
               Text="Are you using a laptop or a desktop PC? This tool is mainly made for laptops. Choosing Desktop PC hides the laptop-only options (ASUS fan mode and hybrid graphics). You can change this any time with the button at the top of the app. Use at your own risk: every change is backed up and can be undone."/>
    <StackPanel Grid.Row="2" Orientation="Horizontal" VerticalAlignment="Bottom" HorizontalAlignment="Center">
      <Button x:Name="btnLap" Content="I'm on a laptop" MinWidth="170" Margin="0,0,12,0"/>
      <Button x:Name="btnPc" Content="I'm on a desktop PC" MinWidth="170"/>
    </StackPanel>
  </Grid>
</Window>
'@
    $script:modeDlg = [Windows.Markup.XamlReader]::Load((New-Object System.Xml.XmlNodeReader $mx))
    $script:modePick = 'Laptop'
    $bLap = $script:modeDlg.FindName('btnLap')
    $bPc  = $script:modeDlg.FindName('btnPc')
    $bLap.Style = $script:Window.FindResource('BtnAccent')
    $bPc.Style  = $script:Window.FindResource('Btn')
    $bLap.Add_Click({ $script:modePick = 'Laptop'; $script:modeDlg.Close() })
    $bPc.Add_Click({ $script:modePick = 'PC'; $script:modeDlg.Close() })
    [void]$script:modeDlg.ShowDialog()
    return $script:modePick
}

function Apply-Mode {
    $lap = ($script:Mode -ne 'PC')
    $vis = 'Visible'
    if (-not $lap) { $vis = 'Collapsed' }
    foreach ($c in $script:Cards) { if ($c.Tweak.LaptopOnly) { $c.Card.Visibility = $vis } }
    $isAsus = $false
    try { $isAsus = ((Get-CimInstance Win32_ComputerSystem).Manufacturer -match 'ASUS') } catch { }
    $script:FanOk = ($lap -and $isAsus)
    $fvis = 'Collapsed'
    if ($script:FanOk) { $fvis = 'Visible' }
    foreach ($n in @('chkFan', 'btnFanOn', 'btnFanOff')) { $script:ui[$n].Visibility = $fvis }
    if (-not $script:FanOk) { $script:ui.chkFan.IsChecked = $false }
    if ($lap -and -not $isAsus) { Log 'Fan control is ASUS-only, so it is hidden. Use your laptop maker performance mode while gaming.' }
    if ($lap) { $script:ui.btnMode.Content = 'Laptop mode (change)' } else { $script:ui.btnMode.Content = 'Desktop PC mode (change)' }
    Log ("Mode: {0}." -f $(if ($lap) { 'laptop' } else { 'desktop PC (laptop-only options hidden)' }))
}

function Apply-Everything {
    $msg = @'
Apply ALL turns on every optimization on every tab, including the ADVANCED / OPTIONAL ones that are meant to be tested one at a time.

What can go wrong:
- Windows Defender stops scanning the Valorant and Fortnite folders in real time (a small security risk).
- The Windows telemetry service is switched off, and Fast Startup, background apps, MPO and transparency effects are changed.
- Power throttling is turned off and the CPU runs at full speed: more heat and more battery drain (keep the charger plugged in).
- ASUS laptops only: the fans go to Turbo while a game runs, which is louder and uses more power.
- Some settings help certain games and hurt others (fullscreen optimizations, MPO, GPU scheduling). Applying them all at once makes it hard to tell which one caused a problem.
- Many changes need a restart, and the app must run as administrator.

Everything is backed up and "Undo all" restores it, but this is not a good idea on a work or school PC.

Apply everything anyway?
'@
    $r = [System.Windows.MessageBox]::Show($msg, "$($script:AppName) - Apply ALL", 'YesNo', 'Warning', 'No')
    if ($r -ne 'Yes') { Log 'Apply ALL cancelled.'; return }
    foreach ($t in $script:Tweaks) {
        if ($t.Group -eq 'Tools') { continue }
        if (-not $t.CanRevert) { continue }
        if ($t.LaptopOnly -and $script:Mode -eq 'PC') { continue }
        Invoke-Tweak $t 'Apply'
    }
    $script:ui.chkPrio.IsChecked = $true
    $script:ui.chkTimer.IsChecked = $true
    if ($script:FanOk) { $script:ui.chkFan.IsChecked = $true }
    Save-GameSettings
    if ($script:GameActive) { Stop-GameBoost }
    Log 'Apply ALL finished: every optimization and the game boost options are on. Restart your PC, then test your games. Use Undo all to go back.'
}

# =============== navigation ===============
function Show-View($name) {
    $script:ui.viewOpt.Visibility   = 'Collapsed'
    $script:ui.viewBench.Visibility = 'Collapsed'
    $script:ui.viewRec.Visibility   = 'Collapsed'
    $script:ui.viewFps.Visibility   = 'Collapsed'
    $plain  = $script:Window.FindResource('Btn')
    $accent = $script:Window.FindResource('BtnAccent')
    $script:ui.btnNavOpt.Style   = $plain
    $script:ui.btnNavBench.Style = $plain
    $script:ui.btnNavRec.Style   = $plain
    $script:ui.btnNavFps.Style   = $plain
    switch ($name) {
        'opt'   { $script:ui.viewOpt.Visibility   = 'Visible'; $script:ui.btnNavOpt.Style   = $accent }
        'bench' { $script:ui.viewBench.Visibility = 'Visible'; $script:ui.btnNavBench.Style = $accent }
        'rec'   { $script:ui.viewRec.Visibility   = 'Visible'; $script:ui.btnNavRec.Style   = $accent }
        'fps'   { $script:ui.viewFps.Visibility   = 'Visible'; $script:ui.btnNavFps.Style   = $accent }
    }
}

# =============== benchmark ===============
function Get-BenchHistory {
    if (-not (Test-Path $script:BenchFile)) { return @() }
    try {
        $o = Get-Content $script:BenchFile -Raw | ConvertFrom-Json
        return @($o)
    } catch { return @() }
}
function Save-BenchHistory($arr) {
    $keep = @($arr | Select-Object -Last 20)
    ConvertTo-Json -InputObject $keep -Depth 4 | Set-Content -Path $script:BenchFile -Encoding UTF8
}

function Fmt-Delta($now, $old) {
    if ($null -eq $old -or $old -le 0 -or $now -le 0) { return '' }
    $p = ($now - $old) / $old * 100
    return ('{0:+0.0;-0.0;0.0}%' -f $p)
}

function Show-BenchResult($run, $prev, $hist) {
    $lines = New-Object System.Collections.ArrayList
    [void]$lines.Add(('{0,-30}{1,12}{2,12}{3,10}' -f 'Test', 'This run', 'Last run', 'Change'))
    [void]$lines.Add(('-' * 64))
    $rows = @(
        @('CPU single-core (M ops/s)', 'Single'),
        @('CPU all-core burst (M ops/s)', 'Multi'),
        @('CPU all-core sustained', 'Sustained'),
        @('Memory copy (GB/s)', 'Mem'),
        @('Disk write (MB/s)', 'Disk')
    )
    foreach ($r in $rows) {
        $now = [double]$run.($r[1])
        $nowTxt = if ($now -le 0) { 'skipped' } else { '{0:0.0}' -f $now }
        $oldTxt = '-'; $delta = ''
        if ($prev) {
            $old = [double]$prev.($r[1])
            if ($old -gt 0) { $oldTxt = '{0:0.0}' -f $old }
            $delta = Fmt-Delta $now $old
        }
        [void]$lines.Add(('{0,-30}{1,12}{2,12}{3,10}' -f $r[0], $nowTxt, $oldTxt, $delta))
    }
    [void]$lines.Add('')
    if ($run.Multi -gt 0) {
        $drop = ($run.Multi - $run.Sustained) / $run.Multi * 100
        if ($drop -lt 0) { $drop = 0 }
        $msg = 'healthy'
        if ($drop -gt 15) { $msg = 'large drop: heat or power limits are holding the CPU back (check Armoury Crate mode and that the charger is plugged in)' }
        elseif ($drop -gt 8) { $msg = 'moderate drop, normal for a thin laptop' }
        [void]$lines.Add(('CPU speed drop over the test: {0:0.0}% - {1}' -f $drop, $msg))
    }
    [void]$lines.Add(('Run: {0}   Logical cores: {1}   Tweaks ON: {2}' -f $run.Time, $run.Threads, $run.Tweaks))
    [void]$lines.Add('')
    [void]$lines.Add('Higher is better. Scores are for comparing this laptop against itself (before and after tweaks), not against other PCs.')
    $h = @($hist | Select-Object -Last 5)
    if ($h.Count -gt 1) {
        [void]$lines.Add('')
        [void]$lines.Add('Recent runs (single / all-core burst / tweaks ON):')
        foreach ($x in $h) { [void]$lines.Add(('  {0}   {1:0.0} / {2:0.0}   tweaks {3}' -f $x.Time, [double]$x.Single, [double]$x.Multi, $x.Tweaks)) }
    }
    $script:ui.benchResult.Text = ($lines -join "`r`n")
}

function Start-Bench {
    if ([LoloBench]::Running -or $script:Recording) { return }
    $script:ui.btnBench.IsEnabled = $false
    $script:ui.benchBar.Value = 0
    $script:ui.benchResult.Text = ''
    Log 'Benchmark started - please do not use the PC until it finishes.'
    [LoloBench]::Start()
    $script:BenchTimer.Start()
}

function Finish-Bench {
    $script:BenchTimer.Stop()
    $script:ui.btnBench.IsEnabled = $true
    $script:ui.benchBar.Value = 100
    $script:ui.benchStatus.Text = 'Done.'
    if ([LoloBench]::Error) { Log ("Benchmark notes: {0}" -f [LoloBench]::Error) }
    $applied = @($script:Cards | Where-Object { $_.Badge.Visibility -eq 'Visible' }).Count
    $run = [pscustomobject]@{
        Time      = (Get-Date).ToString('yyyy-MM-dd HH:mm')
        Tweaks    = $applied
        Threads   = [LoloBench]::Threads
        Single    = [math]::Round([LoloBench]::CpuSingle, 2)
        Multi     = [math]::Round([LoloBench]::CpuMulti, 2)
        Sustained = [math]::Round([LoloBench]::CpuSustained, 2)
        Mem       = [math]::Round([LoloBench]::MemGBs, 2)
        Disk      = [math]::Round([LoloBench]::DiskWriteMBs, 1)
    }
    $hist = @(Get-BenchHistory)
    $prev = $null
    if ($hist.Count -gt 0) { $prev = $hist[$hist.Count - 1] }
    $hist = @($hist) + @($run)
    Save-BenchHistory $hist
    Show-BenchResult $run $prev $hist
    Log 'Benchmark finished.'
}

$script:BenchTimer = New-Object System.Windows.Threading.DispatcherTimer
$script:BenchTimer.Interval = [TimeSpan]::FromMilliseconds(400)
$script:BenchTimer.Add_Tick({
    $script:ui.benchBar.Value = [LoloBench]::Percent
    $script:ui.benchStatus.Text = [LoloBench]::Status
    if ([LoloBench]::Done) { Finish-Bench }
})

function Clear-BenchHistory {
    Remove-Item $script:BenchFile -Force -ErrorAction SilentlyContinue
    $script:ui.benchResult.Text = ''
    Log 'Benchmark history cleared.'
}

# =============== game session recorder ===============
$script:Recording = $false
$script:RecSamples = New-Object System.Collections.ArrayList
$script:RecStartTime = $null
$script:ThrottleField = $null
$script:HasNv = [bool](Get-Command nvidia-smi -ErrorAction SilentlyContinue)

function ConvertTo-Dbl($s) {
    $d = 0.0
    if ([double]::TryParse(([string]$s).Trim(), [Globalization.NumberStyles]::Float, [Globalization.CultureInfo]::InvariantCulture, [ref]$d)) { return $d }
    return 0.0
}

function Find-ThrottleField {
    foreach ($f in @('clocks_throttle_reasons.active', 'clocks_event_reasons.active')) {
        $o = & nvidia-smi "--query-gpu=$f" --format=csv,noheader 2>$null
        if ($o) {
            $first = ($o | Select-Object -First 1).ToString().Trim()
            if ($first -match '^0x') { return $f }
        }
    }
    return $null
}

function Sample-Rec {
    if (-not $script:HasNv) { Sample-RecBasic; return }
    $fields = 'temperature.gpu,utilization.gpu,clocks.gr,power.draw'
    if ($script:ThrottleField) { $fields += ',' + $script:ThrottleField }
    $o = & nvidia-smi "--query-gpu=$fields" --format=csv,noheader,nounits 2>$null
    if (-not $o) { return }
    $v = ($o | Select-Object -First 1) -split ',\s*'
    if ($v.Count -lt 4) { return }
    $thr = 0
    if ($v.Count -ge 5) { try { $thr = [Convert]::ToInt64(($v[4].Trim() -replace '^0x', ''), 16) } catch { $thr = 0 } }
    $os  = Get-CimInstance Win32_OperatingSystem
    $cpu = (Get-CimInstance Win32_Processor | Select-Object -First 1).LoadPercentage
    $sample = [pscustomobject]@{
        Temp  = ConvertTo-Dbl $v[0]
        Util  = ConvertTo-Dbl $v[1]
        Clock = ConvertTo-Dbl $v[2]
        Power = ConvertTo-Dbl $v[3]
        Thr   = $thr
        Cpu   = [double]$cpu
        RamGB = [math]::Round(($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / 1MB, 1)
    }
    [void]$script:RecSamples.Add($sample)
    $el = (Get-Date) - $script:RecStartTime
    $script:ui.recStatus.Text = ("Recording {0:mm\:ss}   GPU {1:0}C  {2:0}%  {3:0} MHz  {4:0} W   CPU {5:0}%   RAM {6} GB" -f $el, $sample.Temp, $sample.Util, $sample.Clock, $sample.Power, $sample.Cpu, $sample.RamGB)
}

function Sample-RecBasic {
    $g = 0.0
    try {
        $rows = @(Get-CimInstance Win32_PerfFormattedData_GPUPerformanceCounters_GPUEngine -ErrorAction Stop | Where-Object { $_.Name -like '*engtype_3D*' })
        if ($rows.Count -gt 0) {
            $per = $rows | Group-Object { $_.Name -replace '^pid_\d+_', '' } | ForEach-Object { ($_.Group | Measure-Object UtilizationPercentage -Sum).Sum }
            $m = ($per | Measure-Object -Maximum).Maximum
            if ($m) { $g = [math]::Min(100.0, [double]$m) }
        }
    } catch { }
    $os  = Get-CimInstance Win32_OperatingSystem
    $cpu = (Get-CimInstance Win32_Processor | Select-Object -First 1).LoadPercentage
    $sample = [pscustomobject]@{
        Temp = 0.0; Util = $g; Clock = 0.0; Power = 0.0; Thr = 0
        Cpu = [double]$cpu
        RamGB = [math]::Round(($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / 1MB, 1)
    }
    [void]$script:RecSamples.Add($sample)
    $el = (Get-Date) - $script:RecStartTime
    $script:ui.recStatus.Text = ("Recording {0:mm\:ss}   GPU {1:0}%   CPU {2:0}%   RAM {3} GB" -f $el, $sample.Util, $sample.Cpu, $sample.RamGB)
}

function Show-RecReportBasic($s) {
    $dur    = $script:RecEnd - $script:RecStartTime
    $avgU   = ($s | Measure-Object Util -Average).Average
    $maxU   = ($s | Measure-Object Util -Maximum).Maximum
    $avgCpu = ($s | Measure-Object Cpu -Average).Average
    $maxRam = ($s | Measure-Object RamGB -Maximum).Maximum
    $os     = Get-CimInstance Win32_OperatingSystem
    $totGB  = [math]::Round($os.TotalVisibleMemorySize / 1MB, 1)
    $out = New-Object System.Collections.ArrayList
    [void]$out.Add(('Recording length : {0:mm\:ss}  ({1} samples)' -f $dur, $s.Count))
    [void]$out.Add(('GPU 3D usage     : avg {0:0}%, max {1:0}%' -f $avgU, $maxU))
    [void]$out.Add(('CPU load         : avg {0:0}%' -f $avgCpu))
    [void]$out.Add(('RAM used         : peak {0} GB of {1} GB' -f $maxRam, $totGB))
    [void]$out.Add('')
    [void]$out.Add('Temperatures, clocks and throttling are only available on NVIDIA cards.')
    [void]$out.Add('')
    [void]$out.Add('What this means:')
    $notes = 0
    if ($avgU -lt 15) {
        [void]$out.Add('- The graphics card was barely used. Either the game was not running, or it is running on the built-in graphics. On a laptop, apply "Force games onto the dedicated GPU" and restart the game.'); $notes++
    } elseif ($avgU -lt 60) {
        [void]$out.Add('- GPU usage is moderate. At very high FPS that is normal; otherwise the game is probably CPU-limited or FPS-capped.'); $notes++
    }
    if ($avgCpu -gt 90) { [void]$out.Add('- The CPU was close to maxed out, which is a common cause of stutter. Close background apps.'); $notes++ }
    if ($totGB -gt 0 -and ($maxRam / $totGB) -gt 0.9) { [void]$out.Add('- RAM was nearly full. Close background apps before gaming.'); $notes++ }
    if ($notes -eq 0) { [void]$out.Add('- Nothing worrying in usage or memory.') }
    $script:ui.recResult.Text = ($out -join "`r`n")
}

function Show-RecReport {
    $s = @($script:RecSamples)
    if ($s.Count -lt 3) {
        $script:ui.recResult.Text = 'Not enough data. Record for at least 10 seconds while the game is running.'
        return
    }
    if (-not $script:HasNv) { Show-RecReportBasic $s; return }
    $dur = $script:RecEnd - $script:RecStartTime
    $load = @($s | Where-Object { $_.Util -ge 30 })
    $loadNote = ''
    if ($load.Count -ge 3) { $basis = $load; $loadNote = ('(clock, power and throttle figures use the {0} samples where the GPU was busy)' -f $load.Count) } else { $basis = $s }
    $avgU   = ($s | Measure-Object Util -Average).Average
    $avgT   = ($s | Measure-Object Temp -Average).Average
    $maxT   = ($s | Measure-Object Temp -Maximum).Maximum
    $avgC   = ($basis | Measure-Object Clock -Average).Average
    $avgP   = ($basis | Measure-Object Power -Average).Average
    $maxP   = ($basis | Measure-Object Power -Maximum).Maximum
    $avgCpu = ($s | Measure-Object Cpu -Average).Average
    $maxRam = ($s | Measure-Object RamGB -Maximum).Maximum
    $thermalMask = 0x20 -bor 0x40
    $powerMask   = 0x4
    $brakeMask   = 0x8 -bor 0x80
    $thermal = 0; $powerCap = 0; $brake = 0
    foreach ($x in $basis) {
        if (($x.Thr -band $thermalMask) -ne 0) { $thermal++ }
        if (($x.Thr -band $powerMask) -ne 0)   { $powerCap++ }
        if (($x.Thr -band $brakeMask) -ne 0)   { $brake++ }
    }
    $n = [double]$basis.Count
    $pThermal = $thermal / $n * 100
    $pPower   = $powerCap / $n * 100
    $pBrake   = $brake / $n * 100

    $out = New-Object System.Collections.ArrayList
    [void]$out.Add(('Recording length : {0:mm\:ss}  ({1} samples)' -f $dur, $s.Count))
    [void]$out.Add(('GPU temperature  : avg {0:0} C, max {1:0} C' -f $avgT, $maxT))
    [void]$out.Add(('GPU usage        : avg {0:0}%' -f $avgU))
    [void]$out.Add(('GPU clock        : avg {0:0} MHz' -f $avgC))
    [void]$out.Add(('GPU power        : avg {0:0} W, max {1:0} W' -f $avgP, $maxP))
    if ($script:ThrottleField) {
        [void]$out.Add(('Thermal throttle : {0:0}% of busy time' -f $pThermal))
        [void]$out.Add(('Power limited    : {0:0}% of busy time' -f $pPower))
    } else {
        [void]$out.Add('Throttle info    : not reported by this driver')
    }
    [void]$out.Add(('CPU load         : avg {0:0}%' -f $avgCpu))
    [void]$out.Add(('RAM used         : peak {0} GB' -f $maxRam))
    if ($loadNote) { [void]$out.Add(''); [void]$out.Add($loadNote) }
    [void]$out.Add('')
    [void]$out.Add('What this means:')
    $notes = 0
    if ($avgU -lt 15) {
        [void]$out.Add('- The GPU was barely used. Either the game was not running, or it is running on the built-in graphics. On a laptop, apply "Force games onto the dedicated GPU" and restart the game.'); $notes++
    } elseif ($avgU -lt 60) {
        [void]$out.Add('- GPU usage is moderate. In Valorant at high FPS that is normal; otherwise the game is probably CPU-limited or FPS-capped.'); $notes++
    }
    if ($maxT -ge 87) { [void]$out.Add('- The GPU got very hot. Use a hard flat surface or a cooling stand, set Turbo fan mode, and clean the vents.'); $notes++ }
    if ($script:ThrottleField -and ($pThermal -gt 1 -or $pBrake -gt 1)) { [void]$out.Add('- The GPU slowed itself down because of heat or a power brake. This is what causes FPS drops in long sessions.'); $notes++ }
    if ($script:ThrottleField -and $pPower -gt 60 -and $avgU -gt 85) { [void]$out.Add('- The GPU is hitting its laptop power limit most of the time. That is normal for laptop graphics; being plugged in with your performance/Turbo mode gives it the most power.'); $notes++ }
    if ($maxRam -gt 14.0) { [void]$out.Add('- RAM was nearly full. Close background apps before gaming.'); $notes++ }
    if ($notes -eq 0) { [void]$out.Add('- Nothing worrying: temperatures, clocks and memory all looked healthy.') }
    $script:ui.recResult.Text = ($out -join "`r`n")
}

function Toggle-Recording {
    if ([LoloBench]::Running) { return }
    if (-not $script:Recording) {
        $script:RecSamples.Clear()
        $script:ThrottleField = $null
        if ($script:HasNv) { $script:ThrottleField = Find-ThrottleField } else { Log 'No NVIDIA card found: recording GPU usage, CPU and RAM only.' }
        $script:RecStartTime = Get-Date
        $script:Recording = $true
        $script:ui.btnRec.Content = 'Stop and show report'
        $script:ui.recResult.Text = ''
        $script:RecTimer.Start()
        Log 'Recording started. Launch your game, play, then come back and press Stop.'
    } else {
        $script:Recording = $false
        $script:RecTimer.Stop()
        $script:RecEnd = Get-Date
        $script:ui.btnRec.Content = 'Start recording'
        $script:ui.recStatus.Text = 'Not recording.'
        Show-RecReport
        Log 'Recording stopped.'
    }
}

$script:RecTimer = New-Object System.Windows.Threading.DispatcherTimer
$script:RecTimer.Interval = [TimeSpan]::FromSeconds(2)
$script:RecTimer.Add_Tick({ try { Sample-Rec } catch { } })

# =============== live stats bar ===============
function Update-Stats {
    if ($script:Recording -or [LoloBench]::Running) { return }
    try {
        $os   = Get-CimInstance Win32_OperatingSystem
        $used = [math]::Round(($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / 1MB, 1)
        $tot  = [math]::Round($os.TotalVisibleMemorySize / 1MB, 1)
        $cpu  = (Get-CimInstance Win32_Processor | Select-Object -First 1).LoadPercentage
        $s = ("CPU {0}%    RAM {1}/{2} GB" -f $cpu, $used, $tot)
        if ($script:HasNv) {
            $g = & nvidia-smi --query-gpu=temperature.gpu,utilization.gpu,memory.used,memory.total,clocks.gr --format=csv,noheader,nounits 2>$null
            if ($g) {
                $v = ($g | Select-Object -First 1) -split ',\s*'
                $s += ("    GPU {0}C  {1}%  VRAM {2}/{3} MB  {4} MHz" -f $v[0], $v[1], $v[2], $v[3], $v[4])
            }
        }
        $script:ui.stats.Text = $s
    } catch { }
}

$script:Timer = New-Object System.Windows.Threading.DispatcherTimer
$script:Timer.Interval = [TimeSpan]::FromSeconds(3)
$script:Timer.Add_Tick({ Update-Stats })

# =============== wire up buttons ===============
$script:ui.btnAll.Add_Click({ Apply-Recommended })
$script:ui.btnEverything.Add_Click({ Apply-Everything })
$script:ui.btnAddGame.Add_Click({ Add-CustomGame })
$script:ui.btnMode.Add_Click({ $script:Mode = Show-ModeDialog; Save-GameSettings; Apply-Mode })
$script:ui.btnUndoAll.Add_Click({ Undo-All })
$script:ui.btnReport.Add_Click({ Show-Report })
$script:ui.btnShortcut.Add_Click({ New-DesktopShortcut })
$script:ui.btnNavOpt.Add_Click({ Show-View 'opt' })
$script:ui.btnNavBench.Add_Click({ Show-View 'bench' })
$script:ui.btnNavRec.Add_Click({ Show-View 'rec' })
$script:ui.btnNavFps.Add_Click({ Show-View 'fps' })

# game boost controls
$script:GameChanged = { Save-GameSettings; if ($script:GameActive) { Stop-GameBoost } }
$script:ui.chkFan.Add_Click($script:GameChanged)
$script:ui.chkPrio.Add_Click($script:GameChanged)
$script:ui.chkTimer.Add_Click($script:GameChanged)
$script:ui.txtGames.Add_LostFocus($script:GameChanged)
$script:ui.btnFanOn.Add_Click({
    $cur = Get-FanMode
    if ($null -eq $cur) { Log 'Fans: could not reach the ASUS fan interface. Use Armoury Crate or Fn+F5 instead.'; return }
    if ($cur -eq 1) { Log 'Fans: already on Turbo.'; return }
    if ($null -eq (Get-FanState)) { Save-FanState $cur }
    if (Set-FanMode 1) { Log 'Fans: Turbo on (test). Press Restore fan mode when you are done.' }
    else { Log 'Fans: the laptop did not accept Turbo mode.'; Clear-FanState }
})
$script:ui.btnFanOff.Add_Click({
    if ($null -eq (Get-FanState)) { Log 'No saved fan mode to restore.' } else { [void](Restore-Fan) }
})

$script:GameTimer = New-Object System.Windows.Threading.DispatcherTimer
$script:GameTimer.Interval = [TimeSpan]::FromSeconds(3)
$script:GameTimer.Add_Tick({ try { Update-GameBoost } catch { } })
$script:ui.btnBench.Add_Click({ Start-Bench })
$script:ui.btnBenchClear.Add_Click({ Clear-BenchHistory })
$script:ui.btnRec.Add_Click({ Toggle-Recording })

$script:Window.Add_Closed({
    try { $script:GameTimer.Stop(); Stop-GameBoost } catch { }
    $script:Timer.Stop()
    $script:BenchTimer.Stop()
    $script:RecTimer.Stop()
})

# =============== start ===============
try {
    $cpuName  = (Get-CimInstance Win32_Processor | Select-Object -First 1).Name.Trim()
    $gpuNames = (Get-CimInstance Win32_VideoController | ForEach-Object { $_.Name }) -join ' + '
    $script:ui.subtitle.Text = "$cpuName   |   $gpuNames"
} catch { $script:ui.subtitle.Text = 'Gaming optimizer' }

# show last benchmark if there is one
try {
    $h0 = @(Get-BenchHistory)
    if ($h0.Count -gt 0) {
        $prev0 = $null
        if ($h0.Count -gt 1) { $prev0 = $h0[$h0.Count - 2] }
        Show-BenchResult $h0[$h0.Count - 1] $prev0 $h0
    } else {
        $script:ui.benchResult.Text = 'No benchmark runs yet. Press Run benchmark to take a baseline before you apply tweaks.'
    }
} catch { }

Update-Badges
Update-Stats
$script:Timer.Start()
Log 'Ready. Tip: run the Benchmark first for a baseline, press Apply recommended, restart, then run it again.'
if (-not (Test-Admin)) { Log 'Note: not running as Administrator. Many tweaks (HKLM ones) will fail until you run it as admin.' }

# game boost: load saved options, undo a fan change left over from a crash, start watching
try {
    Load-GameSettings
    if (-not $script:Mode) { $script:Mode = Show-ModeDialog; Save-GameSettings }
    Apply-Mode
    $p0 = @()
    $nm0 = @(Get-GameNames)
    if ($nm0.Count -gt 0) { $p0 = @(Get-Process -Name $nm0 -ErrorAction SilentlyContinue) }
    if ($p0.Count -eq 0 -and $null -ne (Get-FanState)) {
        Log 'Last session ended while fans were on Turbo. Restoring:'
        [void](Restore-Fan)
    }
    $script:GameTimer.Start()
    Update-GameBoost
} catch { }

[void]$script:Window.ShowDialog()
