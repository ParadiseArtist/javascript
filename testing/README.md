# `import {Testing} from 'Shopify'`

This guide looks at our recommended tools and conventions for testing JavaScript at Shopify. The primary focus of this guide is unit testing, but you can find some discussion of other types of testing in the [types of testing section](#types-of-testing).

This guide does recommend some tools that you can use to get up and running quickly with JavaScript testing. However, we want to provide the opportunity for more experienced developers to expand their understanding and to help us identify great new tools. We’re happy with a developer choosing a tool that isn't our primary recommendation, as long as it adheres to the guidance we’ve set out below for what features to look for (and avoid), and what testing behaviors to encourage.



## Table of contents

1. [Tools](#tools)
1. [Style](#style)
1. [Best practices](#best-practices)
1. [Types of testing](#types-of-testing)
1. [React](#react)
1. [Coverage](#coverage)



## Tools

### Frameworks

Your testing framework is the main API you will be interacting with to write your tests (aside from the API of the code under test, of course!). It is important that the test runner you use has clear conventions around how to break up tests, good support for asynchronous tests, and is an established tool that is unlikely to have major changes in its API.

If you don’t have any familiarity or preference with JavaScript testing, we recommend using the following tools in your tests:

- [**mocha**](https://mochajs.org/) is our preferred test framework. It provides the basic scaffolding for tests including hooks, excellent support for asynchronous code, and more.

- [**chai**](http://chaijs.com/) is our preferred assertion library. It provides many expressive assertions that have descriptive error messages in the case of test failures.

- [**sinon**](http://sinonjs.org/) is our preferred library for handling spies and mocks in JavaScript.

These tools are commonly used together and have many excellent online resources, including solid documentation. In addition to the above, you can use [chai plugins](http://chaijs.com/plugins/) to provide more meaningful assertions. Use caution, however: only include a plugin where it materially helps improve the legibility of tests (otherwise, the burden of other developers tracking down a particular assertion outweighs the benefits you get from having it). [chai-as-promised](http://chaijs.com/plugins/chai-as-promised/) is an excellent example of a chai plugin that makes otherwise awkward tests much easier to read.

If you feel that mocha/ sinon/ chai do not do a good job of addressing the way your team wants to write tests, feel free to use any of the following tools in their place:

- [AVA](https://github.com/avajs/ava) is a very modern testing framework that provides a much simpler, more JavaScript-native API than mocha, and has built-in support for code transpiling using Babel. We also have a shared linting configuration for AVA (just include `plugin:shopify/ava` in your ESLint config). However, AVA does not yet provide a browser test runner, so this tool only works for node.js projects.

- [Tape](https://github.com/substack/tape) provides a similarly-small API surface area, letting your tests take center stage.

- [testdouble](https://github.com/testdouble/testdouble.js) is a great alternative to sinon as a test double library.

If your project is using [ReactJS](https://reactjs.org/) or is built with [sewing-kit](https://github.com/Shopify/sewing-kit), we recommend using the following tools:
- [Jest](http://facebook.github.io/jest/) for the test framework, which also handles spies and mocks.
- [Enzyme](http://airbnb.io/enzyme/) for handling the rendering and manipulating React components.

### Test Runners

In addition to the testing framework, you generally need a test *runner*. Which runner to use will depend on the nature of your project:

- For projects using [Jest](http://facebook.github.io/jest/), you do not need an additional test runner - just use Jest to run your tests and generate test coverage. See [Coverage](#coverage) for more information on test coverage with Jest. Jest will default to providing you a "fake" browser environment with [JSDom](https://github.com/tmpvar/jsdom). If you want to test things in a true browser environment it will not be sufficient to rely on Jest alone as this is not currenlty supported.

- For existing projects that already have tests written in mocha/sinon/chai, we recommend [Karma](https://karma-runner.github.io/1.0/index.html). It allows running tests in the browser and integrates well with other tools/frameworks you are likely using, such as bundlers and code coverage tools. See [Coverage](#coverage) for more information on test coverage with Karma.

### Other

There are a variety of other tools you might need depending on your project. Here are some additional recommendations we make:

- [Teaspoon](https://github.com/modeset/teaspoon) is the default test runner for Rails projects but it is no longer recommended due to the lack of code coverage, slowness, and inability to run tests continuously by auto-watching files for changes. However, if you are using Teaspoon and your project is using [sprockets-commoner](https://github.com/Shopify/sprockets-commoner) to transpile your JavaScript, make sure to include [teaspoon-bundle](https://github.com/Shopify/teaspoon-bundle) to generate your test bundle.

- **Rails fixtures**: if you are testing a component that has a non-trivial HTML component, using the actual partial improves the test’s resilience. We recommend using [Magic Lamp](https://github.com/crismali/magic_lamp) if you need access to real partials in a Rails application (if you are writing a JavaScript application, you should be able to import your partials using JavaScript imports).

[↑ scrollTo('#table-of-contents')](#table-of-contents)



## Style

- Our [shared linting configuration](../packages/eslint-plugin-shopify) provides a mocha-/ sinon-/ chai-focused linting configuration which you can use to prevent many common mistakes. To use it, simply extend `plugin:shopify/mocha` in your ESLint configuration (full details can be found on that package’s README). If you are using AVA, we also provide the `plugin:shopify/ava` configuration.

- Use [TDD-style `assert` syntax](http://chaijs.com/api/assert/) over BDD-style `expect` or `should`.

  > Why? This keeps JavaScript tests more in line with our style for writing Ruby tests, which helps everyone contribute to the test codebase more easily.

- Do not rely on test ordering for your tests to run correctly. Imagine each test was going to be run in a separate process, and must therefore entirely setup any preconditions before running.

  > Why? Tests that depend on anything outside of them will inevitably break as more tests are added. Other developers will assume there is no shared context between tests; don’t violate that assumption!

  ```js
  // bring in assert, either per file or in a test helper
  import {assert} from 'chai';

  // bad
  expect(2 === 1).to.be.false;
  (2 === 1).should.be.false;

  // good
  assert.isFalse(2 === 1);
  ```

- Use a dedicated test double library (like `sinon`) instead of rolling your own version of stubs and spies.

  > Why? Spies and stubs provide a consistent interface in case the tests need to be changed, reduce boilerplate so you can focus on the subject under test, can easily be restored to their original versions, and can be tested with more meaningful assertions.

  ```js
  suite('MyComponent', () => {
    // bad
    suite('#myBadFunc', () => {
      let original;
      let calledWith;

      setup(() => {
        spy = function(firstArg) { calledWith = firtArg; }
        let original = subject.myBadFunc;
        subject.myBadFunc = spy;
      });

      teardown(() => {
        subject.myBadFunc = original;
      });

      test('it works', () => {
        subject.doSomething();
        assert.equal(calledWith, 'some arg');
      });
    });

    // good
    suite('#myGoodFunc', () => {
      let spy;

      setup(() => {
        spy = sinon.spy(subject, 'myGoodFunc');
      });

      teardown(() => {
        spy.restore();
      });

      test('it works', () => {
        subject.doSomething();
        assert.calledWith(spy, 'some arg');
      });
    });
  });
  ```

- Move any common setup/ teardown to dedicated functions. Most test frameworks, including mocha and AVA, provide dedicated hooks for these actions — use them if they are available!

  > Why? Setup and teardown in the test itself draws attention from whatever is being tested.

  ```js
  suite('MyComponent', () => {
    // bad
    suite('#myBadFunc', () => {
      test('it does something', () => {
        const subject = new MyComponent();
        sinon.stub(subject, 'myBadFunc').returns(true);
        assert.true(subject.doSomething())
      });

      test('it does something else', () => {
        const subject = new MyComponent();
        sinon.stub(subject, 'myBadFunc').returns(true);
        assert.false(subject.doSomethingElse())
      });
    });

    // good
    suite('#myBadFunc', () => {
      let subject;

      setup(() => {
        subject = new MyComponent();
        sinon.stub(subject, 'myBadFunc').returns(true);
      });

      test('it does something', () => {
        assert.true(subject.doSomething())
      });

      test('it does something else', () => {
        assert.false(subject.doSomethingElse())
      });
    });
  });
  ```

### mocha, sinon and chai

- Use the Mocha-provided [TDD interface](https://mochajs.org/#tdd), which includes the `suite()`, `test()`, `suiteSetup()`, `suiteTeardown()`, `setup()`, and `teardown()` functions.

  ```js
  // bad
  describe('myFunction', () => {
    beforeEach(() => {
      // Execute before each test case.
      doSomeSetup();
    });

    it('is a function', () => {
      assert.isFunction(myFunction);
    });
    
    after(() => {
      // Execute after running tests.
      doSomeTear();
    });
    
  });

  // good
  suite('myFunction', () => {
    setup(() => {
      // Execute before each test case.
      doSomeSetup();
    });

    test('it is a function', () => {
      assert.isFunction(myFunction);
    });
    
    suiteTeardown(() => {
      // Execute after the suite.
      doSomeTear();
    });
  });
  ```

- Do not attach variables under test to the Mocha test context (`this`). Instead, use variables scoped to the nearest `suite`. If using ESNext, always use arrow functions to ensure that you will not be able to rely on context.

  > Why? Using a shared context makes it too easy for tests to "leak" (have some impact on other tests), especially in other test files. It also makes it harder to track down what should be in scope at any given time.

  ```js
  // bad
  suite('MyComponent', function() {
    setup(function() {
      this.subject = new MyComponent();
      this.doSomethingWithComponent = function() {
        return this.subject.doSomething();
      }
    });

    test(function() {
      assert.isFalse(this.doSomethingWithComponent());
    });
  });

  // good
  suite('MyComponent', () => {
    let subject;
    function doSomethingWithComponent() {
      return subject.doSomething();
    }

    setup(() => {
      subject = new MyComponent();
    });

    test(() => {
      assert.isFalse(doSomethingWithComponent());
    });
  });
  ```

- Use the most meaningful assertion for what you are trying to test. For a list of available assertions, make sure to check out [chai’s `assert` docs](http://chaijs.com/api/assert/).

  > Why? More specific assertions have more specific error messages, making it easier for you (and any developer that deals with this code in the future) to track down and fix test failures.

  ```js
  // bad
  assert.isTrue(foo === bar);

  // good
  assert.strictEqual(foo, bar);
  ```

- Use sinon’s custom spy assertions, exposed to chai’s `assert` object.

  ```js
  // somewhere in the test setup...
  // (see http://sinonjs.org/docs/#assert-expose)
  sinon.assert.expose(assert);

  // bad
  assert.isTrue(spy.calledOnce);

  // good
  assert.calledOnce(spy);
  ```

- Always use a test helper that sets up your global test state. This generally includes adding any chai plugins you need, setting up a global sinon sandbox, and any other global hooks you require. It is often useful to expose `sinon`/ `sandbox` and `assert` to the global object in this helper to avoid needing to import them in every test file.

  ```js
  // Here's an example test-helper file

  import chai, {assert} from 'chai';
  import sinon from 'sinon';
  import chaiAsPromised from 'chai-as-promised';

  chai.use(chaiAsPromised);
  sinon.assert.expose(assert);

  // This creates a sandbox that will automatically restore any
  // stubbed properties performed during tests.
  const sandbox = sinon.sandbox.create();

  global.assert = assert;
  global.sinon = sandbox;

  afterEach(() => {
    sandbox.restore();
  });
  ```

- Use mocha’s ability to automatically finish tests when a returned promise is resolved for tests that are asynchronous and depend on promises.

  ```js
  // bad
  test('it resolves with a value', (done) => {
    doSomethingAsync()
      .then((result) => {
        assert.equal(result, 'something');
        done();
      })
      .catch(() => {
        assert(false, 'The promise was rejected :(');
      });
  });

  // good
  test('it resolves with a value', () => {
    return doSomethingAsync()
      .then((result) => {
        assert.equal(result, 'something');
      });
  });

  // also good, if using async/ await
  test('it resolves with a value', async () => {
    const result = await doSomethingAsync();
    assert.equal(result, 'something');
  });
  ```

[↑ scrollTo('#table-of-contents')](#table-of-contents)



## Best practices

- Test the interface, not the implementation. Ideally, you should not expose methods that you wouldn’t want users of your object to call directly, just so you can test them. Try to find the smallest unit of functionality that you would expect someone to use directly, and test that. Your implementation of that functionality should be able to change without affecting consumers or your tests.

- Try to limit the number of assertions made in a single test. For unit tests, you should only be testing one thing at a time (generally, an individual function, either standalone or as a method of an object), and more than one or two assertions is generally indicative that you should split your test into multiple tests.

- In general, each test has three steps: setup, perform the action under test, and assert based on the result.

- If your tests involve stubbing a lot of different objects, it can be a sign of a fragile technical design. Instead of having many external dependencies that are directly brought in by the module, it can be useful to rely on [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) instead. This design pattern can make code easier to understand and test.

- Avoid using mocks.

  > Why? They reverse the typical order of tests (setup, do something, assertion), which makes them harder to read in the context of other tests. Generally, you can achieve the same thing with stubs instead.

  ```js
  // bad
  const mock = sinon.mock(subject).expects('myMethod').once();
  subject.doSomething();
  mock.verify();

  // good
  const stub = sinon.spy(subject, 'myMethod');
  subject.doSomething();
  assert.calledOnce(stub);
  ```

[↑ scrollTo('#table-of-contents')](#table-of-contents)



## Types of testing

The guide above is mostly focused on unit tests, where a "unit" of code (ideally, the smallest version of an API) is tested in isolation from the rest of the system. However, there are a few other types of testing that are important to consider when writing JavaScript: integration testing, functional testing, and UI testing.

### Integration testing

Integration tests ensure that various parts of a system work properly together. It mostly focuses on the design of an overall architecture rather than individual pieces. In writing integration tests, make sure to keep the following in mind:

- Be careful not to do unit tests for the pieces of the system and integration tests for the system as a whole if the pieces are not part of the public interface. That is, if the pieces will only be used in combination as part of this system, the system is the actual unit of tests, and the pieces are an implementation detail that does not need direct tests (they are, of course, indirectly tested by the testing of the public interface to the system).

### Functional testing

In the context of developing web applications, functional testing is more accurately described as browser testing. The goal with these tests is to ensure that a part of the application works according to the specifications you have set. Keep the following in mind for these types of tests:

- Avoid relying on class names for finding components on the page. CSS selectors are meant as styling hooks, and the visual appearance of the page should be able to change without breaking your functional tests. If you absolutely must find a particular DOM node, prefer IDs or `data-` attributes.

- If your test is merely ensuring that a particular visual element is on the page, or that a particular class is on a component, it is more appropriate to use a visual regression test (detailed below), as you are effectively testing the visuals of the page indirectly through the classes you are querying.

### Visual regression testing

Visual regression testing allows you to be confident that changes made to the components that make up your interface did not affect the visual appearance. This kind of testing is particularly useful when refactoring a particular component, as it can ensure that layout and other visual properties did not change without cause.

[↑ scrollTo('#table-of-contents')](#table-of-contents)


## React

If your project is using React, you can find React-specific testing recommendations in the [React styleguide](https://github.com/Shopify/javascript/tree/master/react#testing).

[↑ scrollTo('#table-of-contents')](#table-of-contents)

## Coverage

Generating a test coverage report can help you identify untested code. Coverage reports can be integrated with CodeCov and Circle CI to ensure team-specific coverage targets are met with every checkin. See the [CodeCov Integration Guide](https://vault.shopify.com/Codecov-integration) for more information.

Here are some recommended ways to generate coverage:

- **Karma**: You can use the [Karma Istanbul Reporter plugin](https://github.com/mattlewis92/karma-coverage-istanbul-reporter) to automatically generate coverage when tests are running.
- **Jest**: You can use the built in coverage reporter by adding the `--coverage` argument when running tests. See (documentation)[https://facebook.github.io/jest/docs/en/cli.html#coverage].
- **Other**: We recommend using [nyc](https://github.com/istanbuljs/nyc) as a standalone command line tool to generate coverage.

It is best practice to generate coverage for all relevant JS files as opposed to only tested files. This will ensure files are not inadvertently missed. There are a couple of ways to generate coverage for all files:
- **Karma**: Add the [`includeAllSources`](https://github.com/karma-runner/karma-coverage/blob/master/docs/configuration.md#includeallsources) option to your Karma `coverageReporter` config:
```js
files: [
  'app/assets/javascripts/**/*.js', // Regexp matching relevant files
],
coverageReporter: { 
  includeAllSources: true
}
```
- **Jest**: If you are using [sewing-kit](https://github.com/Shopify/sewing-kit), coverage for all files should be enabled automatically. If you are not using sewing-kit, you might need to configure the [`collectCoverageFrom`](http://facebook.github.io/jest/docs/en/configuration.html#collectcoveragefrom-array) option:
```js
{
  "collectCoverageFrom": [
    '**/*.{js,jsx}', // Glob pattern for matching/excluding files
    "!**/node_modules/**",
    "!**/vendor/**"
  ]
}
```
- **Other**: Alternatively, you can use a single test entry file that loads all relevant JS modules. This is not recommended since the entry file would have to be updated whenever a new module is added.

If you are writing code in ES6 and are bundling your code, it is best to generate coverage on non-bundled and non-transpiled code for better understandability. This should already be supported by default if you are using Webpack + Babel to bundle your code. If you are using Browserify for bundling, you might need to use the `es2016` preset along with the [browserify-istanbul plugin](https://github.com/devongovett/browserify-istanbul) to generate coverage for non-bundled and non-transpiled code. For example: 
```js
browserify: { 
  extensions: ['.js'],
  transform: [
    ['babelify',
    { presets: ['es2016'], ignore: /node_modules/,
      plugins: ['istanbul', { exclude: ['**/test/**','**/node_modules/**']
    }]
  ]
}
```

[↑ scrollTo('#table-of-contents')](#table-of-contents)
