# Task Planning Frameworks
This is a cautionary tale about some the different task planning frameworks we've tried. If you are ever considering adopting a new framework, review this first and consider if you're treading the same ground here. This covers the 3 task planning frameworks we've used while I've been on this club.

## Home-grown Task System
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

Competition that year was done without any task planning framework and in an [extremely rushed manner](https://github.com/DukeRobotics/robosub-ros/blob/4e0026a654cca2771290437654766553a2ee3eed/onboard/catkin_ws/src/controls/scripts/comp_2023.py). This was not purely SMACH's fault, but we were having some confusing issues and wanted to cut out the parts we were uncertain about.

### async-await Coroutines
By this time a good chunk of the team was ready to just write plain Python functions without any formal framework. This line of thinking heavily influenced our choice for a new framework. I had used a coroutine system before, in the form of C#'s `IEnumerator` and `yield` statements. Something similar was possible in Python using generators, but I opted to explore the possibility of using `async` and `await` to do something similar but better.

The guiding principles of this system were to:
- Require minimal boilerplate and excess complexity above basic Python functions
- Allow for using normal language structures (loops) for control flow

At the time of writing, this system has yet to be fully tested, but I am optimistic about its likelihood of success. The basic principle is "functions but they can be paused and continued later". Calling a task returns a `TaskWrapper` object, which can be stepped through or run as a whole depending on your desires. Any `await Yield()` call causes the function to pause and return all the way back down to the lowest level where `.send()` or `.step()` has been called previously. This should allow tasks to be run in "parallel" without any actual parallelism.

Additionally, you can yield specific values and send back in values through `.send()`, allowing for communication up and down the stack. However, this shouldn't be too complex for the end user (hopefully), as if you don't want to interact with the coroutine system, you can just put it in a box and think "I just put `await` whenever I call another task".

Anyway, conclusions is, fingers crossed, and if you are considering a new task planning framework in the future, make sure you aren't repeating the mistakes of the past.