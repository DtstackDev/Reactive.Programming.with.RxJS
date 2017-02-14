# 响应式之路

在现实世界中，事件发生的随机性、应用程序的冲突以及网络错误等情况会导致一切都一团糟，几乎没有应用程序是完全同步的，为了保持程序的高响应异步代码必不可少，程序开发充斥着痛苦，然而我们并不比如此。

现代的应用程序不仅需要超快的响应能力，同时还要具备准确处理不同数据来源中数据流的能力。然而当前的技术没有提供这种能力，因为当我们增加了并发和应用程序各种状态转换再来处理数据流会让代码复杂度指数级增加。如果硬要这样解决会大大增加开发者的精神负担，从而导致代码越来越复杂和层出不穷的bug。

本章会向你介绍一种更自然更简单的异步代码解决方案——响应式编程。我会向你展示事件流（我们称之为被观察者）是怎样优雅的处理异步代码的，然后再创建一个观察者来告诉我们什么事响应式思维。RxJS可以显著的改进现有的技术，你将发现你的开发会更快乐更有效率。

## 什么是响应式？

我们先从一个点击不同按钮处理不同数据源的小程序来看一看用RxJS是如何响应式编程的。这个小程序首先要满足下列的条件：

* 数据必须来自于两个不同的地址，并且拥有不同的JSON结构
* 最终结果不需要关心是否重复
* 为防止过多请求，要求每秒钟只能处理一次用户的点击行为

用RxJS写出的代码就是这样的：

```javascript
var button = document.getElementById('retrieveDataBtn');
var source1 = Rx.DOM.getJSON('/resource1').pluck('name');
var source2 = Rx.DOM.getJSON('/resource2').pluck('props', 'name');
function getResults(amount) { return source1.merge(source2)
.pluck('names')
.flatMap(function(array) { return Rx.Observable.from(array); }) .distinct()
.take(amount);
}
var clicks = Rx.Observable.fromEvent(button, 'click'); clicks.debounce(1000)
  .flatMap(getResults(5))
  .subscribe(
function(value) { console.log('Received value', value); }, function(err) { console.error(err); },
function() { console.log('All values retrieved!'); }
);
```
先不用担心看不懂这些代码，大致看下代码结构，首先你会发现使用了被观察者的模式，我们用寥寥几行代码就实现了所有的处理逻辑。

每个被观察者都代表一个数据源，程序可以很大程度上表示为数据流。在上面的例子中远端的接口，用户鼠标的点击都是被观察者。事实上，在我们的代码中只有点击按钮获取想要结果的事件是必须的被观察者。

响应式编程是十分高效的，就拿我们例子中的限制鼠标点击次数来说，你可以想象如果我们用回调函数或promise来实现是怎么样的：我们需要首先需要一个每秒重置的计时器，另外还要设置一个状态表示过去的一秒钟用户有没有点击按钮。对于这么小的一个功能这么做就显得过于复杂了，而且这些代码甚至不与你程序的实际功能有关联。在更大的应用程序中，这些小的复杂功能叠加起来很快就会把基础代码搞的一团糟。

在响应式编程中，我们使用反射机制来对点击限流，这确保了两次点击之间至少间隔1秒，多余的点击都被忽略。其实我们不需要关心具体如何实现的，只需要知道想要我们的代码做什么就可以了。

更有趣的是我们接下来就看看响应式编程如何帮助我们的程序更加高效易懂。

## 灵活的电子表格

我们先以电子表格这个典型例子开始研究响应式系统。电子表格我们都使用过，但是很少有人停下来想想它们是多么惊人的直观(how shockingly intuitive they are)。例如我们在电子表格的A1格内设定一个值，并将它关联到其他单元格，这样每当我们改动A1的值，其他关联的单元格中的值也都自动更新了。

![spreadsheet](http://ojo0vdkcl.bkt.clouddn.com/spreadsheet.jpeg)

这样的行为让人感觉非常自然，我们不用告诉电脑去更新A1关联的其他单元格的值，也不用告诉电脑怎样做，这些单元格跟A1的变化是关联的。在电子表格中我们只需定义要处理的问题就行了，完全不用关心电脑是如何计算这些结果的。

这就是响应式编程想要实现的，我们定义成员之间的关联，程序随着这些成员的改变去计算更新后的值。

## 将鼠标操作看成数据流

回顾一下本章开始的例子，我们将事件理解了成数据流。在例子中用户实时点击操作我们用一组循环的事件序列去处理，这种思路是RxJS开创者Erik Meijer在《Your Mouse Is a Database》一文中提出的。

在响应式编程中我们将鼠标点击看作是一系列连续的事件流，而这个事件流是可以查询喝操作的。用事件流代替孤立的值给我们编程打开了一条全新的道路，这种方式中我们可以操作整个数值序列，但实际上它并没有被创建。

静下心来想一想，以往的做法中，我们要将这些值先存到一个地方（数据库或数组），需要的时候再从里面取。但是如果它们不能马上获取到，比如要用过网络请求获取的时候，那么我们只能等待获取到它们之后再继续了。

![](http://ojo0vdkcl.bkt.clouddn.com/WechatIMG1.jpeg)

我们将这一序列想像成了一个数组，只不过这个数组中的值是用时间维度划分而不是内存维度。无论按时间划分还是按内存划分，我们都将得到一序列的元素如下：

![](http://ojo0vdkcl.bkt.clouddn.com/WechatIMG2.jpeg)

将程序看作这么一序列的数据是理解RxJS编程的关键，这需要一些练习但并不困难。事实上我们应用程序处理的绝大多数数据都可以用序列来表达，在本书17页，第二章“深入序列”中我们会学习更多关于序列的概念。

## 查询序列

举一个用javascript传统事件监听处理鼠标事件流的简单例子：每当鼠标点击的时候打印出鼠标的x和y坐标。通常我们会这么写：

```
document.body.addEventListener('mousemove', function(e) { console.log(e.clientX, e.clientY);
});
```
这段代码在每次鼠标点击的时候会打印出x、y坐标如下：
```
252 183 
211 232 
153 323
...
```
这看起来不就是一个序列吗？唯一的问题就是操作事件比操作数组麻烦多了，例如：将上面的例子改成只打印屏幕右侧的前10次点击（请原谅有点儿随意…），那么代码会改成下面这样。
```
var clicks = 0;
document.addEventListener('click', function registerClicks(e) {
	if (clicks < 10) {
		if (e.clientX > window.innerWidth / 2) {
			  console.log(e.clientX, e.clientY);
				clicks += 1; 
		}
	} else {
		document.removeEventListener('click', registerClicks);
	}
});
```
为了满足功能，我们不得不定义一个全局变量click来记录一直以来的点击次数；为了判断两种不同的条件，我们又不得不用双重条件判断。而且当打印结束的时候我们还要释放变量和事件注销以防止内存泄漏。

### 副作用和外部状态

当一个功能触发的时候会对作用域之外的部分造成影响，我们称之为副作用。改变函数外的变量、console函数中打印或者更新数据库中的一个值都是副作用的典型例子。

举个例子，改变我们函数内的变量是安全的，但是如果这个变量在我们的函数中，别的函数却可以修改它，那么这就意味着我们失去了对这个变量的控制，甚至我们都没有办法保证它的值是否还是我们期望的。因此我们必须增加与函数完全不相关的代码逻辑，平白增加了代码的复杂度和出错率。

在构建任何有趣的程序时，有时候不得不加入这种副作用，但是我们应该努力减少这种代码，这在响应式编程中非常重要，因为响应式编程中有很多随时间变化的部分。在本书中，我们将寻求一种避免外部状态和副作用的方法，事实上，在第三章“构建并发程序”中我们将构建一个完全没有副作用的视频游戏。

为了实现一个简单的功能，我们最终写了一段复杂的代码，这对第一次接触它的开发人员来说是很难维护的。更重要的是因为我们需要保持状态，所以未来开发的时候很容易产生隐藏的bug。

在这种情况下我们想要的是查询点击的“数据库”。如果使用关系型数据库，就需要使用声明式语言SQL：
```
SELECT x, y FROM clicks LIMIT 10
```
如果我们将该点击事件流视为可以查询和转换的数据源，该怎么办呢？其实它与数据库的唯一区别就是它是实时发送的，其他并没有什么不同。我们所需要的也就是一个抽象概念的数据类型。

使用RxJS和它的被观察者数据类型如下：
```

Rx.Observable.fromEvent(document, 'click')
	.filter(function(c) { return c.clientX > window.innerWidth / 2; })
	.take(10)
	.subscribe(function(c) { console.log(c.clientX, c.clientY) })
```
这段代码完成的功能跟第四页的代码一样，可以这样理解：

	创建点击事件的被观察者，并过滤出屏幕左侧发生的点击次数，然后只有前10次点击发生的时候在控制台打印出坐标。	
就算你对RxJS不熟悉也能轻松读懂这段代码，而且你再也不用创建独立于业务代码之外的变量来保持状态，这也就更难引入bug。释放变量的问题也不复存在，不用担心忘记注销事件带来的内存泄漏，是不是很完美？

在上面的代码中我们用一个DOM事件创建了一个被观察者，这个被观察者给我们提供了一个可以作用整体操作的事件序列或流，而不是往常的每次一个单独的事件。处理序列给我们带来巨大的力量；我们可以轻松的合并，转换或传递被观察者。我们将无法处理的事件转变为有型的数据结构，就像操作数组一样简单，但更灵活。

在下一节中，我哦们将看到使Observables成为一个伟大工具的原理。

## 观察者和迭代器

要理解被观察者怎么来的，我们先要学点基础的东西：观察者模式和迭代器模式。在本节中我们快速了解一下这两种设计模式，然后再看看Observables如何以简单而强大的方式组合两者的概念。

### 观察者模式

对于一个软件开发人员来说，很少听说Observables，因此也无法从Observables联想到经典的观察者模式。在观察者模式中有一个对象称为生产者（在大多数观察者模式的说明中，这个对象都被称为Subject，为了避免跟RxJS中的Subject混淆，这里我们称之为生产者。），它内部有一些监听器保持着对生产者状态的订阅，每当生产者状态发生改变这些监听器就会被通知到（通过调用它们的update方法）。

用几行代码我们就可以实现一个观察者模式：

ch1/observer_pattern.js
```
function Producer() { 
	this.listeners = [];
}
Producer.prototype.add = function(listener) { 
	this.listeners.push(listener);
};
Producer.prototype.remove = function(listener) { 
	var index = this.listeners.indexOf(listener); this.listeners.splice(index, 1);
};
Producer.prototype.notify = function(message){ 
	this.listeners.forEach(function(listener) {
    listener.update(message);
  });
};
```
上面的Producer对象实例中用一个listeners数组保存监听器的动态列表，每当Producer调用notify方法时，这些监听器都将被更新。下面的代码中，我们创建两个监听通知的对象，Producer的代码如下：

ch1/observer_pattern.js
```
// Any object with an 'update' method would work.
var listener1 = {
update: function(message) {
	console.log('Listener 1 received:', message); }
};
var listener2 = {
update: function(message) {
	console.log('Listener 2 received:', message); }
};
var notifier = new Producer(); 
notifier.add(listener1); 
notifier.add(listener2); 
notifier.notify('Hello there!');
```
运行代码输出如下
```
Listener 1 received: Hello there! 
Listener 2 received: Hello there!
```
不需要我们处理，每当Producer的实例notifier更新它的内部状态，listener1和listener2都会被通知。

### 迭代器模式
被观察者谜题中的另一部分是迭代器模式。迭代器是一个对消费者隐藏实现的对象，它为消费者提供一种简单方法来遍历其内容。

迭代器接口设置很简单，它只有两个方法：获取序列中下一个项目的next()方法，以及检查序列中是否有剩余项目的hasNext()方法。

下面是我们如何编写一个对数字数组进行操作的迭代器，它只生成除数参数倍数的元素：

ch1/iterator.js
```
function iterateOnMultiples(arr, divisor) { 
	this.cursor = 0;
	this.array = arr;
	this.divisor = divisor || 1;
}
iterateOnMultiples.prototype.next = function() { 
	while (this.cursor < this.array.length) {
		var value = this.array[this.cursor++]; 
		if (value % this.divisor === 0) {
			return value; 
		}
	} 
};
iterateOnMultiples.prototype.hasNext = function() { 
	var cur = this.cursor;
	while (cur < this.array.length) {
		if (this.array[cur++] % this.divisor === 0) { 
			return true;
		} 
	}
	return false; 
};
var consumer = new iterateOnMultiples([1, 2, 3, 4, 5, 6, 7, 8, 9, 10], 3);
console.log(consumer.next(), consumer.hasNext()); // 3 true console.log(consumer.next(), consumer.hasNext()); // 6 true console.log(consumer.next(), consumer.hasNext()); // 9 false
```
迭代器很适合封装任何种类数据结构的遍历逻辑，上面例子中可以看出，迭代器在处理不同类型数据或者可以在运行时配置的时候非常有用，就像我们代码中实现了遍历divisor参数的逻辑一样。

## Rx模式和被观察者

观察者模式和迭代器模式在各自的领域内都十分强大，将两者组合起来能带来更多惊喜。我们将这种组合结果称之为Rx模式，命名为Reactive Extensions库，在本书接下来的部分就会使用这种模式。
被观察者序列或者简单被观察者是Rx模式的核心，它像迭代器一样按顺序发出它的值，推送给可用的消费者，而不是消费者请求下一个值。它与Producer在观察者模式中的作用类似：发出值并将它们推送诶其监听器。

	### Pulling vs. Pushing
	
	在编程中，基于推送的行为意味着应用程序的服务器组件向其客户端发送更新，而不是客户端必须轮询服务器以进行这些更新。这就像说：“你不用来找我，我会通知你”。
	RxJS就是基于推送实现的，因此事件源（被观察者）会推送新的值给消费者（观察者），消费者不需要主动请求下一个值。
	
简单的说，一个被观察者就是一个内部状态随时间变得可用的序列。而被观察者的消费者，即观察者，则等同于观察者模式中的监听器。只要观察者订阅了被观察者，每当序列中的值状态变成可用，观察者不用发送请求就会受到这些值。

这看起来与传统的观察者模式没什么不同，但是实际上它们有两点主要不同：

* 被观察者只有被订阅的时候才会开始流项目
* 像迭代器一样，被观察者可以在序列完成时发出信号

有了被观察者，我们可以声明如何响应它们发出的元素序列，而不是对每个元素单独响应，可以有效的复制，转换和查询序列，这些操作将应用于序列的所有元素。

## 创建被观察者
有多种方法可以创建被观察者，create操作符是最明显的。Rx.Observable对象中的create操作符接收一个以观察者为入参的回调函数，改函数定义了被观察者如何发出值。下面的代码创建了一个简单的被观察者：

```
var observable = Rx.Observable.create(function(observer) { 
	observer.onNext('Simon');
	observer.onNext('Jen');
	observer.onNext('Sergi');
	observer.onCompleted(); // We are done 
});
```
当我们订阅了这个observable，它会通过监听器上注册的onNext方法发出3个字符串，然后通过onCompleted方法发送序列完成的信号。但是怎么才能订阅这个observable呢？接下来我们来看看观察者是如何使用的吧。

### 初次接触观察者
观察者是监听被观察者的，一旦被观察者的事件发生，它就会调用所有观察者的对应方法来通知它们。

观察者通常有3个方法：onNext，onCompleted和onError。

onNext 相当于观察者模式中的更新操作。当Observable发出一个新值时，这个方法就被调用。请注意这个名称反映了我们订阅整个序列的事实，而不仅仅是离散值。

onCompleted 通知已经没有数据更新了，onCompleted调用之后再调用onNext都是无效的了。

onError 当Observable发生错误时调用，它被调用之后再调用onNext都是无效的。

下面的代码创建了一个基本的观察者：

```
var observer = Rx.Observer.create(
	function onNext(x) { console.log('Next: ' + x); }, 
	function onError(err) { console.log('Error: ' + err); }, function onCompleted() { console.log('Completed'); }
);
```
Rx.Observer对象中的create方法接受onNext，onCompleted和onError的方法，并返回一个Observer实例。这三个方法时可选的，你可以决定要包含哪些方法。例如，如果我们订阅的是一个无穷序列，比如点击按钮（假设用户会永远不停点击），那么onCompleted就永远不会被调用。如果我们有足够信心保证这个序列不会出错（比如，用一个数字数组作为被观察者），那么onError方式也是不需要的了。

### 将Ajax请求封装成被观察者
到目前为止我们还没有用Observables做出什么有用的功能。那么如何创建一个Observables来检索远程内容呢？要实现这个功能，我们要将XMLHttpRequest对象封装进Rx.Observable.create中：

```
function get(url) {
return Rx.Observable.create(function(observer) {
	// Make a traditional Ajax request
	var req = new XMLHttpRequest(); 
	req.open('GET', url);
	req.onload = function() { 
		if (req.status == 200) {
			// If the status is 200, meaning there have been no problems, 
			// Yield the result to listeners and complete the sequence observer.onNext(req.response);
			observer.onCompleted();
		}
		else {
			// Otherwise, signal to listeners that there has been an error
			observer.onError(new Error(req.statusText)); 
		}
	};
	req.onerror = function() {
		observer.onError(new Error("Unknown Error"));
	};
    req.send();
  });
}
// Create an Ajax Observable
var test = get('/api/contents.json');
```
上面的代码中，get函数使用create来封装XMLHttpRequest，如果这个HTTP的GET请求成功，我们就发出它的返回内容并结束序列（我们的Observable只会发出一个结果）。否则的话我们就会发出一个错误。在最后一行，我们调用具有特定URL的函数来定义请求。这行代码会创建一个Observable对象，但是并不会发送请求，因为还没有观察者订阅它，这一点非常重要。那么让我们来实现这个订阅的功能：

```

// Subscribe an Observer to it
test.subscribe(
	function onNext(x) { console.log('Result: ' + x); }, 
	function onError(err) { console.log('Error: ' + err); }, function onCompleted() { console.log('Completed'); }
);
```
首先要注意的是，我们没有像第11页代码中那样显式的创建Observer，大多数时候我们都使用上面这种更简洁的版本，在Observable的subscribe中传入onNext、onCompleted、onError三个函数，通过调用subscribe实现Observer的三种情况。

在订阅中设置一切逻辑，在订阅之前，我们只是声明了Obervable和Obsrever两者如何交互，只有当我们调用subscribe代码逻辑才会开始运行。

### 总有一个控制器
在RxJS中，转换和查询序列的方法我们称之为operators（运算符）。Operators都定义在Rx.Observable静态对象和Observable实例当中，我们例子中的create方法就是这么一个operator。

create方法是我们创建专门的Obervable时一个非常不错的选择，但是RxJS给我们提供了大量其他易于创建常见源的Observable的oprator。

拿先前的例子来说，像Ajax请求这种常见操作就有现成的operator给我们用。同样的，RxJS DOM库提供了几种从DOM相关源创建Observable的方法。因此，当我们想发送一个GET请求的时候就可以用Rx.DOM.Request.get来完成，改造后的代码就变成下面这样：
```
Rx.DOM.get('/api/contents.json').subscribe(
	function onNext(data) { console.log(data.response); }, 
	function onError(err) { console.error(err); }
);
```
这个代码与我们现前写的大致相同，但是我们省了自己封装XMLHttpRequest的步骤，一切都给我们封装好了。同时我们还省略的onCompleted回调函数，因为Observable完成时我们不打算做什么响应，结果反正只有一个，我们已经在onNext回调函数中使用了它。

在本书中我们将大量使用这种方便的operator，这是RxJS自带的功能包，事实上，这也是它的主要优势之一。

	### 一种数据类型适用所有规则
	
	在RxJS程序中，我们应该努力将所有数据放倒Observable中，而不仅仅是来自异步元的数据。这样做可以很容易的组合来自不同源的数据，例如现有的数组，或者回调的结果，或者用户出发某些事件而触发XMLHttpRequest请求的结果
	例如，如果我们有一个数组数组，需要将它的成员与来自其他地方的数据结合使用，最好的办法是将这个数组变成一个Observable（显然，如果这个数组只是一个不需要组合的中间变量，就没必要这么做了）。通过本书你将学到什么情况下需要将数据类型转换成Observable。
	
RxJS提供了根据大多数javascript数据类型创建Observable的操作符，下面我们来了解一下一些我们会经常用的到：arrays，events和callbacks。

### 用数组创建Observable
我们可以用功能丰富的from操作符将任何类似数组或迭代器的对象转换成一个Observable。from操作符接收一个数组作为参数，并返回一个可以发出其每个元素的Observable。
```
Rx.Observable
.from(['Adrià', 'Jen', 'Sergi']) 
.subscribe(
	function(x) { console.log('Next: ' + x); }, 
	function(err) { console.log('Error:', err); } 
	function() { console.log('Completed'); }
);
```
from和fromEvent是RxJS代码中最方便和使用最频繁的运算符之一。

### 用javascript事件创建被观察者
当我们将一个事件转换为一个Observable时，它会变成一个可以组合传递的一级值。举个例子，我们定义一个Observable，它在鼠标移动的时候发出鼠标指针的坐标：
```
var allMoves = Rx.Observable.fromEvent(document, 'mousemove') allMoves.subscribe(function(e) {
  console.log(e.clientX, e.clientY);
});
```
将事件转换为Observable会释放事件本身的约束，更重要的是，我们可以基于原始的Observable创建新的Observable，这些新的独立对象可以用于不同的任务：
```
var movesOnTheRight = allMoves.filter(function(e) { 
	return e.clientX > window.innerWidth / 2;
});
var movesOnTheLeft = allMoves.filter(function(e) { 
	return e.clientX < window.innerWidth / 2;
});
movesOnTheRight.subscribe(function(e) { 
	console.log('Mouse is on the right:', e.clientX);
});
movesOnTheLeft.subscribe(function(e) { 
	console.log('Mouse is on the left:', e.clientX);
});
```
上面的代码中，我们用原始的allMoves创建了两个Observable，这些特殊的Observable只包含原来的过滤功能：movesOnTheRight包含了鼠标移动到屏幕右侧时候出发的事件，而movesOnTheLeft则是鼠标移动到屏幕左侧触发事件。两者都没有修改原始Observable中的功能：allMoves仍然会在鼠标移动的时候保持发送信息。Observable是不变的，每一个新创建的Observable都会包含所有的运算符。

### 用回调函数创建被观察者
如果您使用第三方JavaScript库，那么就必须与基于回调的代码进行交互。但是我们可以用fromCallback和fromNodeCallback两个方法将回调函数转换成Observable。Node.js遵循以下约定：总是给回调函数设置一个错误的参数，当出现错误则指示存在问题。那么，我们就用fromNodeCallback方法来创建一个类似Node.js风格的Observable：
```
var Rx = require('rx'); // Load RxJS
var fs = require('fs'); // Load Node.js Filesystem module

// Create an Observable from the readdir method
var readdir = Rx.Observable.fromNodeCallback(fs.readdir); 

// Send a delayed message
var source = readdir('/Users/sergi');

var subscription = source.subscribe(
	function(res) { console.log('List of directories: ' + res); }, function(err) { console.log('Error: ' + err); },
	function() { console.log('Done!'); 
});
```
在上面的代码中，我们使用Node.js的fs.readdir方法创建了一个读目录的Observable。fs.readdir方法接收目录路径和回调函数delayedMsg两个参数，一旦检索到目录内容就会调用回调。

我们代码的readdir方法使用了除回调函数之外与原始fs.readdir完全相同的参数，并返回一个Observable。当我们创建一个Observer订阅它之后，它将适时的调用onNext，onError和onCompleted。

## 总结
在本章中，我们探讨了响应式编程的方法，并了解了RxJS如何通过被观察者来解决其他方案的问题，例如回调函数或者promise。现在你明白被观察者这种模式为什么如此强大，并且知道如何创建它们。有了这个基础，我们现在可以继续创建更有趣的响应式程序了。下一章将向您展示如何在web开发中一些常见场景应用被观察者这种模式，创建和编写基于序列的程序。