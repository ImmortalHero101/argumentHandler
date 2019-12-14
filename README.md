# Argument Handler
An argument handling module for spuiiBot.

# Motive
The main theme of the bot is a good User Interface, which is to be able to perform complex tasks with ease of use. However, with time, spuiiBot grew as new commands and functionalities were being added and made the overall code relatively large and complex, and adding new code often involved repeating old functionalities to make the new code exeuctable. Some of these old functionalities are chains of permission and argument checks, which also needed to generate a command-specific Error Message to rightly guide the user in using that command. In most cases, these checks were often larger than the actual functionality code of the command, making a huge clutter and reducing productivity if a command with similar amount of functionality code were to be created. Therefore, I decided to make a module that would handle command-exclusive argument and permission checking separately and with additional utility features that would be hard to implement otherwise.

A Permission Handler was easy to make as I just had to define the permissions in a command's code required to run that command and then intercept this data when the command is being run to resolve and validate permissions of the instances (member/bot) defined in the data. My focus now shifts to Argument Handler, which is a more complex module, and will be my main focus in this article.

## Introduction to Argument Handler
First of all, we have to determine what kind of messages will be accepted as command messages by spuiiBot. We'll be going with the general prefixed-command format (`$command arguments`), where the command name is preceeded by a prefix and followed by the command's arguments. Now to put this to action, we'll be checking if each user message starts with the prefix `$`, and if it does, then extract the command name from the message and check if we have a command with that name. If we do, then we extract the rest of the user message (which will be considered as the command's arguments, as indicated by the command format) and pass it to the Argument Handler along with the [Command Argument Structure Object](#command-argument-structure-object), and once we get the processed arguments, we pass them to the command. This process is done natively in the `message` event callback and separately from the Argument Handler as the point of the Argument Handler is to work with the command's arguments.

Moving on to the Argument Handler, the main function of it is to parse command arguments to a program-interceptable format. Mine does exactly this, but with some additional features to enhance the functionality, some of which are:
- Argument Validator to allow variety of type checks and criteria checks for those types.
- Conversion of user input into program-interceptable data. This includes resolving object of instances such as members, channels, etc.
- Internal data generation to complement other parts of the bots, such as providing any mentioned users to Permission Handler to perform any necessary checks on those users.
- Error generation with a reliable descripton and the command's usage instructions.

Additionally, the User Input Parser (called Argument Parser) has some additional features, such as:
- Considering quoted text as input to a single argument.
- Capture comma-separated arguments as lists, which is a group of data for a single argument. (Provisional)
- Capture text even if quotes aren't complete. (Provisional)
- Auto-adjust arguments if argument quoting/listing isn't clear. (Provisional)

These features will be discussed in detail as we discuss the algorithm of this module.


## Data Structures
### Command Argument Structure Object
### Argument Parse State
### Validator Status Class Object
### Validation Error Object
### Function Output Object


## Parsing


## Pre-validation code flow


## Validation
### Validation Checks
#### Type Checks
#### Criteria Checks
#### Formatted Output generation

### Error Message generation


## Post-validation code flow


## Path Selection

