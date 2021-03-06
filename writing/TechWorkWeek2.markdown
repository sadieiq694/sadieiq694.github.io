#**Week Two: Technical Work**

##**Goals**

After week one, students felt more comfortable with the cars and basic motion commands. So, for week two, groups built upon these skills by combining them with basic computer vision.

Similar to the previous week, there were two main goals to accomplish. The first was to  learn about image processing and become comfortable using OpenCV, an open source computer vision software library (7). The final challenge goal incorporated things learned during week one and week two. The cars had to be able to detect a brightly colored piece of construction paper, or, 'blob', and drive toward it using visual servoing. This blob was either red or green. Once the robot got a certain distance away from it (approximately one meter), it had to make a steering decision based upon this color. If the blob was green, the car had to turn right and follow the left wall to a finish line. If it was red, the car had to turn left and follow the right wall to a different finish line. Teams were to complete these tasks as quickly as possible during the final challenge.

##**Approach**

Again, some background knowledge is necessary before my group's approach can be described.

###**Computer Representation of Images**

A computer deals with color images as 2D pixel arrays where each pixel encodes a color (8). There are several ways in which computers can represent colors, but in the Beaver Works course students mostly worked with RGB and HSV formats.

The human eye perceives color using specialized cells called cones. There are three different types of cones in our eyes that all respond most strongly to different wavelengths of light; red, green, or blue (9). RGB representation is based off the human visual system. Each pixel in the 2D array that represents an image encodes a color with the triplet (R, G, B), indicating the amount of each color of light in that pixel. Because color pixels are typically encoded in 24 bits, or, 8 bits per color, the R, G, and B components take values between 0 and 255 (8). (0, 0, 0) encodes black, (255, 255, 255) encodes white, and any triplet in which all three values are the same represents gray. (255, 0, 0) would be a 'perfect' red.

An alternative to RGB is the HSV color model. This system also uses triplets to represent colors, but the three values represent hue, saturation, and value. Hue is out of 180 degrees. 0 degrees is red, 60 degrees is green, and 120 degrees is blue. Saturation is out of 255 and represents the intensity of the color. Value is also on a scale of 0 to 255 and is a measure of the brightness of a color (8).

![HSV Chart] (https://cloud.githubusercontent.com/assets/18174572/17703787/f45c870e-63a0-11e6-86b2-d12b2cd06902.png)

Figure 1: Diagram showing how colors change in accordance with HSV values (10)

The HSV model works better for blob detection because it is much more robust under illumination changes.

###**Intro To OpenCV Tools**

Students used OpenCV to process the images received from the ZED camera. Specific tools in the library that were particularly useful for week two's challenge were:

* **Creating A Mask**: Using cv2.inRange, one can select upper and lower bounds for color values and create a binary "mask" that highlights a specific color. This method can be used to initially identify blobs of specific colors. (7)

![mask](https://cloud.githubusercontent.com/assets/18174572/17709879/deab533c-63b8-11e6-95a7-1b490beab197.png)

Figure 2: Side by side of original image and binary mask for red range (11)

* **Creating a Contour**: Contours are essentially curves that join all continuous points along a boundary having the same color. They work best on binary images, which is why a mask is often utilized to isolate the specific colors. By using the function cv2.findContours, one can identify contours using the binary image. The contours can then be applied to the original image using cv2.drawContours (7).

![contour](https://cloud.githubusercontent.com/assets/18174572/17710445/4aed708c-63bb-11e6-83fa-fa6570613979.png)

Figure 3: Image with contours applied (11)

* **Using Moments**: Moments (cv2.moments) can be used to find the center of contours, which is useful to know when visual servoing (7). One can compare the x coordinate of the center of a contour (that outlines the desired blob) to the x coordinate of the center of the camera's frame and make steering decisions based on the difference.

![center](https://cloud.githubusercontent.com/assets/18174572/17711449/ac908e7e-63bf-11e6-8275-5bdbf1ecbd89.png)

Figure 4: Image with center identified (11)

* **Creating a Bounding Rectangle**: cv2.boundingrect, as the name suggests, draws a rectangle around a contour. It is also one of several functions that can be used to determine the rough area of a blob, as it computes both height and width (7). However, this approach to finding the area is only accurate when the blob is rectangular in shape.

![bounding rectangle](https://cloud.githubusercontent.com/assets/18174572/17712492/28f3e264-63c4-11e6-9bed-37eb7119b0af.png)

Figure 5: Side by side view of red contour and bounding rectangle based off of that contour (12)

These tools were primarily what groups used to work with blobs for week two's challenge.

###**Custom Messages**

Custom messages were another important tool introduced to students. Sometimes it is necessary to communicate multiple pieces of information between nodes of varying different types (13). This is made simple by custom message types. An example is as follows:

      Header header
      std_msgs/String color
      std_msgs/Float64 size
      geometry_msgs/Point location

This is the message type my group used in the week two challenge to communicate information about the blob. The custom message allowed the color, size, and location of the blob to be sent all together instead of individually.

###**Challenge Approach**

We decided to handle the challenge tasks with two nodes. One would handle vision and blob detection and one would handle motion commands. By writing fewer nodes, we hoped to save time that we could spend working with image processing, which proved to be difficult in the preliminary labs.

The blob detection node would determine the size, location, and color of the blob and pass it to the motion control node, which would use that information to make all driving decisions.

In terms of task delegation, our group split so some of us were working on blob detection and others worked on motion commands. Again, time played a big role in this decision. It would have been preferable for the entire group to work through everything together to ensure unanimous comprehension of the code, but because of time constraints we went with a "divide and conquer" approach.

##**Process**

###**Blob Detection Node**

There was a steep learning curve for image processing. My group struggled with it for several days, spending most of our time debugging and rerunning code that never seemed to work. We struggled to interpret the documentation of several OpenCV functions and encountered multiple problems with the custom message type. On Thursday we finally succeeded in getting the node to publish the correct data.

The blob detection node subscribed to the ''/camera/rgb/image_rect_color' topic, which gave it access to the stream of images captured by the ZED camera. It published to the 'blob_info' and 'image_echo' topics. The custom 'blob_detect' messages were sent over the 'blob_info' topic. The Image topic was used to publish frames from the camera to which contours and points indicating the center of blobs were added. These altered images were used to debug the vision code and tweak color ranges.

**Table 1: Color Ranges for Green and Red Blobs (Saturation and Value are Fractional)**

| Color | Hue Min | Hue Max | Saturation Min | Saturation Max | Value Min | Value Max |
|-------|:-------:|:--------:|:--------------:|:-------------:|:--------:|:----------:|
|Red     |   0      |     15     |       .8         |       1     |    .7        |     1       |
|Green    |   50      |   77      |       .4         |      1      |    .15        |    1        |

The vision node worked by creating two lists of contours: one for red blobs and one for green blobs. The 'official contour' was set to green by default. If the list of green contours was empty, the official contour would be set to red. The center and size of the blob was then determined using moments and the fields of the custom message type were filled out. Finally, the custom message type and the modified camera image were published.

###**Motion Node**

The blob detection node took a lot longer to program than our group anticipated, so we did not get to spend as much time as we wanted to on the motion code. To summarize, the node subscribed to the 'blob_info' and ''/scan' topics. It used the x term of the location tuple and compared it to the x coordinate at center of the camera's view to find the steering error. The error was input into a PID function to determine the steering angle. The size of the blob was used to tell the node when to switch between visual servoing and wall following. If the size of the blob made up more than 10% of the camera view, the node would make the change.

The wall following algorithm we implemented was similar to the one from the previous week; it took a range of points and used the minimum distance of these points as the distance from the wall.

It was necessary to add an additional motion command between visual servoing and wall follow. When the car reached the point that it was close enough to the blob to start wall following, it was perpendicular to the wall. This means that when the car attempted to wall follow in either direction, the LiDAR would detect the smallest distance to be extremely large, so the car would turn hard in that direction and drive in a circle indefinitely. To combat this, we added a manual steering command that would tell the car to steer full right for green and full left for red for a period of .75 seconds. That way, when the car finally switched to wall following, there would actually be a wall for it to detect using the LiDAR.

###**Testing**

We did not get to test until the final day. Before testing the robot on the floor, we put it on a box so we could see what the wheels were doing without it actually moving. We then held a green blob in front of the camera and moved it around to see if the car was successfully steering towards the blob. Once we were sure this worked, we were ready to test on the floor.

To make sure it was making it into the visual servo box, we added temporary code to made the robot stop right before the transition from visual servoing to wall following. This allowed us to tweak the desired area for the blob until the car stopped right inside the box.

We only got to test the entire system a couple of times before the weekly challenge. There were problems with sign mix ups; the car was turning left on green and later it was not following the correct wall. We thought that we had fixed these problems as we did have one successful test run with a green blob, but would have liked to test it many more times to confirm that they were correct.

![WeeklyChallenge](https://cloud.githubusercontent.com/assets/18174572/17713920/5cdf9df0-63cb-11e6-810f-9680f78509df.png)

Figure 6: Diagram of Week 2 Challenge (14)

##**Results**

For week two's challenge, each car had three opportunities to complete the tasks. For each of these three tries, the blob color would be randomly chosen as would the starting orientation of the car. The blob would be either red or green and the car would either start pointing directly at the blob or be slightly rotated to the left or right. The time (if there was one) would be recorded for each run and the single best time would be used to rank the cars at the end.

For our first run, our car was assigned the green blob and was rotated slightly to the right. For the second run, we again were given the green blob but our car was not rotated. In the final run we had to detect a red blob and we started rotated to the right again. We were one of the seven teams that did not complete any runs successfully. This was unsurprising considering we only had one successful test run, but disappointing nonetheless.

During the first and third runs, our car made it into the visual servoing box and turned in the correct direction, but just continued to turn in a circle and did not make it to the finish line. The car's most successful attempt was its second (green no rotation), where it made it into the box, made the correct turn (right), wall followed the left wall for a little while, but about three meters away from the finish line suddenly turned right and collided with the wall on the other side. There are many possible reasons for the car's malfunctions. It could have been something as simple as a sign error in the left and right steer values or something that would take more time to debug, like a faulty turn command. Given more time our team may have made some very different choices, such as splitting the visual servoing and wall following into two separate nodes instead of combining them into one. This week was difficult, but groups were able to accomplish the first goal, gaining experience with image processing, which would help them in the weeks to follow.
