# Aurelia HOVM

Enhance your Aurelia's view models with high order functionality

## Introduction

> - If you are wondering why I built this, go to the [Motivation](#motivation) section.
> - HOVM stands for "High order ViewModel"

Here's a really basic example of what you can achieve using this library:

```js
// myComponent.js
import { enhance } from 'aurelia-hovm';
import LogStatus from './logStatus';

@enhance(LogStatus)
class MyComponent {}

export { MyComponent };

// logStatus.js
class LogStatus {
  attached() {
    console.log('attached :D!');
  }
  detached() {
    console.log('detached D:!');
  }
}
```

When your component gets attached to the DOM (or gets detached), via the enhancement class `LogStatus`, it will log a message on the console; all of this without having to write the functionality on your component class and without needing class inheritance.

## Usage

As you saw on the example above, the way it works is pretty straightforward:

1. Define an enhancement class with methods you want to be called when the ones of the ViewModel are triggered.
2. Add the enhancement to the ViewModel using the `enhance` decorator.

There are also a few extra features that may be useful for you:

### Access the ViewModel

When instantiating an enhancement, the library sends to a reference of the ViewModel instance, so you can access methods and properties on your custom methods.

Let's say you have a component that renders a form where the user saves some important information, and if the user were to leave the route without saving the information, you want to use a prompt to ask for confirmation.

A basic approach would be to have a flag indicating if the changes are saved and then use the `canDeactivate` lifecycle method to decide whether to show the prompt or not:

```js
class MyForm {
  isSaved = false;
  ...
  canDeactivate() {
    // If everything is saved, go away.
    if (isSaved) {
      return true;
    }

    // Let's make the prompt async.
    return new Promise((resolve) => {
      // ask for the user confirmation.
      const answer = confirm('Confirm that you want to leave without saving');
      // resolve the promise with the answer.
      resolve(answer);
    });
  }
}

export { MyForm };
```

It works, is simple and you didn't have to involve any external logic. But now, what if you have to implement this same logic on four more forms? That's when an enhancement is useful.

Let's write an enhancement that uses the reference for the ViewModel in order to verify if the user needs to be prompted.

```js
class FormConfirmation {
  constructor(viewModel) {
    this._viewModel = viewModel;
  }

  canDeactivate() {
    if (this._viewModel.isSaved) {
      return true;
    }

    return new Promise((resolve) => {
      const answer = confirm('Confirm that you want to leave without saving');
      resolve(answer);
    });
  }
}
```

It's the same functionality, but it now checks the `isSaved` property form the ViewModel.

In this case it was really easy because the property is a `boolean` and the `if` checks with _falsy_, but we could've check if the property was defined and use a different default, like "if the ViewModel doesn't have the property, the user can always leave without confirmation".

Ok, let's add it to the form:

```js
import { enhance } from 'aurelia-hovm';
import { FormConfirmation } from '...';

@enhance(FormConfirmation)
class MyForm {
  isSaved = false;
  ...
}

export { MyForm };
```

And that's all, you can now add it to the other four forms using the enhancement.

### Dependency injection

Just like on any other class you use in the Aurelia context, you can use the `@inject` decorator on an enhancement in order to inject dependencies.

Will use the enhancement from the first example and trigger an event before the log messages.

```js
import { inject } from 'aurelia-framework';
import { EventAggregator } from 'aurelia-event-aggregator';

@inject(EventAggregator);
class LogStatus {
  constructor(viewModel, ea) {
    this._viewModel = viewModel;
    this._ea = ea;
  }
  attached() {
    this._ea.publish('something:attached', this._viewModel);
    console.log('attached :D!');
  }
  detached() {
    this._ea.publish('something:detached', this._viewModel);
    console.log('detached D:!');
  }
}
```

Is that simple! Just like a regular component/service.

This the feature that makes this library Aurelia-specific.

### Enhance an enhancement

As mentioned above, dependency injection is the only thing that makes this library Aurelia-specific, and it's an "optional feature", which means that you can enhance any kind of class.

Let's take the example about dependency injection and move the events part to another enhancement:

```js
import { inject } from 'aurelia-framework';
import { EventAggregator } from 'aurelia-event-aggregator';

@inject(EventAggregator);
class PublishStatus {
  constructor(viewModel, ea) {
    this._viewModel = viewModel;
    this._ea = ea;
  }
  attached() {
    this._ea.publish('something:attached', this._viewModel);
  }
  detached() {
    this._ea.publish('something:detached', this._viewModel);
  }
}

export { PublishStatus };
```

Now we can create an enhance `LogStatus` with `PublishStatus`:

```js
import { enhance } from 'aurelia-hovm';
import { PublishStatus } from '...';

@enhance(PublishStatus)
class LogStatus {
  attached() {
    console.log('attached :D!');
  }
  detached() {
    console.log('detached D:!');
  }
}
```

That was just to prove the point that you can enhance an enhancement, but there are two other (and simpler) ways in which you can achieve the same result:

#### The decorator as a function

Decorators are just functions, in this case, a function that returns a function:

```js
enhance(...Enhancements)(ViewModel): Proxy<ViewModel>
```

So, instead of enhancing `LogStatus` with `PublishStatus`, we can create a new enhancement with both of them:

```js
import { enhance } from 'aurelia-hovm';
import { LogStatus } from '...';
import { PublishStatus } from '...';

export const PublishAndLogStatus = enhance(PublishStatus)(LogStatus);
```

#### Multiple enhancements at once

The `enhance` decorator supports multiple enhancements as parameters, so we could just send `LogStatus` and then `PublishStatus` and the result would be the same:

```js
import { enhance } from 'aurelia-hovm';
import { LogStatus } from '...';
import { PublishStatus } from '...';

@enhance(LogStatus, PublishStatus)
class MyComponent {}
```

### Lifecycle method

Let's say we have this enhancement:

```js
class FormConfirmation {
  constructor(viewModel) {
    this._viewModel = viewModel;
  }

  canDeactivate() {
    if (this._viewModel.isSaved) {
      return true;
    }

    return new Promise((resolve) => {
      const answer = confirm('Confirm that you want to leave without saving');
      resolve(answer);
    });
  }
}
```

But on the ViewModel, you want to add some other functionality that also needs to run on the `canDeactivate` but only if the enhanced method returned `false`.

I didn't want to modify the signature of the enhanced method because, if the method has optional parameters, it would end up being a total mess.

The easiest way to solve this is with the lifecycle method the library uses to send the returned value from the enhanced method.

The method, is called `enhanced[OriginalMethodName]Return` (even if the method name starts with lowercase, the library will make the first letter uppercase) and this is the signature:

```js
enhancedCanDeactivateReturn(value, enhancementInstance): void
```

If the ViewModel implements it, the library will call it with whatever value the enhanced method returned, and a reference to the enhancement class instance.

## Development

### NPM/Yarn tasks

| Task       | Description                         |
|------------|-------------------------------------|
| `test`     | Run the project unit tests.         |
| `lint`     | Lint the modified files.            |
| `lint:all` | Lint the entire project code.       |
| `docs`     | Generate the project documentation. |
| `todo`     | List all the pending to-do's.       |

### Repository hooks

I use [husky](https://yarnpkg.com/en/package/husky) to automatically install the repository hooks so the code will be tested and linted before any commit and the dependencies updated after every merge. The configuration is on the `husky` property of the `package.json` and the hooks' files are on `./utils/hooks`.

### Testing

I use [Jest](https://facebook.github.io/jest/) with [Jest-Ex](https://yarnpkg.com/en/package/jest-ex) to test the project. The configuration file is on `./.jestrc.json`, the tests are on `./tests` and the script that runs it is on `./utils/scripts/test`.

### Linting

I use [ESlint](http://eslint.org) with [my own custom configuration](http://yarnpkg.com/en/package/eslint-plugin-homer0) to validate all the JS code. The configuration file for the project code is on `./.eslintrc` and the one for the tests is on `./tests/.eslintrc`. There's also an `./.eslintignore` to exclude some files on the process. The script that runs it is on `./utils/scripts/lint`.

### Documentation

I use [ESDoc](http://esdoc.org) to generate HTML documentation for the project. The configuration file is on `./.esdoc.json` and the script that runs it is on `./utils/scripts/docs`.

### To-Dos

I use `@todo` comments to write all the pending improvements and fixes, and [Leasot](https://yarnpkg.com/en/package/leasot) to generate a report. The script that runs it is on `./utils/scripts/todo`.

## Motivation

TBD
