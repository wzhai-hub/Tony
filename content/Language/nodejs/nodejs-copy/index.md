
---
title: Javascript deeep copy
tags:
  - nodejs
date: '2023-11-04'
summary: Understand Javascript deeep copy
---


# Javascript 深拷贝与浅拷贝

## JavaScript是一种高级的，动态类型化脚本语言。像大多数其他编程语言一样，JavaScript允许支持深层拷贝和浅层拷贝的概念

### 浅拷贝：当参考变量被拷贝到使用赋值运算符一个新的参考变量，在创建时候是引用对象的浅表副本。简而言之，引用变量主要存储它引用的对象的地址。为新参考变量分配旧参考变量的值后，将存储在旧参考变量中的地址拷贝到新参考变量中。这意味着旧参考变量和新参考变量都指向内存中的同一对象。结果，如果对象的状态通过任何参考变量发生变化，则两者都会反映出来。让我们举个例子来更好地理解这些

#### 举例说明

```
interface School {
  student: string;
  teacher: string;
  address: string;
}
const middleschool: School = {
  student: 'xiao',
  teacher: 'mei',
  address: 'china'
}

console.log('school origin:', middleschool);
let refmiddleschool = middleschool;
console.log('school origin:', refmiddleschool);
middleschool.student = 'ss';

console.log('school all changed:', middleschool);
console.log('school all changed:', refmiddleschool);
```

#### result

```
school origin: { student: 'xiao', teacher: 'mei', address: 'china' }
school origin: { student: 'xiao', teacher: 'mei', address: 'china' }
school all changed: { student: 'ss', teacher: 'mei', address: 'china' }
school all changed: { student: 'ss', teacher: 'mei', address: 'china' }
```

#### 说明：从上面示例中可以看出，当middleschool.student名称被修改时，它也反映在refmiddleschool对象上，这可能会导致数据不一致。这称为浅拷贝。新创建的对象与旧对象具有相同的内存地址。因此，对它们中的任何一个所做的任何更改都会更改两者的属性。如果将其中一个从内存中删除，则另一个将不复存在。从某种意义上说，这两个对象是相互依存的。为克服此问题，可以使用深拷贝

### 深层拷贝：与浅层拷贝不同，深层拷贝会拷贝旧对象的所有成员，为新对象分配单独的内存位置，然后将拷贝的成员分配给新对象。这样，两个对象彼此独立，并且在对任何一个对象进行任何修改的情况下，另一个对象均不受影响。同样，如果删除了其中一个对象，则另一个仍保留在内存中。可以使用lodash提供的cloneDeep（）方法做深拷贝，查看源代码可以看出是new了一个新对象

#### 举个例子

```
import * as _ from 'lodash';
interface School {
  student: string;
  teacher: string;
  address: string;
}
const middleschool: School = {
  student: 'xiao',
  teacher: 'mei',
  address: 'china'
}

console.log('school origin:', middleschool);
const refmiddleschool = _.cloneDeep(middleschool);
console.log('school origin:', refmiddleschool);
middleschool.student = 'ss';

console.log('school all changed:', middleschool);
console.log('school all changed:', refmiddleschool);
```

#### 执行结果

```
school origin: { student: 'xiao', teacher: 'mei', address: 'china' }
school origin: { student: 'xiao', teacher: 'mei', address: 'china' }
school all changed: { student: 'ss', teacher: 'mei', address: 'china' }
school all changed: { student: 'xiao', teacher: 'mei', address: 'china' }
```

#### 说明：修改后，两个对象具有不同的属性。同样，每个对象的方法定义不同，并产生不同的输出

##### 参考文档

[1] [geeksforgeeks](https://www.geeksforgeeks.org/what-is-shallow-copy-and-deep-copy-in-javascript/)
[2] [Lodash](https://lodash.com/docs/4.17.15#cloneDeep)
