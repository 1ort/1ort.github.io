---
title: Декларативные конечные автоматы на Python
date: 2024-11-19T19:08:11+05:00
draft: false
---
<br>
С конечными автоматами я впервые познакомился во времена своего увлечения геймдевом. В разработке игр все поголовно используют эту абстракцию. Однако, это далеко не единственная их сфера применения.

Конечные автоматы повсюду вокруг нас, даже если мы их не замечаем, или не знаем, что это такое. Тикет в jira, транзакция в базе данных, страница регистрации пользователя в соцсети. Всё перечисленное объединяет одно - состояние.

## Что такое конечные автоматы?
Не хочу углубляться в математические абстракции, поэтому буду краток.

Конечный автомат определяется следующими компонентами:
- Конечное множество состояний.
- Нахождение только в одном из состояний в определённый момент времени.
- Правила, описывающие возможные переходы между состояниями.

#### Непонятно?
Вот пара примеров:

#### Светофор:
- Обладает конечным множеством состояний: "Красный", "Жёлтый", "Зелёный"
- Имеет правила переходов между состояниями:
	- красный -> жёлтый
	- жёлтый -> зелёный
	- зелёный -> красный
```
.---.    
|red|    
'^-.'    
 |.V----.
 ||green|
 |'.----'
.'-V---. 
|yellow| 
'------' 
```
#### Страница логина в систему:
-  Конечное множество состояний: "Ввод логина", "Ввод пароля", "Доступ разрешен", "Доступ запрещен"
- Правила переходов:
	- Ввод логина → Ввод пароля (логин введен)
	- Ввод пароля → Доступ разрешен (пароль верный)
	- Ввод пароля → Доступ запрещен (пароль неверный)
	- Доступ запрещен → Ввод логина (повторная попытка)

```
.-----.           
|login|           
'^-.--'           
 |.V------------. 
 ||  password   | 
 |'.-----------.' 
.'-V----------.|  
|access denied||  
'-------------'|  
.--------------V-.
| access granted |
'----------------'
```
## Выделяйте состояние явно, а не используйте косвенные признаки
Нам, как программистам нужно уметь вовремя замечать этот паттерн в системе, над которой мы работаем и явно выделить состояние.

Взгляните на этот сниппет:
```python
if (
	support_ticket.last_message.from == 'support'
	and (
		datetime.now() - support_ticket.last_message.time
	).hours > 1
) or (
	  support_ticket.answer_rating is not None
	  and support_ticket.answer_rating >= 3
  ):
    #do some logic with ticket
```
Вы понимаете, что в нём происходит?
У меня требует когнитивных усилий разобраться в нём.
Но ведь это всего лишь проверка на то, что обращение в поддержку закрыто.

А теперь взгляните сюда:
```python
if support_ticket.status is SupportTicketStatus.Closed:
	# do some logic with ticket
```

Всё, что я сделал - это выделил явное состояние. Конечный автомат в системе от этого не появился, он существовал и до этого, просто теперь он явный.

Преимущества такого подхода очевидны:
1. Теперь у меня меньше кода и мне не нужно поддерживать все встречающиеся косвенные признаки состояний. Достаточно в одном месте описать условия перехода.
2. Глядя на объект, я сразу же могу сказать, в каком он состоянии. Мне не нужно держать в голове все косвенные признаки состояния.

## Реализация fsm на python
Напишем минимальный конечный автомат на python.
А чтобы было веселее, придумаем способ описать его декларативно.
[Что такое "декларативно"?](https://en.wikipedia.org/wiki/Declarative_programming)
Для примера возьмём задачу из таск-трекера.
### 1. Перечислим все возможные состояния FSM
1. Для этого идеально подойдёт Enum
```python
from enum import Enum, auto

class TaskStatus(Enum):
    CREATED = auto()
    QUEUED = auto()
    IN_PROGRESS = auto()
    CANCELLED = auto()
    FINISHED = auto()
```

2. Я хочу иметь описать состояние любого объекта и указать начальное состояние.
```python
class Task:
    status = State(TaskStatus, initial_state=TaskStatus.CREATED)
```

3. При создании объекта, его состояние должно соответствовать initial_state
```python
task = Task()
assert.task.status is TaskStatus.CREATED
```

4. Давайте реализуем требуемую логику.  Для задуманного будем использовать [протокол дескриптора](https://docs.python.org/3/howto/descriptor.html)
```python
from enum import Enum


class State:
    def __init__(self, states: type[Enum], initial_state: Enum):
        self._all_states = states
        self._initial_state = initial_state

    def __set_name__(self, owner: type, attr_name: str) -> None:
        """
        Вызывается при создании класса, которому принадлежит.
        """

        self._attr_name = "_" + attr_name

    def __get__(self, instance: object, objtype: type | None):
        """
        Взять напрямую из объекта-хозяина текущее состояние.
        В качестве ключа используется имя, которое мы записали в __set_name__
        """
		if instance is None:  # При обращении из класса вернём сам дескриптор
			return self
        return instance.__dict__.get(
            self._attr_name,
            self._initial_state # значение по умолчанию
        )
```
### 2. Перечислим переходы
1. Раз мы решили идти по декларативному пути, опишем переходы в теле класса как аттрибуты:
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
Переход "cancel" возможен из разных статусов, поэтому в качестве source передадим список значений.
Таким образом мы имеем вот такую схему переходов:
```
.-------.       
|created|       
'.-.-.--'       
 | |.V--------. 
 | || queued  | 
 | |'.-------.' 
 |.V-V------.|  
 ||cancelled||  
 |'^--------'|  
.V-'---------V-.
| in progress  |
'.-------------'
.V-------.      
|finished|      
'--------'      
```
2. Я хочу чтобы нельзя было вручную переключить состояние. Только через указанные переходы:
```python
try:
    task.status = TaskStatus.IN_PROGRESS
except AttributeError:
    ...
```
3. Я хочу чтобы описанные переходы работали как методы:
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
4. State должен контролировать, какой переход вызывается и не позволять нам перейти по некорректному маршруту.
```python
task = Task()
assert task.status is TaskStatus.CREATED

try:
    task.finish()
except ImpossibleTransitionError:
    ...

assert task.status is TaskStatus.CREATED
```
5. Реализуем логику переходов. Для этого дополним класс-декскриптор State :
```python

class ImpossibleTransitionError(Exception):
    pass

class State:
    def __set__(self, instance: object, value: Any):
        """
        Поднять исключение при попытке обновить состояние напрямую.
        """
        raise AttributeError()

    def transition(self, source: Enum | Collection[Enum], dest: Enum):
        """
        Этот метод создаёт замыкание _update_state, которое делает переход
        """
        # Проверка, что передан корректный Enum
        if not dest in self._all_states:
            raise ValueError(f'Destination state {repr(dest)} not found')
        # Для удобства рассматривать одиночный source как единичный список
        if not isinstance(source, Collection):
            source = [source]
        for source_state in source:
            # Такая же проверка как для dest
            if not source_state in self._all_states:
                raise ValueError(f'Source state {repr(source_state)} not found')

        def _update_state(instance):
            """
            Получает текущее состояние объекта и проверяет, возможен ли переход.
            Либо поднимает исключение либо обновляет состояние
            """
            state = self.__get__(instance, None)
            if state in source:
                instance.__dict__[self._attr_name] = dest
            else:
                raise ImpossibleTransitionError()

        return _update_state
```

### 3. Соберём код воедино:
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
        Вызывается при создании класса, которому принадлежит.
        """
        self._attr_name = "_" + attr_name

    def __get__(self, instance: object, objtype: type | None):
        """
        Взять напрямую из объекта-хозяина текущее состояние.
        В качестве ключа используется имя, которое мы записали в __set_name__
        """
        if instance is None:  # При обращении из класса вернём сам дескриптор
            return self
        return instance.__dict__.get(
            self._attr_name,
            self._initial_state # значение по умолчанию
        )

    def __set__(self, instance: object, value: Any):
        """
        Поднять исключение при попытке обновить состояние напрямую.
        """
        raise AttributeError()

    def transition(self, source: Enum | Collection[Enum], dest: Enum):
        """
        Этот метод создаёт замыкание _update_state, которое делает переход
        """
        # Проверка, что передан корректный Enum
        if not dest in self._all_states:
            raise ValueError(f'Destination state {repr(dest)} not found')
        # Для удобства рассматривать одиночный source как единичный список
        if not isinstance(source, Collection):
            source = [source]
        for source_state in source:
            # Такая же проверка как для dest
            if not source_state in self._all_states:
                raise ValueError(f'Source state {repr(source_state)} not found')

        def _update_state(instance):
            """
            Получает текущее состояние объекта и проверяет, возможен ли переход.
            Либо поднимает исключение, либо обновляет состояние
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
    assert False, "Нельзя совершить переход CREATED -> FINISHED"

# Статус не изменился
assert task_b.status == TaskStatus.CREATED

try:
    task_b.status = TaskStatus.IN_PROGRESS
except AttributeError as e:
    assert True
else:
    assert False, "Нельзя менять состояние вручную"

# Статус не изменился
assert task_b.status == TaskStatus.CREATED
```

### 4. Что дальше?
Вот несколько путей, куда можно развить этот код. Предлагаю тебе, читатель, размяться и реализовать эту функциональность:
1. Переключатели и циклы
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
2. Коллбэки
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

## Резюме
Мы разобрались, что такое конечные автоматы и почему важно и полезно их замечать в системах, с которыми работаем.
Мы научились описывать на python конечный автомат декларативным способом и наметили пути развития этой системы.

<br>
## Библиография

- [Wikipedia - Finite state machines](https://en.wikipedia.org/wiki/Finite-state_machine)
- [Wikipedia - Declarative programing](https://en.wikipedia.org/wiki/Declarative_programming)
- [Python.org - Descriptor guide](https://docs.python.org/3/howto/descriptor.html)
- [Python.org - Enum HOWTO](https://docs.python.org/3/howto/enum.html)
- [Luciano Ramalho - Fluent Python chapter 23 - Descriptors](https://www.oreilly.com/library/view/fluent-python-2nd/9781492056348/)

Все графики переходов в этой статье были сгенерированы при помощи [Diagon](https://diagon.arthursonzogni.com/#GraphPlanar)
