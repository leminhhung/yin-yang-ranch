==================================================================
How Computer Vision Tasks are Distributed Among Multiple Computers
==================================================================

A detailed example can show how **imagenode** can be used to capture images on
Raspberry Pi computers, send selected event messages and images to an
**imagehub** and allow further processing in the **librarian**.
This design enables a pipeline of computer vision processing
across multiple computers.

Let's walk through an example where the combination of **imagenode** and
**imagezmq** is used to distribute a computer vision process across a Raspberry
Pi and a Mac. This project involves reading a water meter using a Raspberry Pi
and PiCamera to gather images and do some image processing locally. The
**imagenode** program running on the Raspberry Pi selects some images to be
transmitted via **imagezmq** to a Mac for more complex processing.

This is what a water meter looks like:

.. image:: images/water-meter.jpg

The goal of the project is to use computer vision to manage water use on a small
farm. I can use computer vision to determine if water is flowing or not, read
the gallons used per hour or per day, and save some of the images for analysis.
The project also watches for unusual water flow due to leaks or broken irrigation
controls and sends alerts. When the water is flowing, the large analog needle
spins clockwise. Each full rotation of the needle causes the rightmost
digit of the digital meter to advance by one digit. The small "blue star" dial
is a "leak detector" that spins even when a very small amount of water is
flowing (like a dripping faucet).

Here's what **imagenode** does on the Raspberry Pi:

- The PiCamera gets warmed up and bank of LEDs is turned on to illuminate the
  water meter (via the RPi GPIO pins).
- Images of the water meter are taken at 16 frames per second.
- The images are rotated, cropped and converted to grayscale.
- The images are thresholded and dilated.
- The images are pushed into a Queue as they are gathered.
- Pairs of images in the Queue are differenced to look for needle motion to
  determine if water is flowing or not.
- The water meter status (flowing or not flowing) is sent via **imagezmq**
  along with a series of images that show when the status changed (from flowing
  to not flowing or vice-versa). To minimize network load, images are only sent
  when there has been a status change. The program logic to select which images
  to send is running in the main thread of the **imagenode** program on the
  Raspberry Pi.

Here's what **imagehub** program does on the Mac:

- The images are received via **imagezmq** along with the status messages.
- The images are stored in an image database.
- The event messages are logged in an event log with date and time.

The list of tasks done by the **imagehub** is limited by design. It can gather
images and event messages from multiple Raspberry Pi **imagenode** computers
if it does nothing other than receive snd store images and event messages.

Here's what the **librarian** program does on the Mac:

- Several Regions of Interest (ROI) are extracted from each image and further
  processed:

  - An ROI containing the digital digits portion of the meter face is masked and
    the digits are extracted to compute the numeric value of the meter digits.
    A customized digit classifier algorithm is used on the extracted digits.
  - An ROI containing the "leak detector" is masked and analyzed to create
    an alert if there are leaks or irrigation failures.

- Current water state (flowing or not flowing), water state history, images of
  interest and gallons used are analyzed and put into a database.
- Other programs running on the Mac access the database to answer queries about
  water status, gallons used per hour, per day or per week, etc.
- Another program on the Mac sends text alerts in case of water leaks or irrigation
  control issues.

This distributed computing arrangement optimizes the Raspberry Pi and Mac
computer vision pipeline in a way that allows each computer to do tasks for
which it is best suited:

- No images are stored on the Raspberry Pi. The SD Card storage on the Raspberry
  Pi is very slow for writing large binary image files. Also, there have been
  reports of SD card failures caused by frequent writes of large binary files.
- The Raspberry Pi can control the lighting to gather the best images.
- The Raspberry Pi can also send information from sensors connected to its
  GPIO pins. For example, if there is a temperature sensor connected, the
  temperature can be read and sent at specified intervals. Reading of GPIO
  sensors happens in a secondary thread.
- The Raspberry Pi can do simple image processing, motion detection and can use
  an algoritm to select which images to send via **imagezmq**.
- The Mac can use its larger memory and faster CPU to do more complex image
  processing, including feature extraction and digit classification.
- The Mac can use its fast SSD disk drives to store the images and the extracted
  features of images.
- One Mac can act as a processing hub for multiple Raspberry Pi computers
  sending image and other sensor data. I am running about 8 Raspberry Pi
  computers for each Mac (or Linux computer).

In addition to using a Raspberry Pi to read the water meter, I have Raspberry
Pis running **imagenode** software watching the garage and barn doors, watching the farm
for critters, tracking the weather and solar panels, etc. One Mac can easily
act as a hub for 8 or more Raspberry Pi computers that are doing simple image
processing, motion detection and sending a subset of gathered images.

The code for the Raspberry Pi **imagenode** program is in the
`imagenode GitHub repository <https://github.com/jeffbass/imagenode.git>`_.
The code for the **imagezmq** communication software that allows the node
computers to communicate with the hub computer is in the
`**imagezmq** GitHub repository <https://github.com/jeffbass/imagezmq>`_.
The **imagehub** program is in the
`imagehub GitHub repository <https://github.com/jeffbass/imagehub>`_.

Together they form the **imagenode <--> imagezmq <--> imagehub** distributed
computer vision system.

`Return to main documentation page README.rst <../README.rst>`_
