---
title: Declarative finite state machines in Python
date: 2024-11-19T19:08:11+05:00
draft: false
---
<br>
I first got acquainted with finite state machines during my fascination with gamemade. Everyone uses this abstraction in game development. However, this is not their only field of application.

Finite state machines (FSM) are all around us, even if we don't notice them or don't know what they are. A ticket in jira, a transaction in a database, a user registration page in a social network. All of these things have one thing in common: state.
## What are finite state machines?
I don't want to deepen into mathematical abstractions, so I'll be brief.

A finite state machine is defined by the following components:
- A finite set of states.
- Being in only one of the states at a particular point in time.
- Rules describing possible transitions between states.
#### Not clear? 
There are couple examples:

#### Traffic light:
- Has a finite number of states: “Red”, ‘Yellow’, ‘Green’.
- Has rules for transitions between states: 
	- red -> yellow
	- yellow -> green
	- green -> red
```
┌───┐     
│red│     
└△─┬┘     
 │┌▽─────┐
 ││yellow│
 │└┬─────┘
┌┴─▽──┐   
│green│   
└─────┘   
```
#### System login page:
- A finite number of states: “Enter login”, ‘Enter password’, ‘Access granted’, ‘Access denied’.
- Transition rules:
	- Login Input → Password Input (login entered)
	- Enter password → Access granted (password is correct)
	- Enter password → Access denied (password is incorrect)
	- Access denied → Login entry (retry)

```
┌─────┐           
│login│           
└△─┬──┘           
 │┌▽────────────┐ 
 ││  password   │ 
 │└┬───────────┬┘ 
┌┴─▽──────────┐│  
│access denied││  
└─────────────┘│  
┌──────────────▽─┐
│ access granted │
└────────────────┘
```
## Explicitly emphasize the state rather than using indirect features
We as programmers need to be able to spot this pattern in time in the system we are working on and explicitly highlight the state.

Take a look at this snippet:
```python
if (
	support_ticket.last_message.from == 'support' 
	and (
		datetime.now() - support_ticket.last_message.time
	).hours > 1
) or (
	  support_ticket.answer_rating is not None 
	  and support_ticket.answer_rating>= 3
  ):
    #do some logic with ticket
``````
Do you understand what's going on in it?
It takes me a cognitive effort to figure it out.
But it's just a check that the support ticket is closed.

Now take a look at this:
```python
if support_ticket.status is SupportTicketStatus.Closed:
	# do some logic with ticket
```

All I did was emphasize the explicit state. The finite state machine did not appear in the system, it existed before, it's just that now it's explicit.

The advantages of this approach seem obvious:
1. Now I have less code and I don't need to maintain all the indirect state attributes. It is enough to describe the transition conditions in one place.
2. Looking at an object, I can immediately tell what state it is in. I don't need to keep all the indirect state features in my head.

## Implementing FSM in python
Let's write a minimal finite state machine in python.
And to make it more fun, let's figure out a way to describe it declaratively.
[What is “declaratively”?](https://en.wikipedia.org/wiki/Declarative_programming).
Let's take a task from the task tracker as an example.
### 1. Let's list all the possible states of the FSM
1. `Enum` is perfect for this
```python
from enum import Enum, auto

class TaskStatus(Enum):
    CREATED = auto()
    QUEUED = auto()
    IN_PROGRESS = auto()
    CANCELLED = auto()
    FINISHED = auto()
```

2. I want to be able to describe the state of any object and specify the initial state. 
```python 
class Task:
    status = State(TaskStatus, initial_state=TaskStatus.CREATED)
```

3. When an object is created, its state must match `initial_state`
```python 
task = Task()
assert.task.status is TaskStatus.CREATED
```

4. Let's implement the required logic.  We will use [descriptor protocol](https://docs.python.org/3/howto/descriptor.html) to conceive it
```python
from enum import Enum


class State:
    def __init__(self, states: type[Enum], initial_state: Enum):
        self._all_states = states
        self._initial_state = initial_state

    def __set_name__(self, owner: type, attr_name: str) -> None:
        """
        Called when creating the class that owns.
        """

        self._attr_name = "_" + attr_name

    def __get__(self, instance: object, objtype: type | None):
        """
        Take the current state directly from the host object.
        The name we wrote in __set_name__ is used as the key
        """
		if instance is None:  # When accessed from a class, return itself
			return self
        return instance.__dict__.get(
            self._attr_name, 
            self._initial_state # default value
        )
```
### 2. Let's list the transitions
1. Since we decided to go the declarative route, let's describe the transitions in the class body as attributes:
```python 
class Task:
    status = State(TaskStatus, initial_state=TaskStatus.CREATED)

    enqueue = status.transition(
	    source=TaskStatus.CREATED, 
	    dest=TaskStatus.QUEUED,
	)
    proceed = status.transition(
	    source=TaskStatus.QUEUED, 
	    dest=TaskStatus.IN_PROGRESS,
	)
    prioritize = status.transition(
	    source=TaskStatus.CREATED, 
	    dest=TaskStatus.IN_PROGRESS,
	)
    cancel = status.transition(
	    source=[
		    TaskStatus.CREATED, 
		    TaskStatus.QUEUED,
		    TaskStatus.IN_PROGRESS
		], 
		dest=TaskStatus.CANCELLED,
	)
    finish = status.transition(
	    source=TaskStatus.IN_PROGRESS, 
	    dest=TaskStatus.FINISHED,
	)
```
The “cancel” transition is possible from different statuses, so we pass a list of values as source.
Thus we have this scheme of transitions:
```
┌───────┐       
│created│       
└┬─┬─┬──┘       
 │ │┌▽────────┐ 
 │ ││ queued  │ 
 │ │└┬───────┬┘ 
 │┌▽─▽──────┐│  
 ││cancelled││  
 │└△────────┘│  
┌▽─┴─────────▽─┐
│ in_progress  │
└┬─────────────┘
┌▽───────┐      
│finished│      
└────────┘      
```
2. I want it to be impossible to manually change state. Only through the specified transitions:
```python
try:
    task.status = TaskStatus.IN_PROGRESS
except AttributeError:
    ...
```
3. I want the described transitions to work as methods:
```python
task = Task()
assert task.status is TaskStatus.CREATED

task.enqueue()
assert task.status is TaskStatus.QUEUED

task.proceed()
assert task.status is TaskStatus.IN_PROGRESS

task.finish()
assert task.status is TaskStatus.FINISHED
```
4. `State` should control which transition is called and prevent us from taking the incorrect route.
```python
task = Task()
assert task.status is TaskStatus.CREATED

try:
    task.finish()
except ImpossibleTransitionError:
    ...

assert task.status is TaskStatus.CREATED
```
5. Let's implement the logic of transitions. For this purpose, let's extend the class-descriptor `State`:
```python

class ImpossibleTransitionError(Exception):
    pass
    
class State:
    def __set__(self, instance: object, value: Any):
        """
        Raise an exception when trying to update a state directly.
        """
        raise AttributeError()

    def transition(self, source: Enum | Collection[Enum], dest: Enum):
        """
        This method creates a _update_state closure that does the transition
        """
        # Check that the correct Enum is passed in
        if not dest in self._all_states:
            raise ValueError(f'Destination state {repr(dest)} not found')
        # For convenience, treat a single source as a single-element list
        if not isinstance(source, Collection):
            source = [source]
        for source_state in source:
            # Same check as for dest
            if not source_state in self._all_states:
                raise ValueError(f'Source state {repr(source_state)} not found')

        def _update_state(instance):
            """
            Gets the current state and checks if the transition is possible.
            Either raises an exception or updates the state
            """
            state = self.__get__(instance, None)
            if state in source:
                instance.__dict__[self._attr_name] = dest
            else:
                raise ImpossibleTransitionError()

        return _update_state
```

### 3. Let's put the code together:
1. fsm.py
``` python
from collections.abc import Collection
from typing import Any
from enum import Enum

class ImpossibleTransitionError(Exception):
    pass

class State:
    def __init__(self, states: type[Enum], initial_state: Enum):
        self._all_states = states
        self._initial_state = initial_state

    def __set_name__(self, owner: type, attr_name: str) -> None:
        """
        Called when creating the class that owns.
        """

        self._attr_name = "_" + attr_name

    def __get__(self, instance: object, objtype: type | None):
        """
        Take the current state directly from the host object.
        The name we wrote in __set_name__ is used as the key
        """
		if instance is None:  # When accessed from a class, return itself
			return self
        return instance.__dict__.get(
            self._attr_name, 
            self._initial_state # default value
        )

    def __set__(self, instance: object, value: Any):
        """
        Raise an exception when trying to update a state directly.
        """
        raise AttributeError()

    def transition(self, source: Enum | Collection[Enum], dest: Enum):
        """
        This method creates a _update_state closure that does the transition
        """
        # Check that the correct Enum is passed in
        if not dest in self._all_states:
            raise ValueError(f'Destination state {repr(dest)} not found')
        # For convenience, treat a single source as a single-element list
        if not isinstance(source, Collection):
            source = [source]
        for source_state in source:
            # Same check as for dest
            if not source_state in self._all_states:
                raise ValueError(f'Source state {repr(source_state)} not found')

        def _update_state(instance):
            """
            Gets the current state and checks if the transition is possible.
            Either raises an exception or updates the state
            """
            state = self.__get__(instance, None)
            if state in source:
                instance.__dict__[self._attr_name] = dest
            else:
                raise ImpossibleTransitionError()

        return _update_state
```
2. task.py
```python
from enum import Enum, auto
from fsm import State, ImpossibleTransitionError

class TaskStatus(Enum):
    CREATED = auto()
    QUEUED = auto()
    IN_PROGRESS = auto()
    CANCELLED = auto()
    FINISHED = auto()

class Task:
    status = State(TaskStatus, initial_state=TaskStatus.CREATED)

    enqueue = status.transition(
        source=TaskStatus.CREATED,
        dest=TaskStatus.QUEUED,
    )
    proceed = status.transition(
        source=TaskStatus.QUEUED,
        dest=TaskStatus.IN_PROGRESS,
    )
    prioritize = status.transition(
        source=TaskStatus.CREATED,
        dest=TaskStatus.IN_PROGRESS,
    )
    cancel = status.transition(
        source=[TaskStatus.CREATED, TaskStatus.QUEUED, TaskStatus.IN_PROGRESS],
        dest=TaskStatus.CANCELLED,
    )
    finish = status.transition(
        source=TaskStatus.IN_PROGRESS,
        dest=TaskStatus.FINISHED,
    )

task_a = Task()
assert task_a.status is TaskStatus.CREATED

task_a.enqueue()
assert task_a.status is TaskStatus.QUEUED

task_a.proceed()
assert task_a.status is TaskStatus.IN_PROGRESS

task_a.finish()
assert task_a.status is TaskStatus.FINISHED


task_b = Task()
assert task_b.status is TaskStatus.CREATED

try:
    task_b.finish()
except ImpossibleTransitionError as e:
    assert True
else:
    assert False, "Can't make the transition CREATED -> FINISHED”

# Status has not changed
assert task_b.status == TaskStatus.CREATED

try:
    task_b.status = TaskStatus.IN_PROGRESS
except AttributeError as e:
    assert True
else:
    assert False, "Can't change the status manually”
# Status has not changed
assert task_b.status == TaskStatus.CREATED
```

### 4. What's next.
Here are a few ways where you can develop this code. I suggest you, reader, stretch your brain and implement these features yourself:
1. Switches and loops
```python
class LeverState(Enum):
    ON = auto()
    OFF = auto()

class Lever:
    state = State(LeverState, initial_state = LeverState.OFF)
    switch = state.cycle(SwitchState.OFF, SwitchState.ON)

lever = Lever()
assert lever.state is LeverState.OFF

lever.switch()
assert lever.state is LeverState.ON

lever.switch()
assert lever.state is LeverState.OFF
```

```python
class TrafficLightColor(Enum):
    GREEN = auto()
    RED = auto()
    YELLOW = auto()

class TrafficLight:
    color = State(TrafficLightColor, initial_state=TrafficLightColor.RED)
    change_color = color.cycle(
        TrafficLightColor.RED,
        TrafficLightColor.GREEN, 
        TrafficLightColor.YELLOW
    )

traffic_light = TrafficLight()
assert traffic_light.color is TrafficLightColor.RED

traffic_light.change_color()
assert traffic_light.color is TrafficLightColor.GREEN

...  # and so on
```
2. Callbacks
```python
class Task:
    status = State(TaskStatus, initial_state=TaskStatus.CREATED)

    enqueue = status.transition(
        source=TaskStatus.CREATED,
        dest=TaskStatus.QUEUED,
    )
    
    proceed = status.transition(
        source=TaskStatus.QUEUED,
        dest=TaskStatus.IN_PROGRESS,
    )

    @status.on_transition
    def on_transition(source: TaskStatus, dest: TaskStatus):
        print("Transitioning from {source} to {destination}")

    @status.on_transition(proceed)
    def on_proceed(source: TaskStatus, dest: TaskStatus):
        print("Proceeding task")
```

## Summary
We've figured out what finite state machines are and why it's important and useful to notice them in the systems we work with.
We learned how to describe a finite state machine in python in a declarative way and outlined ways to develop this system.

<br>
## Bibliography

- [Wikipedia - Finite state machines](https://en.wikipedia.org/wiki/Finite-state_machine)
- [Wikipedia - Declarative programing](https://en.wikipedia.org/wiki/Declarative_programming)
- [Python.org - Descriptor guide](https://docs.python.org/3/howto/descriptor.html)
- [Python.org - Enum HOWTO](https://docs.python.org/3/howto/enum.html)
- [Luciano Ramalho - Fluent Python chapter 23 - Descriptors](https://www.oreilly.com/library/view/fluent-python-2nd/9781492056348/)


All transition graphs in this article were generated using [Diagon](https://diagon.arthursonzogni.com/#GraphPlanar).