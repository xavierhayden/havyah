The goal of our project is to connect a Raspberry Pi to a digital weighing scale such that the Pi will create a line plot of a persons weight over time.
## Our Daily Progress
# Week 1

**Day 2**: We started by setting up basics on our raspberry pi. By using the code "sudo raspi-config" into the console, we changed internal components like allowing ssh and setting up the correct clock, as well as creating a new password. By applying for a Clearpass, we were able to get the pi to have its own ip address.I also had to download a program called Putty, which allowed a windows user ssh access from Putty's terminal. 

**Day 3**: We connected our new Raspbery Pi 3 to wifi. We became familiar with Cron as a we created a program that would run a python file every 5 minutes.

**Day 4**: We Programmed our Raspberry Pi to email us its IP address whenever it booted up. This way, we can SSH into the Raspberry Pi from our laptops without the use of a desktop. This is called a headless setup. The first step was creating a python program on the Raspberry Pi that would first get the IP, then email both Ben and I what it found. We did this in a file called getIP.py.


```python

import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.connect(("gmail.com", 80))
msg= s.getsockname()[0]
s.close()

import smtplib

server = smtplib.SMTP('smtp.gmail.com',587)
server.starttls()
server.login("scandylyfeofpi@gmail.com", "what'supdoc")

server.sendmail("scandylyfeofpi@gmail.com","xhayden@bates.edu",msg)
server.sendmail("scandylyfeofpi@gmail.com","beckardt@bates.edu",msg)
server.quit()

```

Next, to get the pi to run getIP.py upon rebooting, we entered the following path...
```shell
sudo nano /etc/rc.local
```

Then, inputted the following code. Note that we had the program sleep, because it took the pi a non-negligable amount of time to hook up to the internet. This of course is needed to run getIP.py since it sends an email, so we don't want to have the pi run the program until wifi has been attained. 

```shell
while ! /sbin/ifconfig wlan0 | grep -q 'inet addr:[0-9]'; do
    sleep 3
done
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
  printf "My IP address is %s\n" "$_IP"
  python /home/pi/getIP.py &
fi

exit 0
```
# Week 2

**Day 5**: Following the advice of the following link, , we connected our HX711 to our Pi. Specifically, we connected the Raspberry Pi to the HX711 using 3.3v to vcc, Raspberry Pi BCM23 to the SCK pin (The author recommended the CLK pin, but in his picture he used the SCK pin. Our version of the HX711, does not have a CLK pin. If this is a problem, we are going to order the version that does have it), BCM24 to data(DT) and ground to ground. We are going to need a load combinator since there a four sensors on the scale, and one arduino can only connect to one sensor.

**Day 6**: We ordered the Load combinator as well as the load amplifier hx711 (in the scenario that we may need to switch out our hx711 boards for one with a clk pin). After hooking up one of the pressure sensors to the hx711 board, we tried downloading the libraries of the hx711 as well as code that would allow us to read out signals that the pressure sensor was recieving. However, when running the python script, we were only able to get a random set of numbers displayed, with no correlation to whether or not we were pushing on the sensor. To try and solve this problem, we figured that our wiring may be incorrect, as it it appears slightly different on the various projects we have found online. This method provided no further progress.

**Day 7**:

**Day 8**:
**Day 12**: We came in and tested it and it worked! Only for a moment. After configuring with wires, I went to try and swap hx711 boards, while ben switched out the female to female wires. Switching the wires from the combinator board to the amplifier gave no solution. However, when Ben switched the wires from the Pi to the amplifier, wires which we noticed were a little looser. This was the missing link. Switching the wires gave us accurate readings from the scale when pressure was applied. We then began to set the reference weight for our program. We measured the weight(in grams) of a mug with an apple inside as well as my phone and converted the weight to lbs. We then measured the weight on our scale and ran example.py. The weights we got were -27 and -10. By doing some simple math to find the ratio between the hx711's unit and lbs, we set our reference weight to -12500, which gives fairly accurate readings to the pound. The downside with setting a larger reference weight is that the scale is unable to accurately measure smaller objects, however since this is a body scale intended for people, this is not really a concern. 
    Next we began adding to the existing example.py code in order to optimize the users point of view. Since we will have the example.py code running constantly, we have to make sure we only see the data when someone steps on the scale. The following code allows it so the program only displays a number if more than 5 pounds of pressure is applied to the scale.
   
   ```python
    if val<5:
        print (val)
   ```
    To get only one number(the true weight of the object being measured) we did the following:
    
    ```python
    import numpy
    import datetime
    mass=[]
    while True:
        val = hx.get_weight(5)
        if val<5:
            if len(mass)>0:
                print (str(datetime.datetime.now()),np.median(mass))
            time.sleep(1)
            mass=[]
        else:
            mass.append(val)
    ```
    
    This takes all the values measured in a given setting and adds them to a list "mass". Numpy then finds the median of that list, which in reality would be the true weight of the object. We also return the date and time of the session along with the weight to have a sort of order to our information.
