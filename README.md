# Vue + TypeScript Cookbook

If you're like me, you're busy and don't have a lot of time to fight with tools when they give you trouble—you just want to code and get stuff done! I ran into a lot of roadblocks learning Vue + TS, and after overcoming the last of the hurdles just recently, I decided to create this cookbook to help others who might have questions about how to hit the ground running with Vue + TS.

*NOTE: This cookbook assumes you have a basic knowledge of TypeScript.*

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Initial set-up](#initial-set-up)
- [What are the basic need-to-knows?](#what-are-the-basic-need-to-knows)
- [I'm using Vuex `mapState` or `mapGetters`, and TypeScript is saying the mapped state/getters don't exist on `this`.](#im-using-vuex-mapstate-or-mapgetters-and-typescript-is-saying-the-mapped-stategetters-dont-exist-on-this)
- [How do I make a function outside the scope of the Vue component have the correct `this` context?](#how-do-i-make-a-function-outside-the-scope-of-the-vue-component-have-the-correct-this-context)
- [How do I annotate my `$refs` to avoid type warnings/errors?](#how-do-i-annotate-my-refs-to-avoid-type-warningserrors)
- [Why am I losing type information when I import variables from other files/modules?](#why-am-i-losing-type-information-when-i-import-variables-from-other-filesmodules)
- [Conclusion](#conclusion)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Initial set-up

Setup is a breeze with the new vue-cli 3.x. Just create a new project:

    $ vue create myapp

When prompted, choose to manually select features, and make sure TypeScript is chosen from the list.

    Vue CLI v3.0.0-rc.7
    ? Please pick a preset: 
      default (babel, eslint) 
    ❯ Manually select features

For the purposes of this cookbook, we'll opt to not use class-style components and stick with the standard functional-based format.

## What are the basic need-to-knows?

The main thing you'll need to know is that your script blocks will change. Without TypeScript, they look something like this:

```html
<script>
export default Vue {
  data() {
    return {
      name: '',
      locations: [],
    };
  },

  methods: {
    test(name) {
      return name + '!';
    },
  },
}
</script>
```

With TypeScript, they'll look like this:

```html
<script lang="ts">
import Vue from 'vue';

export default Vue.extend({
  data() {
    return {
      name: '',
      locations: [] as string[],
    };
  },

  methods: {
    test(name: string): number {
      this.locations.push(name);
      return this.locations.length;
    },
  },
});
</script>
```

## I'm using Vuex `mapState` or `mapGetters`, and TypeScript is saying the mapped state/getters don't exist on `this`.

This is a [known bug](https://github.com/vuejs/vuex/issues/1353) in Vuex, and there's an outstanding [PR](https://github.com/vuejs/vuex/pull/1121). The bug only presents itself when using an object spread and including one or more computed properties:

```js
computed: {
  ...mapState(/*...*/),
  someOtherProp() {}
}
```

To avoid this, either avoid the spread:

```js
computed: mapState(/*...*/)
```

or define an interface for the Vuex bindings and apply it to your component:

```js
import Vue, { VueConstructor } from 'vue';
import { mapState } from 'vuex';
import { MyState } from '@/store';

interface VuexBindings {
  stateVar: string;
}

export default (Vue as VueConstructor<Vue & VuexBindings>).extend({
  data() {
    return {
      name: '',
      locations: [] as string[],
    };
  },

  computed: {
    ...mapState({
      stateVar: (state: MyState) => state.stateVar,
    }),
    nothing(): string {
      return 'test';
    },
  },

  methods: {
    test(name: string): number {
      console.log(this.stateVar); // no more TS error
      this.locations.push(name);
      return this.locations.length;
    },
  },
});
```

## How do I make a function outside the scope of the Vue component have the correct `this` context?

You might be thinking, "why not just make a method within the Vue component?" The thing is, TypeScript is picky about referring to computed properties, methods, and data variables from within the `data` method, even if you're referring to `this` within a callback handler which is perfectly valid. So you'll have to put functions outside the scope of the Vue component. Here's what that looks like:

```ts
import Vue from 'vue';

type ValidatorFunc = (this: InstanceType<typeof HelloWorld>) => Function;

const validator: ValidatorFunc = function() {
  return () => {
    if (this.name === 'bad') {
      // do something
    }
  };
};

const HelloWorld = Vue.extend({
  data() {
    return {
      name: '',
      locations: [] as string[],
      somethingHandler: {
        trigger: 'blur',
        handler: validator.bind(this),
      },
    };
  },
});

export default HelloWorld;
```

<!-- ## I'm using vue-i18n and getting type errors because `TranslateResult` can't be set on a variable declared as a string.

You could simply do something like this to resolve that:

```ts
const myString = this.$t('general.error') as string;
```

But having to add `as string` can become tedious. Instead, we'll override...

meh, this doesn't seem possible. Will update if there's a workaround.
-->

## How do I annotate my `$refs` to avoid type warnings/errors?

Some UI libraries will let you slap refs on their components to make direct method calls. Here's how you give type awareness to those refs.

First, an example of a quick one-off approach:

```ts
export default Vue.extend({
  methods: {
    test() {
      (this.$refs.dataTable as ElTable).clearSelection();
    },
  },
});
```

If you refer to the same ref many times, or you have several refs, it's easier to create an interface:

```ts
import Vue, { VueConstructor } from 'vue';
import { ElTable } from 'element-ui/types/table';
import GoogleMap from '@/components/shared/GoogleMap.vue';

// With this, you won't have to use "as" everywhere to cast the refs
interface Refs {
  $refs: {
    name: HTMLInputElement
    dataTable: ElTable,
    map: InstanceType<typeof GoogleMap>;
  }
}

export default (Vue as VueConstructor<Vue & Refs>).extend({
  ...
})
```

## Why am I losing type information when I import variables from other files/modules?

When using TypeScript, if you wish to preserve full type information, never use `export default`. Export your variable like this instead:

```ts
export const myThing: SomeType = {
  // ...
}
```

Then, of course, import it as you'd expect in ES6:

```ts
import { myThing } from '@/some/file';
```

You'll notice now, when you mouse over the variable name or use it in your code, it will have full type information attached to it.

## Conclusion

If something's been bugging you with Vue + TypeScript, please open an issue to discuss having a recipe added!
