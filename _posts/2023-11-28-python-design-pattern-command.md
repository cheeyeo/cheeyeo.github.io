---
layout: post
show_meta: true
title: Using the Command design pattern
header: Using the Command design pattern
date: 2023-11-28 00:00:00
summary: Using the Command design pattern in refactoring python code
categories: python design-pattern command
author: Chee Yeo
---


[Command Pattern]: https://refactoring.guru/design-patterns/command
[referenced source]: https://refactoring.guru/design-patterns/command/python/example#lang-features

In a previous post, I wrote about refactoring some python code which has a requirement to run different algorithms provided as inputs using the `Strategy design pattern`. However, it is not an accurate usage of the pattern as the `Strategy` pattern is designed as a way of describing different ways of performing the same task. We have a single context object where we apply or swap different algorithms to apply to the inputs.

In the problem statement, we are given a list of commands and associated inputs to apply to a fictious cloud service. A more appropriate design pattern to use would be the [Command Pattern]

The [Command Pattern] pattern converts each command into a dedicated class, from which we instantiate a command object. Any inputs for the command are passed as parameters into the command object. 

The command objects are not invoked directly. Instead, we need to create a `Invoker` object which would hold a reference to these command objects, which would execute it based on a client's actions.

This approach has the following advantages:

* Clear separation of commands into individual classes mean that each command can be tested and also makes it easier to add new commands in the future.

* The `Invoker` object acts as a middleman between the client and command classes. Since the commands are not executed directly by the client, this allows for more complex actions such as storing a history of commands for undo/redo; deferred execution of commands; or to assemble a series of simpler commands into a more complex command.

The downside of using the `command` pattern is the verbosity of the code and the usage of the `Invoker` object. The other downside to this pattern is that the commands to be used has to be known beforehand.

However, for our given example, I think its a better fit in describing the codebase.

To start, we will create a base `Command` class which will be inherited by concrete command classes. It has a single function `execute` which needs to be implemented:

{% highlight python %}
from abc import abstractmethod, ABC


class Command(ABC):
    @abstractmethod
    def execute(self) -> None:
        raise Exception('Implement in subclasses')
{% endhighlight %}

Next, we create the two concrete command classes which will be in use in the example:

{% highlight python %}
# Define core cloud svc commands here
class AddFile(Command):
    def __init__(self, storage: dict, file: str, filesize: str) -> None:
        self._file = file
        self._filesize = filesize
        self._storage = storage

    
    def execute(self) -> str:
        if not self._file in self._storage.keys():
            self._storage[self._file] = self._filesize
            return 'true'
        else:
            print(f'AddFile: {self._file} already exists!')
            return 'false'
        

class CopyFile(Command):
    def __init__(self, storage: dict, source: str, dest: str) -> None:
        self._storage = storage
        self._source = source
        self._dest = dest


    def execute(self) -> str:
        if self._source not in self._storage.keys():
            print(f'CopyFile: {self._source} does not exist')
            return 'false'
        
        if self._source == self._dest:
            print(f'CopyFile: {self._source} cannot be the same as {self._dest}')
            return 'false'
        

        self._storage[self._dest] = self._storage[self._source]
        print(f'CopyFile: {self._source} copied to {self._dest}')
        return 'true'
{% endhighlight %}

Next, we create our invoker class, which will be the main object the client interacts with to run our commands:

{% highlight python %}
class Invoker:
    def __init__(self, storage: dict, cmds: list[str] | None = []) -> None:
        self._cmds = cmds
        self._storage = storage
        self._results = list()
        self._cmd_history = list()
        self._on_start = None
        self._on_finish = None
        self._parse_cmds()

    
    @property
    def results(self) -> list[str]:
        return self._results
    
    @property
    def history(self) -> list[Command]:
        return self._cmd_history


    def _parse_cmds(self) -> None:
        """
        Parses the passed in cmds
        """

        for cmd in self._cmds:
            algo, *params = cmd

            if algo == 'ADD_FILE':
                params = {
                    'storage': self._storage,
                    'file': params[0],
                    'filesize': params[1]
                }
                cmdx = AddFile(**params)
            elif algo == 'COPY_FILE':
                params = {
                    'storage': self._storage,
                    'source': params[0],
                    'dest': params[1]
                }
                
                cmdx = CopyFile(**params)
            
            self._cmd_history.append(cmdx)

    
    def on_start(self, cmd: Command | None = None):
        self._on_start = cmd

    
    def on_finish(self, cmd: Command | None = None):
        self._on_finish = cmd


    def execute(self) -> None:
        if self._on_start is not None:
            print(f'On start execute: {self._on_start}')
            self._on_start.execute()


        for cmd in self._cmd_history:
            self._results.append(cmd.execute())


        if self._on_finish is not None:
            print(f'On finish execute: {self._on_finish}')
            self._on_finish.execute()
            
{% endhighlight %}

There are several points of note to the implementation above, which is slightly different from the [referenced source]:

* We pass a reference of a storage dict and a list of string commands into the constructor. Rather than have the client code create the commands, we delegate it to a private function `_parse_cmds`, which creates a command object based on the value of the first tuple, which represents the name of the command to run. The remaining values are passed as parameters to the command constructor. Note that each command object has different arguments.

* We store a reference of each command into the private `_cmd_history` list. We also store the results of running each command into a `_results` list. 

* We can also run before and after actions via the `on_start` and `on_finish` functions, which each take a command object to run before and after the command list. 

* The `execute` function runs the commands stored in the `_cmd_history` list. It also runs a before and after command if it exists.


{% highlight python %}
cmds = [
    ("ADD_FILE", "/data/file.txt", "10"),
    ("ADD_FILE", "/data/file.txt", "10"),
    ("COPY_FILE", "/data/file.txt", "/data/file2.txt"),
    ("COPY_FILE", "/data/file.txt", "/data/file.txt"),
    ("COPY_FILE", "/data/non-exists.txt", "/data/file2.txt"),
    ("COPY_FILE", "/data/file2.txt", "/data/file3.txt"),
]

storage = dict()

invoker = Invoker(storage=storage, cmds=cmds)

print(invoker.history)

invoker.execute()

print(invoker.results)
assert invoker.results == ['true', 'false', 'true', 'false', 'false', 'true']

print(storage)
assert len(storage) == 3

{% endhighlight %}

As an addendum, each of the command classes can also have a reference to a `Receiver` object which allows the delegation of more complex commands to an external object. This is optional but is mentioned in the [referenced source], which shows one of the command classes with a `_receiver` attribute.

To summarize, we refactored the previous application of the `Strategy` pattern to use the `Command` pattern. The `Command` pattern allows us to specify each command as an object on its own, which is invoked by passing an instance of the command object into an `Invoker` object that acts as an interface between the client and the commands.

Hope it helps. H4PPY H4CK1NG