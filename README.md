# nemo-fixo

nemo-fixo is a [nemo](https://github.com/paypal/nemo) plugin to manage your test fixtures.

Test fixtures can be complex when you need to run your tests for different environments and locales, as there tends to be a lot of data duplication.

`nemo-fixo` solves this issue with [`fixo`](https://github.com/paypal/fixo), an extension to JSON parser to extend and compose JSON objects. `fixo` instance is accessible from `nemo.fixo`; `nemo-fixo` also provides `load` and `iterate` functions to use the fixture data to drive your mocha test cases.

## Installation

```sh
$ npm install nemo-fixo
```

## How to use it

Edit Nemo `config.json` file, add `nemo-fixo` plugin settings:

```json
{
  "plugins": {
    "nemo-fixo": {
      "module": "nemo-fixo",
      "arguments": [{
         "srcDir": "test/fixture"
      }]
    }
  }
}
```

### Options

* **srcDir**: Test fixtures directory (_required_, default: `test/fixture`)
* **specDir**: Test specs directory (_optional_, default: `undefined`), more info [here](#load-fixture-with-__filename).
* **defaultIterateList** : Default list loaded by `iterate` helper function (_optional_, default: `all`), more info [here](#iterate-config-profiles-key).

### Fixo instance

```js
// Load fixture
nemo.fixo.load('card', function(err, card) { ...  });

// Or with Promise
nemo.fixo.load('card').then(function(card) { ...  });

// Load fixture property for a profile
nemo.fixo.load('card.visa', 'GB', function(card) { ...  });

// Load multiple fixtures
nemo.fixo.load(['card', 'bank'], function(fixtures) { ...  });

// Load fixture in a folder
nemo.fixo.load('folder/card', function(card) { ...  });
```

#### Fixture file

##### Inheritance and Override

Fixo supports object inheritance and override, example:
- `GB` extends from `master`
- `GB-en` extends from `master`, `GB`, `en`
- `GB-en-dev` extends from `master`, `GB`, `en`, `GB-en`

```json
{
  "master": {
    "visa": {
       "account_number": "1111111111111111",
       "expiry_date": "08/18",
     "cvv": "123"
  },
    "diners": {
      "account_number": "22222222222222",
       "expiry_date": "08/18",
       "cvv": "123"
    },
    "amex": {
       "account_number": "333333333333333",
     "expiry_date": "08/18",
     "cvv": "123"
    }
  },
  "GB": {
    "visa": {
      "account_number": "4444444444444444"
    }
  },
  "en": {},
  "GB-en": {},
  "GB-en-GB": {}
}
```

##### Default values

Shared default values. Visa, Diners, Amex extend the default values:

```js
{
  "master": {
    "default": {
       "expiry_date": "08/18",
       "cvv": "123"
    },
    "visa": {
       "account_number": "1111111111111111",
       "cvv": "123"
    },
    "diners": {
       "account_number": "22222222222222",
    },
    "amex": {
       "account_number": "333333333333333"
    }
  }
}
```

##### Resolvers and Macros

`Resolvers` allow you to transform an object property value as a whole, while `Macros` allow you to replace parts of a string value with dynamic content.

You could easily create custom resolvers and macros, refer to [fixo documentation](https://github.com/paypal/fixo) for more information.

```js
{
  "master": {
    "profile": {
      "name": "Walter Mitty",
      "email": "test-{random}@domain.com"
    }
    "name": "get:profile.name",
    "wallet": {
      "bank": "include:bank.citibank",
      "card_GB": "include.GB:card.visa"
    }
  },
  "GB": {
    "profile": {
      "name": "William"
    }
  }
}
```

### Load and iterate functions

Use your test fixture data to drive your test cases. `nemo-fixo` provides `load` and `iterate` functions. They load fixtures synchronously, so you could use the fixture data to customize your test spec:
* `nemo-fixo/load` — to load one or multiple fixture
* `nemo-fixo/iterate` — to load one or multiple fixtures and iterate over different profiles

#### Load fixture function

```js
var load = require('nemo-fixo/load');
var fixture = load('fixture-name');

if (fixture.flagOn) {
  it('should test this', function() { ... }
} else {
  it('should test that', function() { ... }
}

// Load multiple fixtures
var fixtures = load(['card', 'bank']);
```

#### Iterate fixtures function

To repeat test scenario for each country in the iterate list:

```js
var iterate = require('nemo-fixo/iterate');

iterate('fixture-name', ['US', 'GB', 'CA'], function(profile, fixture, index) {
  describe('Test scenario for profile ' + profile, function() {
    // Use fixture data to further customize your test spec
    fixture.elements.forEach(function(element) {
      it('should... for ' + element.name, function(done) {
        ...
      });

      // Load a new fixture instance for each test case
      it('should... for ' + element.name, function(done, newFixture) {
        ...
      });
    });
  });
});
```

##### Iterate profile list

Iterate profile list is determined from multiple sources in the following precedence:

1. `FIXO_ITERATE_PROFILES` environment variable, a comma-seperated value of profile list. Used in development to force iterate a fixed set of profiles.
2. Profile list argument passed to `iterate` function
3. Profile list configured in the fixture file under `iterate` property name

```js
// test/fixture/card.json
{
  "master": {
    "visa": {},
    "amex": {}
  },
  "US": {},
  "GB": {},
  "CA": {},
  ...
  "iterate": {
    "all": ["US", "GB", "CA"]
  }
}
```

```js
// test/spec/test-card.js
// Profile list is retrieved from the card.json, i.e. ['US', 'GB', 'CA']
iterate('card', function(profile, fixture) {
  ...
});
```

##### Multiple profile lists

You could configure multiple profile lists in your fixture file. By default, iterate method will look for the `all` list:

```js
// test/fixture/card.json
{
  "master": {},
  ...
  "iterate": {
    "all": [...],
    "small": [...],
    "medium": [...],
    "large": [...]
  }
}
```

If you need to change the default list name, set `defaultIterateList` option when configuring nemo fixo plugin. To load a specific profile list, you could pass the `list` option:

```js
iterate('card', undefined, { list: 'small' }, function(profile, card) { ... });
```

When configuring your CI environment, you could also set `FIXO_ITERATE_LIST` environment variable.

##### Load fixture matching the spec filename

To load fixture matching the spec name and the directory structure, configure `specDir` option in Nemo `config.json` and pass `__filename` for the fixture name.

```js
// Nemo config.json
{
  "plugins": {
    "nemo-fixo": {
      "modules": "nemo-fixo"
      "arguments": [{
         "srcDir": "test/fixture"
         "specDir": "test/spec"       // Comment off to ignore fixture directory structure
      }]
    }
  }
}

// test/spec/nested/test-card.js
// fixture will load from test/fixture/nested/test-card.json
load(__filename, function(fixture) { ... });
iterate(__filename, function(profile) { ... });
```

## License

Copyright (C) 2016 PayPal

Licensed under the Apache License, Version 2.0 (the "License"); you may
not use this file except in compliance with the License. You may obtain
a copy of the License at http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
