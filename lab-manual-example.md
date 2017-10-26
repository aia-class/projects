
# Forensics 101
#### Evidence collection on a compromised Windows Server

We will follow a sound forensics process to extract evidence from a running Windows 2003 server.  This old OS was selected because it only needs 384MB of RAM to run and has a tiny 4GB hard drive.  This makes collecting the evidence much faster.  We are also using a custom built incident response tools CD-Rom because it has trusted Windows binaries that are compatible with this older compromised server.  For our forensics workstation, we are using the [SANS SIFT](https://digital-forensics.sans.org/community/downloads#overview) linux distribution built on Ubuntu 16.04.  

#### Learning Objectives

1. Following the order of volatility, use trusted forensic tools to collect digital evidence on a compromised live Windows server.
1. Transfer evidence to a forensics workstation and use specialized tools to analyze evidence. 

#### Prepare Forensics Workstation
The first thing we must do is prepare the SIFT system to receive the evidence.  We will start by setting our IP address, and then configuring a ncat server to receive evidence over the network.  

Now, click on the **SIFT** button at the top of the screen to open a console.  Next, login to SIFT with Username: **examiner**  Password: **tartans@1**.  Right click on the desktop and select *Open Terminal* and then type the following commands after the *$* prompt.

    $ mkdir evidence    
    $ sudo service network-manager start
    $ sudo ip addr add 10.0.0.1/24 dev ens32
    $ nc -l -p 8001 >./evidence/memdump.dd &
    $ nc -l -p 8002 >./evidence/mod_times.txt &
    $ nc -l -p 8003 >./evidence/access_times.txt &
    $ nc -l -p 8004 >./evidence/create_times.txt &
    $ nc -l -p 8005 >./evidence/fred.txt &
    
#### Collect Evidence on Compromised Server

Now, click on the **Windows-Compromised** button at the top of the screen to open a console.  If locked, click the **Cog** icon on the top left of the console window and select **Ctrl Alt Del**.  Login with Username: **jsmith**  Password: **tartans**.  

Now Click *Start* -> *Run* and type:

     D:\Shells\cmdxp.exe

From the trusted command shell, type the below commands after the **>** prompt:

     > netsh interface ip set address "Local Area Connection 2" static 10.0.0.2 255.255.255.0
     > cd ..\ir\wft
     > dd.exe  if=\\.\PhysicalMemory | d:\ir\cygwin\nc.exe 10.0.0.1 8001

Wait a few minutes while the 384MB of memory are piped over the netcat connection.  When you see the **records in /records** out lines, type **Ctrl C** to return to the command prompt.  Now, let's grab some Modified, Accessed, and Created times from the server's filesystem.  Hit the *Up Arrow* button to show the previous command and edit it to match this:

     > dir c:\ /A /S /T:W | d:\ir\cygwin\nc.exe 10.0.0.1 8002

Wait about 1 minute, Hit the *Up Arrow* button again and change to:

    > dir c:\ /A /S /T:A | d:\ir\cygwin\nc.exe 10.0.0.1 8003

Wait about 1 minute, Hit the *Up Arrow* button again and change to:

    > dir c:\ /A /S /T:C | d:\ir\cygwin\nc.exe 10.0.0.1 8004

Wait about 1 minute, Hit the *Up Arrow* button again and change to:

    > type d:\ir\fred.bat

Scroll up on the command window to inspect the contents of the FRED script.  FRED stands for First Responders Evidence Disk.  It was created by the US Air Force CERT and still works well for older versions of Windows.  Several newer collection scripts exist like [IR-Rescue](https://github.com/diogo-fernan/ir-rescue), however FRED is suitable for this example. Now let's run the script and pipe the contents over netcat to complete our live evidence collection on this Windows server.  Type the below command to execute a FRED helper script that automates the netcat transfer.

    > d:\ir\call-fred-nc.bat 10.0.0.1 8005

Let's wait a few minutes for the script to run and for the output to reach our forensics workstation.  You can check its progress by opening a console on the SIFT machine and then repeatedly running ls-al on the evidence directory until the file size stops growing.

Lastly, we will collect a live disk image of the compromised server.  Normally, it is preferable to collect a "dead" image by shutting the system down and booting into a 

####Inspect Evidence on Forensics Workstation

Click on the SIFT Console tab in the top of your browser.  You will probably have to refresh the page to reconnect to this console.

From a terminal shell, type the below commands to open the evidence directory, list its contents, and then create MD5 hashes of the evidence files.  You will also merge the collected file system MAC times into a single file and then run strings on the memdump.dd file to extract ascii text from the server's RAM.  Finally, you will inspect the evidence files.

     $ cd evidence
     $ ls -al
     $ md5deep ./* > evidence_hashes.txt
     $ cat mod_times.txt > mac_times.txt
     $ cat access_times.txt >> mac_times.txt
     $ cat create_times.txt >> mac_times.txt
     $ strings memdump.dd >./memstrings.txt
     $ more evidence_hashes.txt     
     $ more mac_times.txt
     $ more fred.txt
     $ more memstrings.txt
