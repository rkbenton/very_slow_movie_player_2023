# Building an ePaper SlowMovie Display

Walkthru of my 2023 Very Slow Movie Player build on a Raspi Zero W 2

For this project, I got a Waveshare [800×480, 7.5inch E-Ink display HAT for Raspberry Pi](https://www.waveshare.com/product/displays/e-paper/epaper-1/7.5inch-e-paper-hat.htm) (Item model number 7.5inch E-Ink Display HAT-1) via Amazon. It was $56.

I also picked up a [Raspberri Pi Zero W 2](https://www.amazon.com/Raspberry-Zero-Bluetooth-RPi-2W/dp/B09LH5SBPS) for $26, and I picked up a couple of [SanDisk Ultra microSDHC UHS-I cards](https://www.amazon.com/dp/B08J4HJ98L?ref=ppx_yo2ov_dt_b_product_details&th=1) (Item model number SDSQUA4-032G-GN6MT) for $15. Were I to do it again, I'd pick up the version with the pre-soldered headers. Soldering 40 pins can be a tedious chore. :)

There are many guides for setting up a Very Slow Movie Player. I settled on this one: https://feng.lu/2021/12/04/Building-a-Very-Slow-Movie-Player/

# Set up the Raspberri Pi Zero W 2

https://desertbot.io/blog/headless-raspberry-pi-zero-w-2-ssh-wifi-setup-mac-windows-10-steps

Note: the DNS name for this installation is `jefferson.local`, login name is `becky`.

## Remote Access and SSH

See: https://www.raspberrypi.com/documentation/computers/remote-access.html

The Raspi WiFi works on 2.4g. Network bridging allows you to SSH into the unit from the 5g network.

```
ssh becky@jefferson.local
```

## Passwordless SSH Access

You can optionally use rsa keys to speed access. See:

https://www.raspberrypi.com/documentation/computers/remote-access.html#passwordless-ssh-access

To add your ssh-key to the Mac's keychain:

```
ssh-add --apple-use-keychain ~/.ssh/id_rsa
```

Note that if you've re-flashed the OS onto the Pi, your SSH key may fail because the Mac  sees that it's 
connecting to the Pi but the Pi's identification has changed, so it thinks that someone malicious is trying 
to spoof the Pi's address.

You'll have to edit the file at `/Users/becky/.ssh/known_hosts` on your Mac and 
remove the line that contains the key for the Pi. Then, the next time you SSH, your ssh tool will recognize 
that there's no existing key and prompt you as to whether it should add the new one to your known_hosts file.


## Get the latest updates

Once connected SSH, the next thing you should do on the Pi is run some updates:

```
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt dist-upgrade -y
```

If you're really into being thorough (this may take quite a while, like an hour):

```
sudo apt-get update                          \
    && sudo apt-get dist-upgrade --yes       \
    && sudo apt-get autoremove --purge --yes \
    && sudo apt-get autoclean                \
    && sudo apt-get clean                    \
    && sudo reboot
```

## Keep the Pi from going to sleep

```
iwconfig
```

Under wlan0 you may see this line:

*Power Management:on*

if it shows "on", you can turn that off using this line:

```
sudo /sbin/iw wlan0 set power_save off
```

that's temporary; to make it permanent you need to do the following:

```
sudo nano /etc/rc.local
```

Above the line that says `exit 0` insert this command and save the file:

```
/sbin/iw wlan0 set power_save off
```

To confirm that the setting is permanent, `sudo reboot)` and run iwconfig again.


#### Enable SPI Interface

```
sudo raspi-config
Choose Interfacing Options -> SPI -> Yes Enable SPI interface
sudo reboot
```


# Install SlowMovie

**20231212 notes on my installation process.**

I'm following along with this how-to:

https://feng.lu/2021/12/04/Building-a-Very-Slow-Movie-Player/

First thing was to install Whitwell's SlowMovie package:

https://github.com/TomWhitwell/SlowMovie

I used the simple install:

```
bash <(curl https://raw.githubusercontent.com/TomWhitwell/SlowMovie/main/Install/install.sh)
```

Pick option 1, then enter yes so it starts up every time.


```
sudo raspi-config
Choose Interfacing Options -> SPI -> Yes Enable SPI interface
sudo reboot
```

It was a long install, but eventually, I got this:

```
SlowMovie install/update complete. To test, run 'source /home/becky/SlowMovie/.SlowMovie/bin/activate; python3 /home/becky/SlowMovie/slowmovie.py'
Created symlink /etc/systemd/system/multi-user.target.wants/slowmovie.service → /etc/systemd/system/slowmovie.service.
SlowMovie service installed! Use sudo systemctl start slowmovie to test
```

I rebooted, and SlowMovie started right up, automatically. It showed frames from the `test.mp4` video in `SlowMovie/Videos/`.

Starting and stopping the movie service:

```
sudo systemctl stop slowmovie
sudo systemctl start slowmovie
```

# Movies for SlowMovie

I followed this guide to slim down a movie and size it for the display:

https://github.com/TomWhitwell/SlowMovie/wiki/Preparing-Video-Files

worked great.

To copy movies from my Mac over, I stopped the service and, on the Mac, copied the movies over to the Pi using SCP:

```
scp /Path/To/Some.mp4 becky@jefferson.local:SlowMovie/Videos/
```

for example, in Terminal on your Mac:

```
scp /Users/becky/Downloads/shrunk_videos_for_epaper/n-wEvzqdDZg_shrunk.mp4 becky@jefferson.local:SlowMovie/Videos/
```

## Pysical Display Size

According the to spec, the displayable surface is a 2:3 aspect ratio: 163.2 × 97.92mm (6.47in by 3.88in), while the full dimensions of the device are 170.2 × 111.2mm.

See: https://www.waveshare.com/product/displays/e-paper/epaper-1/7.5inch-e-paper-hat.htm



# Enable VNC if you want a remote desktop

SlowMovie does not require a desktop. You can build and run SlowMovie easily by SSHing into the Raspi.

However, as an exercise, I did ran with a desktop for a while and figured out how to remote in.

> The remoting situation *recently changed* on Raspi. RealVNC is no longer supported. Use TigerVNC instead. See https://picockpit.com/raspberry-pi/tigervnc-and-realvnc-on-raspberry-pi-bookworm-os/ for more info. (Raspi may support RealVNC at some date in the future, but for now, out of the box, it's Tiger.)

Note: to find the IP address of your pi, do this:

hostname -I

if you know the dns name of the pi, you could ping it from another machine:

```
ping franklin.local
```

By default a Raspberry Pi does not allow a remote VNC connection. You have to enable it through raspi-config. It will also install the required VNC server.

```
sudo raspi-config
```

Then:

    - Select Interfacing Options
    - Select VNC
    - For the prompt to enable VNC, select Yes (Y)
    - For the confirmation, select Ok

### Important! Change the default screen resolution

There is a weird quirk where you must change the screen resolution or VNC will report "Cannot currently show the desktop."

Still from within raspi-config:

    - Select Display
    - Select Resolution
    - Select anything but the default (example: 1024x768)
    - Select Ok

Once you've established that it works, you can go back and try other screen resolutions.

    Select Finish
    For the reboot prompt, select Yes


### Install TigerVNC for remote desktop

```
sudo apt install tigervnc-standalone-server 
```

you’ll then need to edit the configuration file. For this, you can go to:

```
sudo nano /etc/tigervnc/vncserver-config-mandatory
```


## Using Secure Copy

from: https://www.raspberrypi.com/documentation/computers/remote-access.html#using-secure-copy

Copying Files to your Raspberry Pi

Copy the file myfile.txt from your computer to the pi user’s home folder of your Raspberry Pi at the IP address 192.168.1.3 with the following command:

scp myfile.txt pi@192.168.1.3:

Copy the file to the /home/pi/project/ directory on your Raspberry Pi (the project folder must already exist):

scp myfile.txt pi@192.168.1.3:project/

Copying Files from your Raspberry Pi

Copy the file myfile.txt from your Raspberry Pi to the current directory on your other computer:

scp pi@192.168.1.3:myfile.txt .

Copying Multiple Files

Copy multiple files by separating them with spaces:

scp myfile.txt myfile2.txt pi@192.168.1.3:

### Copying a Whole Directory

Copy the directory project/ from your computer to the pi user’s home folder of your Raspberry Pi at the IP address 192.168.1.3 with the following command:

scp -r project/ pi@192.168.1.3:


# Setting up Python

I wound up *not* setting up Python for the Very Slow Movie Player project. VSMP's installer handles the entire setup on a more-or-less virgin Pi install all by itself.

These instructions are here as a reminder just in case I want to do further things in Python on the Pi.

### Use a virtual environment

venv documentation: https://docs.python.org/3/tutorial/venv.html

Nice overview: https://realpython.com/python-virtual-environments-a-primer/

Python virtual environments tutorial: https://realpython.com/courses/working-python-virtual-environments/

sudo apt-get install python3-venv python3-setuptools python3-pip python3-wheel

`python3 -m venv my_venv` creates a virtual environment named `my_venv` for your Python 3 interpreter in the current directory.

	- `python3`: This specifies the Python 3 interpreter to use.
	- `-m`: This tells Python to run the module specified after it as a script.
	- `venv`: This is the name of the Python module for creating virtual environments.
	- `my_venv`: This is the name of the virtual environment to be created.

Once you have created the virtual environment, you need to activate it. This tells your system to use the virtual environment's Python interpreter and packages instead of the system's default ones. activate the environment by executing a script that comes with the installation (Before you run this command, make sure that you’re in the folder that contains the virtual environment you just created):

```
source my_venv/bin/activate
```

Once you can see the name of your virtual environment—in this case (my_venv)—in your command prompt, then you know that your virtual environment is active. You’re all set and ready to install your external packages!

### Installing dependencies in your virtual environment

```
python -m pip install <package-name>
```

This command is the default command that you should use to install external Python packages with pip. Because you first created and activated the virtual environment, pip will install the packages in an isolated location.

> Note: Because you’ve created your virtual environment using a version of Python 3, you don’t need to call python3 or pip3 explicitly. As long as your virtual environment is active, python and pip link to the same executable files that python3 and pip3 do.

### Leaving (deactivating) the virtual environment

Once you’re done working with this virtual environment, you can deactivate it:

```
(my_venv) $ deactivate
```

If you want to go back into a virtual environment that you’ve created before, you again need to run the activate script of that virtual environment.

### Dependencies for the epaper demo from Waveshare


Demo tutorial: https://www.waveshare.com/wiki/7.5inch_e-Paper_HAT_Manual#Working_With_Raspberry_Pi

```
python -m pip install RPi.GPIO
python -m pip install spidev
python -m pip install gpiozero
python -m pip install pillow
```

```
cd ~/my_venv/
git clone https://github.com/waveshare/e-Paper.git
cd e-Paper/RaspberryPi_JetsonNano/
```

