---
layout: post
title:  "Redux和React的关系"
date:   2019-06-24 22:55:00 +0800
categories: ReactJS
---
### Redux是什么 
> Redux is a predictable state container for JavaScript apps. 官方说法 
 
1. Redux 是一个可预测的JS APP的状态容器，独立于React 以及其他框架或者库。
2. Redux 是一个唯一的状态仓库，也就是在一个APP里面只能有一个
3. 仓库里面的状态值只读
4. 修改状态值只能通过JS 方法来修改，由这个Action来告诉Redux 修改哪一个值以及怎么修改， 这个Action实际经常用用一个function来获得，由两个数据：**不同类型的值 Action 需要指明不同的类型  以及  这个类型的值**
5. 知道了怎么修改 ，也有需要有人来执行这个操作，这个人就是Reducer根据和action约定的不同的规则来做不同的修改
6. 已经知道了怎么做，那么到底怎么实施的呢？```var store = Redux.createStore(favoriteColors);```用reducer作为一个初始化参数才创建仓库，以及有个入口方法来接收action
```
store.dispatch(addColor("blue"));
store.dispatch(addColor("yellow"));
store.dispatch(addColor("green"));
store.dispatch(addColor("red"));
```


