# Argument Handler
An argument handling module for spuiiBot.

# Motive
The main theme of the bot is a good User Interface, which is to be able to perform complex tasks with ease of use. However, with time, spuiiBot grew as new commands and functionalities were being added and made the overall code relatively large and complex, and adding new code often involved repeating old functionalities to make the new code exeuctable. Some of these old functionalities are chains of permission and argument checks, which also needed to generate a command-specific Error Message to rightly guide the user in using that command. In most cases, these checks were often larger than the actual functionality code of the command, making a huge clutter and reducing productivity in developing these commands, or if a command with similar amount of functionality code were to be created. Therefore, I decided to make a module that would handle command-exclusive argument and permission checking and with additional utility features that would be hard to implement otherwise. This would be yet another complex feature implemented to the bot to maintain  the developing productivity and improve user experience as a result of reduced errors and unexpected behaviour caused by invalid input data.

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
Before we discuss the algorithm, we will look at data structures that the algorithm is going to work with in this module. Having a fixed data structure is necessary as it holds data that is necessary for the progression of the code and so that there aren't any unexpected results.
### Command Argument Structure Object (Read-Only)
Starting off, we'll discuss the **Command Argument Structrue Object** (`argStructure` for short) which holds the argument structure of a command, against which arguments of its command will be evaulated.
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
We will be breaking down this data structure into three parts: [Main Object](#main-object), [Paths Object](#paths-object) and [Components Object](#components-object).
#### Main Object
The Main Object contains details about the command relevant to the `$help` command, and holds the argument structure in the `paths` property.
```js
{
  title: String,
  description: String,
  blank: String,
  paths: []
}
```
- `title`: Name of the command to be displayed in the Help menu.
- ``description``: Descripton of the command to be displayed in the Help menu.
- ``blank``: Description of what the command does if no arguments are provided. (Leaving this blank will make it mandatory to include arguments)
- **Paths:** Array containing different combinations of command usage by containing structures for those combinations. Contains [Paths Object](#paths-object) data structure.

#### Paths Object
The Paths Object is the data structure contained inside an array of the [Main Object](#main-object)'s property `paths`. This data structure allows us to have different usage combinations of the command, each combination containing a mandatory argument structure different from the other combinations of the same paths array, to avoid any unexpected behaviour.
```js
{
  title: String,
  description: String,
  components: []
}
```
- `title`: Title of the path to be displayed in the Help menu.
- `description`: Description of the path to be displayed in the Help menu.
- `components`:  Array containing the argument structures of the path, each position in Array representing the argument number it represents and will be evaluated against.

#### Components Object
Finally, the Components Object (`pathComponent` as reference name) that contains (or defines) the structure of the argument that would be in the position same as of this data structure in its array. There should be at least one `pathComponent` per path to avoid any unexpected behaviour. 
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
  - `descriptor`: Alternative name of the parameter to be shown in the error message for formatting purposes.
- `type`: The data type the argument should be in, and the data type of the processed value. This is a form of argument validation. For more details regarding this field's accepted values, refer to [Types avaliable](#types-available).
- `associate`: Name by which this parameter's processed argument will be stored as, for internal dependency. For more details regarding this field's meaning for certain values, refer to [Meaningful Associates](#meaningful-associates).
- `action`: Action to be shown in error messages for the current parameter, for formatting purposes.
- `values`: The only values to accept. This is also used for help menu.
- `isOptional`: (Not implemented) Determines whether the current parameter is optional and jumps to the next argStructure with the current argument in case there is an error in the current argStructure. Keeps on jumping to the next argStructure in case of error, till a mandatory argStructure is encountered.
- `ignoreArg`: (Provisional, currently stuck on and might be dumped) Object containing `associate:value` pairs, which will check respective associates with the value provided. If there is a match, then this argStructure will not be evaulated and validator will jump to the next path.
- `criteria`: Criteria options as `criteria:value` pairs based on the selected `type`. The argument will be evaulated against these criteria options, and an error will be thrown on a mismatch. Refer to [Types avaliable](#types-available) for each `type`'s accepted criteria.
- `fromPrev`: *Incomplete description*


##### Types available
**Base Types:**
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
*Incomplete section*



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

