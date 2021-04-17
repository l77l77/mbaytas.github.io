---
title: Human-Drone Interaction with the Crazyflie and Motion Capture
---

> WORKING DRAFT, IN PROGRESS

# Building Interactions with the Bitcraze Crazyflie and Motion Capture

*March 2021*

In our research we experiment with autonomous "drones" -- or flying robots -- for at-home leisure and wellnes applications. Our tech stack is based on the [Bitcraze Crazyflie](https://www.bitcraze.io/), which serves this purpose well for two reasons.

First, it is modular and pretty much entirely open-source. We find a variety of "decks" with sensors, lights, etc. to extend the core hardware without adding unnecessary weight. We can publish knowledge and artifacts based on this ecosystem, unconstrained by IP considerations. There are challenges in this kind of ecosystem; for example, more centralized documentation would make our lives easier. However, all things considered, the Crazyflie is a rich platform for creative applications. I liken it to a a flying Arduino.

Most importantly, it is small. There are other drones – like the [Tello EDU](https://www.ryzerobotics.com/tello-edu) – which come with programming tools. But the Tello, capable of outdoor flight, is close to 200mm wide and weighs around 80g. Indoors, its noise and prop wash are very uncomfortable, especially when close to the human body. It breaks things on impact. The Crazyflie is under 100mm motor-to-motor, and weighs less than 30g. I can come very close to it with my body and my hands without discomfort. Having crashed hundreds of times -- into my head, my hands, my computer screen, and countless other objects -- I can vouch that it is safe.

Another part of our tech stack is a [Qualisys](https://www.qualisys.com/) motion capture system. For many of our prototypes, we do fine with sensors on the drone, like the [Multi-ranger](https://www.bitcraze.io/products/multi-ranger-deck/) and [Flow](https://www.bitcraze.io/products/flow-deck-v2/) decks. But mocap can precisely track anything we can put markers onto, within the same calibrated coordinate system as the drone itself.

*Full disclosure: My work is sponsored by Qualisys, but the company does not direct or screen our projects and publications in any way.*

The way that we utilize this tech stack is somewhat uncommon. The Crazyflie is often used by engineering labs, and the Qualisys is popular for human biomechanics. Their documentation and resources speak to these audiences. We couldn't find a good entry point for hackers and designers to experiment with the Crazyflie, tracked by an external motion capture system. So we decided to write one.


## Preliminaries

A basic building block for an interactive drone is to have the drone follow another object in real time. We use a Python script to do this.

Our script connects to [QTM](https://www.qualisys.com/software/qualisys-track-manager/), receiving pose data for the drone and any other tracked objects in real time; and stream this data to the Crazyflie: we let the Crazyflie know about its own pose for [closed loop control](https://en.wikipedia.org/wiki/Control_theory), and set a target for it to move to, based on the position of another object.

### Hardware Setup

### Program Structure

Our script consists of various functions and classes that keep things structured, globally accessible variables to keep track of options and states, and the business end of the script: a main loop.

*This is a structure that is common in real-time interactive applications like games and physical computing. It's how [Arduino](https://www.arduino.cc/) and [Processing](https://processing.org/) programs are structured, and how [Pygame](https://www.pygame.org/) works.*

```python
# Set things up
...

with SyncCrazyflie(cf_uri, cf=Crazyflie(rw_cache='./cache')) as scf:

# Set other things up
...

# FLY
while(fly == True):

    # Do flying things
    ...

# Land
...
```

## Safety First

Even though the Crazyflie is small and light, and generally not capable of serious damage; it has a very expensive self-harm habit. So we need to take some safety measures to protect the drone.

### Speed

Using the Crazylie's parameters system, we can set internal limits on how fast it's allowed to go. There are two parameters for the speed limit `xyVelMax` for lateral movement, and `zVelMax` for vertical movement.

The actual value for these is set at the beginning of the script (in a unit of m/s), and passed to `cf.param.set_value()` right before takeoff.

```python
# Options: Physical Config
cf_max_vel = 0.4 # in m/s

...

# Slow down
cf.param.set_value('posCtlPid.xyVelMax', cf_max_vel)
cf.param.set_value('posCtlPid.zVelMax', cf_max_vel)
```
    
A slow speed limit drastically improves stability and safety. It lowers your probability of crashing, as well as the damage done <del>if</del> when you do.

Unfortunately, at this time, there is no way to set a rotational speed limit. It would have been very useful to be able to limit the yaw rate – our #1 reason for crashing at the moment is when the Crazyflie is unable to stabilize itself after rotating a little too eagerly.

*We found that Bitcraze hasn't produced clear documentation for the parameters system at this time, but their [GUI client](https://github.com/bitcraze/crazyflie-clients-python) offers a Parameters tab if you'd like to explore them.*

### Landing

Other than crashes, a frequent reason why the Crazyflie is prone to hurting itself is that it doesn't automatically perform a calm landing when we stop sending commands to it. For this reason, we must program a landing sequence into our script.

The landing sequence initiates whenever we break out of the flight loop for whatever reason, and by using the `cf.commander.send_hover_setpoint()` command it is able to cut thrust gradually, even if the tracking (and thereby closed loop control) is disrupted.

```python
while (fly == True):
    # Do things
    ...

# Land calmly if fly loop is broken
print("Landing...")
for z in range(5, 0, -1):
    cf.commander.send_hover_setpoint(0, 0, 0, float(z) / 10)
    time.sleep(0.2)
```

## Fence

It's always a good idea to have a physical fence aroung the drone. We use a cat net we bought from a pet store.

We also build a virtual fence in our code.

![Virtual fence confining drone to a safe zone](/img/crazyflie_fence.png)

```python
# Land if drone strays out of bounding box
if not x_min - safeZone_margin < cf_pose.x < x_max + safeZone_margin
or not y_min - safeZone_margin < cf_pose.y < y_max + safeZone_margin
or not z_min - safeZone_margin < cf_pose.z < z_max + safeZone_margin:
    print("DRONE HAS LEFT SAFE ZONE!")
    break
# Land if drone disappears
if cf_trackingLoss > cf_trackingLoss_treshold:
    print("TRACKING LOST FOR " + str(cf_trackingLoss_treshold) + " FRAMES!")
```

# Interactivity

## The Pose Class

[Pose](https://en.wikipedia.org/wiki/Pose_(computer_vision)) is the term used in engineering to refer to the combination of position (x, y, z coordinates) and orientation (rotations around x, y, z axes), describing an object's whereabouts in [six degrees of freedom (6DoF)](https://en.wikipedia.org/wiki/Six_degrees_of_freedom).

To keep the rest of the code clean, I implemented a `Pose` class which we will use to keep track of this information for each object that we are tracking, and to hold some utility functions for dealing with pose data.

We can retrieve pose from QTM in two ways: one represents orientation as [Euler angles](https://en.wikipedia.org/wiki/Euler_angles), and the other provides a 3x3 [rotation matrix](https://en.wikipedia.org/wiki/Rotation_matrix). Both can be useful for us, so the `Pose` class has [class methods](https://stackoverflow.com/questions/12179271/meaning-of-classmethod-and-staticmethod-for-beginner) that can instantiate it from both kinds of information. Within the class, the `roll`, `pitch`, and `yaw` for the Euler angles and the `rot` attribute for the rotation matrix are independent -- updating one does not affect the other. This is counterintuitive, but has no practical bearing for our purpose. I haven't implemented a method to convert between the two because we can retrieve either from QTM at any time.

```python
class Pose:
	"""Holds pose data with euler angles and/or rotation matrix"""
    def __init__(self, x, y, z, roll=None, pitch=None, yaw=None, rot=None):
        self.x = x
        self.y = y
        self.z = z
        self.roll = roll
        self.pitch = pitch
        self.yaw = yaw
        self.rot = rot

    @classmethod
    def from_qtm_6d(cls, qtm_6d):
    	"""Build pose from rigid body data in QTM 6d component"""
        ...

    @classmethod
    def from_qtm_6deuler(cls, qtm_6deuler):
    	"""Build pose from rigid body data in QTM 6deuler component"""
    	...
        
    def distance_to(self, other_point):
        """Calculate linear distance between two poses"""
        ...
        
    def is_valid(self):
        """Check if any of the coodinates are NaN."""
        ...

    def __str__(self):
        ...
```

QTM will be streaming data to our application asynchronously. We will not poll the motion capture system for data. Handling position data must be done via asynchronous callbacks.

*The [QTM real-time protocol](https://docs.qualisys.com/qtm-rt-protocol/) does allow polling if that's what you'd like to do, but the best-documented examples for the [Qualisys Python SDK](https://github.com/qualisys/qualisys_python_sdk) are all built on asynchronous streaming, so that's what we have used.*

We utilize global variables to keep track of where things are at all times.

```python
# Global vars
...
cf_pose = Pose(0, 0, 0)
controller_poses = []
for i, controller_body_name in enumerate(controller_body_names):
	controller_poses[i] = Pose(0, 0, 0)
```

### Global Variables

We use global variables to keep track of where things are and expose this data to different parts of our program.

```python
# Global vars
fly = True
cf_trackingLoss = 0
cf_pose = Pose(0, 0, 0)
controller_poses = []
for i, controller_body_name in enumerate(controller_body_names):
	controller_poses[i] = Pose(0, 0, 0)
controller_select = 0
```

`fly` is the "on switch" for our drone. Setting this to false at any time will terminate the flight and cause the drone to (hopefully) execute the landing sequence.

`cf_trackingloss` is a counter that is incremented if we receive a packet from QTM which does not contain a valid pose for the Crazyflie. If we don't receive pose from QTM, that means we are not in control anymore. It's actually fine if this happens for a fraction of a second (which it often does), but if we don't get pose for more than a few seconds the drone will become unstable. To prevent this from happening we constantly check this counter against the `cf_trackingLoss_treshold` option that we set in the beginning. If `cf_trackingloss` exceeds `cf_trackingLoss_treshold`, it's time to land.

`cf_pose` and `controller_poses` hold the pose data for all the things we're tracking.

We'll get to `controller_select` in a second, below.



