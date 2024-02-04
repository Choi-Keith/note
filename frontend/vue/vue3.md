
## Vue2到Vue3的一个变化
vue2不再使用new Vue({el: '#app'})的方式实例化一个实例，而是采用createApp({})的方式创建， 并且createApp还移除了filter、$on、$off、$set、$delete等API

## 插槽


```vue
<WelcomeItem>
    <template #icon>
        <DocumentationIcon />
    </template>
    <template #heading>Documentation</template>

    Vue’s
    <a href="https://vuejs.org/" target="_blank" rel="noopener">official documentation</a>
    provides you with all information you need to get started.
</WelcomeItem>
------------------------------------------
<template>
  <div class="item">
    <i>
        // 具名插槽
      <slot name="icon"></slot>
    </i>
    <div class="details">
      <h3>
        <slot name="heading"></slot>
      </h3>
      // 匿名插槽
      <slot></slot>
    </div>
  </div>
</template>
```

## watch和watchEffect的区别


## watch和computed的区别


## 父组件和子组件的生命周期


## Vue3相对于Vue2有哪些改变的地方