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

**Day 5**: To get an idea of how the scale is hooked up, there are four load cell sensors in each corner of the scale that each have three wires coming out of them, giving a total of twelve wires. They then connect to a chip at the top of the scale.

![img_9516](https://cloud.githubusercontent.com/assets/28270466/26218372/bf648cc0-3bd8-11e7-8df2-29876d11f447.JPG)

Following the advice of the following link [link](http://graham.tech/digital-scale-hack/) , we connected our HX711 to our Pi. Specifically, we connected the Raspberry Pi to the HX711 using 3.3v to vcc, Raspberry Pi BCM23 to the SCK pin (The author recommended the CLK pin, but in his picture he used the SCK pin. Our version of the HX711, does not have a CLK pin. If this is a problem, we are going to order the version that does have it), BCM24 to data(DT) and ground to gnd.

![img_9511](https://cloud.githubusercontent.com/assets/28270466/26218983/075154c6-3bdb-11e7-9ae5-a760815e9e98.JPG)

We discovered that we will need a very important part called a load combinator. This will allow us to combine all 4 load cell sensors to essentially get one output out of it. We ordered this from Sparkfun.com. 

**Day 6**: We ordered the Load combinator as well as the load amplifier hx711 (in the scenario that we may need to switch out our hx711 boards for one with a clk pin). After hooking up one of the pressure sensors to the hx711 board, we tried downloading the libraries of the hx711 as well as code that would allow us to read out signals that the pressure sensor was recieving [link](https://github.com/tatobari/hx711py). Without much editing we copied the hx711.py and example.py and ran them in an attempt to get any sort of reading from the sensors(at this point we did not have the load combinator and had only on cell hooked up to the pi). The example.py originally looked like this:

```python
import RPi.GPIO as GPIO
import time
import sys
from hx711 import HX711

def cleanAndExit():
    print "Cleaning..."
    GPIO.cleanup()
    print "Bye!"
    sys.exit()

hx = HX711(5, 6)

# I've found out that, for some reason, the order of the bytes is not always the same between versions of python, numpy and the hx711 itself.
# Still need to figure out why does it change.
# If you're experiencing super random values, change these values to MSB or LSB until to get more stable values.
# There is some code below to debug and log the order of the bits and the bytes.
# The first parameter is the order in which the bytes are used to build the "long" value.
# The second paramter is the order of the bits inside each byte.
# According to the HX711 Datasheet, the second parameter is MSB so you shouldn't need to modify it.
hx.set_reading_format("LSB", "MSB")

# HOW TO CALCULATE THE REFFERENCE UNIT
# To set the reference unit to 1. Put 1kg on your sensor or anything you have and know exactly how much it weights.
# In this case, 92 is 1 gram because, with 1 as a reference unit I got numbers near 0 without any weight
# and I got numbers around 184000 when I added 2kg. So, according to the rule of thirds:
# If 2000 grams is 184000 then 1000 grams is 184000 / 2000 = 92.
#hx.set_reference_unit(113)
hx.set_reference_unit(92)

hx.reset()
hx.tare()

while True:
    try:
        # These three lines are usefull to debug wether to use MSB or LSB in the reading formats
        # for the first parameter of "hx.set_reading_format("LSB", "MSB")".
        # Comment the two lines "val = hx.get_weight(5)" and "print val" and uncomment the three lines to see what it prints.
        #np_arr8_string = hx.get_np_arr8_string()
        #binary_string = hx.get_binary_string()
        #print binary_string + " " + np_arr8_string
        
        # Prints the weight. Comment if you're debbuging the MSB and LSB issue.
        val = hx.get_weight(5)
        print val

        hx.power_down()
        hx.power_up()
        time.sleep(0.5)
    except (KeyboardInterrupt, SystemExit):
        cleanAndExit()
```
However we were met with a string of seemingly random numbers, with no correlation to input from the sensor at all. The failure with this method made us realize that the hx711 was 'missing' a secondary input, and that our method required the use of the load combinator to function correctly.

**Day 7**: Since our wiring yesterday failed to yield any results, we had the idea that it was possibly the flimsy wires to blame(we figured the code wasn't the direct problem since we werent getting actual changes in our readings). We unsoldered the wires from the chip on the scale, which would take away the LED display(bypassing the inner components of the scale altogether), but would allow us to have more freedom of movement with the wires. We then tried using a breadboard to connect the load cell sensor wires to the hx711. The problem we faced with this was again, the quality of the wires from the cells. I tried fortifying them by soldering the tips but this process was very ineffiecient as well as ineffective. This method returned to us another failure, however it did solidify the fact that we needed the use of the combinator board in order to get the correct inputs the hx711 chip needed in order to make the analog to digital conversion. 

**Day 8**: So today the load combinator arrived! We carefully unsoldered the remaining wires from the chip in the scale and began the process of soldering them to the load combinator. 

load combinator arrives We stripped the wires from the load cell sensors and soldered them directly to the load combinator

# Week Three

**Day 9**:  

**Day 10&11**: Today we continued reinforcing the wires from the load cell sensors. This was mainly all I did today. I ended with connecting the reinforced wires to the load combinator after removing the old solder from it. No results.

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

**Day 13**: 

**Day 14**: The LCD screen came in the mail today, so I spent the whole class soldering it together. Not much else to say really.

**Day 15**: funky wiring tricks, downloading lcd libraries, fiddled with contrast, fiddled with code to set up a basic user selection, one that would run as the user steps off and resets for each session.

**Day 16**: A classmate brought in her personal scale to help us correctly calibrate the reference unit for the hx711 conversion. The important thing we did here was do multiple tests with her scale. We were finding that the scale wasnt giving accurate readings until about 20 minutes into using it, which was strange. We then used basic math to deterimine the ratio between the the hx711 scale and the reference unit. Using the formula (hx711 weight)/(reference unit)=(person's true weight)/(true reference unit), we were able to get within a +/- 3lbs range of the scale. Then we continued to adjust the reference unit until we were getting almost identical readings as the store bought scale that our classmate brought. 
    We also fiddled with the code for the LCD board. A problem we ran into was that since we were running our code (now called example2.py) in rc.local, when we would manually run it in the terminal it would be running two instances of the code and mess up the display on the LCD screen. We set up a pretty basic selection menu for who was being weighed, as well as made our own personal google spreadsheet page and implemented them into the code. 

**Day 17**: We honestly did not have much to do today. What we have left is basically to improve the interface of the LCD screen in order to make it more streamline. We also began work on the final website.
