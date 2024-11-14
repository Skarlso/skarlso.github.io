+++
author = "hannibal"
categories = ["devops", "problem solving"]
date = "2015-07-16"
tags = ["jenkins", "Packer", "vagrant"]
title = "Selenium Testing with Packer and Vagrant"
url = "/2015/07/16/selenium-testing-with-packer-and-vagrant/"

+++

So, recently, the tester team talked to me, that their build takes too long, and why is that? A quick look at their configuration and build scripts showed me, that they are actually using a vagrant box, which never gets destroyed or re-started at least. To remedy this problem, I came up with the following solution.

<!--more-->

# Same old.

Same as in my previous post, we are going to build a Windows Machine for this purpose. The only addition to my previous settings, will be some Java install, downloading selenium and installing Chrome, and Firefox.

# Installation

#### Answer File

Here is the configuration and setup of Windows before the provision phase.

~~~xml

...
               <SynchronousCommand wcm:action="add">
                  <CommandLine>cmd.exe /c C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -File a:\jdk_inst.ps1 -AutoStart</CommandLine>
                  <Description>Install Java</Description>
                  <Order>103</Order>
                  <RequiresUserInput>true</RequiresUserInput>
               </SynchronousCommand>
...
~~~

This is the part were I'm installing Java. The script for the jdk_inst.ps1 is in my previous post, but I'll paste it here for ease of read.

~~~powershell

function LogWrite {
   Param ([string]$logstring)
   $now = Get-Date -format s
   Add-Content $Logfile -value "$now $logstring"
   Write-Host $logstring
}

$Logfile = "C:\Windows\Temp\jdk-install.log"

$JDK_VER="7u75"
$JDK_FULL_VER="7u75-b13"
$JDK_PATH="1.7.0_75"
$source86 = "http://download.oracle.com/otn-pub/java/jdk/$JDK_FULL_VER/jdk-$JDK_VER-windows-i586.exe"
$source64 = "http://download.oracle.com/otn-pub/java/jdk/$JDK_FULL_VER/jdk-$JDK_VER-windows-x64.exe"
$destination86 = "C:\Windows\Temp\$JDK_VER-x86.exe"
$destination64 = "C:\Windows\Temp\$JDK_VER-x64.exe"
$client = new-object System.Net.WebClient
$cookie = "oraclelicense=accept-securebackup-cookie"
$client.Headers.Add([System.Net.HttpRequestHeader]::Cookie, $cookie)

LogWrite "Setting Execution Policy level to Bypass"
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy Bypass -Force

LogWrite 'Checking if Java is already installed'
if ((Test-Path "c:\Program Files (x86)\Java") -Or (Test-Path "c:\Program Files\Java")) {
    LogWrite 'No need to Install Java'
    Exit
}

LogWrite 'Downloading x86 to $destination86'
try {
  $client.downloadFile($source86, $destination86)
  if (!(Test-Path $destination86)) {
      LogWrite "Downloading $destination86 failed"
      Exit
  }
  LogWrite 'Downloading x64 to $destination64'

  $client.downloadFile($source64, $destination64)
  if (!(Test-Path $destination64)) {
      LogWrite "Downloading $destination64 failed"
      Exit
  }
} catch [Exception] {
  LogWrite '$_.Exception is' $_.Exception
}

try {
    LogWrite 'Installing JDK-x64'
    $proc1 = Start-Process -FilePath "$destination64" -ArgumentList "/s REBOOT=ReallySuppress" -Wait -PassThru
    $proc1.waitForExit()
    LogWrite 'Installation Done.'

    LogWrite 'Installing JDK-x86'
    $proc2 = Start-Process -FilePath "$destination86" -ArgumentList "/s REBOOT=ReallySuppress" -Wait -PassThru
    $proc2.waitForExit()
    LogWrite 'Installtion Done.'
} catch [exception] {
    LogWrite '$_ is' $_
    LogWrite '$_.GetType().FullName is' $_.GetType().FullName
    LogWrite '$_.Exception is' $_.Exception
    LogWrite '$_.Exception.GetType().FullName is' $_.Exception.GetType().FullName
    LogWrite '$_.Exception.Message is' $_.Exception.Message
}

if ((Test-Path "c:\Program Files (x86)\Java") -Or (Test-Path "c:\Program Files\Java")) {
    LogWrite 'Java installed successfully.'
} else {
    LogWrite 'Java install Failed!'
}
LogWrite 'Setting up Path variables.'
[System.Environment]::SetEnvironmentVariable("JAVA_HOME", "c:\Program Files (x86)\Java\jdk$JDK_PATH", "Machine")
[System.Environment]::SetEnvironmentVariable("PATH", $Env:Path + ";c:\Program Files (x86)\Java\jdk$JDK_PATH\bin", "Machine")
LogWrite 'Done. Goodbye.'
~~~

This installs both x86 and 64 bit version of Java.

# Provision

I decided to put these into the provision phase to get log messages written out properly. Because in the unattended file, you can't see any progress.

#### Chrome And Firefox

Installing these two proved a little bit more difficult. Chrome didn't really like me to download their installer without accepting something first, like Java. Luckily, after a LOT of digging, I found a chrome installer which lets you install silently. Here is the script to install the two.

~~~powershell

function LogWrite {
    Param ([string]$logstring)
    $now = Get-Date -format s
    Add-Content $Logfile -value "$now $logstring"
    Write-Host $logstring
}

function CheckLocation {
    Param ([string]$location)
    if (!(Test-Path  $location)) {
        throw [System.IO.FileNotFoundException] "Could not download to Destination $location."
    }
}

$Logfile = "C:\Windows\Temp\chrome-firefox-install.log"

$chrome_source = "http://dl.google.com/chrome/install/375.126/chrome_installer.exe"
$chrome_destination = "C:\Windows\Temp\chrome_installer.exe"
$firefox_source = "https://download-installer.cdn.mozilla.net/pub/firefox/releases/39.0/win32/hu/Firefox%20Setup%2039.0.exe"
$firefox_destination = "C:\Windows\Temp\firefoxinstaller.exe"

LogWrite 'Starting to download files.'
try {
    LogWrite 'Downloading Chrome...'
    (New-Object System.Net.WebClient).DownloadFile($chrome_source, $chrome_destination)
    CheckLocation $chrome_destination
    LogWrite 'Done...'
    LogWrite 'Downloading Firefox...'
    (New-Object System.Net.WebClient).DownloadFile($firefox_source, $firefox_destination)
    CheckLocation $firefox_destination
} catch [Exception] {
    LogWrite "Exception during download. Probable cause could be that the directory or the file didn't exist."
    LogWrite '$_.Exception is' $_.Exception
}

LogWrite 'Starting firefox install process.'
try {
    Start-Process -FilePath $firefox_destination -ArgumentList "-ms" -Wait -PassThru
} catch [Exception] {
    LogWrite 'Exception during install process.'
    LogWrite '$_.Exception is' $_.Exception
}
LogWrite 'Starting chrome install process.'

try {
    Start-Process -FilePath $chrome_destination -ArgumentList "/silent /install" -Wait -PassThru
} catch [Exception] {
    LogWrite 'Exception during install process.'
    LogWrite '$_.Exception is' $_.Exception
}

LogWrite 'All done. Goodbye.'
~~~

They both install silently. Pretty neat.

#### Selenium

This only has to be downloaded, so this is pretty simple. Vagrant will handle the startup of course when it does a vagrant up.

~~~powershell

function LogWrite {
   Param ([string]$logstring)
   $now = Get-Date -format s
   Add-Content $Logfile -value "$now $logstring"
   Write-Host $logstring
}

$Logfile = "C:\Windows\Temp\selenium-install.log"

$source = "http://selenium-release.storage.googleapis.com/2.46/selenium-server-standalone-2.46.0.jar"
$destination = "C:\Windows\Temp\selenium-server.jar"
LogWrite 'Starting to download selenium file.'
try {
  (New-Object System.Net.WebClient).DownloadFile($source, $destination)
} catch [Exception] {
  LogWrite "Exception during download. Probable cause could be that the directory or the file didn't exist."
  LogWrite '$_.Exception is' $_.Exception
}
LogWrite 'Download done. Checking if file exists.'
if (!(Test-Path $destination)) {
  LogWrite 'Downloading dotnet Failed!'
} else {
  LogWrite 'Download successful.'
}

LogWrite 'All done. Goodbye.'
~~~

Straightforward.

#### The Packer Json File

So putting this all together, here is the Packer JSON file for this:

~~~json

{
      "variables": {
      "vm_name": "win7x64selenium",
      "output_dir": "output_win7_x64_selenium",
      "vagrant_box_output": "box_output",
      "cpu_number": "2",
      "memory_size": "4096",
      "machine_type": "pc-1.2",
      "accelerator": "kvm",
      "disk_format": "qcow2",
      "disk_interface": "virtio",
      "net_device": "virtio-net",
      "cpu_model": "host",
      "disk_cache": "writeback",
      "disk_io": "native"
   },

  "builders": [
    {
      "type": "virtualbox-iso",
      "iso_url": "/home/user/vms/windows7.iso",
      "iso_checksum_type": "sha1",
      "iso_checksum": "0BCFC54019EA175B1EE51F6D2B207A3D14DD2B58",
      "headless": true,
      "boot_wait": "2m",
      "ssh_username": "vagrant",
      "ssh_password": "vagrant",
      "ssh_wait_timeout": "8h",
      "shutdown_command": "shutdown /s /t 10 /f /d p:4:1 /c \"Packer Shutdown\"",
      "guest_os_type": "Windows7_64",
      "disk_size": 61440,
      "floppy_files": [
        "./answer_files/7-selenium/Autounattend.xml",
        "./scripts/dis-updates.ps1",
        "./scripts/microsoft-updates.bat",
        "./scripts/openssh.ps1",
        "./scripts/jdk_inst.ps1"
      ],
      "vboxmanage": [
        [
          "modifyvm",
          "{{.Name}}",
          "--memory",
          "{{user `memory_size`}}"
        ],
        [
          "modifyvm",
          "{{.Name}}",
          "--cpus",
          "{{user `cpu_number`}}"
        ]
      ]
    }
  ],
  "provisioners": [
    {
      "type": "powershell",
      "scripts" : [
        "./scripts/install-selenium-server.ps1",
        "./scripts/install-chrome-firefox.ps1"
      ]
    },{
      "type": "shell",
      "remote_path": "/tmp/script.bat",
      "execute_command": "{{.Vars}} cmd /c C:/Windows/Temp/script.bat",
      "scripts": [
        "./scripts/vm-guest-tools.bat",
        "./scripts/vagrant-ssh.bat",
        "./scripts/rsync.bat",
        "./scripts/enable-rdp.bat"
      ]
    }
  ],
    "post-processors": [
    {
      "type": "vagrant",
      "keep_input_artifact": false,
      "output": "{{user `vm_name`}}_{{.Provider}}.box",
      "vagrantfile_template": "vagrantfile-template"
    }
    ]
}
~~~

#### Additional Software

This is not done here. Obviously, in order to test your stuff, you first need to install your software on this box. Ideally, everything you need should be in the code you clone to this box, and should be contained mostly. And your application deployment should take core of that. But, if you require something like a DB, postgres, oracle, whatnot, than this is the place where you would install all that.

# Vagrant and Using the Packer Box

Now, this has been interesting so far, but how do you actually go about using this image? That's the real question now, isn't it? Having a box, just sitting on a shared folder, doesn't do you too much good. So let's create a Jenkins job, which utilizes this box in a job which runs a bunch of tests for some application.

#### Vagrantfile

Your vagrant file, could either be generated automatically, under source control ( which is preferred ) or sitting somewhere entirely elsewhere. In any case, it would look something like this.

~~~ruby

# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.provider "virtualbox"

  config.vm.define "selenium-box" do |vs2013|
    vs2013.vm.box = "windows7-x64-04-selenium"
    vs2013.vm.box_url = "path/to/your/share/win7x64_selenium_virtualbox.box"
  end

  config.env.enable

  config.vm.guest = :windows
  config.vm.communicator = "winrm"
  config.winrm.username = "vagrant"
  config.winrm.password = "vagrant"
  config.windows.set_work_network = true
  config.vm.network :forwarded_port, guest: 3389, host: ENV['RDESKTOP_PORT'], host_ip: "0.0.0.0"
  config.vm.network :forwarded_port, guest: 5985, host: 5985, id: "winrm", auto_correct: true, host_ip: "0.0.0.0"
  config.vm.network :forwarded_port, guest: 9991, host: 9991, id: "selenium", auto_correct: true, host_ip: "0.0.0.0"
  config.vm.provider :virtualbox do |vbox|
    vbox.gui = false
    vbox.memory = 4096
    vbox.cpus = 2
  end


  config.winrm.max_tries = 10
  config.vm.synced_folder ".", "/vagrant", type: "rsync"
  config.vm.provision "shell", path: "init.bat"
  config.vm.provision "shell", path: "utils_inst.bat"
  config.vm.provision "shell", path: "jenkins_reg.ps1"
  config.vm.provision "shell", path: "start_selenium.bat"
end
~~~

Easy, no? Here is the script to start selenium.

~~~
java -jar c:\Windows\Temp\selenium-server.jar -Dhttp.proxyPort=9991
~~~

Straight forward. We also are forwarding the port on which Selenium is running in order for the test to see it.

#### The Jenkins Job

The job can be anything. This is actually too large to cover here. It could be a gradle job, a maven job, an ant, a nant - or whatever is running the test -, job; it's up to you.

Just make sure that before the test runs, do a **vagrant up** and after the test finishes, in an ALWAYS TO BE EXECUTED HOOK -like gradle's finalizedBy , call a **vagrant destroy**. This way, your test will always run on a clean instance that has the necessary stuff on it.

# Closing words

So, there you have it. It's relatively simple. Tying this all into your infrastructure might prove difficult though depending on how rigid your deployment is. But it will always help you make your tests a bit more robust.

Also, you could run the whole deployment and test phase on a vagrant box, from the start, which is tied to jenkins as a slave and gets started when the job starts and destroyed when the job ends. That way you wouldn't have to create a, box in a box running on a box, kind of effect.

Thanks for reading,

Gergely.