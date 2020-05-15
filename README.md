# Citrix-Fedora32
Notes on getting Citrix Workspace to work on Fedora32

# Citrix Workspace client on Fedora 32 

I have recently started working on a system where I need to connect to Citrix Workspace to access the systems.

Navigating to the Citrix download site here:
https://www.citrix.com/en-gb/downloads/workspace-app/
you can find a number of packages, of which one is an RPM packaged version for RedHat style distributions.

Downloading this onto my Fedora 32 machine and installing the RPM, it has created a link to the wfica client which you can see with the following:

```
$ xdg-mime query default application/x-ica
wfica.desktop
```

However - when I launched the Citrix Desktop this was failing silently, and it turned out to be a core dump.

```
$ coredumpctl info 13980
           PID: 13980 (wfica)
           UID: 1000 (chris)
           GID: 1000 (chris)
        Signal: 11 (SEGV)
     Timestamp: Wed 2020-05-13 19:16:44 BST (2h 37min ago)
  Command Line: /opt/Citrix/ICAClient/wfica -icaroot /opt/Citrix/ICAClient /tmp/mozilla_chris0/<redactedid>.ica
  ```

  Cue a lot of searching and I found this blog post with some great hints: https://ask.fedoraproject.org/t/howto-citrix-receiver-on-fedora-30/1274/21 

  The summary would be that you need to make sure that you have the right certifcates and also the Citrix workspace client only works with an older version of libssl (no comment on security of this). So if you also need to get this working then hopefully this will help.

  ## Steps to make things work

  * Install the Citrix Workspace App RPM from the citrix download pages above.
  *  The version of libcrypto that you need is 1.0.2o based, so I used the following command to install the older compat library 
  ```
  $ sudo dnf install compat-openssl10*
  ```
  This results in the install of the following:
  ```
  $ dnf search compat-openssl*
...
compat-openssl10.x86_64 : Compatibility version of the OpenSSL library
compat-openssl10.i686 : Compatibility version of the OpenSSL library
compat-openssl10-devel.x86_64 : Files for development of applications which have to use OpenSSL-1.0.2
compat-openssl10-devel.i686 : Files for development of applications which have to use OpenSSL-1.0.2
  ```
  * Once I have the correct library, I now need to force this to be used. The ICA file for me is downloaded by Firefox when I tried (and failed) to launch it. So I am going to re-launch the file from the command line using LD_PRELOAD to over-ride the library to the older compat version above:
  ```
  $ export LD_PRELOAD="/lib64/libcrypto.so.1.0.2o"
  /opt/Citrix/ICAClient/wfica /tmp/mozilla_chris0/<redactedid>.ica 
```
That was enough to get the citrix receiver to now run.

Addeddum - how to get a different resolution as the Citrix receiver was consuming all 3 of my screens. To force a different geometry use:

```
-geometry 3840x2160
```
As an additional command line flag to the wfica command (assuming you also have a 4K screen).

## Additional Steps if you need a custom certificate

If you also need a custom certificate, then you will also need to follow these steps replacing the DigiSign cert I needed with the one you are missing.

* First step is to identify the certificate from the Error message - Mine was missing the Digi Cert Global Root G2 it complained. So I found the PEM file for this here: https://cacerts.digicert.com/DigiCertGlobalRootG2.crt.pem 

* Now that I have the pem file, this needs to be copied (as root) to the following location:
```
cp <cert.pem> /opt/Citrix/ICAClient/keystore/cacerts/
```
You also need to generate some internal data for citrix to use this certificate (as per Citrix article CTX231524) and you need to run the following command:
```
/opt/Citrix/ICAClient/util/ctx_rehash
```
Once I had these both in place, the custom certificate and the older library, I could launch the Citrix Workspace App on Fedora 32.

