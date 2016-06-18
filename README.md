Possible CSS modules style guide
================================

* Only use one class name per element, and define these component-specific classes in the same file as the React component. (If you need to conditionally change only some styles, still only use one class at a time, but share the common styles between the classes using "composes:".)
  - Reason: Keeps the styles for the component completely separate from the markup, but still scoped to the component.
* For component-specific classes, only use "composes:" with shared classes to define the styles for that component, don't actually use any style definitions in the same as the React component. (The shared classes with their style definitions could all be kept in various modules in a shared/ folder, like shared/layout.js, shared/colors.js, etc.)
  - Reason: Really just to force yourself to reuse styles as much as possible. Exceptions can be made if you're really sure that the style for that component won't be used anywhere else.
* For component-specific classes, always use semantic class names (e.g. .container, .button, etc.). However, for shared classes that are meant to be composed, it's okay to use very specific names, like "bright-red", "wide-horizontal-padding", etc.
  - Reason: the idea is that class composition should theoretically be able to replace variables in CSS. Instead of having a $primary-color variable, you can have a .primary-color class that has nothing but "color: #xxxxxx;" in it. In order to be composable, most classes should only have one or two style definitions (however, you can make higher-order classes that are just compositions of other shared classes).


Usage with Aphrodite
====================

There are two basic ways you could use this strict-composition style of CSS modules with Aphrodite. You can either compose the styles using `className={css(styles.foo, styles.bar)}`, or `className={[css(styles.foo), css(styles.bar)].join(' ')}`.

The first way actually merges the styles in each of the classes in the order they're given and creates a new class name. For example, if styles looks like:

```
const styles = StyleSheet.create({
  foo: {
    color: 'blue',
  },
  bar: {
    backgroundColor: 'gray',
    color: 'red',
  }
});
```

then `css(styles.foo, styles.bar)` would generate CSS similar to:

```
.foo_bar {
  color: red;
  background-color: gray;
}
```

and `css(styles.bar, styles.foo)` would generate something like:

```
.bar_foo {
  color: blue;
  background-color: gray;
}
```

Then, the singular class would be applied to the element (e.g. `class="foo_bar"`).

This method is fine, but it could potentially insert a lot of duplicated CSS styles into the style tag during runtime, because if you are using the class composition style properly then you'd end up creating a ton of class names that are permutations of your various shared composable classes. For example:

```
.primary-color_light-background_large-text { ... }
.primary-color_dark-background_large-text { ... }
.secondary-color_dark-background_small-text { ... }
...
```

I have no idea if this would actually cause any noticeable performance issues, but that might be something to measure. It could potentially use up a lot of memory on mobile devices, though.

One other problem with this method is that the order you specify the shared classes when you compose them could matter if you're not careful to make sure that two of the classes you are composing do not assign different values to the same CSS property (e.g. .button-primary and .button-large apply different values to font-size). This might not seem like a big deal, but it's basically just a less bad version of the global namespacing conflicts problem that we're trying to avoid in the first place. Plus, it feels very unintuitive to me for the order of composition to have side effects, and I'm pretty sure I'd forget about it eventually and make a mistake.

The other method of composing the classes is similar to how the "composes:" keyword works in CSS modules. `className={[css(styles.foo), css(styles.bar)].join(' ')}` would generate the CSS for the classes normally and assign both classes to the element, like `class="foo bar"`.

However, one disadvantage of this method is that if two of the composed classes assign different values to the same CSS property, which value gets applied is undefined (it just depends on which class happened to get inserted into the style tag last). This is just a slight variation of the same problem as the other method.

Conflict checking
-----------------

In order to avoid this problem, I think it would be best to make a small wrapper around Aphrodite that checks for conflicts for you and throws an error if there are ever any classes that get composed together that share CSS properties. Because the behavior is either unintuitive (IMHO) or undefined when composed classes share properties, it's probably best to force the programmer to reorganize their classes to be properly composable. (Of course, this can be disabled in production both for performance and to make sure the user doesn't get an error telling them to reorganize their classes...)

This might look something like this:

```
render() {
  const styles = composeStyles({
    container: [layout.centered, layout.wide, background.primaryGradient, color.secondary, border.roundedInset],
    button: [background.secondary, background.secondaryHover, color.primary, layout.largeTopMargin],
  });

  return <div className={styles.container}>
    <a className={styles.button}>Button</a>
  </div>;
}
```

(Side note: it's a little bit weird for the :hover style to be in a separate class, I'm not sure if that's a good idea or not. You don't have to do that with this method, though, I'm just trying to be super compose-y.)

This would go through the definitions of all of the composed styles and make sure that they share no properties (and throw an error if they do). If the checks pass, then it would be equivalent to just doing `container: [css(layout.centered), css(layout.wide), ...].join(' ')`.

Higher-order composition
------------------------

In order to do higher-order composition, you could just use arrays of composable classes. For example, if you made a `components` module containing:

```
export default {
  baseContainer: [layout.centered, layout.wide, border.roundedInset],
  baseButton: [background.secondary, background.secondaryHover, color.primary],
};
```

Then it could be used like this:

```
  const styles = composeStyles({
    container: components.baseContainer.concat([background.primaryGradient, color.secondary]),
    button: components.baseButton.concat([layout.largeTopMargin]),
  });
```

And in reality composeStyles would probably just recursively flatten the arrays given to it so that you can use higher-order classes the same way as normal ones:

```
  const styles = composeStyles({
    container: [components.baseContainer, background.primaryGradient, color.secondary],
    button: [components.baseButton, layout.largeTopMargin]),
  });
```

"composes:" in Aphrodite
------------------------

It might be even better to create a complete wrapper around Aphrodite's API (or just a fork of Aphrodite) that adds a 'composes:' keyword just like CSS modules:

`App.js:`
```
import React, { Component } from 'react';
import { StyleSheet, css } from 'aphrodite-composes';
import { layout, background, color, components } from './styles';

class App extends Component {
    render() {
        const styles = {
            container: css(layout.centered, layout.wide, background.primaryGradient, color.secondary, border.roundedInset),
            button: css(background.secondary, background.secondaryHover, color.primary, layout.largeTopMargin),
        };

        return <div className={styles.container}>
            <p>This is a button using only shared styles:</p>
            <a className={styles.button}>Button</a>
            
            <p>This is a button with some styles that are only used on this component and nowhere else:</p>
            <a className={styles.specialButton}>Special Button</a>
        </div>;
    }
}

const styles = StyleSheet.create({
    container: { composes: [components.baseContainer, background.primaryGradient, color.secondary] }
    button: { composes: [components.baseButton, layout.largeTopMargin] },
    specialButton: {
        composes: [components.baseButton],
        color: #beefee,
    },
});
```

`styles.js:`
```
import { StyleSheet } from 'aphrodite-composes';

const layout = StyleSheet.create({
    block: {
        display: 'block',
    },
    centered: {
        // Strings could be used to reference styles within the same stylesheet.
        composes: ['block'],
        marginLeft: 'auto',
        marginRight: 'auto',
    },
    ...
});

const color = StyleSheet.create({
    ...
});

const background = StyleSheet.create({
    ...
});

const border = StyleSheet.create({
    ...
});

const components = StyleSheet.create({
    baseContainer: { composes: [layout.centered, layout.wide, border.roundedInset] },
    baseButton: { composes: [background.secondary, background.secondaryHover, color.primary] },
});

export default { layout, color, background, border, components };
```
