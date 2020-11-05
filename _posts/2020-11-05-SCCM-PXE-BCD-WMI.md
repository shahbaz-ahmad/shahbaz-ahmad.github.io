---
layout: post
title:  SCCM PXE Boot Image BCD Class WMI Error 80041002
categories: [SCCM,PXE,BCD,Multicast]
---
Troubleshooting PXE and Boot image distribution error "Failed to connect to wmi class 'BcdStore' GenerateBootBcd() failed. 80041002".

Today I received an escalation that the operations team are not able to build computers over the network (Distribution Point, PXE & multicast). 

This service was critical as it relates to the business continuity environment of a major client. 

The service was broken after a SCCM infrastructure were upgrade from 1906 to 2002 version.

The operational teams had already tried uninstallation of WDS and distribution point, and reinstallation of these roles but the problem didn't resolved.

Upon troubleshooting, noticed that the boot image distribution after SCCM upgrade was failing.
Looking into the log files on distribution point, found the following error:

```
Failed to connect to wmi class 'BcdStore'	
GenerateBootBcd() failed. 80041002.	
Failed adding image G:\RemoteInstall\SMSImages\PRD007C9\boot.PRD007C9.wim. Will Retry.. Not found (Error: 80041002; Source: WMI)
```

![](/images/sccm/SCCM_PXE_BCD_80041002.png)

### Troubleshooting & Solution
The error in log file above is pointing to the BCD store classes in WMI repository. To check if these WMI classes are available, log on the distribution point and run the following PowerShell command:

```gwmi -name root\wmi -list bcd*```

The above command didn't return any output that means BCD classes were missing on the server.

To fix the issue, perform the following steps on the distribution point server:

1. Open addministrative command prompt at and change directory to ```C:\Windows\System32 ```
2. Run the following command to register bcdprov.dll:

    ```regsvr32 bcdprov.dll```

3. Run the following command to register bcdsrv.dll:

    ```regsvr32 bcdsrv.dll```

4. Change directory to ```C:\windows\system32\wbem```
5. Stop the WMI Service (make note of the dependent services it will stop also)

    ```net stop /y winmgmt``` 

6. Compile the bcd mof file so it creates the classes

    ```mofcomp bcd.mof```

7. Start the WMI service 
    ```net start winmgmt```

8. Start other services those were stopped in step 5 above

9. Start SCCM Services e.g. SMS Agent Host, SMS Executive 

Verify that the BCD classes are now present by running the PowerShell command:

```gwmi -name root\wmi -list bcd*```

The output will look similar to this:
![](/images/sccm/SCCM_PXE_BCD_Classes.png)

You can now distribute the boot images and verify that the error in the log file has disappeared. 

After the fix and successful distribution of boot images, users were able to PXE boot workstations and continue with the build.

