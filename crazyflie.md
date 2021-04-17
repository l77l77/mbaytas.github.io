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


# Preliminaries

A basic building block for an interactive drone is to have the drone follow another object in real time. We use a Python script to do this.

Our script connects to [QTM](https://www.qualisys.com/software/qualisys-track-manager/), receiving pose data for the drone and any other tracked objects in real time; and stream this data to the Crazyflie: we let the Crazyflie know about its own pose for [closed loop control](https://en.wikipedia.org/wiki/Control_theory), and set a target for it to move to, based on the position of another object.

## Hardware Setup

## Program Structure

Our script consists of various functions and classes that keep things structured, globally accessible variables to keep track of options and states, and the business end of the script: a main loop.

*This is a structure that is common in real-time interactive applications like games and physical computing. It's how [Arduino](https://www.arduino.cc/) and [Processing](https://processing.org/) programs are structured, and how [Pygame](https://www.pygame.org/) works.*

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

# Safety First

## Speed

Turn down the speed

    # Slow down
    cf.param.set_value('posCtlPid.xyVelMax', cf_max_vel)
    cf.param.set_value('posCtlPid.zVelMax', cf_max_vel)

## Landing

The main fly loop...

    while (fly == True):
        # Do things
        ...
    
    # Land calmly if fly loop is broken
    print("Landing...")
    for z in range(5, 0, -1):
        cf.commander.send_hover_setpoint(0, 0, 0, float(z) / 10)
        time.sleep(0.2)

## Fence

It's always a good idea to have a physical fence aroung the drone. We use a cat net we bought from a pet store.

![Virtual fence confining drone to a safe zone](/img/crazyflie_fence.png)

    # Land if drone strays out of bounding box
    if not x_min - safeZone_margin < cf_pose.x < x_max + safeZone_margin
    or not y_min - safeZone_margin < cf_pose.y < y_max + safeZone_margin
    or not z_min - safeZone_margin < cf_pose.z < z_max + safeZone_margin:
        print("DRONE HAS LEFT SAFE ZONE!")
        break
    # Land if drone disappears
    if cf_trackingLoss > cf_trackingLoss_treshold:
        print("TRACKING LOST FOR " + str(cf_trackingLoss_treshold) + " FRAMES!")

# Interactivity

## The Pose Class

QTM will be streaming data to our application asynchronously. This means that we will not be able to "poll" the motion capture system for data. Handling position data must be done via asynchronous callbacks.

To make things a bit easier, we will utilize global variables to keep track of where things are at all times.

    fly = True
    cf_xyz = [0, 0, 0]
    target_xyz = [0, 0, 0]
    
