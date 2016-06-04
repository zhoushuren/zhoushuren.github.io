
## JavaScript原型链的基本概念以及继承的原理

#### 作者：周俊。
##### Email：zhoujun247@gmail.com

 有任何问题欢迎交流

* 你是不是对JS的原型链感到困惑，是不是非常绕，很难理解？
读完周老师写的文章你就自然明白了。不要太感谢我。

* 先背一句绕口令：
  1. 每个对象都有\__proto__属性。这个属性指向他的构造函数的prototype属性

  如：

  ```js
  var a = {};
  a.__proto__ === Object.prototype   //true
  ```

  2.一个对象的的值在自己身上找不到，他就会在自己的__proto__属性上找，（也就是说在他的构造函数的prototype上找）

* 改变构造函数的prototype就是让他继承其他函数的属性和其他函数的\__proto__上的值（其他函数的构造器的prototype）
以上这句难理解的请耐心往下看。

### 形象的说：
* 一个对象是一个人。变量 var p  他生下来的时候什么都没有，但是家里有的东西肯定都是他的，这叫继承。（就像你继承你爸爸的财产一样）

* 他是怎么来的。诞生的时候肯定是爸妈生的。（废话）
* 他爸爸默认就是Object

* 所以创建对象是
```js
var p = {}; //下面的方式其实是一样只是写法不同
var p = new Object();
```

* 这时候p刚生下来，光秃秃的对象，什么都没有。但他继承了他爸爸的属性，他爸爸有什么，他就有什么。

其他例子也是同理
```js
//创建字符串对象
var s = ‘abc’; //等同于下面这句。如果为什么等同不能理解，自己去找资料。
var s = new String(‘abc');
```
```js
//创建函数对象
var f = function(){};  //这两个一样的
var f = new Function();
```

* 以上创建了一个光秃秃的对象s，我们并没有在s上定义任何属性，那么它为什么会有length属性呢？
s.length //他就有这个属性。这个属性是他老爸String.prototype上的。

* 他是怎么继承的呢？

* 以上说的第一个绕口令，每个对象都有\__proto\__ . 可以理解为他有个小口袋 \__proto__，连接着爸爸的prototype .

* 在他自己身上找不到length属性，就会在小口袋里找
通过小口袋__proto__ 找到了length属性，他就拿来用了。

* 如果你还不理解。我再说通俗一点。每个人都有信用卡。当自己卡没有钱的时候，可以刷爸爸的额度。
自己的信用卡是 自己.\__proto__    爸爸的信用卡是 爸爸.prototype 他们之间是共享的。

#### 再不理解我真的帮不了你了。

### 这里必需要理解的是（如果不理解就死背下来这几句口令）

1. 每个对象都一定有小口袋\__proto__指向自己构造器的prototype。

2. 每个函数都有prototype这个属性，并且这个属性自己也是个对象

3. 每个函数都可以当作构造器（爸爸），构造（生）一个对象。

4. 每个函数的 prototype.constructor 都默认指向自己

5. 因为protytope自己也是一个对象。自己当然也有\__proto__

## 一个对象怎么继承另一个对象呢？
下面声明两个构造函数。A B （构造函数其实就是普通的函数，你用new来创建对象，它就是构造函数。）
```js
function A(){
 this.name=‘我是a’;
 this.m = ‘我有一百万';
}
```
```js
function B(){
 this.name=‘我是b我是穷惯蛋’;
}
```
下面大B要生小b了。
```js
var b = new B();

console.log(b.name);  //我是b我是穷惯蛋

console.log(b.m);  //undefined  (他没有钱)
```
为什么没有钱呢？
因为B的prototype指向的是Object,是所有对象的祖先，他也没钱。。怎么办。所以儿子的儿子就没有钱。

那么B就想办法认一个有钱的干爹A。
```js
B.prototype = new A();
```
因为A他里面有m属性。（它是有钱人）

这个时候B.prototype.constructor 也指向了A

b的__proto__指向B.prototype的，就是说。B.prototype有的方法。b的口袋里都有。

那么B.prototype 指向了 new A(); 所以，new A()有的方法。b也有。

很容易理解吧！

### b在访问一个自身属性的时候的步骤。
1. 看看自己身上有没有。
2. 没有的话，看看身上的\__proto\__里有没有 (\__proto__连接着爸爸的prototype)其实就是看构造器的prototype有么有。
3. 如果构造器的prototype身上也没有。（上面说过prototype也是一个对象）就会在prototype.\__proto__里面找。
4. 既然prototype是对象，对象都是Object构造出来的。那么prototype.\__proto__就指向Object.prototype
5. Object.prototype 也是对象，但是Object.prototype.\__proto__指向 null 它是末端了，人家祖坟被你挖出来了，实在没东西挖了。


#### 欢迎和作者交流JS问题，作者是JS发烧狂，没事爱研究JS问题。先告一段落了，下周更新其他知识。
