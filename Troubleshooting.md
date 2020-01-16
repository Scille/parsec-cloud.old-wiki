# Windows 7

Parsec may have a hard time running on a Windows 7 machine. This is probably due to Windows detecting WinFSP as not correctly signed.

![](https://i.ibb.co/6wGfbFz/winfsp-win7.png)

See https://docs.microsoft.com/en-us/security-updates/SecurityAdvisories/2015/3033929?redirectedfrom=MSDN for information on this issue.

A patch from Microsoft has to be installed:
- For x64: https://www.microsoft.com/en-us/download/details.aspx?id=46148
- For x86: https://www.microsoft.com/en-us/download/details.aspx?id=46078

If in doubt, the x64 is probably the right one. This patch requires that your Windows 7 is up-to-date.