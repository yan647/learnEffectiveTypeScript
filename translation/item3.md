# item 3 明白代码生成与类型无关
在高层次，tsc做了两件事：
* 它将下一代的TS/JS转换成能在浏览器中运行的老版本的JS（编译）。
* 它检查代码是否存在类型错误。

令人惊讶的是，这两种行为是完全独立的。换句话说，代码中的类型不会影响由TS到JS的编译过程。由于JS是可执行的，这就意味着类型不能影响代码的运行方式。
这里有一些令人惊讶的暗示，而且会告诉你TS能做什么和不能做什么。
带有类型错误的代码可以产生输出
由于代码输出与类型检查无关，因此有类型错误的代码可以产生输出。

```
$ cat test.ts
let x = 'hello';
x = 1234;
$ tsc test.ts
test.ts:2:1 - error TS2322: Type '1234' is not assignable to type 'string'
​
2 x = 1234;
~
​
$ cat test.js
var x = 'hello';
x = 1234;
```
如果你使用过C或Java这样的语言，他们的类型检查和输出是同时进行的，那TS的这种情况可能会非常令人惊讶。你可以认为所有的TS报错与这些语言中的警告类似：他们很可能表明了一个问题，值得探究，但他们不会停止构建。

在实践中，在代码中发现错误是很有用的。如果你正在构建一个Web应用程序，你可能知道某个特定部分存在问题。但是因为TS在存在错误的时候仍然可以生成代码，你可以在修改他们之前测试其他部分的代码。

当你提交代码的时候你的目标是零错误，以免落入必须记住预料之中或意外的错误。如果你希望禁止错误输出，可以在tsconfig.json中使用noEmitOnErrors选项，或者在构建工具中使用类似选项。

## 不能在运行时检查TS类型
你可能会写这样的代码：

```TypeScript
interface Square {
  width: number;
}
interface Rectangle extends Square {
  height: number;
}
type Shape = Square | Rectangle;

function calculateArea(shape: Shape) {
    if (shape instanceof Rectangle) {
                        // ~~~~~~~~~ 'Rectangle' only refers to a type,
                        //           but is being used as a value here
    return shape.width * shape.height;
                    //         ~~~~~~ Property 'height' does not exist
                    //                on type 'Shape'
    } else {
      return shape.width * shape.width;
    }
}
```
Instanceof的检查发生在运行时，但是Rectangle是一种类型，因此它不能影响代码的运行时行为。类型是“可擦除的”: 编译到JavaScript的部分会被简单地从代码中删除所有的接口、类型和类型注释（todo）。

为了确定要处理的类型的形状，你需要一些方法能在运行时重构它的类型。在这种情况下，你可以检查height属性是否存在：
```typescript
function calculateArea(shape: Shape) {
  if ('height' in shape) {
    shape;  // Type is Rectangle
    return shape.width * shape.height;
  } else {
    shape;  // Type is Square
    return shape.width * shape.width;
  }
}
```

这是起作用的，因为属性检查只涉及在运行时可用的值，但仍允许类型检查器把shape的类型细化为Rectangle。

另一种方法引入一个"tag"，以运行时可用的方法显式存储类型：

```TypeScript
interface Square {
  kind: 'square';
  width: number;
}
interface Rectangle {
  kind: 'rectangle';
  height: number;
  width: number;
}
type Shape = Square | Rectangle;

function calculateArea(shape: Shape) {
  if (shape.kind === 'rectangle') {
    shape;  // Type is Rectangle
    return shape.width * shape.height;
  } else {
    shape;  // Type is Square
    return shape.width * shape.width;
  }
}
```

这里的Shape类型是被标记的联合类型（tagged union）的一个例子。因为这种方法使得在运行时恢复类型信息变得非常容易，所以标记联合类型在TypeScript中是无处不在。

一些构造引入了一个类型（在运行时不可用）和一个值（在运行时可用）。class关键字就是其中之一。创建Square和Rectangle类可能是修复错误的另一种方法：

```TypeScript
class Square {
  constructor(public width: number) {}
}
class Rectangle extends Square {
  constructor(public width: number, public height: number) {
    super(width);
  }
}
type Shape = Square | Rectangle;

function calculateArea(shape: Shape) {
  if (shape instanceof Rectangle) {
    shape;  // Type is Rectangle
    return shape.width * shape.height;
  } else {
    shape;  // Type is Square
    return shape.width * shape.width;  // OK
  }
}
```

这是因为class Rectangle同时引入了类型和值，而interface只引入了类型。

在type Shape =Square | Rectangle 中的Rectangle指的是类型，而shape instanceof Rectangle 中的Rectangle指的是值。这个区别对于理解是很重要的，但是可能很微妙，详情参见Item8。

## 类型操作不能影响运行时的值


