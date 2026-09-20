#main 
# Welcome
welcome to my docker documentation, this documentation was made for me and anyone whom want to use it, I made it as a cheatsheet and exlpaination for docker commands and some commands options 


# what is docker in one centence  
containerisation technology used to wrap the application and the libraries used in it and the dependencies into single container so every software  works the same on every machine that runs it 


---
---
---
---
---
---
# IMPORTANT NOTE
anything you did not find in this documentation, that means its easy to find and read using the `--help`, yes some things easy to find in the `--help` are here BUT I made it this way because I want it this way

my documentation , my decisions

if I am working in a team thats a different call I will decument every thing , but in my personal documentation for my self its **MY DECISION TO MAKE IT THE WAY I WANT ( •̀ ᴖ •́ )** 

thanks for understanding :) 

---
---
---
---
---
---

# what is in here 
here you will find 
- [Docker cheat sheet](Docker cheat sheet)
- [[#some advices]] I find usefull 
  >doesnt mean its the best,  its just things I tried and think it might be usefull
- [[#some projects to practice]]


# Docker cheat sheet 
- Docker [[2.baisics]]
- options explained with examples for comples/ have manny options commands
	- [[A.docker_run1]]
	- [[A.docker_run2]]
	- [[B.docker_build]]
	- [[C.docker.exec]]
	- [[D.Docker_Inspect]]
	- [[E.Docker_Volume]]

---
# some advices

1. install the docker image from a trusted place like [Github](https://github.com/) or [Docker hub](https://hub.docker.com/)  
2. try something fun while you are learning (like the retro games image in the [[#some projects to practice]])
3. when making your image make sure that you documented the dockerfile 
4. take notes (this repo is built on the notes I made) it will help you and save some time in case you wanted to search for something since you wrote it you know where to search  
5. take your project slowly and device the work 
6.  and the most important to me, WRITE SH FILES TO START THE CONTAINER THAT YOU WANT that saves time and lower the (`I am stupid I forgot that flag again`) situation





# some projects to practice
here is a list of fun projects (`at least for me`) you can make that will help you practice some things 

### 1. Nginx
Nginx is a 4 in 1 package that contains web server, reverse proxy, load balancer, and content cache all in one 
you can install it and run your own website or web application to practice docker baisics (installing an image , running a container ..)

### 2. Retroassembly
this image contains many games from the 90's (`old but gold`) you can install the image and configure it and then run it 
[Download](https://hub.docker.com/r/arianrhodsandlot/retroassembly) 

### 3.Docker minecraft server
sick of paying companies money to host a minecraft server for you 
you have an old laptop that no body uses 

with docker you can install and host you own minecraft server and have fun 
>NOOOOOOOOOOOOTE
>when you are hosting your minecraft server please make sure that you have applied security to the device and the docker container 
>I advise you to follow a youtube tutorial just to be safe like [this one](https://www.youtube.com/watch?v=jWrrSbtr8hU&t=168s)

[Download](https://github.com/itzg/docker-minecraft-server)

### 4. server admin page and container managment
this one might not be as fun as the others but I have enjoyed making it 
>NOOOOOOOOOOE
>by the date I uploaded this repo the project is not finished yet

the idea is simple 
☑️ ~~made a website that shows the containers you have on the server and the services that you have on it~~
☑️  ~~add the ability to start , stop a container or a service~~ 

☑️  ~~add a log page for each container/ service~~

- [ ] enforce security rules on the server using IPTABLES so no one logs in the website from outside 

- [ ] add data base so you can make users and give each one of them the privilages that they can have (alex have access to the retro games but not the nginx for example)

### 5.wiki home server
I did not make this one personally (`as I use obsidian and github to write and store my documentations and notes`) but I thought its a good idea to mention it 

a wiki server is web application that uses wiki software to let users create, edit, and collaborate on web pages through a browser

its useful for storing your notes or documentations, maybe you are documenting a project you are making or just taking notes for a course you are taking 
