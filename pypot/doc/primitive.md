# Primitives

In the previous sections, we have shown how to make a simple behavior
thanks to the [pypot.robot.robot.Robot](pypot.robot.html#pypot.robot.robot.Robot) abstraction.
[pypot.primitive.primitive.Primitive](pypot.primitive.html#pypot.primitive.primitive.Primitive) allows you to create more
complexe, parallelized behavior really easily.

## What is a "Primitive"?

We call [pypot.primitive.primitive.Primitive](pypot.primitive.html#pypot.primitive.primitive.Primitive) any simple or complex
behavior applied to a [pypot.robot.robot.Robot](pypot.robot.html#pypot.robot.robot.Robot). A primitive can access
all sensors and effectors in the robot. It is started in a thread and
can therefore run in parallel with other primitives.

All primitives implement the
[pypot.primitive.primitive.Primitive.start](pypot.primitive.html#pypot.primitive.primitive.Primitive),
[pypot.primitive.primitive.Primitive.stop](pypot.primitive.html#pypot.primitive.primitive.Primitive),
[pypot.utils.stoppablethread.StoppableThread.pause](pypot.utils.html#pypot.utils.stoppablethread.StoppableThread) and
[pypot.utils.stoppablethread.StoppableThread.resume](pypot.utils.html#pypot.utils.stoppablethread.StoppableThread). Unlike regular
python thread, primitive can be restart by calling again the
[pypot.primitive.primitive.Primitive.start](pypot.primitive.html#pypot.primitive.primitive.Primitive) method.

To check if a primitive is finished, use the
[pypot.primitive.primitive.Primitive.start.is\_alive](pypot.primitive.html#pypot.primitive.primitive.Primitive) method (will output True
if primitive is paused but False if it has been stopped or if it's finished). 

The [pypot.primitive.primitive.PrimitiveLoop](pypot.primitive.html#pypot.primitive.primitive.PrimitiveLoop) is a
[pypot.primitive.primitive.Primitive](pypot.primitive.html#pypot.primitive.primitive.Primitive) that repeats its behavior forever.

A primitive is supposed to be independent of other primitives. In
particular, a primitive is not aware of the other primitives running on
the robot at the same time.

This is really important when you create complex behaviors - such as
balance - where many primitives are needed. Adding another primitive -
such as walking - should be direct and not force you to rewrite
everything. Furthermore, the balance primitive could also be combined
with another behavior - such as shoot a ball - without modifying it.

### Primitive manager

To ensure this independence, the primitive is running in a sort of
sandbox. More precisely, this means that the primitive has not direct
access to the robot. It can only request commands (e.g. set a new goal
position of a motor) to a [pypot.primitive.manager.PrimitiveManager](pypot.primitive.html#pypot.primitive.manager.PrimitiveManager)
which transmits them to the "real" robot. As multiple primitives can run
on the robot at the same time, their request orders are combined by the
manager.

The primitives all share the same manager. In further versions, we would
like to move from this linear combination of all primitives to a
hierarchical structure and have different layer of managers.

The manager uses a filter function to combine all orders sent by
primitives. By default, this filter function is a simple mean but you
can choose your own specific filter (e.g. add function).

> **warning**
>
> You should not mix control through primitives and direct control
> through the [pypot.robot.robot.Robot](pypot.robot.html#pypot.robot.robot.Robot). Indeed, the primitive manager
> will overwrite your orders at its refresh frequency: i.e. it will look
> like only the commands send through primitives will be taken into
> account.

### Default primitives

An example on how to run a primitive is shown
[here](quickstart-primitive.html).

Another primitive provided with Pypot is the
[pypot.primitive.utils.Sinus](pypot.primitive.html#pypot.primitive.utils.Sinus) one. It allows you to apply a sinusoidal
move of a given frequency and intensity to a motor or a list of motors:

    from pypot.primitive.utils import Sinus

    sinus_prim_z = Sinus(poppy, 50, "head_z", amp=30, freq=0.5, offset=0, phase=0)

    #set phase to 90° to have the y sinus 1/4 of phase late compared to the z sinus
    sinus_prim_y = Sinus(poppy, 50, "head_y", amp=20, freq=1, offset=0, phase=90)

    print "start moving"    
    sinus_prim_z.start()
    sinus_prim_y.start()

    time.sleep(20)

    print "pause move" 
    sinus_prim_z.pause()
    sinus_prim_y.pause()

    time.sleep(5)

    print "restart move"    
    sinus_prim_z.resume()
    sinus_prim_y.resume()

    time.sleep(20)

    print "stop moving" 
    sinus_prim_z.stop()
    sinus_prim_y.stop()

Other default primitives are:

-   [pypot.primitive.utils.Cosinus](pypot.primitive.html#pypot.primitive.utils.Cosinus) for a cosinus move
-   [pypot.primitive.utils.Square](pypot.primitive.html#pypot.primitive.utils.Square) for a square (go to max position,
    wait duty\*cycle time, go to minposition, wait (1 - duty)\*cycle
    time)
-   [pypot.primitive.utils.PositionWatcher](pypot.primitive.html#pypot.primitive.utils.PositionWatcher): records and saves all
    positions of the given motors. You have a plot function to see your
    data.
-   [pypot.primitive.utils.SimplePosture](pypot.primitive.html#pypot.primitive.utils.SimplePosture): you should define a
    target\_position as a dictionnary and the robot will go to this
    position

## Writing your own primitive

To write you own primitive, you have to subclass the
[pypot.primitive.primitive.Primitive](pypot.primitive.html#pypot.primitive.primitive.Primitive) class. It provides you with basic
mechanisms (e.g. connection to the manager, setup of the thread) to
allow you to directly "plug" your primitive to your robot and run it.

### Important instructions

When writing your own primitive, you should always keep in mind that you
should never directly pass the robot or its motors as argument and
access them directly. You have to access them through the self.robot and
self.robot.motors properties.

Indeed, at instantiation the [pypot.robot.robot.Robot](pypot.robot.html#pypot.robot.robot.Robot) (resp.
[pypot.dynamixel.motor.DxlMotor](pypot.dynamixel.html#pypot.dynamixel.motor.DxlMotor)) instance is transformed into a
[pypot.primitive.primitive.MockupRobot](pypot.primitive.html#pypot.primitive.primitive.MockupRobot) (resp.
[pypot.primitive.primitive.MockupMotor](pypot.primitive.html#pypot.primitive.primitive.MockupMotor)). Those class are used to
intercept the orders sent and forward them to the
[pypot.primitive.manager.PrimitiveManager](pypot.primitive.html#pypot.primitive.manager.PrimitiveManager) which will combine them. By
directly accessing the "real" motor or robot you circumvent this
mechanism and break the sandboxing.

If you have to specify a list of motors to your primitive (e.g. apply
the sinusoid primitive to the specified motors), you should either give
the motors name and access the motors within the primitive or transform
the list of [pypot.dynamixel.motor.DxlMotor](pypot.dynamixel.html#pypot.dynamixel.motor.DxlMotor) into
[pypot.primitive.primitive.MockupMotor](pypot.primitive.html#pypot.primitive.primitive.MockupMotor) thanks to the
[pypot.primitive.primitive.Primitive.get\_mockup\_motor](pypot.primitive.html#pypot.primitive.primitive.Primitive) method. For
instance:

    class MyDummyPrimitive(pypot.primitive.Primitive):
        def run(self, motors_name):
            motors = [getattr(self.robot, name) for name in motors_name]

            while True:
                for m in motors:
                    ...

or:

    class MyDummyPrimitive(pypot.primitive.Primitive):
        def run(self, motors):
            fake_motors = [self.get_mockup_motor(m) for m in motors]

            while True:
                for m in fake_motors:
                    ...

When overriding the [pypot.primitive.primitive.Primitive](pypot.primitive.html#pypot.primitive.primitive.Primitive), you are
responsible for correctly handling those events. For instance, the stop
method will only trigger the should stop event that you should watch in
your run loop and break it when the event is set. In particular, you
should check the
[pypot.utils.stoppablethread.StoppableThread.should\_stop](pypot.utils.html#pypot.utils.stoppablethread.StoppableThread) and
[pypot.utils.stoppablethread.StoppableThread.should\_pause](pypot.utils.html#pypot.utils.stoppablethread.StoppableThread)  in your run
loop. You can also use the
[pypot.utils.stoppablethread.StoppableThread.wait\_to\_stop](pypot.utils.html#pypot.utils.stoppablethread.StoppableThread) and
[pypot.utils.stoppablethread.StoppableThread.wait\_to\_resume](pypot.utils.html#pypot.utils.stoppablethread.StoppableThread) to wait
until the commands have really been executed.

> **note**
>
> You can refer to the source code of the
> [pypot.primitive.primitive.LoopPrimitive](pypot.primitive.html#pypot.primitive.primitive.LoopPrimitive) for an example of how to
> correctly handle all these events.

### Examples

As an example, let's write a simple primitive that recreate the dance
behavior written in the dance\_ section. Notice that to pass arguments
to your primitive, you have to override the
[pypot.primitive.primitive.Primitive.\_\_init\_\_](pypot.primitive.html#pypot.primitive.primitive.Primitive) method.

> **note**
>
> You should always call the super constructor if you override the
> [pypot.primitive.primitive.Primitive.\_\_init\_\_](pypot.primitive.html#pypot.primitive.primitive.Primitive) method.

    import time

    import pypot.primitive
    class DancePrimitive(pypot.primitive.Primitive):

        def __init__(self, robot, amp=30, freq=0.5):
            self.robot = robot
            self.amp = amp
            self.freq = freq
            pypot.primitive.Primitive.__init__(self, robot)

        def run(self):
            amp = self.amp
            freq = self.freq
            # self.elapsed_time gives you the time (in s) since the primitive has been running
            while self.elapsed_time < 30:
                x = amp * numpy.sin(2 * numpy.pi * freq * self.elapsed_time)

                self.robot.base_pan.goal_position = x
                self.robot.head_pan.goal_position = -x

                time.sleep(0.02)

To run this primitive on your robot, you simply have to do:

    ergo_robot = pypot.robot.from_config(...)

    dance = DancePrimitive(ergo_robot,amp=60, freq=0.6)
    dance.start()

If you want to make the dance primitive infinite you can use the
[pypot.primitive.primitive.LoopPrimitive](pypot.primitive.html#pypot.primitive.primitive.LoopPrimitive) class:

    class LoopDancePrimitive(pypot.primitive.LoopPrimitive):
        def __init__(self, robot, refresh_freq, amp=30, freq=0.5):
            self.robot = robot
            self.amp = amp
            self.freq = freq
            LoopPrimitive.__init__(self, robot, refresh_freq)

        # The update function is automatically called at the frequency given on the constructor
        def update(self):
            amp = self.amp
            freq = self.freq
            x = amp * numpy.sin(2 * numpy.pi * freq * self.elapsed_time)

            self.robot.base_pan.goal_position = x
            self.robot.head_pan.goal_position = -x

And then run it with:

    ergo_robot = pypot.robot.from_config(...)

    dance = LoopDancePrimitive(ergo_robot, 50, amp = 40, freq = 0.3)
    # The robot will dance until you call dance.stop()
    dance.start()

## Attaching a primitive to the robot

You can also attach a primitive to the robot (at start up for example)
and then use it more easily.

For example with our DancePrimitive:

    ergo_robot = pypot.robot.from_config(...)

    ergo_robot.attach_primitive(DancePrimitive(ergo_robot), 'dance')
    ergo_robot.dance.start()

By attaching a primitive to the robot, you make it accessible from
within other primitive:

    class SelectorPrimitive(pypot.primitive.Primitive):
        def run(self):
            if song == 'my_favorite_song_to_dance' and not self.robot.dance.is_alive():
                self.robot.dance.start()

> **note**
>
> In this case, instantiating the DancePrimitive within the
> SelectorPrimitive would be another solution.