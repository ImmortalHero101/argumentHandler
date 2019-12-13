# Argument Handler
An argument handling module for spuiiBot.

# Motive
The main theme of the bot is a good User Interface, which is to perform complex and unique tasks with ease of use. However, with time, spuiiBot grew as new commands and functionalities were being added and made the overall code relatively large and complex, and adding new code often involved repeating old functionalities to make the code exeuctable. Some of these functionalities involved chains of permission checks and argument checking, which also needed to generate a command specific error message to rightly guide the user. In most cases, this code was often larger than the actual functionality code of the command, making a huge clutter and reducing productivity if a command with similar amount of functionality code were to be created. Therefore, I decided to make a module that would handle command-exclusive argument and permission checking separately and with additional features that would be hard to implement otherwise.

A Permission Handler was easy to make as I just had to define the permissions in a command's code required to run that command and then intercept this data when the command is being run to resolve and validate permissions of the instances (member/bot) defined in the data. My focus now shifts to Argument Handler, which is a more complex module, and will be my main focus in this article.

## Introduction to Argument Handler
The main function of an argument handler is to parse user input to a interceptable format for the program. Mine does exactly this, but with some additional features to enhance the functionality, some of which are:
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
### Command Argument Object
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

