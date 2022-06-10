# Table Toys

A collection of utilities centered around table top gaming used as a focus for exploring and learning rather than efficiency or completeness.

## Dice Roller

### Personas

#### Richa

Richa, a senior boot camp student, is writing a browser-based board game for her final project. Part of her assignment is integrating 3rd-party services so she's decided to incorporate a cloud-based dice roller.

She's looking for a service that is:

- high-availability given that rolling dice is a core part of her game
- stable, without requiring her to update her code if new versions are released
- self-documenting, so she can focus on her code
- a forgiving parser that doesn't require extensive pre-processing
- JSON-based response
- well-defined limits, with a generous free tier
- doesn't require sign-up

#### James

James, a table top player, is looking for a dice roller to use with his online RPG group that allows sharing player rolls. The mechanism should be unobtrusive so players aren't distracted from the main game play. Given that everyone in the group is a software developer they've each decided to write their own UI. James has decided to write a CLI.

He's looking for:

- a cloud-based service that tracks the rolls
- provides a free tier that doesn't require sign-up, with enough roll requests to handle a weekend campaign
- persists rolls for at least an hour
- shows the result of each die as well as the total
- supports modifiers (+/-)

It would be nice if it:

- allowed for an optional description or tag
- allowed for grouping of rolls to make tracking his group easier
- offered exporting of rolls

He understands that:

- all rolls and descriptions are public

#### Naomi

Naomi is the developer that made the mistake of saying, "I can write that for you!" and is now spending their weekend coding this thing. They have tons of great ideas but Richa is under a deadline so they decide to start with the basics. They start their design thinking of how Richa and James will interact with their service.

## Public API

Both Richa and James are familiar with URL-based APIs.

### Request

Naomi wants this to be easily consumed, without too much setup, so deicides on a URL+param value pattern:

```
https://tabletoys.com/dice?roll=3d6
```

Considerations

- Handling special characters (`+`, `-`, spaces)
- Preventing rolls from being cached

### Response

JSON works best for Richa and James. Richa only needs the final results while James wants to see each die. Given the small response payload (less than 512k) of an average roll Naomi includes both by default.

All responses contain two sections, `status` and `result`. 

- `status` - indicates whether the operation was a success or had errors. Valid values are `success` or `error`.
- `result` - contains either successful results or an array of all errors

**Success**

```json
{
    "status": "success",
    "roll": {
        "version": 1,
        "roll": "3d6+3",
        "result": 12,
        "dice":
        [
            { "die": 6, "result": 3 },
            { "die": 6, "result": 5 },
            { "die": 6, "result": 1  }
        ],
        "modifiers":
        [
            { "modifier": 3 }
        ]
    }
}
```

**Error**

```json
{
    "status": "error",
    "errors": [
        {
            "code": "albatross|beaver|camel",
            "error": "error doing something",
            "details" ""
        }
    ]
}
```

---

## The Components

### The Parser

This is responsible for taking a string and turning it into a quantity and type of dice to be rolled, along with any modifiers. She's using the same notion seen in most RPGs, 3d6 to mean roll three six-sided dice, with an optional + or minus followed by a number to add or subtract from the total.

#### Syntax

Input is a string conforming to the following format:

```
quantity = [0-9]+ - number of dice to roll, at least a single digit, including 0
die-type = [0-9]+ - type of die to roll, at least a single digit, including 0
modifier = [-,0-9] - optional modifier, at least a single digit, including negative values

[quantity](d/D)[die-type](+/-)[modifier]
```

#### Valid Inputs

- `d20` - quantity is optional if rolling a single die
- `d` or `D` - required, case insensitive
- `0` is allowed in all numeric positions
- `3d6+-3` - negative modifiers are allowed and will be applied as given
- `1 d 20 + -3` - white space is ignored but is preserved when persisting a roll

#### Errors

The following inputs result in an error response:

- `2+3d6` - out-of-position modifiers (unknown format)
- `abc3d6` - non-numeric dice quantity (unable to parse number of dice)
- `20`, `2+4` - missing dice indicator (missing the d)
- `3d6+2+2` - multiple modifiers (multiple modifiers found, only one allowed)
- `-3d20`, `1d-20` - negative quantity or die-type (negative quantities and die-types not allowed)
- `1d20 come on give me a crit` - trailing text (trailing non-numeric values not allowed)
- `3d6 + 1d4` - rolls as modifiers aren't supported (yet) (invalid modifier format)

#### Output

Output is a JSON object containing the original roll, an entry per die, and an optional modifier array. Modifiers are treated as an array even though the parser doesn't yet support multiple modifiers.

- An entry per die allows for eventual support for multiple die-types such as, `3d6 + 1d4`
- An entry per modifier allows for eventual support for chained modifiers, `d20 + 1 + 5 - -2`

```json
{
    "roll": "3d6+3",
    "dice":
    [
        { "die": 6 },
        { "die": 6 },
        { "die": 6 }
    ],
    "modifiers":
    [
        { "modifier": 3 }
    ]
}
```

### The Roller

The roller generates a random value within the given range, as determined by the die-type. It handles a single roll at a time and has no context or awareness of the other dice within a roll. It's main function is generating random numbers.

#### Design Considerations

Naomi pauses to consider the roller's inputs. Should it take the entire roll, with all the dice, or should the caller handle breaking the roll apart and send a single die at a time. Naomi already knows they want to add additional features and doesn't want to break the roller each time the format changes so they opt to handle a single die at a time, leaving the coordination up to the processing worker.

#### Inputs

- A single integer representing the die-type (the random number range), from 0 to MAXINT.
- Negative values are not allowed and will return an error

#### Outputs

- A random integer value between 1 and die-type
- 0 is only returned when die-type is 0

---

### Storage

Storage handles persisting a roll to a data store for later retrieval. It returns the unique ID that can be used to retrieve a prior roll.

Considerations

- Rolls only need to be retained for an hour
- The short retention period allows for non-database options like in-memory key/value datasets
- Limiting search to the ID opens up less-structured options such as NoSql/document storage
- The storage format should be independent of the final response to allow for multiple versions and greater freedom for change
- Long-term storage has a higher cost
- Items are immutable, removing the need for an update operation
- Item deletion will be triggered by a separate process based on time (everything older than an hour from creation date)
- Consumers shouldn't care or be aware of the underlying mechanism
- Adding a new roll should happen quickly and return an ID, without waiting for a lengthy write operation
- Eventual consistency is considered acceptable by her friends

Naomi weighs the ease of using an in-memory key/value store against a database and decides to start with in-memory knowing they'd like to move to a nosql document store if longer retention is needed or if the memory store proves unstable.

#### Storage API

A minimal set of operations are needed for V1. A sea of great ideas and options fill Naomi's mind but they only have the weekend so they focus on the bare minimum.

`int AddRoll(document)`

- accepts a JSON string
- verifies that a "version" attribute is present
- returns a unique ID used to retrieve the same document that was passed in
- should happen async
- does no duplication checking

`document = GetRoll(id)`

- returns the document associated with the given ID
- returns an error if a document with the given ID can't be found

`int DeleteRollsOlderThan(time-in-hours)`

- deletes all rolls with a creation date older than an hour
- datetime is UTC
- negative values result in an error
- 0 is valid, it just doesn't do anything
- returns the number of rolls deleted
- not finding any rolls to delete is a valid operation and will return 0

#### Document Format

Naomi decides to store the results as given but with a required version field that upper-layers can use to translate between versions.

- [x] how to handle setting the creation date?
  - creation date should be part of the document model
- [ ] how to make finding old rolls quick/non-expensive
- [x] does the caller need to see creation date, would they care?
  - start without it
- a common attribute for status allows greater reuse of the status parsing code

**Document Storage Format**

```json
{
    "created-on": UTC datetime,
    "document": passed in JSON document
}
```

**Document Response Format - Success**

```json
{
    "status": "success",
    "created-on": UTC datetime,
    "document": passed in JSON document    
}
```

**Document Response Format - Error**

```json
{
    "status": "error",
    "errors": [
        {
            "code": "albatross|beaver|camel",
            "error": "error doing something",
            "details" ""
        }
    ]
}
```

Adding error codes feels a little extra but Naomi likes being able to quickly determine the error category via a known code instead of parsing message strings. Details is for extra debugging information such as call stacks or passing along inner exceptions.

---

## General Architecture

1. A roll string is passed into the Dice service
1. The string is passed to the Parser
1. Each die is sent individually to the roller
1. The die is updated with the result
1. Once all dice have been rolled the total is computed
1. Modifers are added to the total
1. The total roll value is added to the roll
1. The roll result is saved by Storage
1. The roll and its ID are returned to the caller

### Going Bonkers

All of this could be handled by a single function or service, but where's the fun in that?

1. A roll string is sent to a lambda
1. The roll string is added to a parse queue
1. The Parser lambda is triggered by the addition of a new item to the queue
1. Parser pulls the item off the queue
1. Parse results are put into a Roller Queue
1. Roller lambda is triggered by addition
1. Roller processes each die
1. Roll results are put in a Done queue

The big questions are:

1. How to do all of this while still returning quickly?
1. How to reassemble the results of the Roller queue and keep track of everything?
1. If going 202 how to get the ID quickly?
   - ID becomes job ID which is passed to the look-up or a status check?
