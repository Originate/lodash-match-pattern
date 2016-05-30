# Match Pattern

![CircleCI badge](https://circleci.com/gh/mjhm/lodash-match-pattern.svg?style=shield&circle-token=:circle-token)

This is a general purpose validation tool for JSON objects. It includes facilities for deep matching, partial matching, unordered lists, and several advanced features for complex patterns.  It also includes a variety of validation functions from the `lodash-checkit` module (a [`lodash`](https://lodash.com/docs) extension mashup with [`checkit`](https://github.com/tgriesser/checkit)), and it allows for custom checking and mapping functions.

The primary goal of this and the supporting modules is to enable the highly flexible, expressive, and resilient feature testing of JSON based APIs.

#### Basic Usage
```
npm install lodash-match-pattern --save-dev
```
In your test file insert
```
var matchPattern = require('lodash-match-pattern');
var _ = matchPattern.getLodashModule(); // Use our lodash extensions (recommended)

// Example usage:

var testValue = {a: 1, b: 'abc'};

var successResult = matchPattern(testValue, {a: 1, b: _.isString});
// returns null for a successful match.

var failResult = matchPattern(testValue, {a: _.isString, b: 'abc'});
// returns "{a: 1} didn't match target {a: '_.isString'}"
```

#### Features Index

Here are the main features. You probably won't need all of them, but there's plenty of flexibility to allow you to adapt to the details of your specific use cases. All of the examples below are illustrated in the [`examples/example1/features/basic.feature`] as cucumber-js tests.

1. [Deep JSON matching](#deep-json-matching)
1. [Matching property types](#matching-property-types)
1. [Partial objects](#partial-objects)
1. [Partial and superset matches of arrays](#partial-and-superset-matches-of-arrays)
1. [Omitted items](#omitted-items)
1. [Parametrized matchers](#parametrized-matchers)
1. [Unsorted arrays](#unsorted-arrays)
1. [Transforms](#transforms)
1. [Multiple matchers](#multiple-matchers)
1. [Customization](#customization)

## JavaScript Objects vs "JSON Pattern Notation"

There are two similar ways to specify patterns to match. JavaScript objects are more convenient for `mocha` and JavaScript test runners that aren't multiline string friendly. "JSON Pattern Notation" is more readable in `cucumber` tests and other environments that support multiline strings. Almost all patterns can be expressed in either form. In most cases below the examples will be shown in both forms.

## Deep JSON matching

Just for starters, suppose we have a "joeUser" object and want to validate its exact contents.  Then `matchPattern` will do a deep match of the object and succeed as expected.
<table><tr>
<th>JavaScript Objects (mocha)</th><th>JSON Pattern Notation (cucumber)</th>
</tr>
<tr><td><pre>
var matchPattern = require('lodash-match-pattern');
var joeUser = getJoeUser();

describe('basic match', function () {
  it('matches joeUser', function () {
    var matchResult = matchPattern(joeUser,
{
  id: 43,
  email: 'joe@matchapattern.org',
  website: 'http://matchapattern.org',
  firstName: 'Joe',
  lastName: 'Matcher',
  createDate: '2016-05-22T00:23:23.343Z',
  tvshows: [
    'Match Game',
    'Sopranos',
    'House of Cards'
  ],
  mother: {
    id: 23,
    email: 'mom@aol.com'
  },
  friends: [
    {id: 21, email: 'bob@mp.co', active: true},
    {id: 89, email: 'jerry@mp.co', active: false},
    {id: 14, email: 'dan@mp.co', active: true}
  ]
};
    if (matchResult) throw(new Error(matchResult));
  });
});
</pre></td><td><pre>
  Given I have joeUser
  Then joeUser matches the pattern
    """
{
  id: 43,
  email: 'joe@matchapattern.org',
  website: 'http://matchapattern.org',
  firstName: 'Joe',
  lastName: 'Matcher',
  createDate: '2016-05-22T00:23:23.343Z',
  tvshows: [
    'Match Game',
    'Sopranos',
    'House of Cards'
  ],
  mother: {
    id: 23,
    email: 'mom@aol.com'
  },
  friends: [
    {id: 21, email: 'bob@mp.co', active: true},
    {id: 89, email: 'jerry@mp.co', active: false},
    {id: 14, email: 'dan@mp.co', active: true}
  ]
});
    """
</pre></td></tr>
</table>

##### Notes
* In this case the JS Object and the Pattern Notation are visually identical. The only difference is the first is a JS object and the second is a string.
* For all the following examples we'll leave out the surrounding test boiler plate.
* For completeness example the cucumber step definitions could be defined as:

```
// steps.js
var matchPattern = require('lodash-match-pattern');
module.exports = function () {
  var self = this;

  self.Given(/^I have joeUser$/, function () {
    self.user = {
      id: 43,
      email: 'joe@matchapattern.org',
      ...
    }
  });

  self.Then(
    /^joeUser matches the pattern$/,
    function (targetPattern) {
      var matchResult = matchPattern(self.user, targetPattern);
      if (matchResult) throw matchResult;
    }
  );
};
```

Unfortunately, deep matching of exact JSON patterns creates over-specified and brittle feature tests. In practice such deep matches are only useful in small isolated feature tests and occasional unit tests. Just for example, suppose you wanted to match the exact `createDate` of the above user. Then you might need to do some complex mocking of the database to spoof a testable exact value. But the good news is that we don't really care about the exact date, and we can trust that the database generated it correctly. All we really care about is that the date looks like a date. To solve this and other over-specification problems `lodash-match-pattern` enables a rich and extensible facility for data type checking.


## Matching property types

The pattern below may look a little odd at first, but main idea is that there's a bucket full of `_.isXxxx` matchers available to check the property types. All you need to do is slug in the pattern matching function and that function will be applied to the corresponding candidate value.
<table><tr>
<th>JavaScript Objects and JSON Pattern Notation</th>
</tr>
<tr><td><pre>
{
  id: _.isInteger,
  email: _.isEmail,
  website: _.isUrl,
  firstName: /[A-Z][a-z]+/,
  lastName: _.isString,
  createDate: _.isDateString,
  tvshows: [
    _.isString,
    _.isString,
    _.isString
  ],
  mother: _.isObject,
  friends: _.isArray
}
</pre></td></tr>
</table>

##### Notes
* Again the two forms are visually identical. However there's one significant difference. For the JS Objects the matching functions (e.g `_.isString`) can be any function in scope. In contrast the corresponding Pattern Notation functions are required to be members of our lodash extension module and are required to begin with "is".

* The available matching functions are
  1. All `isXxxx` functions from `lodash`.
  1. All validation functions from `checkit` with `is` prepended.
  1. Case convention matchers constructed from lodash's `...Case` functions.
  1. Any regular expression -- intepreted as `/<regex>/.test(<testval>)`.
  1. `isDateString`, `isSize`, `isOmitted`
  1. Any `isXxxx` function you insert as a lodash mixin through [customization](#customization).

To see the full list run this:
```
console.log(
  Object.keys(require('lodash-match-pattern').getLodashModule())
  .filter(function (fname) { return /^is[A-Z]/.test(fname) })
);
```

## Partial objects

Most of the time feature tests are interested in how objects change, and we don't need be concerned with properties of an object that aren't involved in the change.  Matching only partial objects can create a huge simplification which focuses on the subject of the test. For example if we only wanted to test changing our user's email to say "billybob@duckduck.go" then we can simply match the pattern:
<table><tr>
<th>JavaScript Objects (mocha)</th><th>JSON Pattern Notation (cucumber)</th>
</tr>
<tr><td><pre>
{
  id: _.isInteger,
  email: "billybob@duckduck.go",
  "...": ""
}
</pre></td><td><pre>
{
  id: _.isInteger,
  email: "billybob@duckduck.go",
  ...
}
</pre></td></tr></table>

The `"..."` object key indicates that only the specified keys are matched, and all others in `joeUser` are ignored.

_Note: from here on all the examples will use partial matching, and all will successfully match "joeUser"._

## Partial, superset, and equalset matches of arrays

Similarly partial arrays can be matched with a couple caveats:

1. The array entries must be numbers or strings, no nested objects or arrays.
2. The partial arrays are matched as sets -- no order assumed.

<table><tr>
<th>JavaScript Objects (mocha)</th><th>JSON Pattern Notation (cucumber)</th>
</tr>
<tr><td><pre>
{
  tvshows: [
    "House of Cards",
    "Sopranos",
    "..."
  ],
  "...": ""
}
</pre></td><td><pre>
{
  tvshows: [
    "House of Cards",
    "Sopranos",
    ...
  ],
  ...
}
</pre></td></tr></table>

Note that the above specifies both a partial array (for `joeUser.tvshows`) and a partial object (for `joeUser`).

Supersets are similarly specified by "^^^". This following says that `joeUser.tvshows` is a subset of the list in the pattern below:
<table><tr>
<th>JavaScript Objects (mocha)</th><th>JSON Pattern Notation (cucumber)</th>
</tr>
<tr><td><pre>
{
  tvshows: [
    "House of Cards",
    "Match Game",
    "Sopranos",
    "Grey's Anatomy",
    "^^^"
  ],
  "...": ""
}
</pre></td><td><pre>
{
  tvshows: [
    "House of Cards",
    "Match Game",
    "Sopranos",
    "Grey's Anatomy",
    ^^^
  ],
  ...
}
</pre></td></tr></table>

Or to compare equality of arrays as sets by unordered membership, use "===":

<table><tr>
<th>JavaScript Objects (mocha)</th><th>JSON Pattern Notation (cucumber)</th>
</tr>
<tr><td><pre>
{
  tvshows: [
    "House of Cards",
    "Match Game",
    "Sopranos",
    "==="
  ],
  "...": ""
}
</pre></td><td><pre>
{
  tvshows: [
    "House of Cards",
    "Match Game",
    "Sopranos",
    ...
  ],
  ...
}
</pre></td></tr></table>

Note that the JS Object form adds the set matching symbols as extra array entries. If you actually need to literally match `"..."`, `"^^^"`, or `"^^^"` in an array see the [customization](#customization) example below.

## Omitted items

Sometimes an important API requirement specifies fields that should not be present, such as a `password`. This can be validated with an explicit `_.isOmitted` check (an alias of `_.isUndefined`). Note that it works properly with partial objects.
```
    {
      "id": 43,
      "password": "_.isOmitted",
      "...": ""
    }
```

## Parametrized matchers

Some of the matching functions take one or two parameters. These can be specified with ":" separators at the end of the matching function.
```
    {
      "id": "_.isBetween:42.9:43.1",
      "tvshows": "_.isContainerFor:House of Cards",
      "...": ""
    }
```

## Unsorted arrays
Often database rows are returned with no guaranteed order, this is problematic for matching specific array elements. For simple arrays with number and string entries, it's sufficient to just specify the partial array ordering ellipsis `"..."`, since it is defined to be an unsorted array match.  However for more complex arrays a `_.sortBy` transform function needs to be applied to the array values to put them in a predictable order.
```
    {
      "tvshows": [
        "Sopranos",
        "House of Cards",
        "Match Game",
        "..."
      ],
      "friends": {
        "_.sortBy:email": [
          {"id": 21, "email": "bob@matchpattern.org", "active": true},
          {"id": 14, "email": "dan@matchpattern.org", "active": true},
          {"id": 89, "email": "jerry@matchpattern.org", "active": false}
        ]
      },
      "...": ""
    }
```


## Transforms

Hopefully you will need complex transform functions only occasionally because they reduce the clarity of test patterns. However they are nevertheless useful and ultimately quite powerful.

Transforms such as `_.sortBy:email` above are inserted as a sole key value that wraps the target pattern. It's a little unintuitive but the transform functions are applied to the values under test, not to the pattern.

Just to illustrate another transform function, a different approach for checking the contents of the friends list is to use the `_.map` function to pull the "email" values, then compare the result with the ellipsis to indicate and unsorted array check.

```
    {
      "friends": {
        "_.map:email": [
          "bob@matchpattern.org",
          "dan@matchpattern.org",
          "jerry@matchpattern.org",
          "..."
        ]
      },
      "...": ""
    }
```

`_.size` is an simple transform function which can be used to match the size of an array. (Alhough the `_.isSize` matching function is even more convenient.)

```
    {
      "friends": {
        "_.size": 3
      },
      "...": ""
    }
```


Transforms may be composed. However note that they are applied to the values under test in reverse order to how they are specified on the pattern.
```
    {
      "friends": {
        "_.filter:active": {
          "_.size": 2
        }
      },
      "...": ""
    }
```
Here `_.filter(..., 'active')` is first applied to the test `friends` list, resulting in an array of the two active friends. Then `_.size` is applied to the result, and is compared successfully to the specified target value "2".

## Multiple matchers

Occasionally it may be convenient to apply multiple matching assertions in the same pattern. This can be accomplished with the `_.arrayOfDups` transform.
```
    {
      "tvshows": {
        "_.arrayOfDups:2": [
          "_.isSize:3",
          "_.isContainerFor:Sopranos"
        ]
      },
      "...": ""
    }
```
In this example `_.arrayOfDups:2` creates two copies of the "tvshows" array. The first copy is matched against `_.isSize:3` and the second copy is checked against by `_.isContainerFor:Sopranos`.

## Customization

In many cases application of transforms will create unintuitive and hard to understand pattern specifications. Fortunately creating custom matchers and custom transforms is easily accomplished via lodash mixins. For our examples we've added two lodash mixins in our example code:
```
var _ = require('lodash-checkit');
_.mixin({
  literalEllipsis: function (array) {
    return array.map(function (elem) {
      if (elem === '...') return 'LITERAL...';
      if (elem === '---') return 'LITERAL---';
      return elem;
    });
  },

  isSizeAndIncludes: function (array, size, includes) {
    if (! _.isArray(array)) return false;
    if (_.size(array) !== size) return false;
    return _.includes(array, includes);
  }
});
chaiMatchPattern.use(_);
```

We can then use `isSizeAndIncludes` to simplify the previous transform example into:
```
  {
    "tvshows": "_.isSizeAndIncludes:3:Sopranos",
    "...": ""
  }
```

The custom `literalEllipsis` transform can be used to enable literal pattern matching of "..." and "---" in arrays. So for example if for some reason `joeUser` had this as his `tvshows`:
```
  [
    "Mannix",
    "Game of Thrones",
    "...",
    "---"
  ]
```
Then the following now has a successful pattern match:
```
    {
      "tvshows": {
        "_.literalEllipsis": [
          "Mannix",
          "Game of Thrones",
          "LITERAL...",
          "LITERAL---"
        ]
      },
      "...": ""
    }
```
