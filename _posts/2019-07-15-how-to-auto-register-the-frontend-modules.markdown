---
layout: post
title:  "How to auto-register the frontend modules?"
date:   2019-07-15 00:00:00
categories: Javascript
---

Sometimes, we have a requirement of the frontend modules' auto-registration, that because we have to split the frontend team into multiple module teams when the frontend is getting bigger and bigger, and they only care about their modules.

Before we start, have to list the aspects for an auto-registrable module.

1. Should contain a description file which contains some information like id, name, sub-route table, and so on.
1. Should put the module files under a specific folder.
1. Read the module's configuration file then register the menu, route, etc. against the configuration.

### How to read the modules' configuration files?

Can use the `require.context` to list the configuration files. like this:  
```javascript
const files = require.context('.', true, /^\.\/\w+\/index\.js$/)
```

Then, iterate `files` to register the modules one by one.
```javascript
files.keys().forEach((key) => {
  const name = /^\.\/(\w+)\/index\.js$/.exec(key)[1]
  const m = files(key).default
  modules[name] = m
  // TODO module configuration has been loaded, then need to register the menu items and combine the route tables.
})
```

For the module's configuration, should contain any information that belongs to the independence module.  
For example:   
```javascript
const Code  = () => import('./Code')

export default {
  menu () {
    return {
      text: '编号管理',
      icon: 'code',
      link: '/'
    }
    // It's an example to show us we can load the menu from the internet.
    // return new Promise(function(resolve) {
    //   setTimeout(function() {
    //     resolve({
    //       text: '编号管理',
    //       icon: 'code',
    //       link: '/'
    //     });
    //   }, 3000);
    // });
  },
  route: {
    component: Code
  }
}
```

After the modules' route table has been assembed, can register it into the global route table.
```javascript
import Vue from 'vue'
import Router from 'vue-router'
import X404 from './views/404'
import { routes } from './modules'

Vue.use(Router)

export default new Router({
  mode: 'history',
  base: process.env.BASE_URL,
  routes: [
    // Global routes, will override the modules' configuration.
    { path: '/404', name: '404', component: X404 },
    // This is a fallback catch.
    { path: '*', component: X404 },
    ...routes
  ]
})
```

For the full example, check the repository [example-auto-registerable-web-modules](https://github.com/zhangyanwei/example-auto-registerable-web-modules).
