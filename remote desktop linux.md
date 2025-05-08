### x11vnc

#### On remote PC
```
sudo apt-get install x11vnc
x11vnc -storepasswd
x11vnc -rfbauth ~/.vnc/passwd -display :1 -forever -shared -geometry 1280x800
```
- without password: `x11vnc -display :1 -forever -shared -geometry 1328x830`
- Find display automatically: `x11vnc -find -forever -shared -geometry 1328x830

In case this doesn't work change display number from 1 to 0 or 2

If you are not logged in the computer (manually once) then use one of the following options
##### Option 2
- Find the auth file for gdm3 (or your manager) using 
	- `ps aux | grep auth`
	- you will get output something like
	- `root        1428  0.1  0.0 27628360 127872 tty1  Sl+  14:42   0:09 /usr/lib/xorg/Xorg vt1 -displayfd 3 -auth /run/user/125/gdm/Xauthority -background none -noreset -keeptty -verbose 3`
	- This means the auth file is `/run/user/125/gdm/Xauthority`
- Now run the following command
	- `sudo x11vnc -auth /run/user/125/gdm/Xauthority -display :0 -forever -shared
	- This will require the main password. After this log into to the computer (via remote vnc on your local pc mentioned below)
- If you see black screen after logging in the close the vnc server and run the following command again
	- `x11vnc -display :1 -forever -shared`
	- This happens because your display changes once you log in
	- Find display automatically: `x11vnc -find -forever -shared -geometry 1328x830

##### Option 3
If still doesn't work
- Install following: `sudo apt install xvfb xfce4 xfce4-goodies`
- Then run following on one terminal
```
Xvfb :1 -screen 0 1024x768x24 &
DISPLAY=:1 xfce4-session &
```
- On another terminal run: `x11vnc -rfbauth ~/.vnc/passwd -display :1 -forever -shared -geometry 1280x800`
- To kill `Xvfb` afterwards you can do following
	- `ps aux | grep Xvfb`
	- then use `kill -9 <pid>` where `<pid>` is from the above command

##### Option 4
unset XAUTHORITY
unset DISPLAY
export DISPLAY=:1
sudo -u user DISPLAY=:1 XAUTHORITY=/run/user/1000/gdm/Xauthority x11vnc -display :1 -auth /run/user/1000/gdm/Xauthority -forever -shared -ncache 10

##### Option 5
1. Run Virtual Framebuffer X Server
Xvfb :99 -screen 0 1024x768x24 &
export DISPLAY=:99

2. Start the desktop environment
startxfce4 &

3. Then run x11vnc (in new ssh terminal)
x11vnc -display :99 -forever -shared &

#### On local PC
```
sudo apt install tigervnc-viewer
vncviewer 10.9.1.178:5900
```
