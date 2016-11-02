# BIOS Sledgehammer
Automated BIOS update, TPM firmware update and BIOS settings for HP devices.

```
            _
    jgs   ./ |   BIOS Sledgehammer 
         /  /      
       /'  /     Copyright © 2016 Michael 'Tex' Hex  
      /   /      
     /    \      https://github.com/texhex/BiosSledgehammer
    |      ``\     
    |        |                                ___________________
    |        |___________________...-------'''- - -  =- - =  - = `.
   /|        |                   \-  =  = -  -= - =  - =-   =  - =|
  ( |        |                    |= -= - = - = - = - =--= = - = =|
   \|        |___________________/- = - -= =_- =_-=_- -=_=-=_=_= -|
    |        |                   `` -------...___________________.'
    |________|      
      \    /     
      |    |     This is *NOT* an official HP tool.                         
    ,-'    `-,   This is *NOT* sponsored or endorsed by HP.
    |        |   Use at your own risk. 
    `--------'    

```
ASCII banner from: http://chris.com/ascii/index.php?art=objects/tools

## <a name="warning">Disclaimer</a> 

* BIOS Sledgehammer is **NOT** an official HP tool.                         
* This is **NOT** sponsored or endorsed by HP.
* HP was **NOT** involved in developing BIOS Sledgehammer.
* You computer can become [FUBAR](https://en.wikipedia.org/wiki/List_of_military_slang_terms#FUBAR) in the process. 

## <a name="about">About</a>

<!--
If you deploy the devices you use in your enterprise automatically (e.g. MDT or SCCM), changes are good that somewhere in that automatic process you also have one step to automatically update the BIOS firmware. 

This is normally plain easy: If model/type is of that type X, execute ``HPBIOSUPDREC64.exe -params X`` and you are done. Except if the BIOS you deployed was buggy. And HP fixed it already on newer devices. But your script didn’t check the installed BIOS version so you downgraded all devices to the faulty BIOS version. Woops. So, you create a new script, try to parse the BIOS version and roll it out. Just to note three weeks that another BIOS version is delivered that breaks your parsing and you can start over again....

The upcoming change to Windows 10 increases this complexity by several magnitudes:
-->

Suppose you get a workitem like this: 

> For the Windows 10 rollout, we need you to support ten different hardware models and all of them need to be updated to the newest BIOS version. Some devices require a TPM firmware update to use new features that depend on TPM 2.0. And of course, some BIOS settings like Secure Boot, Fast Boot etc. need to be changed as well. Oh, and a new BIOS password would also be a big plus because we currently have twenty different passwords in use. 

You can now waste precious life time to try to script this, or you can just use BIOS Sledgehammer:
* You can support several BIOS passwords for your devices, it will simply try all password you specify until the correct one is found
* You define which BIOS version the devices should have. Newer version of the BIOS will not trigger a downgrade and the BIOS version parsing works from a rather old 2570p up to a 1040 G3.
* Define which TPM firmware and/or Specification (1.2 or 2.0) the device should have. Of course, there are checks so BIOS Sledgehammer won’t try to flash “Update 6.40 to 7.41” on a device that has firmware 6.41
* The BIOS password can be set individual per model or you just set all devices to the same password.
* BIOS settings are changed individual so when something goes wrong, you know exactly what the problem was.

If this sounds good to you, see [Process](#process) how BIOS Sledgehammer works or download it from [Releases](https://github.com/texhex/BiosSledgehammer/releases).

## <a name="requirements">System requirements</a>

* PowerShell 4.0 or higher
* Windows 7 64-bit or Windows 10 64-bit - Windows 8 should also work, but wasn't tested
* [HP BIOS Configuation Utility](https://ftp.hp.com/pub/caps-softpaq/cmit/HP_BCU.html) (BCU) 4.0.15.1 stored in the folder ``\BCU-4.0.15.1``
* [BIOS Update](http://www.hp.com/drivers) files for the models you want to support
* [TPM Update files](http://h20564.www2.hp.com/hpsc/doc/public/display?docId=emr_na-c05192291) (Advisory with download link) if a TPM update is desired

## <a name="process">Process</a>

When starting BiosSledgehammer.ps1, the following will happen:

* A log file ``BiosSledgehammer.ps1.log-XX.txt`` is created, where XX is sequentially increased value with each run. See [Logfile](#logfile) for details.
* It checks if the environment is ready (64-bit OS, required folders found, device is from HP etc.).
* A check is made if communication between BCU (*BiosConfigUtility64.exe*) and the BIOS through WMI is possible by reading the value of *Universally Unique Identifier (UUID)*.
* It tries to figure out the password the device is using by going through all files in the [PwdFiles folder](#pwdfilesfolder) and trying to change the value of *Asset Tracking Number* to a random value (it will be reverted to the original value at the end). An empty password is always tried first.  
* A search is performed below the [Models folder](#modelsfolder) to locate the matching folder for the current model (this is a partial search, so a sub folder named *1040 G1* will match the model *HP EliteBook Folio 1040 G1*). All configuration is then read from this folder only. 
* If the file **BIOS-Update.txt** is found, it is read and checked if a BIOS update is required. If so, the BIOS update files are locally copied and the update is performed. Any **.log* file generated by the update tool is attached to the BIOS Sledgehammer log file.  Finally, a restart is requested because the actual update is performed during POST. See [BIOS Update](#biosupdate) for more details.
* If the file **TPM-Update.txt** exists, it is read and checked if a TPM update is required. This is done by checking if the TPM Specification version (1.2 or 2.0) or the TPM firmware are below the configured versions. If so, the TPM updates files are locally copied and executed. Any **.log* file generated by the update tool is attached to the BIOS Sledgehammer log file.  Finally, a restart is requested because the actual update is performed during POST. See [TPM Update](#tpmupdate) for more details.
* If the file **BIOS-Password.txt** is found, it is checked if the device is already set to use this password. The password is not specified directly (clear), but using a *.bin file name that stores the password encrypted. If the passwords differ, the configured *.bin file is read from the [PwdFiles folder](#pwdfilesfolder) and the password is changed. See [BIOS Password](#biospassword) for more details.
* If the file **BIOS-Settings.txt** exists, it is read and each entry is the name of a BIOS setting that needs to be changed. Each entry will be performed as single change (not all in a batch) to detect faulty settings more easily. See [BIOS Settings](#biossettings) for more details.

Return codes (exit code):

* 0 if everything was successful
* 3010 (ERROR_SUCCESS_REBOOT_REQUIRED) if a restart is required because of a BIOS or TPM update
* 666 if something didn't worked (Error) 

## <a name="configformat">Configuration files format</a>

BIOS Sledgehammer uses several configuration files that all follow the same ``NAME==VALUE`` syntax. There are saved as *.txt files to make it easy to change them with a text editor. 

```
# This is a comment and ignored
; So is this line. 
#And this.
;And this also.

# The general format is NAME==VALUE
Version==1.08

# Leading or trailing white spaces are ignored so this is the same as the line before
 Version == 1.08


# Empty line are ignored, add as many as you wish
```

## <a name="logfile">Logfile</a>

Each time BIOS Sledgehammer is executed, a new logfile with the pattern ``BiosSledgehammer.ps1.log-XX.txt`` is created - ``XX`` is sequentially increased so the first run can be found in ``BiosSledgehammer.ps1.log-01.txt``, the second run in ``BiosSledgehammer.ps1.log-02.txt`` and so on. The location where these files are stored depends on if MDT/SCCM is active:

* By default the logfile is saved to ``C:\Windows\Temp\``.

* If it is executed in a task sequence by MDT or SCCM, the task sequence variable ``LogPath`` is used: ``C:\MININT\SMSOSD\OSDLOGS\`` or ``C:\Windows\Temp\DeploymentLogs\``.


## <a name="pwdfilesfolder">*PwdFiles* folder</a>

The ``\PwdFiles`` folder stores all BIOS passwords that your devices might use. When BIOS Sledgehammer starts, it tries every file in this folder until the password for the device has been found (an empty password is automatically added, there is no file for this). If no password file matches, an error is generated. 

The order, in which they are tried, is determined by sorting the files by name: a file called *01_Standard.bin* is tried before *02_Standard.bin*. The most commonly used password should always come first because some BIOS versions enforce how many times you can try a wrong BIOS password. When this limit is reached, any password is rejected until the computer is restarted. 

To create these files, execute ``HPQPswd64.exe`` (found in the BCU folder) and save the file to the ``\PwdFiles`` folder as *.BIN file.  


## <a name="modelsfolder">*Models* folder</a>

It is expected that each model (type) of hardware you want to support, has a separate sub folder below ``\Models``. The model (type) is displayed automatically by BIOS Sledgehammer, so simply run it once for each hardware to know the model. 

The sub folder will contain all settings files together with the source files for any updates. Please note that BIOS Sledgehammer does not support “sharing” update files between several models, each model requires its own set of files. That’s because sharing files between models has proven to cause problems for older models each time the shared folders are updated for new models. 

To locate the model folder, a partial search with the current model is used. If you execute it on a ``HP EliteBook Folio 1040 G1``, the folder can be called exactly like that, or, if you are lazy, ``1040 G1``. 

Where this partial search help a lot is for models that are technical identical but have differnt model name. For example the *ProDesk 600 G1* comes in different form factors, each with a unique name: *HP ProDesk 600 G1 TWR* (Tower), *HP ProDesk 600 G1 SFF* (Small Format Factor) and so on. If you create one folder *HP ProDesk 600 G1* this folder will match all this form factors.  

If you do not want to change anything for a given model, simply create an empty folder. If no model folder at all is found, an error is generated. 

## <a name="biosupdate">BIOS Update</a>

The settings for a BIOS update need to be stored in the file ``BIOS-Update.txt`` for the given model. For example, if you execute it on a *HP EliteBook 850 G1* and the model folder is called *\Models\HP EliteBook 850 G1\*, the entire file path is *\Models\HP EliteBook 850 G1\BIOS-Update.txt*. Here is an example file for this device:

```
# 850 G1 BIOS Update

#The BIOS version the device should have
Version == 1.37

#Command to be executed for the BIOS update
Command==HPBiosUpdRec64.exe 

#Arguments to pass to COMMAND

# Silent
Arg1 == -s
# Do not restart automatically
Arg2 == -r
# Disable BitLocker during upgrade
Arg3 == -b
# Password file (parameter will be removed if device has no BIOS password)
Arg4 == -p"@@PASSWORD_FILE@@"
```
**Note**: BIOS Sledgehammer enforces that the source files are stored in a sub folder called ``BIOS-<VERSION>``. If the desired BIOS version is 1.37, the BIOS files need to be stored in ``\BIOS-1.37\``. Given that the current model folder is ``\Models\HP EliteBook 850 G1``, the entire path would be ``\Models\HP EliteBook 850 G1\BIOS-1.37``. 

The source folder is then copied to %TEMP% (to avoid any network issues) and the process is started from there. Because the update utility sometimes restarts itself, it is waited until the process noted in COMMAND does no longer appear as running process. If any **.log* file was generated in the local folder, the content is added to the normal BIOS Sledgehammer log. A restart is requested after that because the “real” update process happens during POST, after the restart. 

If anything goes wrong during the process, an error is generated. 

## <a name="tpmupdate">TPM Update</a>

The settings for a TPM update need to be stored in the file ``TPM-Update.txt`` for the given model. For example, if you execute it on a *HP EliteBook Folio 1040 G3* and the model folder is called *\Models\HP EliteBook Folio 1040 G3\*, the entire file path is *\Models\HP EliteBook Folio 1040 G3\TPM-Update.txt*. Here is an example file for this device:

```
# 1040 G3 TPM Update

# Manufacturer of the TPM. 
# If the value exists, the device must have this vendor or no update takes place
Manufacturer == 1229346816
#1229346816 is IFX

# The TPM Spec version we want this this device to have
SpecVersion == 2.0

# The Firmware version we want this device to have 
FirmwareVersion == 7.41

# Define the upgrade file to be used for each firmware
# The firmware active on the device must match an entry here or no upgrade can be performed
6.40 == Firmware\TPM12_6.40.190.0_to_TPM20_7.41.2375.0.BIN
6.41 == Firmware\TPM12_6.41.197.0_to_TPM20_7.41.2375.0.BIN
7.40 == Firmware\TPM20_7.40.2098.0_to_TPM20_7.41.2375.0.BIN

# Command to be used to perform the TPM firmware upgrade
Command == TPMConfig64.exe

#Arguments passed to COMMAND
Arg1 == -s
Arg2 == -f"@@FIRMWARE_FILE@@"
Arg3 == -p"@@PASSWORD_FILE@@"
```
The first setting **Manufacturer** is optional and can be used to ensure that the TPM firmware vendor for the device matches the update files. If it's not defined, the TPM firmware vendor is ignored. 

To detect if an TPM update is required, two versions need to be checked: The TPM Specification version (**SpecVersion**) and the firmware version (**FirmwareVersion**). 

The reason is that all TPM firmware is developed by 3rd parties so a change from TPM 1.2 to 2.0 can result in a LOWER firmware version when the vendor is changed (see [this article on the Dell wiki]( http://en.community.dell.com/techcenter/enterprise-client/w/wiki/11850.how-to-change-tpm-modes-1-2-2-0) – TPM Spec 1.2 is firmware 5.81 from WEC, TPM Spec 2.0 is firmware 1.3 from NTC). BIOS Sledgehammer checks both versions and if any of those two are higher than the current device reports, a TPM update is started. 

The current TPM firmware version of the device is retrieved and it is checked if the settings file contains an entry for this firmware version. Given that the current device has TPM firmware 6.40, the update can be performed as an entry for this version exists (**6.40 == Firmware\TPM12....**). However, if the device would have firmware 6.22 the update would fail because no entry for this version exists. 

As a TPM update also requires that BitLocker is completely turned off (as any BitLocker keys are lost during the upgrade), BIOS Sledgehammer will check if the system drive C: is encrypted with BitLocker and starts an automatic decryption before executing the update. This works for Windows 10, but fails in Windows 7 because the required BitLocker PowerShell module does not exist. 

Once this is all set and done, the source folder is copied to %TEMP% (to avoid any network issues) and the process is started from there.

**Note**: BIOS Sledgehammer enforces that the source files are stored in a sub folder called ``TPM-<VERSION>``. If the desired TPM firmware version is 7.41, the TPM files need to be stored in ``\TPM-7.41\``. Given that the current model folder is *\Models\HP EliteBook Folio 1040 G3*, the entire path would be *\Models\HP EliteBook Folio 1040 G3\TPM-7.41*. 

Because the update utility sometimes restarts itself, it is waited until the process noted in COMMAND does no longer appear as running process. If any **.log* file was generated in the local folder, the content of the log is added to the normal BIOS Sledgehammer log. A restart is requested after that because the “real” update process happens during POST, after the restart. 

If anything goes wrong during the process, an error is generated. 


## <a name="biospassword">BIOS Password</a>

To set a BIOS password, you define the password file (containing the desired password) in ``BIOS-Password.txt``. This file must be stored in the model folder. For example, if you execute it on a *HP EliteBook 850 G1* and the model folder is called *\Models\HP EliteBook 850 G1\*, the entire file path is *\Models\HP EliteBook 850 G1\BIOS-Password.txt*. Here is an example file:

```
# Use our standard password
PasswordFile == 01_W2f4x7t8NxD4xUH.bin
```
**IMPORTANT** This is insecure and just an example! Do not use the password itself as file name!   

This file has to be stored in the [PwdFiles folder](#pwdfilesfolder) (see this section how to create the files). If you want to use an empty password, just leave the value empty like this:
```
PasswordFile == 
```
Regarding BIOS passwords, please note the following:
* Passwords need to meet minimum complexity rules (must contain upper and lower case letters and a number) or the BIOS will reject the password change but won't issue any specific error message - it will simply return "Invalid password file". The only exception of this rule is an empty password which is always allowed.
* There are only some password changes allowed per power cycle. If the password change just doesn’t work although it has worked before, turn the device off and on.

## <a name="biossettings">BIOS Settings</a>

The configuration for BIOS settings need to be stored in the file ``BIOS-Settings.txt`` for the given model. For example, if you execute it on a *HP EliteBook 850 G1* and the model folder is called *\Models\HP EliteBook 850 G1\*, the entire file path is *\Models\HP EliteBook 850 G1\BIOS-Settings.txt*. Here is an example:
```
# 850 G1 
LAN/WLAN Switching == Enable

Intel (R) HT Technology == Enable

Floppy boot == Disable

Asset Tracking Number == @@COMPUTERNAME@@
```
Each entry simply lists how the BIOS setting is called, e.g. **LAN/WLAN Switching**, and to which value, e.g. **Enable** it should be configured. To get a complete list which BIOS settings the current device support, execute ``GetAllBiosSettings.bat`` (as Administrator) in the BCU folder. Please note that the list of available settings differs which each model and can also change when the BIOS is updated. Always first update the BIOS, then check the list of available settings.
 
Each setting is individual changed and not as batch so if any setting is wrong, it is very clear which setting is causing an issue. BIOS Sledgehammer also tries to “translate” the error code of BCU to an explanation what went wrong (as defined by the error code list from the BCU User Guide PDF).
 
A single replacement value is supported: *@@COMPUTERNAME@@*. If this value is detected, the value is replaced with the computer name of the device executing BIOS Sledgehammer. 

Please note that on some devices, some BIOS settings (e.g. TPM Activation Policy on a 2570p), require a password to be set or they can’t be activated. If this happens, BCU just fails with error 32769.

Also, BCU performs some basic checks to prevent changes that are not compatible with the current OS. For example, setting “SecureBoot” to “Enable” fails with error code 32769 when using Windows 7 but works directly on Windows 10. However, other settings are not checked so don’t count on it that BCU will prevent any incompatible change.


## <a name="sccmmdt">Using it from MDT or SCCM</a>

By default, MDT/SCCM will run all scripts hidden in order to hide any sensitive information. If you are okay with that, just run ``BiosSledgehammer.ps1`` as PowerShell script, but remember to tick the box for "Disable 64bit file system redirection" so it is run as 64-bit PowerShell process. This settings applies only for SCCM - MDT always runs PowerShell native.

If you want to see what BIOS Sledgehammer is doing, run the provided batch file ``RunVisble.bat`` with this command line in MDT/SCCM: ``cmd.exe /c "%SCRIPTROOT%\BiosSledgehammer\RunVisible.bat"`` (given you stored it in the *\Scripts* folder). 

This batch automatically uses the correct (native) version of PowerShell. It will also set the ``-WaitAtEnd`` parameter which causes BIOS Sledgehammer to pause for 30 seconds when finished so you can have a quick look at the results.

It is recommended to start BIOS Sledgehammer **four** times and restart the device after each run. If a device requires a BIOS Update, a TPM update and BIOS setting changes, three executions are needed. The final one is to make sure everything really worked.    


## <a name="contributions">Contributions</a>
Any constructive contribution is very welcome! 

If you encounter a bug, please start BIOS Sledgehammer with the option -Verbose (``.\BiosSledgehammer.ps1 -Verbose``) and attach the logfile to [new issue](https://github.com/texhex/BiosSledgehammer/issues/new).

## <a name="license">License</a>
Copyright © 2016 [Michael Hex](http://www.texhex.info/). Licensed under the **Apache 2 License**. For details, please see LICENSE.txt.



<!--  
TPM Information

* [Dell: TPM 1.2 vs. 2.0 Features](http://en.community.dell.com/techcenter/enterprise-client/w/wiki/11849.tpm-1-2-vs-2-0-features)
* [Dell: Get TPM information using PowerShell](http://en.community.dell.com/techcenter/b/techcenter/archive/2015/12/09/retrieve-trusted-platform-module-tpm-version-information-using-powershell)
* [Dell: How to change TPM Modes](http://en.community.dell.com/techcenter/enterprise-client/w/wiki/11850.how-to-change-tpm-modes-1-2-2-0)
* http://www.dell.com/support/article/de/de/debsdt1/SLN300906/en
* http://h20564.www2.hp.com/hpsc/doc/public/display?docId=emr_na-c05192291
  #>
-->