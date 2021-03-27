<div class="container container-narrow py-5 mx-auto" markdown="1">

> This post is not done yet.
 
March 2021

Together with colleagues at Chalmers University of Technology, I am developing prototypes with autonomous "drones" -- or flying robots -- for at-home leisure and wellness-related applications.

Our tech stack is based on the [Bitcraze Crazyflie](https://www.bitcraze.io/). As interaction design researchers, we think the Crazyflie is a very exciting platform, mainly because of two reasons.

First, the Crazyflie is versatile, programmable, modular and pretty much entirely open-source. We have programming tools for building an endless variety of applications.  We find a variety of "decks" with sensors, lights, etc. to extend the core hardware without adding unnecessary weight. We can publish knowledge and artifacts based on this ecosystem, unconstrained by considerations re: intellectual property. There are challenges to working with this kind of ecosystem; for example, the documentation could benefit from some more centralization. However, all things considered, the Crazyflie is an excellent platform for prototyping creative applications. I liken it to a a flying Arduino.

Then, the Crazyflie is small. There are other drones – for example, the Tello EDU – which come with the programming tools we need and we can invent ways of extending the hardware if needed. But drones offering robustness and programmability come in larger packages. The Tello, designed to be able to fly outdoors, measures close to 200mm wide and weighs around 80g. Flown indoors, it produces noise and prop wash (air blown down by the propellers) that is very uncomfortable in close range to the human body, and has the potential to break things on impact. The Crazyflie is under 100mm motor-to-motor, and weighs less than 30g. I'm able to come very close to it with my body and my hands without significant discomfort, and having crashed hundreds of times it -- into my head, my hands, my computer screen, and countless other objects -- I can vouch for its safety.


Qualisys system: tracking Crazyflie and ANY object in the same coordinate system in a seamless fashion.

Even though the infrastructure we have from Bitcraze and Qualisys is top notch, we found that the way we use it is somewhat rare. And we could not find any documentation that would have helped us as we develop applications. So we decided to write one.

Let's consider the case of the Crazyflie following another object in real time.

PICTURE OF OBJECT FOLLOWING IN BOUNDED SPACE

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

## Safety

It's always a good idea to have a physical fence aroung the drone. We use a cat net we bought from a pet store.

    # Land if drone strays out of bounding box
    if not x_min - safeZone_margin < cf_pose.x < x_max + safeZone_margin
    or not y_min - safeZone_margin < cf_pose.y < y_max + safeZone_margin
    or not z_min - safeZone_margin < cf_pose.z < z_max + safeZone_margin:
        print("DRONE HAS LEFT SAFE ZONE!")
        break
    # Land if drone disappears
    if cf_trackingLoss > cf_trackingLoss_treshold:
        print("TRACKING LOST FOR " + str(cf_trackingLoss_treshold) + " FRAMES!")

Turn down the speed

    # Slow down
    cf.param.set_value('posCtlPid.xyVelMax', cf_max_vel)
    cf.param.set_value('posCtlPid.zVelMax', cf_max_vel)

## The Pose Class

QTM will be streaming data to our application asynchronously. This means that we will not be able to "poll" the motion capture system for data. Handling position data must be done via asynchronous callbacks.

To make things a bit easier, we will utilize global variables to keep track of where things are at all times.

    fly = True
    cf_xyz = [0, 0, 0]
    target_xyz = [0, 0, 0]
    
</div>
