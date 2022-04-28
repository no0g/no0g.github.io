---
layout: post
title: "Backend Writeup/Walkthrough - HackTheBox"
comments: true
description: "Writeup for HackTheBox's Backend"
tags: "HackTheBox CTF Writeups Linux Pentesting"
---

# Backend
![[]](/assets/backendHTB/Pasted image 20220429042950.png)


# Initial Recon
Let's start with the basic Nmap scan 
## Nmap
- Command
  `nmap -A -T5 -oN nmap.txt -vv 10.10.11.161`
- Result
```shell
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ea:84:21:a3:22:4a:7d:f9:b5:25:51:79:83:a4:f5:f2 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDZBURYGCLr4lZI1F55bUh/6vKCfmeGumtAhhNrg9lH4UNDB/wCjPbD+xovPp3UdbrOgNdqTCdZcOk5rQDyRK2YH6tq8NlP59myIQV/zXC9WQnhxn131jf/KlW78vzWaLfMU+m52e1k+YpomT5PuSMG8EhGwE5bL4o0Jb8Unafn13CJKZ1oj3awp31fRJDzYGhTjl910PROJAzlOQinxRYdUkc4ZT0qZRohNlecGVsKPpP+2Ql+gVuusUEQt7gPFPBNKw3aLtbLVTlgEW09RB9KZe6Fuh8JszZhlRpIXDf9b2O0rINAyek8etQyFFfxkDBVueZA50wjBjtgOtxLRkvfqlxWS8R75Urz8AR2Nr23AcAGheIfYPgG8HzBsUuSN5fI8jsBCekYf/ZjPA/YDM4aiyHbUWfCyjTqtAVTf3P4iqbEkw9DONGeohBlyTtEIN7pY3YM5X3UuEFIgCjlqyjLw6QTL4cGC5zBbrZml7eZQTcmgzfU6pu220wRo5GtQ3U=
|   256 b8:39:9e:f4:88:be:aa:01:73:2d:10:fb:44:7f:84:61 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJZPKXFj3JfSmJZFAHDyqUDFHLHBRBRvlesLRVAqq0WwRFbeYdKwVIVv0DBufhYXHHcUSsBRw3/on9QM24kymD0=
|   256 22:21:e9:f4:85:90:87:45:16:1f:73:36:41:ee:3b:32 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEDIBMvrXLaYc6DXKPZaypaAv4yZ3DNLe1YaBpbpB8aY
80/tcp open  http    syn-ack ttl 63 uvicorn
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, GenericLines, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     content-type: text/plain; charset=utf-8
|     Connection: close
|     Invalid HTTP request received.
|   FourOhFourRequest: 
|     HTTP/1.1 404 Not Found
|     date: Thu, 28 Apr 2022 22:24:00 GMT
|     server: uvicorn
|     content-length: 22
|     content-type: application/json
|     Connection: close
|     {"detail":"Not Found"}
|   GetRequest: 
|     HTTP/1.1 200 OK
|     date: Thu, 28 Apr 2022 22:23:49 GMT
|     server: uvicorn
|     content-length: 29
|     content-type: application/json
|     Connection: close
|     {"msg":"UHC API Version 1.0"}
|   HTTPOptions: 
|     HTTP/1.1 405 Method Not Allowed
|     date: Thu, 28 Apr 2022 22:23:55 GMT
|     server: uvicorn
|     content-length: 31
|     content-type: application/json
|     Connection: close
|_    {"detail":"Method Not Allowed"}
| http-methods: 
|_  Supported Methods: GET
|_http-title: Site doesn't have a title (application/json).
|_http-server-header: uvicorn
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-...
```
From the scan, we can see that there is an SSH server and HTTP server running. The HTTP server runing on port 80 works like an API which returns JSON value everytime we send a request.
```bash
# HTTP Server Responses based on Nmap Scanning
|   FourOhFourRequest: 
|     HTTP/1.1 404 Not Found
|     date: Thu, 28 Apr 2022 22:24:00 GMT
|     server: uvicorn
|     content-length: 22
|     content-type: application/json
|     Connection: close
|     {"detail":"Not Found"}
|   GetRequest: 
|     HTTP/1.1 200 OK
|     date: Thu, 28 Apr 2022 22:23:49 GMT
|     server: uvicorn
|     content-length: 29
|     content-type: application/json
|     Connection: close
|     {"msg":"UHC API Version 1.0"}
|   HTTPOptions: 
|     HTTP/1.1 405 Method Not Allowed
|     date: Thu, 28 Apr 2022 22:23:55 GMT
|     server: uvicorn
|     content-length: 31
|     content-type: application/json
|     Connection: close
|_    {"detail":"Method Not Allowed"}
```

## Content Discovery
### Dirbuster 
I used old-fashioned dirbuster because my machine was newly downloaded Kali VM and I haven't installed the lovely `gobuster` or `wfuzz`.

![[]](/assets/backendHTB/Pasted image 20220429013632.png)   
From dirbuster, we know that we have:   
```bash
/api/
/docs/
/api/v1/
```
#### Trying to access documentation
Going to `/docs` will tell us that there is some kind of authentication in this API.   
![[]](/assets/backendHTB/Pasted image 20220429014411.png)   
This means, in the API itself there's a user management system that involves authentication and possibly also a registration system.  
### Trying to access the API itself
![[]](/assets/backendHTB/Pasted image 20220429014948.png)   
From this response, we can see that there are two possible endpoints which are `/user` and `/admin`   
![[]](/assets/backendHTB/Pasted image 20220429015601.png)   
Fuzzing it deeper give us:   
``` bash
/admin/file

and
# every valid number returns 200
/user/{valid number}
```
We can assume that the `/user` endpoint will return user based on `id` which is represented by a number. We can find a valid user id by filtering out a small-sized and repetitive response (in this case it's 4 and 104) from `gobuster`.    

![[]](/assets/backendHTB/Pasted image 20220429020111.png)   
As expected, the valid user returns with size 141 and we have found another endpoint which is `/cgi-bin`. We can validate this finding by accessing each user data (apparently it does not require authentication)   
![[]](/assets/backendHTB/Pasted image 20220429020311.png)   
We still havent found anything related to authentication so we will try fuzzin once again with POST request instead of GET   
![[]](/assets/backendHTB/Pasted image 20220429022107.png)   
**Finally**, We found `/login` and `/signup` accessible through `POST` request.   

### Signing Up New User
Assuming that the signup will require email and password. We can register a new user.   
`curl -XPOST http://10.10.11.161/api/v1/user/signup -H "Content-Type: application/json" -d '{"email":"ejak@ejak.ejak","password":"ejak"}'`   
![[]](/assets/backendHTB/Pasted image 20220429024710.png)   
### Login as Our User
![[]](/assets/backendHTB/Pasted image 20220429025211.png)   
For no absolute reason, we can't login using a json request but we can login using normal POST url form format. Now, using this token, we can access the documentation and most probably find a way to exploit the API.   

## Authenticated Enum
Using *Modify Header Value* extension in firefox, I was able to use the token and freely browse the API from browser. The API is a FastAPI instance, a Python-based API.   

![[]](/assets/backendHTB/Pasted image 20220429025900.png)   

From the docs, there is this one endpoint called `/api/v1/user/SecretFlagEndpoint` which is accessible with `PUT` request.   
![[]](/assets/backendHTB/Pasted image 20220429030301.png)   
Surprisingly, this returns our `user.txt` :).   

### Interesting Endpoint
![[]](/assets/backendHTB/Pasted image 20220429030557.png)   
The next interesting endpoint is the `updatepass` functionality. This endpoint requires user to be authenticated and provide `guid` value. If there is no validation done to verify the authenticated user and the given guid, we can easily update admin password by providing admin user's guid.   

### Updating Admin Pass
With the exploitation idea explained above, we can change admin password and get access to the admin functionality.   
![[]](/assets/backendHTB/Pasted image 20220429031312.png)   

# Exploitation (Initial Foothold)
After successfully authenticated as admin, we can use the token to execute arbitrary code using the `/exec` endpoint accessible only for admin.   
![[]](/assets/backendHTB/Pasted image 20220429031547.png)   
Apparently, we need a 'debug' value inside our JWT to be able to execute this functionality. To do this we need to find the JWT secret to modify our JWT using the `/file` functionality to read files.   

## Finding JWT Secret
We can find the root directory of the API by accessing the environment variable stored in `/proc/self/environ`.   
![[]](/assets/backendHTB/Pasted image 20220429033248.png)   
From this, we can see the root directory is `/home/htb/uhc` and the main function is inside `app/main.py`.   
![[]](/assets/backendHTB/Pasted image 20220429033641.png)   
From `main.py` we know that it is importing `app/deps.py` to do `deps.parse_token` . Inside `/deps.py`, we can see that the JWT secret is imported from `settings` (`/app/core/config.py`)    
![[]](/assets/backendHTB/Pasted image 20220429034119.png)   
And here it is, **The Secret**   
![[]](/assets/backendHTB/Pasted image 20220429034221.png)      
So, let `PyJWT` do the magic.   
![[]](/assets/backendHTB/Pasted image 20220429035411.png)   
With this new token we will be able to execute command!   
![[]](/assets/backendHTB/Pasted image 20220429035449.png)      
## Getting a Shell
Well its the basic, you can either insert your public key to `.ssh/authorized_keys` or execute a reverse shell command. In my case, I just do `echo {base64 of reverse shell payload}|base64 -d|bash`. Make sure it's url encoded!   
![[]](/assets/backendHTB/Pasted image 20220429041905.png)   
![[]](/assets/backendHTB/Pasted image 20220429041925.png)   


# Enum & PrivEsc
This one is the easiest part of this machine. The app is on "Debug" mode with auth.log logging every authentication attempt. Reading it will give us the password for root.
![[]](/assets/backendHTB/Pasted image 20220429042229.png)   

# Conclusion
![[]](/assets/backendHTB/Pasted image 20220429042405.png)   
Well yeah, thats it for this box. We played around with FastAPI and JWT to get the foothold and simply use a reused password to get root shell. The main learning point from this machine is the absent of validation in `/updatepass` functionality, allowing attacker to update any user's password easily. I personally was expecting more for the root part but not reusing your password is still a good advice I guess.   

## External Resources
[PyJWT](https://pypi.org/project/PyJWT/)   
[Modify Header Value](https://addons.mozilla.org/id/firefox/addon/modify-header-value/)   
[JWT.IO](https://jwt.io/)