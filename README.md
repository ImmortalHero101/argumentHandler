# Argument Handler
An argument handling module for spuiiBot.

##### Table of Contents  
[Motive](#motive)  
 - [Introduction to Argument Handler](#introduction-to-argument-handling)
   - [General Command Usage](#general-command-usage)
     - [Prefix](#prefix)
     - [Command](#command)
     - [Arguments](#arguments)
   - [Implementation](#intro-implementation)
 - [Data Structures](#data-structures)
   - [Command Argument Structure Object](#command-argument-structure-object-read-only)
     - [Main Object](#main-object)
     - [Paths Object](#paths-object)
     - [Components Object](#components-object)
       - [Types Available](#types-available)
       - [Meaningful Associates](#meaningful-associates)
   - [Progress Data Object (Read and Write)](#progress-data-object-read-and-write)
   - [Validator Status Class Object (Read and Write)](#validator-state-class-object-read-and-write)
   - [Component State Object (Read Only)](#component-state-object-read-only)
   - [Validation Error Object (Read and Write)](#validation-error-object-read-and-write)
   - [Validation Output Object (Write Only)](#validation-output-object-write-only)
 - [Parsing](#parsing)
 - [Pre-validation code flow](#pre-validation-code-flow)
 - [Validation](#validation)
   - [Validation Checks](#validation-checks)
     - [Type Checks](#type-checks)
     - [Criteria Checks](#criteria-checks)
     - [Formatted Output generation](#formatted-output-generation)
   - [Error Message generation](#error-message-generation)
 - [Post-validation code flow](#post-validation-code-flow)
 - [Path Selection](#path-selection)

---

# Motive
The main theme of the bot is a good User Experience (UX), which is to be able to perform complex tasks with ease of use. However, with time, spuiiBot grew as new commands and functionalities were being added and made the overall code relatively large and complex, and adding new code often involved repeating old functionalities to make the new code exeuctable. Some of these old functionalities are chains of permission and argument checks, which also needed to generate a command-specific Error Message to rightly guide the user in using that command. In most cases, these checks were often larger than the actual functionality code of the command, making a huge clutter and reducing productivity in developing these commands, or if a command with similar amount of functionality code were to be created. Therefore, I decided to make a module that would handle command-exclusive argument and permission checking and with additional utility features that would be hard to implement otherwise. This would be yet another complex feature implemented to the bot to maintain the developing productivity and improve user experience as a result of reduced errors and unexpected behaviour caused by invalid input data.

A Permission Handler was easy to make as I just had to define the permissions in a command's code required to run that command and then intercept this data when the command is being run to resolve and validate permissions of the instances (member/bot) defined in the data. My focus now shifts to **Argument Handler**, which is a more complex module, and will be my main focus in this article.

## Introduction to Argument Handling
First of all, we have to determine a format for messages that resemble a *Command Message*, which is a message sent with an intent to execute a command. The user will have to comply with this format each time they want to run a command, with some execptions where the bot will rightly guide the user as necessary.

### General Command Usage
We'll be using the general prefixed-command-arguments format which looks something like this: `$command arguments?`. If you've developed a bot then you should be familiar with this (but should not skip this section). We'll be breaking down this format into three parts: Prefix, Command and Arguments.

#### Prefix
```
$command arguments?
^
```
A prefix is a word, letter or number added before another. In the Command Usage format, a prefix is a series of characters at the beginning of a message that serves the purpose of an indication of a Command Message to the bot. By default, the prefix is `$` and this can be accessed and changed through the `$prefix` command. It can be changed to any series of non-whitespace characters that contains at least one symbol character and has at least 1 character and at most 3 characters. If the user provides an incorrect prefix, the bot will simply ignore the message as it failed to meet the Command Message format criteria.

#### Command
```
$command arguments?
 ^^^^^^^
```
A command is a case-insensitive word or character(s) immediately following the prefix which is the name of the command that the user is intending to use. The length of a command name depends on the commands that are registered on the bot. In case an invalid command name is provided, the bot will again simply ignore the message as the user could have used the characters of the prefix in their message without an intent to run a command.

#### Arguments
```
$command arguments?
         ^^^^^^^^^^
```
The arguments are the user's input to a command according to that command's *parameters*, which is the data a command accepts in order to function in a certain way. There are cases such as when arguments are not accepted by a command if no parameters are defined, both providing and not providing arguments is accepted when parameters are defined but the command still works without arguments, and where it is mandatory to provide arguments when parameters are defined and the command does not work without data provided.

This data the user provides as arguments to each command has to be in a certain format too. When providing arguments to parameters, the user intends to provide a *certain value* to a *specific parameter*, usually in series if the command has multiple parameters (thus accepts multiple arguments). This series of arguments has to be in an order, which is defined in the command and can be seen through the help menu of that command. There also has to be a *delimiter* in order for the bot to be able to distinguish between arguments in order to specify for which parameters the *argument segments* are provided to. This will be further discussed in [Argument Parsing](#parsing).

The Argument Handler will be handling *this* section of Command Messages as it is where we can invest some complexity to introduce a range of new functionalities to the bot.

### <span id="intro-implementation">Implementation</span>
Now that you have understood the General Command Usage format, we can jump to discussing the implementation of handling Command Messages. First, the bot will check if the user's message's content (accessed through the [`content` property](https://discord.js.org/#/docs/main/stable/class/Message?scrollTo=content) of each [Message Object](https://discord.js.org/#/docs/main/stable/class/Message), received from the Discord API) begins with a prefix, which indicates that the user might be intending to use a command. If it does, the bot will extract the command name from the message and check if a command goes by that extracted command name. If the command is found, then it will extract the rest of the message's content (which will be considered as the command's arguments in series) and pass it to the **Argument Handler** along with the selected command's [Command Argument Structure Object](#command-argument-structure-object) for validation. The Argument Handler will process the arguments to give the desired format of the arguments in program-interceptable data so that they can be passed to the selected command for it to execute, or in case of any errors, give the error data so that the bot can send them back to the channel where the user sent the message in so that the user can see this error and respond to it. This process is done natively in the `message` event callback and is ***not*** a part of the Argument Handler, but complements it by extracting and providing it the necessary data for it to function.

Before moving onto the algorithm and implementation of Argument Handler, you will need to go through the [first data structure](#command-argument-structure-object-read-only) under the [Data Structures section](#data-structures) below. After that, you can choose to proceed to the [next section](#parsing) as the rest of the data structures have their context given throughout this article, and you can visit that data structure's section when mentioned in the article.


## Data Structures
This section contains data structures that are developed for this module. Having a fixed data structure is highly recommended as it provides expected behaviour by giving fixed meaning (and the data type in most cases) to each property of the data structure and the only modification applied to it is scaling of the value of each meaning. This helps maintaining code readibility and makes it easy to debug.

### Command Argument Structure Object (Read-Only)
Starting off, we'll discuss the **Command Argument Structrue Object** (`argStructure` for short) which holds the overall argument structure of a command, which will be used to evaulate the arguments of its commands. 

> You can jump to the [next section](#parsing) right after reading about this data structure as you are only required to know about *this* data structure before continuing this article. The rest of the data structures will be mentioned when you need to know about them to continue reading.

```js
{
  title: String,
  description: String,
  blank: String,
  paths: [{
      title: String,
      description: String,
      components: [{
        name: {
          display: String,
          reference: String,
          descriptor: String
        },
        type: String,
        associate: String,
        action: String,
        values: [String],
        isOptional: Boolean,
        ignoreArg: String,
        criteria: Object,
        fromPrev: [{
          type: String,
          descriptor: String,
          values: [String],
          criteria: Object
        }]
      }]
  }]
}
```
To make it simpler, this data structure can be broken down into three parts: [Main Object](#main-object), [Paths Object](#paths-object) and [Components Object](#components-object), each playing a distinctive role in giving meaning to this data structure.

#### Main Object
The **Main Object** contains details about the command relevant to the `$help` command, and holds the command's [Paths Object](#paths-object) in the `paths` property, which allows us to have different combinations of the command's usage.
```js
{
  title: String,
  description: String,
  blank: String,
  paths: Array
}
```
- `title`: Name of the command to be displayed in the Help menu.
- ``description``: Descripton of the command to be displayed in the Help menu.
- ``blank``: Description of what the command does if no arguments are provided. (Leaving this blank will make it mandatory to include arguments)
- **Paths:** Array containing different combinations of command usage by containing structures for those combinations. Contains [Paths Object](#paths-object) data structure.

#### Paths Object
The **Paths Object** is the data structure contained inside an array of the property `paths` of the [Main Object](#main-object), and represents the argument structure of a single path (or combination). It contains data relevant to both the `$help` command and the **Argument Handler**.

To avoid unexpected behaviour, there are some requisites to constructing this part of the **Command Argument Structure Object**. They are:
 - Each combination should contain a [Components Object](#components-object) marked mandatory, different from the other combinations of the same paths array, to avoid any unexpected behaviour.
 
```js
{
  title: String,
  description: String,
  components: Array
}
```
- `title`: Title of the path to be displayed in the Help menu.
- `description`: Description of the path to be displayed in the Help menu.
- `components`:  Array containing the argument structures of the path, each position in Array representing the argument number it represents and will be evaluated against.

#### Components Object
Finally, the **Components Object** (`pathComponent` as reference name, component as alternative name) that contains (or defines) the structure of the argument that would be in the position same as of this data structure in its array.

To avoid unexpected behaviour, there are some requisites to constructing this part of the **Command Argument Structure Object**. They are:
- There should be at least one mandatory component per path.
- The first component object should not contain the `fromPrev` property, as there is no property behind it.
- There should be at least one specified associate component property before that associate is used.

```js
{
  name: {
    display: String,
    reference: String,
    descriptor: String
  },
  type: String,
  associate: String,
  action: String,
  values: [String],
  isOptional: Boolean,
  ignoreArg: Object,
  criteria: Object,
  fromPrev: [{
    type: String,
    descriptor: String,
    values: [String],
    criteria: Object
  }]
      }
```
- `name`: Object containing name-related data.
  - `display`: Name of the parameter to be displayed in the help menu and error messages.
  - `reference`: String value that will be used as a key to the argument's processed data value in the output object.
  - `descriptor`?: Alternative name of the parameter to be shown in the error message for formatting purposes.
- `type`: The data type the argument should be in, and the data type of the processed value. This is a form of argument validation. For more details regarding this field's accepted values, refer to [Types avaliable](#types-available).
- `associate`?: Name by which this parameter's processed argument will be stored as, for internal dependency. For more details regarding this field's meaning for certain values, refer to [Meaningful Associates](#meaningful-associates).
- `action`?: Action to be shown in error messages for the current parameter, for formatting purposes.
- `values`?: The only values to accept. This is also used for help menu.
- `isOptional`?: (Not implemented) Determines whether the current parameter is optional and jumps to the next argStructure with the current argument in case there is an error in the current argStructure. Keeps on jumping to the next argStructure in case of error, till a mandatory argStructure is encountered.
- `ignoreArg`?: (Provisional, currently stuck on and might be dumped) Object containing `associate:value` pairs, which will check respective associates with the value provided. If there is a match, then this argStructure will not be evaulated and validator will jump to the next path.
- `criteria`?: Criteria options as `criteria:value` pairs based on the selected `type` option. The argument will be evaulated against these criteria options, and an error will be thrown on a mismatch. Refer to [Types avaliable](#types-available) for each `type`'s accepted criteria.
- `fromPrev`?: Sets the current component's properties according to the previous component's argument selected from its `values` array property. Properties accepted are: `type`, `descriptor`, `values` and `criteria`.


##### Types available
**Base Types**
- `String`: Accepts any data type, which will be processed to a String.

| Criteria Options  | Description |
| ----------------- | ----------- |
| `min`  | (Integer) Minimum length of the text to accept. Defaults to `1` and cannot be smaller than `1` and larger than `1990` or `max` property. |
| `max` | (Integer) Maximum length of the text to accept. Defaults to `2000`, and cannot be larger than `2000` and smaller than `1` or the `min` property. |
| `long`  | (Boolean, non-functional) To accept arguments containing more than one word (these are generally quoted arguments). Defaults to `false`. |
| `format`  | (RegExp, non-functional) To check the argument against a Regular Expression. Defaults to `null` |

- `Number`: Accepts a number argument, which will be processed to Number value.

| Criteria Options | Description |
| ---------------- | ----------- |
| `min` | (Number) Lower bound of the number to accept. Defaults to `-Infinity`, and cannot be `+Infinity` or larger than `max` property. |
|`max` | (Number) Upper bound of the number to accept. Defaults to `Infinity`, and cannot be `-Infinity` or smaller than `min` property. |
| `intOnly` | (Boolean) To accept a number as an integer. Defaults to `false`.
- `Boolean`: Accepts either `"true"` or `"false"`.
 

**Composite Types** (Not implemented yet, depend on base types or each other. All types listed are provisional and may be changed or removed)
- `Guild`: Accepts Guild ID, which will be processed to Guild Object.
  - Derived from String with defined `format` criteria.
  
| Criteria Options | Description |
| ---------------- | ----------- |
| `multiple` | (Boolean) To allow user to have multiple instances included in this argument. Defaults to `false` |
| `reject` | (Array[GuildID]) Array containing ID's of guilds to reject |
| `rejectError` | (String) Error message to show if a rejected instance is encountered. Doesn't show an error message by default. |

- `User`: Accepts User ID, Mention or Username/Nickname, which will be processed to User Object.
  - Derived from String with defined `format` criteria.
  
| Criteria Options | Description |
| ---------------- | ----------- |
| `multiple` | (Boolean) To allow user to have multiple instances included in this argument. Defaults to `false` |
| `reject` | (Array[UserID]) Array containing ID's of users to reject |
| `rejectError` | (String) Error message to show if a rejected instance is encountered. Doesn't show an error message by default. |

- `Member`: Accepts Member ID, Mention or Username/Nickname, which will be processed to Member Object of the current guild.
  - Derived from User.
  
| Criteria Options | Description |
| ---------------- | ----------- |
| `multiple` | (Boolean) To allow user to have multiple instances included in this argument. Defaults to `false` |
| `reject` | (Array[MemberID]) Array containing ID's of members to reject |
| `rejectError` | (String) Error message to show if a rejected instance is encountered. Doesn't show an error message by default. |
- `Link`: Accepts a link.
  - Derived from String with defined `format` criteria.

##### Meaningful Associates
- `action`: The action that is being performed through using the command. Mainly used for constructing error message `description` property.
- `field`: The field an action is being performed on. Mainly used for constructing error message `description` property.

---

### Argument Parse State (Read and Write)
This data structure determines the parse state of each parsed argument segment, returned as a collection in an array (called **Parse State Array**) by the Argument Parser. It is used to determine a potential flaw in argument syntax and used by the Control Flow to readjust arguments according to argument segment structure types.

```js
{
  state: Integer,
  value: String | Array
}
```
- `state`: Represents the state of the argument segment.
  - `0`: `UNPARSED`; Is set to the state object on its creation, and most likely would not be seen outside of the Argument Parser.
  - `1`: `PARSED`; Successful parse
  - `2`: `INCOMPLETE_QUOTES`; The value captured did not have a closing quote and is the substring of the rest of the argument. This should be the last Parse State in the Parse State array. (Unimplemented)
  - `3`: `INCOMPLETE_LIST`; The list captured did not have a delimiter (quotes, semi-colon at the end, brackets) and the parser made one according to the best match. (Unimplemented)
- `value`: The value(s) captured

### Progress Data Object (Read and Write)
This data structure contains data relevant to the progression of a path, generated by Flow Control, and will be used in [Path Selection](#path-selection) for selecting a single path relevant to its progression data.

```js
{
  componentIndex: Integer,
  progress: Float,
  unparsedArg: Integer,
  unparsedComp: Integer
}
```
- `componentIndex`: The index of last *selected* component by Flow control.
- `progress`: Percentage (in decimal) of total components processed relative to total components present.
- `unparsedArg`: Number of argument segments unprocessed by the end of Flow Control loop of the current path.
- `unparsedComp`: Number of components unprocessed by the end of Flow Control loop of the current path.

### Validator State Class Object (Read and Write)
This data structure is generated through a class (called **ValidatorStatus**) and is modified internally in the class. It contains data related to the current path, including path's progress data, associates and data related to validation of its components. It also contains a inherited prototypial method (from its class) that validates a component.

```js
{
  progressData: Object,
  associates: Object,
  state: Integer,
  validatedComponents: Array,
  validateComponent: [Prototypial Method](value: String, pathComponent: Object)
}
```
- `progressData`: Contains [Progress Data Object](#progress-data-object-read-and-write)
- `associates`: Contains associates in `associate:value` pairs where `associate` is the name of an associate.
- `state`: Overall state of the parsing of the path's components. Usually the last element of `validatedComponents` contains the errored component if the state is errored.
  - `0`: `UNVALIDATED`; Is set on object creation, and most likely would not be seen outside of the class.
  - `1`: `VALIDATED`; All components parsed successfully.
  - `2`: `ERROR`; Validation unsuccessful due to validation error in (one of) the component(s).
- `validatedComponents`: An array containing [Component State Objects](#component-state-class-object-read-only).
- `validateComponent`: A prototypial (inherited) method that processes the next argument segment against its relevant component, both of which are provided to the method as arguments.

### Component State Class Object (Read Only)
This data structure is generated through a class (called **TypeClass**) and is modified internally in the class. It contains data related to the state of the current component. Its class also contains a static method to generate the `usage` property value according to the value of the `type` property of the current component.

```js
{
  value: Any,
  error: String
}
```
- `value`: Contains the unprocessed value of argument segment on initialization, and is most likely changed to program-interceptable variant of the modified value later on in the code.
- `error`: Contains [Validation Error Object](#validation-error-object-read-and-write)

### Validation Error Object (Read and Write)
This data structure contains a user-interceptable error encountered during validation of the argument segment for rightly guiding the user.

```js
{
  code: Integer,
  usage: String,
  description: String
}
```
- `code`: Code of the error, which is used as index to the Argument Errors array containing the first part of the error. These errors are:
  - `0`: `INCOMPLETE_QUOTES`; "You may have forgotten to close the quotes"
  - `1`: `INVALID`; "You provided an invalid"
  - `2`: `UNACCEPTED`; "You provided an unaccepted"
  - '3': `BLANK`; "You did not provide the"
  - '4': `OUT_OF_RANGE`: "The provided value is out of range for the"
- `usage`: Message telling what kind of value to provide.
- `description`: Message telling about the error, such as guiding where the error occured or where the user went wrong.

### Validation Output Object (Write Only)
*This data structure is incomplete as it wasn't planned properly.*

This data structure represents the outcome of the validation of arguments, and is generated after [Path Selection](#path-selection).

```js
{
  state: Integer,
  status: String,
  values: Object,
  error: {
    description: String,
    usage: String,
    fault: {
      detectedPath: String
    }
  }
}
```
- `state`: State of the outcome, relative to the selected path.
- `status`: Error name relative  to the `state` property's value.
- `values`: Object containing `name.reference:value` pairs where `name.reference` is the reference name set as the property name of the registered processed argument as `value`.
- `error`: Object containing error details (if there is an error present on the selected path). Contains [Validation Error Object](#validation-error-object-read-and-write) combined with addtional properties (unplanned).


## Parsing
*This section is incomplete*
The first step is to parse the arguments. This part involves splitting the argument string into separate parts, each representing a single argument *to* one of the command's parameter (in order of left to right). We'll be looking out for 2 details when determining whether a part of this argument string is a single argument: Single words and Quoted text.

Picking out single words is relatively easy as each word is a series of character, in series, that is *not* a space character (other whitespace characters are considered as a part of the word). Therefore, we can use a space character as the split key to break down the argument into its segments, and each segment would almost represent a single argument. As the space character is used as a split key and the argument can have multiple space characters in series, the resulting Array would contain blank elements due to this, which can be worked around with by filtering the array to remove any 'falsey' values (by passing the Boolean constructor as the callback to the filter method).

Next thing we need to look at are Quoted text as single arguments. A piece of text is quoted if it is wrapped around quotes (we'll be considering both single and double quotes). The technique we chose is to work with the argument pieces array to find elements that start with a quote, group together the following elements till an element ending with the same quote is encountered (we'll be considering the different quote as a part of the text). Accordingly, a loop is set up to iterate through the argument segments array to find and group quoted text, as well as make a [status object](#argument-parse-state-read-and-write) for both single arguments and quoted arguments to form a resulting array. In case the loop was unable to find an element ending with the same quote, it will throw an error (this will be changed to return the processed result along with the status code). Since the loop  was unable to find the ending quote, it would have captured the rest of the argument, meaning that if there were distinct arguments after the quoted argument, then it got included in the quoted argument too. This gives the rise to the idea of readjusting the arguments from the parameter accepting quoted text as a single argument onwards, which will be done by validating whether argument pieces from the end match their types as stated in their assigned component object.

The resulting array is then returned.

Ohter planned features include:
- Capture comma-separated arguments as lists, which is a group of data for a single parameter. (Planned)
- Auto-adjust arguments if its quoting/listing isn't clear. (Provisional)

## Pre-validation Code Flow
The next step would be to validate these freshly processed arguments, which would be done by linking each argument segment to a component and then validating it according to the data in that component. Starting off, we have to figure out how to get through the [Command Argument Structure Object](#command-argument-structure-object-read-only) (aka. `argStructure`) to each component object so that we can link it with an argument segment. We know that there is a possibility of multiple paths with each containing a different component data structure combination, thus we need to provide the *same* argument segments to each of the components of each of the paths. We also don't know which path fits the arguments provided and we'll be determining this through evaulation in [Path Selection](#path-selection) after collecting relevant data.

### Control Flow Loops
An approach to this situation would be to iterate through all the paths using a loop (called `pathLoop`), exposing each path's properties in the iteration and ultimately allowing us to access the componens of the path, and since the `paths` property of `argStructure` is an Array, we can use its length as the number of iterations to do and use the iteration number to access the path object from the `paths` array. It doesn't stop there, we still have to figure out how to get the components object from each path object and link argument segments to them. We can use the same technique as we did with the paths (but we'll be calling the loop `componentLoop`) as the `components` property of a path object contains an array of components, but this will be done *inside* of `pathLoop` as it is where the path object is exposed, and ultimately the `components` array. We again use the length of this array as the number of iterations and use the current iteration number to access the component object. Now that we have access to each of the components of each of the paths inside `componentLoop` loop, we can start forming links with argument segments and then continue to validation.

To form links beween arguments and components, we need to find a common pattern between them. Since the arguments are being provided in the order of the usage of the command, and where each segment is associated to a command parameter, the pattern here is that each command is provided in order to the usage to the command, therefore each segment has to be associated to a component in order. We can make use of iteration number of `componentLoop` to form links with argument segments by retrieving the segment from the same iteration number. This way we are exposed to components of paths orderly and retrieve argument segments orderly too. This logic helps us iterate through the same argument segments for each of the paths as we are relying on the iteration number of `componentLoop`, which resets to 0 after each iteration of `pathLoop`, therefore allowing us to retrieve the argument segments from the start for each path.

### Variable Initialization and Pre-defined Classes
We then proceed to doing the validation check on the current argument segment, but we also need to save the data as when the validation is successful, we need to store the returned processed value so that it is later accessible by the command, or otherwise store the validation errors. Therefore, we'll be creating a [state object](#component-state-class-object-read-only) (Component State Object) for each validation, and these state objects will be ordered to represent each component orderly. Since there are multiple paths, we'll need to store these ordered states according to their paths, which in turn will be stored in an orderly manner.

Remember that we don't know which path the user intended to use, so we have to figure it out by ourselves. We will need to see which path the argument seems to be matching the best. Knowing this, we need to collect data produced in each path for comparison that will take place later on in the code (in [Path Selection](#path-selection)). We also only need to collect data for each path till we've encountered an error as this indicates that any following evaulations will be erroneous too, so we'll be modifying the `componentLoop` to stop when an error is encountered (This is covered in [Loop Escape Logic](#loop-escape-logic)). Now we're going to attempt to collect relevant data on the basis of the path's progression. We came to know that there is a possibility that some components might be left unattended, which means that the `componentLoop` wouldn't always iterate according to the number of components a path has. So we'll be storing the iteration number of `componentLoop`, the number of paths left unattended (`unattendedComp`) and percentage completion (in decimal) of the loop. Then we also have a possibility that the user provided more or less number of argument segments than they should have so we'll be storing the number of those segments left unattended too. This data accounts for the progress of a path, and will be stored in a different data structure called [Progress Data](#progress-data-object-read-and-write).

We'll also be creating an associates object for each path to contain data that needs to be stored for later components of that path. The purpose and the use of this data structure will be discussed later in this article.

In order to store this data, we'll be creating another [state object](#validator-state-class-object-read-and-write) which will be created using a class. Each path will store this data as well as the path's Component State Objects, so that it contains all the data related to that path.

Moving on, the variables being created are:
- `parsedArguments` which contains processed argument segments in order. This is going to be fixed throughout the evaulation.
- `pathStatuses` which contains the state objects of paths in order of each path's index in the `paths` property array of [Command Argument Structure Object](command-argument-structure-object-read-only), which will be done using the iteration number of `pathLoop` as it iterates from the first index to the last index.
- `pathIndex` to hold the iteration number of `pathLoop`. Initialized with the loop.
- `progressData` which will be created inside of `pathLoop` so that it is initialized on each iteration and also so that the path is exposed throughout that iteration and each iteration in `componentLoop` so that we can constantly update it.
- `associates`, same as `progressData`, but holds different data.
- `componentIndex` to hold the iteration number of `componentLoop`. Initialized with the loop.

Next comes the validation part, which will be done inside the Path State and Component State classes, and will be covered in the next section.

## Validation
*This section is incomplete*
Argument Validator to allow variety of type checks and criteria checks for those types.
Conversion of user input into program-interceptable data. This includes resolving object of instances such as members, channels, etc.
Internal data generation to complement other parts of the bots, such as providing any mentioned users to Permission Handler to perform any necessary checks on those users.
Error generation with a reliable descripton and the command's usage instructions.

### Validation Checks
*This section is incomplete*
#### Type Checks
*This section is incomplete*
#### Criteria Checks
*This section is incomplete*
#### Formatted Output generation
*This section is incomplete*

### Error Message generation
*This section is incomplete*

## Post-validation Code Flow
*This section is incomplete*

### Loop Escape Logic

## Path Selection
*This section is incomplete*
