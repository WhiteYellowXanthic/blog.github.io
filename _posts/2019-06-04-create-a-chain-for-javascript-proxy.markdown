---
layout: post
title:  "Create a chain for Javascript proxy"
date:   2019-06-04 08:00:00
categories: Javascript
---

In `Java`, we are using the annotations to enhance the methods, because we have chances to read the annotations and do tricks before invoking the methods.

**How to do the same thing in pure Javascript?**

We know `Javascript` is a dynamic programming language, and it's highly flexible, that means we can modify the functions and instances at runtime.
Yup, we can do that, but it's dangerous! Can we find a safe way to handle it?

The answer is Yes! Since `ECMAScript 2015 (ES6)`, introduced a new built-in object, that is [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy). With Proxy, it's able to define custom behaviors. 

The last thing is, how to annotate the functions in Javascript? There is no compiler for it, also no grammar sugars, but I have a simple method to describe the additional features for Javascript functions, that is by using a pattern on the function name.

```javascript
// It's same as in Java, enhance the class or instance to make it more powerful.
const proxiedObject = ProxyChain.proxy(obj, { FeatureProxy, AnotherFeatureProxy });

// Then the enhanced object will contain the feature methods.
proxiedObject
  .__withFeature(options)
  .__withAnotherFeature(options)
  .realFunction()
```

Here is my code snippet, I have defined two class ProxyChain and ProxyChainInvoker, they are respectively for defining the proxy chain and defining the invoker of the proxy chain.

```javascript
class ProxyChainInvoker {
  constructor(plugins, context) {
    this.plugins = plugins;
    this.context = context;
    this.index = 0;
  }

  doNext(target, that, args) {
    if (this.index < this.plugins.length) {
      const [name, plugin] = this.plugins[this.index++];
      const func = plugin.__apply;
      return func.call(plugin, this.context, () => {
        return this.doNext(target, that, args);
      });
    }
    return target.apply(that, args);
  }
}

class ProxyChain {
  constructor(target, plugins) {
    this.target = target;
    this.plugins = this.initPlugins(plugins);
  }

  initPlugins(plugins) {
    const initedPlugins = new Array();
    for (const [name, plugin] of Object.entries(plugins)) {
      initedPlugins.push([
        name,
        typeof plugin == "function" ? new plugin() : plugin
      ]);
    }
    return initedPlugins;
  }

  findConfiguers(prop) {
    return this.plugins.filter(([name, plugin]) => {
      const configuer = plugin[prop];
      return !!configuer && typeof configuer == "function";
    });
  }

  newInvoker(context) {
    return new ProxyChainInvoker(this.plugins, context);
  }
  
  handlerTemplate(context = null) {
    const chain = this;
    return {
      get(target, prop, receiver) {
        if (prop == "__context") {
          return context;
        }
        const configuers = chain.findConfiguers(prop);
        const c = receiver.__context || {};
        if (configuers.length > 0) {
          return (...args) => {
            // call all configuer functions if exists
            for (const [name, plugin] of configuers) {
              plugin[prop].call(plugin, c, ...args);
            }
            return new Proxy(target, chain.handlerTemplate(c));
          };
        }

        const attr = target[prop];
        if (typeof attr === "function") {
          return new Proxy(attr, {
            apply(target, that, args) {
              // invoke the target in a new chain.
              // return target.apply(that, args);
              return chain.newInvoker(c).doNext(target, that, args);
            }
          });
        }
        return Reflect.get(...arguments);
      }
    };
  }

  static proxy(target, plugins) {
    const chain = new ProxyChain(target, plugins);
    return new Proxy(target, chain.handlerTemplate());
  }
}
```
For the above code snippet, there are two important functions I should describe.

* **ProxyChainInvoker.doNext**  
  In this method will find the `__apply` method from the plugin, we can do the additional tricks in this method. 
* **ProxyChain.handlerTemplate**  
  In this method will find the same name function from the plugins, that means we should define not conflict functions in the plugin for plugin configuration.

Next, it's time to write the proxy plugins.

```javascript
class CacheProxy {
  constructor() {
    this.cache = new Map();
  }

  __withAsyncCache(context, key) {
    context.key = key;
  }

  __apply(context, next) {
    let key = context.key;
    let cache = this.cache;
    if (cache.has(key)) {
      return new Promise(resolve => {
        setTimeout(() => {
          log("from cache");
          resolve(cache.get(key));
        });
      });
    } else {
      return next()
        .then(resp => {
          log("from remote, then update cache");
          cache.set(key, resp);
          return resp;
        })
        .catch(reason => {
          log(reason);
        });
    }
  }
}
```
This plugin can be used to cache the promise result.
You can check this example on **[codepen](https://codepen.io/zhangyanwei/pen/KrXrbP)**.