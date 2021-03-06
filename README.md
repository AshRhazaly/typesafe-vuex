# typesafe-vuex [![Build Status](https://travis-ci.org/jackkoppa/typesafe-vuex.svg?branch=master)](https://travis-ci.org/jackkoppa/typesafe-vuex) [![Coverage Status](https://coveralls.io/repos/github/jackkoppa/typesafe-vuex/badge.svg)](https://coveralls.io/github/jackkoppa/typesafe-vuex) [![npm version](https://badge.fury.io/js/typesafe-vuex.svg)](https://badge.fury.io/js/typesafe-vuex)

A simple way to get static typing, static code analysis and intellisense with Vuex library - maintained fork of [`vuex-typescript`](https://github.com/istrib/vuex-typescript)

The above is a great project by [@istrib](https://github.com/istrib), that seems to be unmaintained as of 11/13/2017 ([last commit](https://github.com/istrib/vuex-typescript/commit/105d21977d0f870910d86f7706c06ee8bd3bded9)). This project is intended to provide a maintained version, addressing some issues that have arisen since then.


![](doc/Intellisense.png)

We get full end-to-end compile-time safety and code navigability:
* No string literals or constants for action/mutation/getter names
* No action/mutation/getter misuse by providing wrong payload type
* Intellisense giving unambiguous hints on what type of payload or getter arguments is expected
* Refactoring made easy
* Supports vuex with and without modules (though use of modules and namespaces seems to produce better structured code).

## The idea

This library does not change the way how vuex handlers are defined (in particular it does not make you 
use classes (though it does not stop you, either).

The library changes the way how you *call* the store, once you have its instance: you **don’t use store’s 
compile-time-unsafe methods** but you use **strongly typed *accessors*** instead. This approach is remotely similar to the pattern of 
[Redux Action Creators](http://redux.js.org/docs/basics/Actions.html#action-creators), though much less boilerplate is needed thanks 
to higher-order functions provided by this library which do the dull work.

## Example 

Full working example available [here](https://github.com/jackkoppa/typesafe-vuex/tree/master/src/tests/withModules/store/basket/basket.ts). Excerpt:

```js
import { ActionContext, Store } from "vuex";
import { getStoreAccessors } from "typesafe-vuex";
import { State as RootState } from "../state";
import { BasketState, Product, ProductInBasket } from "./basketState";

// This part is a vanilla Vuex module, nothing fancy:

type BasketContext = ActionContext<BasketState, RootState>;

export const basket = {
    namespaced: true,

    state: {
        items: [],
        totalAmount: 0,
    },

    getters: {
        getProductNames(state: BasketState) {
            return state.items.map((item) => item.product.name);
        },
		...
    },

    mutations: {
        appendItem(state: BasketState, item: { product: Product; atTheEnd: boolean }) {
            state.items.push({ product: item.product, isSelected: false });
        },
		...
    },

    actions: {
        async updateTotalAmount(context: BasketContext, discount: number): Promise<void> {
            const totalBeforeDiscount = readTotalAmountWithoutDiscount(context);
            const totalAfterDiscount = await callServer(totalBeforeDiscount, discount);
            commitSetTotalAmount(context, totalAfterDiscount);
        },
		...
    },
};

// This is where the typesafe-vuex specific stuff begins:
//
// We want to expose static functions which will call get, dispatch or commit method
// on a store instance taking correct type of payload (or getter signature).
// Instead of writing these "store accessor" functions by hand, we use set of higher-order
// functions provided by typesafe-vuex. These functions will produce statically typed
// functions which we want. Note that no type annotation is required at this point.
// Types of arguments are inferred from signature of vanilla vuex handlers defined above:

const { commit, read, dispatch } =
     getStoreAccessors<BasketState, RootState>("basket"); // We pass namespace here, if we make the module namespaced: true.

export const readProductNames = read(basket.getters.getProductNames);
export const dispatchUpdateTotalAmount = dispatch(basket.actions.updateTotalAmount);
export const commitAppendItem = commit(basket.mutations.appendItem);

```

And then in your Vue component:

```js
import * as basket from "./store/basket"; // Or better import specific accessor to be explicit about what you use
...

getterResult = basket.readProductNames(this.$store); // This returns Product[] 
basket.dispatchUpdateTotalAmount(this.$store, 0.5); // This accepts number (discount) - you'd normally use an object as arguments. Returns promise.
basket.commitAppendItem(this.$store, newItem); // This will give compilation error if you don't pass { product: Product; atTheEnd: boolean } in
```

## Vuex version compatibility

This library has explicit dependency on Vuex.
A new version of typesafe-vuex is released following each major release of Vuex. This way breaking API changes introduced into Vuex are guaranteed to be followed up and tested in typesafe-vuex. Currently, this library only supports Vuex 3.

## Functions or objects

This lib is deliberately designed with functions rather than classes. This does not stop you from grouping accessors into objects. 
Note however that this makes little sense as the accessors are loosely or not related to each other.
Importing and using functions rather than objects makes it explicit which accessor you actually use rather than stating which accessor in an object you may be using.

If you wish to define your vuex handlers as class members then you must decorate these methods with `@Handler`
decorator exported from this library as shown in [this test](https://github.com/jackkoppa/typesafe-vuex/tree/master/src/tests/withModules/store/system/system.ts).

## Vue CLI

If you are using Vue CLI, when it compiles for production, it will minify the code.

But typesafe-vuex depends on the names of the functions to be preserved.

We provide a helper method to configure Webpack for you, so that function names are preserved. This should solve issues you see with Vuex when running `vue-cli-service --mode production`, and when running your project in IE 11.

**To use in Vue CLI 3**

```javascript
// vue.config.js
const { preserveFunctionNamesWithTerser } = require('typesafe-vuex/helpers')

module.exports = {
    // other Vue CLI options, like devServer

    configureWebpack: (config) => {
        preserveFunctionNamesWithTerser(config)
    }
}
```

**References:**

* [typesafe-vuex, Issue #4](https://github.com/jackkoppa/typesafe-vuex/issues/4)
* [mrcrowl/vuex-typex](https://github.com/mrcrowl/vuex-typex/issues/22#issuecomment-475524894)
* [istrib/vuex-typescript](https://github.com/istrib/vuex-typescript/issues/13#issuecomment-475524502)

## Custom compilers

If you are using Webpack directly or any other custom compilation strategy, make sure that the names of the functions for Vuex are preserved. You can use the helper method above, `preserveFunctionNamesWithTerser`, or set your own configuration to ensure that function names remain.

## More

https://github.com/mrcrowl/vuex-typex also uses higher-order functions but takes a few steps more: there the store is implicitly built while defining accessors for each handler (in contrast here we create store with standard Vuex options and then wrap handlers into accessors). It is also not required to pass $store as argument to accessors. Definitely worth checking out.

## Contributing

```
npm run build
npm run build-watch

npm test
npm run test-watch
npm run test-debug
npm run coverage
```
