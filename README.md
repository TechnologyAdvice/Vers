# Vers [![Build Status](https://travis-ci.org/TomFrost/Vers.svg?branch=master)](https://travis-ci.org/TomFrost/Vers)
Effortless data model versioning for Javascript and Node.js

## Why do I need model versioning?
- Support versioned REST APIs without cluttering your API code with an endpoint
for every individual version
- Only code against the latest version of your data -- no messy if-statements
throughout your codebase to check for earlier versions or properties in
different places
- Update your backends and your frontends on their own schedules 
- Roll out new schemas and update your code for the new changes
independently, without coordinating a precise rollout, and with no downtime
- For the love of all things holy _stop trying to update every record in your
document store every time you update your data model._ Every time you pull a new
record from your database, just call `toLatest` on it. Done. If you save it,
it saves as the new version. If you don't, it stays the old version and saves
you the bandwidth/request time. Your code only ever deals with the latest
version and you never scan through and update your entire database ever again.
Doesn't that feel better?

## Quick start
```javascript
const Vers = require('vers')
const baseUser = {
  version: 1,
  firstName: 'Doug',
  lastName: 'Funnie'
}
const userVers = new Vers()

// Tell Vers how to convert from version 1 to version 2, and back again.
// We could also use "1.0.1" to "2.4.3", 100 to 200, or "cow" to "chicken"
userVers.addConverter(1, 2, obj => {
  // Version 2 has user initials
  obj.version = 2
  obj.initials = obj.firstName[0] + obj.lastName[0]
}, obj => {
  // Version 1 does not
  obj.version = 1
  delete obj.initials
})

// And now to go from version 2 to version 3 and back
userVers.addConverter(2, 3, obj => {
  // Version 3 combines the names into a single name field 
  obj.version = 3
  obj.name = obj.firstName + ' ' + obj.lastName
  delete obj.firstName
  delete obj.lastName
}, obj => {
  // To go back to version 2, we'd need to split them up again
  obj.version = 2
  const names = obj.name.split(' ')
  obj.firstName = names[0]
  obj.lastName = names.pop()
  delete obj.name
})

userVers.toLatest(baseUser).then(user => {
  // user is now:
  // {
  //   version: 3,
  //   initials: 'DF',
  //   name: 'Doug Funnie'
  // }
  return userVers.to(1, user)
}).then(user => {
  // user is back to the original version defined in baseUser
})
```

## Installation
Vers requires an environment that supports the
[Promise/A+](https://promisesaplus.com/) specification as standardized in ES6.
Node.js version 0.12.0 and up is great right out of the box (no --harmony flag
necessary), as well as the latest versions of many browsers. To support older
browsers, just include a Promise library such as
[Bluebird](https://github.com/petkaantonov/bluebird).

To install, just type:

    npm install vers --save

## API
### new Vers([options])
> _Function_ **options.getVersion**: A function that accepts an object as its
only argument, and returns either the current version identifier of the object
as a number or string, or a Promise that resolves to the current version
identifier. By default, Vers will use the object's `version` property if it
exists, or `1` if it doesn't.  
> _number|string_ **options.latest**: The latest version identifier available for
this model. If not specified, Vers will detect the latest version by calling
`Math.max` on each version specified in `addConverter()`. For string-based
versions, this option should be specified.

Constructs a new instance of Vers. Each data model should have one instance to
define all of its versions. The options object is optional.

### addConverter(fromVer, toVer, forward, [back])
> _number|string_ **fromVer**: The version to convert from  
> _number|string_ **toVer**: The version to convert to  
> _Function_ **forward**: A function that accepts an object to be converted,
and moves it from `fromVer` to `toVer`.  
> _Function_ **back**: An optional function that accepts an object and moves it
from `toVer` to `fromVer`.

Adds a converter to this instance that knows how to change an object from one
version to another, and optionally, how to go back again. If you're using Vers
to power a versioned REST API, then telling it how to go back again is
essential. If your versioning scheme uses numbers, Vers will use `Math.max` to
determine what your latest version is so you don't have to specify that in the
constructor.

Vers is smart: if you need to upgrade from Version 1 to Version 5, it will
upgrade from 1 to 2, then from 2 to 3, and so on up to 5, assuming that you've
added converters for each of those. However, if you add a converter that
shortcuts that in any way to jump over some of those versions, Vers will always
find the shortest path possible to the target -- even if that means upgrading
from 1 to 6, then downgrading to 5. 

Note that the `forward` and `back` functions are called with the object to be
converted as their only argument. These functions can:
- modify the object directly and return nothing
- return a new object
- return a Promise that resolves with the modified or new object

Any method is fine! But keep in mind: modifying the object directly _will_ also
modify the source object. Vers doesn't clone your objects.

### fromTo(fromVer, toVer, obj)
> **Returns a _Promise_ that resolves to an _Object_**  
> _number|string_ **fromVer**: The starting version  
> _number|string_ **toVer**: The target version  
> _Object_ **obj**: The object to be converted

Converts an object from one version to another, using the provided `fromVer` as
the current version instead of trying to detect it. The result is passed on
in the form of a Promise that resolves with the object in its target version.

### fromToLatest(fromVer, obj)
> **Returns a _Promise_ that resolves to an _Object_**  
> _number|string_ **fromVer**: The starting version  
> _Object_ **obj**: The object to be converted

Converts an object from its current version to the latest version available,
using the provided `fromVer` as the current version instead of trying to detect
it. The result is passed on in the form of a Promise that resolves with the
object in its target version.

### to(toVer, obj)
> **Returns a _Promise_ that resolves to an _Object_**  
> _number|string_ **toVer**: The target version  
> _Object_ **obj**: The object to be converted

Converts an object from its auto-detected current version to the `toVer`
version. The result is passed on in the form of a Promise that resolves with
the object in its target version.

### toLatest(obj)
> **Returns a _Promise_ that resolves to an _Object_**  
> _Object_ **obj**: The object to be converted

Converts an object from its auto-detected current version to the latest version
available. The result is passed on in the form of a Promise that resolves with
the object in its target version.

## License
Vers is licensed under the MIT license. Please see `LICENSE.txt` for full
details.

## Credits
Vers was originally created at [TechnologyAdvice](http://technologyadvice.com) in Nashville, TN.

