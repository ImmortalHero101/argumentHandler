# Argument Handler
An argument handling module for spuiiBot.

##### Table of Contents  
[Motive](#motive)  
 - [Introduction to Argument Handler](#introduction-to-argument-handler)
 - [Data Structures](#data-structures)
   - [Command Argument Structure Object](#command-argument-structure-object-read-only)
     - [Main Object](#main-object)
     - [Paths Object](#paths-object)
     - [Components Object](#components-object)
       - [Types Available](#types-available)
       - [Meaningful Associates](#meaningful-associates)
   - [Progress Data Object (Read and Write)](#progress-data-object-read-and-write)
   - [Validator Status Class Object (Read and Write)](#validator-status-class-object-read-and-write)
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
The main theme of the bot is a good User Interface, which is to be able to perform complex tasks with ease of use. However, with time, spuiiBot grew as new commands and functionalities were being added and made the overall code relatively large and complex, and adding new code often involved repeating old functionalities to make the new code exeuctable. Some of these old functionalities are chains of permission and argument checks, which also needed to generate a command-specific Error Message to rightly guide the user in using that command. In most cases, these checks were often larger than the actual functionality code of the command, making a huge clutter and reducing productivity in developing these commands, or if a command with similar amount of functionality code were to be created. Therefore, I decided to make a module that would handle command-exclusive argument and permission checking and with additional utility features that would be hard to implement otherwise. This would be yet another complex feature implemented to the bot to maintain  the developing productivity and improve user experience as a result of reduced errors and unexpected behaviour caused by invalid input data.

A Permission Handler was easy to make as I just had to define the permissions in a command's code required to run that command and then intercept this data when the command is being run to resolve and validate permissions of the instances (member/bot) defined in the data. My focus now shifts to Argument Handler, which is a more complex module, and will be my main focus in this article.

## Introduction to Argument Handler
First of all, we have to determine what kind of messages will trigger command message responses by spuiiBot, so we'll be going with the default prefix-command-arguments format (`$command arguments`, `$` is the default prefix), where `command` is the command name preceeded by a prefix and followed by `arguments`, which is the command's arguments. The bot will check if the [`content` property](https://discord.js.org/#/docs/main/stable/class/Message?scrollTo=content) of each [Message object](https://discord.js.org/#/docs/main/stable/class/Message) (received from Discord API) starts with the prefix `$`, and if it does, then it will extract the command name from the message and check if it has a command with that name. If we do, then we extract the rest of the user message (which will be considered as the command's arguments, as indicated by the command format) and pass it to the **Argument Handler** along with the selected command's [Command Argument Structure Object](#command-argument-structure-object). The Argument handler will process the arguments to give us the desired format of the raw arguments in program-interceptable data. Once we receive the processed arguments, we pass them to the command for it to execute. This process is done natively in the `message` event callback.

Moving on to the Argument Handler, the main function of it is to parse command arguments to a program-interceptable format and perform validation checks to verify the data, along with performing some additional features to enhance the functionality.


## Data Structures
Before we jump to the algorithm, we will look at data structures that are developed for this module. Having a fixed data structure is necessary as it provides expected behaviour by giving fixed meaning to each key of the data structure, maintaining code readibility and easy to debug.

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

### Validator Status Class Object (Read and Write)
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
- `validatedComponents`: An array containing [Component State Objects](#component-state-object-read-only).
- `validateComponent`: A prototypial (inherited) method that processes the next argument segment against its relevant component, both of which are provided to the method as arguments.

### Component State Object (Read Only)
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
The first step is to parse the arguments. The **Argument Parser** will split the argument string into separate parts, each representing a single argument *to* one of the command's parameter (in order of left to right). Some features include:
- Splitting the argument string into argument segments for each space character encountered.
- Considering quoted text as argument to a single command parameter.
- Capture comma-separated arguments as lists, which is a group of data for a single parameter. (Planned)
- Auto-adjust arguments if its quoting/listing isn't clear. (Provisional)

## Pre-validation code flow
*This section is incomplete*

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

## Post-validation code flow
*This section is incomplete*

## Path Selection
*This section is incomplete*
