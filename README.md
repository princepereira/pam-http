Intro
=====

This module is heavily inspired by the pam-http module by Kragen Sitaker. I rewrote it largely because I wanted to MIT license it (instead of GPL) and because there was some profanity in the source.  Also, the version I modeled this off of didn't even compile because it used an old version of libcurl.

This works with libcurl v. 7.21.3 (the one in Ubuntu's repositories).

To build, just type `make`. It will create `mypam.so` and a `test` executable.

Simple Usage
------------

The .so file should be put in `/lib/security` and the PAM config files will need to be edited accordingly.

The config files are located in `/etc/pam.d/` and the one I changed was `/etc/pam.d/common-auth`. This is NOT the best place to put it, as sudo uses this file and you could get unexpected results (like an HTTP user could gain root access; cool huh?).

Put something like this in one of the config files (change the URL to whatever you like):

	auth sufficient mypam.so url=https://localhost:2000
	account sufficient mypam.so

Sufficient basically means that if this authentication method succeeds, the user is given access.

To test, run the test program with a single argument, the username. I have provided a sample HTTPS server (you'll need your own certificate) that will accept all usernames and passwords. This module does not check the validity of certificates, so a custom one will do.


Steps
-----


sudo apt-get install -y libpam-modules libpam0g-dev libcurl4-nss-dev git telnet net-tools python-dev python-pip rsyslog
git clone https://github.com/princepereira/pam-http.git
cd pam-http
make
cp pam-http/mypam.so /lib/x86_64-linux-gnu/security/.


# vi /etc/ssh/sshd_config

UsePAM yes
PasswordAuthentication yes
ChallengeResponseAuthentication yes
PermitRootLogin yes
PubkeyAuthentication no


vi /etc/pam.d/sshd

// comment following line
# session    required     pam_loginuid.so

Vi /etc/pam.d/common-auth
#==============================================================#
auth sufficient mypam.so url=http://localhost:2000
account sufficient mypam.so
#==============================================================#


#----------------------------------------#
#               Python Server            #
#----------------------------------------#

import SocketServer
import SimpleHTTPServer
import re
import os, crypt, time

from base64 import b64decode, b64encode

PORT = 2000

def createUser(name,username,password):
    encPass = crypt.crypt(password,"22")   
    return  os.system("useradd -p "+encPass+ " -s "+ "/bin/bash "+ "-d "+ "/home/" + username+ " -m "+ " -c \""+ name+"\" " + username)

class CustomHandler(SimpleHTTPServer.SimpleHTTPRequestHandler):

    def do_GET(self):
        stri = self.headers.getheader('Authorization')
        upEncoded = stri.split('Basic ')[1]
        upDecoded = b64decode(upEncoded).decode()
        unSplitted = upDecoded.split(":")
        uname = unSplitted[0]
        pwd = unSplitted[1]
        print "Username " ,uname
        print "Password " ,pwd
        createUser(uname,uname,pwd)
        time.sleep(5)
        print "New user ",uname," created.!!!"
        self.send_response(200)
        self.send_header('Content-type','text/html')
        self.end_headers()
        return

httpd = SocketServer.ThreadingTCPServer(('', PORT),CustomHandler)

print "serving at port", PORT
httpd.serve_forever()

=================================


Ref: https://github.com/beatgammit/pam-http


