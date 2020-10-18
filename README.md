# nextcloud-timestamp-PDFs

[Draft Notes]

# Objective
* Monitor a folder for new PDF files
* Add a header to new PDFs containing the original uploaded filename and the date and time of the upload

# Prerequisites
* Nextcloud 
* Ubuntu 18.0.4
  * inotify-tools
  * snapd
  * enscript, ps2pdf
  * pdftk (via snap)

# Setup
* Move Nextcloud data to /media
  
  Snap does not allow access to files and folders outside of /home or /media.  
  
  In my case I needed to move my Nextcloud NFS data folder from /data to /media/data. Other notes on moving the data folder can be found online.
  * Move the NFS mount point
    ```sudo -i
    alias occ='sudo -u www-data php /var/www/nextcloud/occ'
    mkdir /media/data
    chattr +i /media/data
    occ maintenance:mode --on
    umount /data
    #
    # edit /etc/fstab
    # - <nfs_server>:/<nfs_export> /data nfs4 rsize=8192,wsize=8192,timeo=14,intr
    # + <nfs_server>:/<nfs_export> /media/data nfs4 rsize=8192,wsize=8192,timeo=14,intr
    #
    mount /media/data
    
    #
    # replace the old /data folder with a symlink
    # (I have lots of saved code snippets and possibly scripts that look for Nextcloud data in /data...)
    chattr -i /data
    rmdir /data
    ln -s /media/data /
    ````

  * Reconfigure Nextcloud
    ```#
    # edit <NC>/config/config.php
    # -   'datadirectory' => '/data',
    # +   'datadirectory' => '/media/data',
    ```
    
  * Test
  
    ```occ maintenance:mode --off```
    
    * confirm that Nextcloud is working correctly
      
* Install and configure prerequisites
  * inotify-tools
    ```inotify``` will be used to monitor the upload folder for new files
    [still untested]
    
    ```
    apt install inotify-tools
    ```
    
  * enscript
    ```enscript``` will be used to create postscript containing the text to be stamped on uploads
    
    ```
    apt install enscript
    ```
    
  * ps2pdf
    ```ps2pdf``` is used to convert the postscript created by enscript into the PDF required by pdftk.
    
    ```ps2pdf``` was already installed on my system.  If you don't have it, install ```ghostscript```
    
    ```
    apt install ghostscript
    ```
    
  * snapd
    ```snap``` is the only clearly-defined method of installing pdftk on ubuntu 18.04.  Extensive searching found other install methods but none seemed simple or well documented (to me...)
    
    ```
    apt install snapd
    ```
    
  * pdftk
    ```pdftk``` can perform many operations on PDF files, including applying a background to an existing (transparent) PDF or applying a stamp in the foreground.
    
    ```
    # install pdftk
    snap install pdftkIN=/media/data/mmccarn/files/Temp/Techsoup\ 2017-07-17.pdf
    
    # posts seen online indicate this is needed to access files outside of /home...
    sudo ln -fs /snap/pdftk/current/usr/bin/pdftk /usr/bin/pdftk
    
    # and this allows pdftk to "see" files in the /media folder
    snap connect pdftk:removable-media
    ```

# The commands...

```
# The file to be scanned
IN=/media/data/mmccarn/files/Temp/Techsoup\ 2017-07-17.pdf

# The temp filename to contain the stamp
# needs to be in /media/ so the pdftk snap can "see" it.
sudo -u www-data mkdir -p /media/data/.tmp
STAMP=/media/data/.tmp/stamp$$.pdf

# The filename as it should be 'stamped' 
F=$(basename "${IN}")

# The directory to use for the output file
D=$(dirname "${IN})")

# create the temporary stamp pdf
# use the header (no color / filename at left / timestamp at right)
#echo "" |enscript --escapes=~ --margins=15:15:15:15 --header="$F| |Uploaded $(date)" -p - |sudo -u www-data ps2pdf - - >"${STAMP}"
# no header / blue text / filename <new line> timestamp
printf "~color{0 0 1}${F/.[pP][dD][fF]/_stamped.pdf}\n$(date)" |enscript --escapes=~ --margins=5:5:5:5 --no-header -p - |sudo -u www-data ps2pdf - - >"${STAMP}"


# apply the stamp to the input file and save the output as "..._stamped.pdf"
sudo -u www-data pdftk "${IN}" stamp "${STAMP}" output "${IN/.pdf/_stamped.pdf}" 

# Run occ files:scan on the input folder so NC "sees" the new file
occ files:scan --path=${D/\/media\/data\//}

# remove the temporary stamp file
rm "${STAMP}"
```

# TODO
* Work out how to incorporate inotify
* Create a script
* Ignore files where there is already a matching "..._stamped.pdf" file
* Get the color instructions working for headers, or figure out how to right-justify normal text
      
