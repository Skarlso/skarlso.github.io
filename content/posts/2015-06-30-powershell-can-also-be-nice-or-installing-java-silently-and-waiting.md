+++
author = "hannibal"
categories = ["devops", "problem solving"]
date = "2015-06-30"
tags = ["powershell", "unattended", "Windows"]
title = "Powershell can also be nice -Or Installing Java silently and waiting"
url = "/2015/06/30/powershell-can-also-be-nice-or-installing-java-silently-and-waiting/"

+++

Hello folks.

Today, I would like to show you a small script. It installs Java JDK, both version, x86 and 64 bit, silently, and wait for that process to finish.

The wait is necessary because /s on a java install has the nasty habit of running in the background. If you are using a .bat file, **you shouldn't**, than you would use something like: start /w jdk-setup.exe /s. This gets it done, but is ugly. Also, if you are using Packer and PowerShell provisioning, you might want to set up some environment variables as well for the next script. And you want that property to be available and you don't want to mess it up with setting a path into a file and then re-setting your path on the begin of your other script. Or pass it around with Packer. No. Use a proper PowerShell script. Learn it. It's not that hard. Be a professional. Don't hack something together for the next person to suffer at.

Here is how I did it. Hope it helps somebody out.

~~~powershell

$JDK_VER="7u75"
$JDK_FULL_VER="7u75-b13"
$JDK_PATH="1.7.0_75"
$source86 = "http://download.oracle.com/otn-pub/java/jdk/$JDK_FULL_VER/jdk-$JDK_VER-windows-i586.exe"
$source64 = "http://download.oracle.com/otn-pub/java/jdk/$JDK_FULL_VER/jdk-$JDK_VER-windows-x64.exe"
$destination86 = "C:\vagrant\$JDK_VER-x86.exe"
$destination64 = "C:\vagrant\$JDK_VER-x64.exe"
$client = new-object System.Net.WebClient
$cookie = "oraclelicense=accept-securebackup-cookie"
$client.Headers.Add([System.Net.HttpRequestHeader]::Cookie, $cookie)

Write-Host 'Checking if Java is already installed'
if ((Test-Path "c:\Program Files (x86)\Java") -Or (Test-Path "c:\Program Files\Java")) {
    Write-Host 'No need to Install Java'
    Exit
}

Write-Host 'Downloading x86 to $destination86'

$client.downloadFile($source86, $destination86)
if (!(Test-Path $destination86)) {
    Write-Host "Downloading $destination86 failed"
    Exit
}
Write-Host 'Downloading x64 to $destination64'

$client.downloadFile($source64, $destination64)
if (!(Test-Path $destination64)) {
    Write-Host "Downloading $destination64 failed"
    Exit
}


try {
    Write-Host 'Installing JDK-x64'
    $proc1 = Start-Process -FilePath "$destination64" -ArgumentList "/s REBOOT=ReallySuppress" -Wait -PassThru
    $proc1.waitForExit()
    Write-Host 'Installation Done.'

    Write-Host 'Installing JDK-x86'
    $proc2 = Start-Process -FilePath "$destination86" -ArgumentList "/s REBOOT=ReallySuppress" -Wait -PassThru
    $proc2.waitForExit()
    Write-Host 'Installtion Done.'
} catch [exception] {
    write-host '$_ is' $_
    write-host '$_.GetType().FullName is' $_.GetType().FullName
    write-host '$_.Exception is' $_.Exception
    write-host '$_.Exception.GetType().FullName is' $_.Exception.GetType().FullName
    write-host '$_.Exception.Message is' $_.Exception.Message
}

if ((Test-Path "c:\Program Files (x86)\Java") -Or (Test-Path "c:\Program Files\Java")) {
    Write-Host 'Java installed successfully.'
}
Write-Host 'Setting up Path variables.'
[System.Environment]::SetEnvironmentVariable("JAVA_HOME", "c:\Program Files (x86)\Java\jdk$JDK_PATH", "Machine")
[System.Environment]::SetEnvironmentVariable("PATH", $Env:Path + ";c:\Program Files (x86)\Java\jdk$JDK_PATH\bin", "Machine")
Write-Host 'Done. Goodbye.'
~~~

Now, there is room for improvement here. Like checking exit code, doing something extra after a failed exit. Throwing an exception, and so on and so forth. But this is a much needed improvement from calling a BAT file.

And you would use this in a Packer JSON file like this..

~~~json

{
      "type": "powershell",
      "scripts": [
        "./scripts/jdk_inst.ps1"
      ]
}
~~~

Easy. And at the end, the System.Environment actually writes out into the registry permanently so no need to pass it around in a file or something ugly like that.

Hope this helps somebody.

Thanks for reading.

Gergely.