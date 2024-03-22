# Task Planning Frameworks
This is a cautionary tale about some the different task planning frameworks we've tried. If you are ever considering adopting a new framework, review this first and consider if you're treading the same ground here. This covers the three task planning frameworks we've used between 2020-2024.

## Homegrown Task System
The first of these three was mostly created by members of the team and centered around a basic [`Task`](https://github.com/DukeRobotics/robosub-ros/blob/c0a74e1f9032f63046a9f0f660616d0d40624bfe/onboard/catkin_ws/src/task_planning/scripts/task.py) class. Tasks would override the `_on_task_start()` and `_on_task_run()` methods to do initialization and update work respectively. The `_on_task_run()` method was intended to be generally non-blocking. This meant that any state had to be stored as members of the task object.

To composite our smaller tasks into larger tasks, we had ["combination tasks"](https://github.com/DukeRobotics/robosub-ros/blob/c0a74e1f9032f63046a9f0f660616d0d40624bfe/onboard/catkin_ws/src/task_planning/scripts/combination_tasks.py), which were tasks taking in other tasks to do common things like run several tasks at the same time and exit if any finish. These were useful, but probably could be better stated as plain code.

The core issue here is that if we want to represent a loop in a task, we're doing something like:
```python
def _on_task_start(self):
    <loop initialization>

def _on_task_run(self):
    if <loop condition>:
        <contents of loop>
        <loop increment>
```
This isn't terrible, but gets problematic when we want to do anything more complex than a single loop. Keeping track of this state across method calls gets kind of awkward.

## SMACH
[SMACH](https://wiki.ros.org/smach) is a package that can be used for managing different kinds of state machines. We [adapted our existing `Task` class](https://github.com/DukeRobotics/robosub-ros/blob/5fe1df1778a17dfd1b6e9976a63f23dd90880afe/onboard/catkin_ws/src/task_planning/scripts/task.py) to serve as a wrapper for SMACH's own `State` class. The main purpose of this was to reuse the parts of the code used for accessing robot state through properties of the `Task` object.

When selecting SMACH, our intention was to allow tasks to block, then use SMACH's `Concurrence` class to run tasks in parallel when we needed it. We adapted a good chunk of our code before concluding that not only is actual threading a sketchy way to handle tasks running at the same time, but SMACH had an [unpleasant amount of boilerplate](https://github.com/DukeRobotics/robosub-ros/blob/4e0026a654cca2771290437654766553a2ee3eed/onboard/catkin_ws/src/task_planning/scripts/buoy_task.py) and was often confusing to read for anyone not familiar with it. We ended up deciding to make states non-blocking (returning a different state transition depending on if it should finish or repeat), which reduced a lot of the benefit we were hoping to gain from this system. There were a few other nails in the coffin:
- Passing data between states was only done using a `userdata` object, which required any read or written values to be declared in the state initialization.
- Repeatedly cycling through a task had somewhat unpredictable timing, which might have interacted weirdly with our current controls system (requiring a specific rate).

RoboSub 2023 was done without any task planning framework. Instead, all task planning code was put into [one large file](https://github.com/DukeRobotics/robosub-ros/blob/4e0026a654cca2771290437654766553a2ee3eed/onboard/catkin_ws/src/controls/scripts/comp_2023.py). This was not purely SMACH's fault, but didn't have the time to debug the issues we were having with it at competition and fell back to a simpler system using pure Python.

## async-await Coroutines
Post-RoboSub 2023, a good chunk of the team was ready to just write plain Python functions without a rigid framework.

The guiding principles of this system were to:
- Require minimal boilerplate and excess complexity above basic Python functions
- Allow for using normal language structures (loops) for control flow

We chose to use coroutines. The basic principle is "functions that can be paused and continued later". Calling a task coroutine returns a `Task` object, which can be stepped through or run as a whole depending on your desires. Any `await Yield()` call causes the function to pause and return all the way back down to the lowest level where `.send()` or `.step()` has been called previously. This should allow tasks to be run in "parallel" without any actual parallelism.

Additionally, you can yield specific values and send back in values through `.send()`, allowing for communication up and down the stack. And, frequent yielding provides parent tasks fine-grained control over the child tasks.

## Conclusion
If you are considering a new task planning framework in the future, make sure you aren't repeating the mistakes of the past.

Good luck!
