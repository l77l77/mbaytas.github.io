<div class="container container-lg py-5 mx-auto" markdown="1">

> This post is not done yet.

# Building Interactivity with the Bitcraze Crazyflie and Motion Capture

At Chalmers University of Technology, we are developing prototypes with autonomous "drones" -- or flying robots -- for at-home leisure and wellnes applications. Our tech stack is based on the [Bitcraze Crazyflie](https://www.bitcraze.io/). As design researchers, we find the Crazyflie is a very exciting platform, mainly for two reasons.

First, the Crazyflie is versatile, programmable, modular and pretty much entirely open-source. We have programming tools for building an endless variety of applications.  We find a variety of "decks" with sensors, lights, etc. to extend the core hardware without adding unnecessary weight. We can publish knowledge and artifacts based on this ecosystem, unconstrained by considerations re: intellectual property. There are challenges to working with this kind of ecosystem; for example, the documentation could benefit from some more centralization. However, all things considered, the Crazyflie is an excellent platform for prototyping creative applications. I liken it to a a flying Arduino.

Then, the Crazyflie is small. There are other drones – for example, the [Tello EDU](https://www.ryzerobotics.com/tello-edu) – which come with the programming tools we need and we can invent ways of extending the hardware if needed. But drones offering robustness and programmability come in larger packages. The Tello, designed to be capable of outdoor flight, measures close to 200mm wide and weighs around 80g. Flown indoors, it produces noise and prop wash (air blown down by the propellers) that is very uncomfortable in close range to the human body, and has the potential to break things on impact. The Crazyflie is under 100mm motor-to-motor, and weighs less than 30g. I'm able to come very close to it with my body and my hands without significant discomfort, and having crashed hundreds of times it -- into my head, my hands, my computer screen, and countless other objects -- I can vouch for its safety.

Another part of our tech stack is a [Qualisys](https://www.qualisys.com/) motion capture system. For many of our prototypes, we do fine with sensors mounted on the drone, such as the [Multi-ranger](https://www.bitcraze.io/products/multi-ranger-deck/) and [Flow](https://www.bitcraze.io/products/flow-deck-v2/) decks. Bitcraze also offers [Loco Positioning System](https://www.bitcraze.io/products/loco-positioning-system/) and [Lighthouse](https://www.bitcraze.io/products/lighthouse-positioning-deck/) integrations, but mocap is uniquely versatile among these options because it gives us the ability to precisely track anything we can put markers onto, within the same calibrated coordinate system as the drone itself.

<p class="small">Full disclosure: My work is sponsored by Qualisys, but the company does not direct or screen our projects and publications in any way.</p>

The drawback is that way that we utilize this tech stack is somewhat uncommon. The Crazyflie is commonly found in robotics engineering labs, and the Qualisys motion capture system is popular for human biomechanics. Much of the documentation and resources on these systems is rich with low-level details that speak to these audiences. Thus, this tutorial is meant as an entry point for hackers and designers who would like to experiment with interactive applications using the Crazyflie, tracked by an external motion capture system.


# Preliminaries

A basic building block for interactive applications with a micro-drone is to have the drone follow another object in real time.

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
    
