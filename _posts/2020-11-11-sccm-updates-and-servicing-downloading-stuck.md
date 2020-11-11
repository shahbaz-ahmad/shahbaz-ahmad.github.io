---
layout: post
title:  SCCM Updates and Servicing, Update stuck downloading state
categories: [SCCM,Updates and Servicing,Current Branch]
---
This post is about troubleshooting and fixing SCCM Current Branch update package downloading issues.

---

### Scenario
When trying to upgrade SCCM to the latest current branch version, the update is stuck in **downloading** state for many days.
![](/images/sccm/sccm_2006_downloadingstuck.png)

The operations team was unable to proceed with the SCCM current branch upgrade. 

*Although this post shows example of SCCM Current Branch 2006, but this fix should be applicable to any version with similar symptoms.*


### Step by Step Troubleshooting and Fix 
To troubleshooting downloading issues, the starting point should be reading **dmpdownloader.log** log file.

One of the error I found in the log was:

*Failed to download Admin UI content payload with sception: The remote server returned an error: (403) Forbidden.*

![](/images/sccm/sccm_2006_downloading_error403.png)

This lead to us to check if the Site Server can make a connection to the proxy server and is reachable to the internet. Our Site Server was behind a proxy and we only allowed specific URLs.

Upon checking, found out that the proxy authentication account was locked out. Once that was fixed, this error was disappeared.

However, the Configuration Manager update 2006 was still stuck in downloading state.

After going through the logs and checking the contents in **EasySetupPayload\a200e842-5cb5-4bd3-ab77-2999b59721ae** folder, found that setup files were already downloaded, but it is having problem downloading the redistributable (redist) files. This can also be verified by browsing to ```Program Files\Microsoft Configuration Manager\EasySetupPayload\a200e842-5cb5-4bd3-ab77-2999b59721ae\redist```

There was only couple of manifest cab files with 0 KB in size.
![](/images/sccm/sccm_2006_downloading_redist.png)

We can try to download *redist* files manually. First we have to make a command with appropriate parameters. 

1. Browse to the ```Program Files\Microsoft Configuration Manager\EasySetupPayload\a200e842-5cb5-4bd3-ab77-2999b59721ae\SMSSETUP\BIN\X64```. Where **a200e842-5cb5-4bd3-ab77-2999b59721ae** is the GUID of SCCM 2006 Upgrade.

2. Find the **manifest.xml** file and open in Notepad
3. Search and copy the following attributes values:
    ```
    <RedistManifestVersion>202006</RedistManifestVersion>
    <Redist ManifestUrl="https://go.microsoft.com/fwlink/?LinkID=2115626" />
    <LanguagePack ManifestUrl="https://go.microsoft.com/fwlink/?LinkID=2115542" />
    ```
4. Now put your values as parameters in the setupdl.exe command. Your command will become like this:

    ```setupdl.exe /RedistUrl https://go.microsoft.com/fwlink/?LinkID=2115626 /LnManifestUrl https://go.microsoft.com/fwlink/?LinkID=2115542 /RedistVersion 202006 "C:\Temp\SCCM_2006\redist"```

    *C:\Temp\SCCM_2006\redist is the directory in which redistributable files will be downloaded.*

5. Open administrative command prompt, change directory path to ```Program Files\Microsoft Configuration Manager\EasySetupPayload\a200e842-5cb5-4bd3-ab77-2999b59721ae\SMSSETUP\BIN\X64``` and run the command: <br />
```setupdl.exe /RedistUrl https://go.microsoft.com/fwlink/?LinkID=2115626 /LnManifestUrl https://go.microsoft.com/fwlink/?LinkID=2115542 /RedistVersion 202006 "C:\Temp\SCCM_2006\redist"```

Running this command on the Site Server returned an error that it cannot download contents. This verified that the Server is not able to download through Proxy due to connection being blocked. 

You can further verify if all of the URLs are whitelisted as per Microsoft documentation <a href="https://docs.microsoft.com/en-us/mem/configmgr/core/plan-design/network/internet-endpoints" target="_blank">here</a>.

An alternate method is to download redist files on another machine that has internet connectivity e.g. a workstation. To do that:

- Copy the entire contents of the directory ```Program Files\Microsoft Configuration Manager\EasySetupPayload\a200e842-5cb5-4bd3-ab77-2999b59721ae\SMSSETUP\BIN\X64``` on the workstation which has internet connectivity (copying only setupdl.exe will not work)

- Open the administrative command prompt and change the directory path to ```Program Files\Microsoft Configuration Manager\EasySetupPayload\a200e842-5cb5-4bd3-ab77-2999b59721ae\SMSSETUP\BIN\X64``` 

- Run the command: <br/>
```setupdl.exe /RedistUrl https://go.microsoft.com/fwlink/?LinkID=2115626 /LnManifestUrl https://go.microsoft.com/fwlink/?LinkID=2115542 /RedistVersion 202006 "C:\Temp\SCCM_2006\redist"```

- You would see downloading dialog window

- Once the downloading is finished, copy the entire contents of``` C:\Temp\SCCM_2006\redist``` and paste in ```Program Files\Microsoft Configuration Manager\EasySetupPayload\a200e842-5cb5-4bd3-ab77-2999b59721ae\redist``` on the Site Server

- Now to force SCCM to pick up the newly downloaded files quickly either restart the Server, or SMS_EXEC service or only SMS_DMPDownloader thread.

- To restart SMS_DMPDownloader thread, open the SCCM Console and go to *Monitoring \ System Status \ Site Status*

- On the ribbon menu, click Start and open **Configuration Manager Service Manager**
![](/images/sccm/sccm_2006_service_manager.png)

- Search for **SMS_DMP Downloader** thread
- Click the *Query* button ( ! ) this will tell you that the thread is running
- Click the *Stop* button to stop the thread
- Click the *Query* button ( ! ) again to find if thread is stopped
- Click the *Start* button to start the thread. To find if successfully started, click the *Query* button ( ! ) again
![](/images/sccm/sccm_2006_sms_dmp_downloader.png)

- Monitor the **dmpdownloader.log** log file. You will see that the thread was stopped/started and information is being process. It will generate a state message for package a200e842-5cb5-4bd3-ab77-2999b59721ae and create a notification file. 

- In few minutes, you will see that the status of the SCCM 2006 release will change from Downloading to **Ready to Install** state
![](/images/sccm/sccm_2006_readytoinstall.png)


The above fix worked in my customer(s) environment where Proxy was blocking the connections partially. 

However, in addition to the above, sometimes you may need to reset the update package by using **CMUpdateReset.exe** utility where Proxy is not an issue.

More about this later as it is time for a coffee braak now !!
