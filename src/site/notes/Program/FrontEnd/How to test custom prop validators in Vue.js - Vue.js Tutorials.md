---
{"dg-publish":true,"permalink":"/Program/FrontEnd/How to test custom prop validators in Vue.js - Vue.js Tutorials/","noteIcon":""}
---

In this article, I will show you how you can test props and custom prop validators in a simple and performant way, without spying on the console error.

## Testing Vue.js component props

In this tutorial, we're going to use Jest and Vue Test Utils to test a `NotificationMessage` component. The component has two props:

A `message` prop, which uses standard [prop validation](https://vuejs.org/v2/guide/components-props.html#Prop-Validation)

```jsx
props: {
  message: {
    required: true,
    type: String
  }
}
```

And a `type` prop, which uses custom prop validation. It must be a string, and any of these values: `info` `error` `success`

```jsx
props: {
  message: { ... },
    type: {
      required: false,
      type: String,
      default: 'info',
      validator: (value) => [
            'info', 
            'error', 
        '  success'
        ].includes(value.toLowerCase())
  }
}
```

The type dictates the design of our notification message. When rendered, it looks like this:  
![](https://vueschool.io/articles/wp-content/uploads/2020/05/testing_vuejs_prop_validators_vueschool-1024x472.png)

If we try to mount our component with an invalid type, Vue will warn us and show a warning like this in the console.

![](https://vueschool.io/articles/wp-content/uploads/2020/05/testing_vuejs_prop_validators_console_error_vueschool-1024x209.png)

> Want to learn how to make [custom Vue.js prop validators](https://vueschool.io/lessons/vuejs-custom-prop-validation)? [Watch our free lesson](https://vueschool.io/lessons/vuejs-custom-prop-validation)

## How to test custom prop validators in Vue.js

I'll show you three ways of testing your custom prop validators.

1.  Spying on the console error (not recommended)
2.  Using the prop validator through the vue instance (better, not recommended)
3.  Using the prop validator without mounting the component (recommended, best performance)

## 1. Test custom prop validators using a spy (not recommended)

A common solution to testing custom prop validators is to spy on the `console` object for the warning output.

```jsx
// I advise against using this approach

it('does not throw console error when info type', () => {
  const spy = jest.spyOn(console, 'error')

  mount(NotificationMessage, {
    propsData: {
      type: 'info',
      message: 'I like turtles'
    }
  })

  expect(spy).not.toHaveBeenCalledWith(expect.stringContaining('[Vue warn]: Invalid prop: custom validator check failed for prop "type".'))
})
```

This technically works, but it is a poor, low-quality test. If we remove the validator completely from our component, the test will still pass, because Vue doesn't throw a warning.

**There is a much better way to testing your custom prop validators.**

## 2. Test by using the prop validator through the Vue instance (not recommended)

If you think about it, our prop validator is really just a simple function that accepts an argument and returns a boolean.

```jsx
type: {
  // ...
  validator: (value) => [
        'info', 
        'error', 
        'success'
    ].includes(value.toLowerCase())
}
```

We can use and invoke the validator method directly in our tests!

We can access all of the Vue component's options through the `$options` property of the instance. And with Vue Test Utils, we can access the instance through `wrapper.vm` after it's mounted.

```jsx
// I advise against using this approach for performance reasons

it('passes with info type', () => {
  const wrapper = mount(NotificationMessage, {
    propsData: {
      message: 'I like turtles!'
    }
  })
  // Extract the validator
  const validator = wrapper.vm.$options.props.type.validator
  // invoke the validator
  expect(validator('info')).toBe(true)
})
```

This is a much better approach to testing custom prop validators than spying on the console warning.

-   We're testing the code directly in isolation.
-   If we remove the validator from our component, the test will fail.
-   If the warning message changes in a future update, it won't break our tests.

However, there is one thing we can do to significantly improve the performance of this approach.

## 3. Test by using the prop validator without mounting the component (recommended)

We can significantly improve the performance of our test suite by not mounting the component.

Our components are compiled when we run our tests, which means that we have access to the props object (and the validator) **without mounting the component.** This means we can write less code and get improved performance, for free.

This is how I would test a Vue.js prop validator without mounting the component:

```jsx
it('it passes with info type', () => {
  const validator = NotificationMessage.props.type.validator
    expect(validator('info')).toBe(true))
  expect(validator('batman')).toBe(false)
})
```

Isn't that cool? So to test all of the cases of our validator, I would do the following:

```jsx
it('it accepts valid type props', function () {
  const validTypes = ['info', 'error', 'success']
  const validator = NotificationMessage.props.type.validator
  validTypes.forEach((valid) => expect(validator(valid)).toBe(true))
  expect(validator('batman')).toBe(false)
})
```

There you go, a simple, reliable way of testing your custom prop validators in Vue.js!

## Testing standard prop validation in Vue.js

We can use the same performant approach to test any component prop.

When we test standard prop validation, it's enough to assert that the prop object is configured correctly. We can use Jest's `toMatchObject` matcher to make sure a `message` prop is defined, required, and must be a string.

```jsx
it('message prop is required and must be a string', function () {
  expect(NotificationMessage.props).toMatchObject({
    message: {
      type: String,
      required: true
    }
  })
})
```

This is a solid test case. If we remove or alter the prop configuration in the `NotificationMessage` component, our test will catch it and fail.

Want to learn more about testing Vue.js and JavaScript?

I hope you've learned something and feel inspired to back your custom prop validators with tests. I've prepared a [companion repository](https://github.com/rahaug/testing-vuejs-custom-prop-validators) for this article if you want to play around with the examples.

If you want to learn more about testing in JavaScript and Vue.js applications, I recommend that you check out our testing courses:

-   [JavaScript Testing Fundamentals](https://vueschool.io/courses/javascript-testing-fundamentals)
-   [Test with Jest](https://vueschool.io/courses/test-with-jest)
-   [Testing Vue.js Components](https://vueschool.io/courses/learn-how-to-test-vuejs-components)

We also do [in-person training](https://vueschool.io/workshops) and [remote Vue.js workshops](https://vueschool.io/workshops), if you want to elevate your entire team or yourself.

 [https://vueschool.io/articles/vuejs-tutorials/how-to-test-custom-prop-validators-in-vuejs/](https://vueschool.io/articles/vuejs-tutorials/how-to-test-custom-prop-validators-in-vuejs/)
