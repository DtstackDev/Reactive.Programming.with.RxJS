# 一、通过Scheduler掌控时间
当我接触到Rxjs，很快在项目中使用它。有段时间我认为我已经掌握它，并在项目中游刃有余。但是我遇到一个很纠结的问题：

我怎么知道我正在用的operator是异步还是同步？换句话说，就是什么时候利用operator发出通知？这看起来是正确使用Rxjs关键之处，但是它又让我有些地
方感觉模糊。就延迟operator，很明显是一个异步操作，所以它必须是类似setTimeout 方式发出。但是假如我用它在一个范围呢？它也会发出异步的吗？或者它能阻止事件循环吗？那又怎样，我还是到处使用这些operators。即使我不懂关于它们内部的并发模型。

后来我就学会了调度器 

调度器能够精确管理你程序的并发性，并且是一种很有效的方式。它允许你改变它们的并发模型，进而你可以掌握一个Observable是如何发出通知的。
在本章，我们将学习怎样使用调度器以一些常见的应用场景。我们的重点是测试，调度器在这方面就会现的尤为有用，并且你将学会开发自己的调度器。

##   1).使用Schedulers
调度器是一种能调度发生在未来行为的机制，RxJS中的每个operator都在内部使用一个scheduler，以在最可能的情况下提供最佳性能.

让我们看看如何改变operator的scheduler以及这样做的后果吧。首先让我们创建一个包含1,000个整数的数组：
```
var arr = [];
for (var i=0; i<1000; i++) {
 arr.push(i);
}

```
然后，我们从arr创建一个Observable，并强制它通过订阅来发出所有的通知。 在代码中，我们还会测量发出所有通知所需的时间：
```
var timeStart = Date.now();
Rx.Observable.from(arr).subscribe(
function onNext() {},
function onError() {},
function onCompleted() {
console.log('Total time: ' + (Date.now() - timeStart) + 'ms'); 
});
```
>    "Total time: 6ms”

六毫秒 - 不错！ 从内部使用Rx.Scheduler.currentThread，在任何当前工作完成后，调度工作运行。 一旦启动，它将同步处理所有通知。
 
现在我们将调度程序更改为Rx.Scheduler.default

```

var timeStart = Date.now();
Rx.Observable.from(arr, null, null, Rx.Scheduler.default).subscribe(
function onNext() {},
function onError() {},
function onCompleted() {
console.log('Total time: ' + (Date.now() - timeStart) + 'ms');
});
```
>"Total time: 5337ms"

哇，我们的代码比currentThread Scheduler慢几千倍。 这是因为默认Scheduler会不同地运行每个通知。 我们可以通过在订阅之后添加简单的日志语句来验证这一点。

使用currentThread Scheduler
```
Rx.Observable.from(arr).subscribe( ... );
console.log('Hi there!’);
```
> "Total time: 8ms"  
"Hi there!"

使用 default Scheduler
```
Rx.Observable.from(arr, null, null,Rx.Scheduler.timeout).subscribe( ... );
console.log('Hi there!’);
```
> "Hi there!"  
"Total time: 5423ms"

因为使用默认调度程序的Observer会异步发出它的通知，所以我们的console.log语句（这是同步的）在Observable甚至开始发出任何通知之前执行。

而使用currentThread
Scheduler，所有通知都会同步发生，所以只有当Observable发出了所有的通知时，才会执行console.log语句。
因此，Scheduler真的可以改变我们的Observables的工作原理。 在我们的例子中，性能真的受到异步处理大量已经可用的阵列的影响。 但是我们实际上可以使用Scheduler来提高性能。 例如，我们可以在对Observable进行昂贵的操作之前即时切换Scheduler：

```
arr.groupBy(function(value) {
return value % 2 === 0; })
.map(function(value) {
return value.observeOn(Rx.Scheduler.default);
}) .map(function(groupedObservable) {
return expensiveOperation(groupedObservable); });
```

在上面的代码中，我们将数组中的所有值分为两组：均匀和不均匀的值。 groupBy返回一个Observable，为每个创建的组发出一个Observable。 这里是很酷的部分：在对每个分组的Observable中的项目执行代价大的操作之前，我们使用observeOn将Scheduler切换到默认值，以便昂贵的操作将异步执行，而不会阻止事件循环。
## 2).观察和订阅
在上一节中，我们使用observeOn运算符来更改某些Observables中的Scheduler。 observeOn和subscribeOn是返回Observable实例的副本的实例运算符，但是使用我们传递的Scheduler作为一个参数。  
observeOn需要一个Scheduler并返回一个使用该Scheduler的新Observable。 它将使每个onNext调用在新的Scheduler中运行。  

subscribeOn强制可以在特定的调度程序上运行Observable的订阅和非订阅工作（而不是通知）。 像observeOn一样，它接受一个Scheduler作为参数。 subscribeOn是非常有用的，例如当我们正在运行浏览器，并在subscribe调用中执行一些很重要的任务，但是我们不想阻止UI线程。
## 3).基本Rx Schedulers
在我们刚刚使用的Scheduler中，我们来深入一下。 RxJS的operators最常使用的是immediate，default和currentThread
### 1.即时Scheduler
即时调度程序从Observable同步发出通知，所以每当在紧急调度程序上调度一个动作时，它将立即执行，阻止线程。 Rx.Observable.range是在内部使用即时调度程序的运算符之一：
```
Rx.Observable.range(1, 5) .do(function(a) {
console.log('Processing value', a); })
.map(function(value) { return value * value; }) .subscribe(function(value) { console.log('Emitted', value); });
console.log('After subscription');
```
> Before subscription  
Processing value 1  
Emitted 1  
Processing value 2  
Emitted 4  
Processing value 3  
Emitted 9  
Processing value 4  
Emitted 16  
Processing value 5   
Emitted 25  
After subscription
 
 程序输出按照我们期望的顺序发生。 每个console.log语句在当前项目的通知之前运行  
  
###### 何时使用
即时Scheduler非常适合在每个通知中执行可预测且不太昂贵的操作的Observables。 此外，Observable必须最终调用onCompleted  
## 
### 2.默认Scheduler   
默认Scheduler异步运行操作。可以将其视为与零毫秒延迟的setTimeout的粗略等价，以保持序列中的顺序。它运行在最有效的异步实现的平台上（例如，Node.js中的process.nextTick或浏览器中的“设置超时”）。  

我们来看看前面的范例，并在默认的Scheduler上运行。 为此，我们将使用observeOn运算符：
```
console.log('Before subscription'); Rx.Observable.range(1, 5)
.do(function(value) { console.log('Processing value', value);
})
.observeOn(Rx.Scheduler.default)
.map(function(value) { return value * value; }) .subscribe(function(value) { console.log('Emitted', value); });
console.log('After subscription');
```
> Before subscription  
Processing value 1  
Processing value 2  
Processing value 3  
Processing value 4  
Processing value 5   
After subscription  
Emitted 1  
Emitted 4  
Emitted 9  
Emitted 16  
Emitted 25   

这个输出有显着差异。 我们的同步console.log语句立即运行每个值，但是我们使Observable在默认的Scheduler上运行，它会异步生成每个值。 这意味着我们的操作符中的日志语句在平方值之前被处理。
###### 何时使用
默认调度程序从不阻止事件循环，因此它非常适合涉及时间的操作，如异步请求。它也可以用于从未完成的观察，因为它在等待新的通知（这可能永远不会发生）时不阻止程序。

### 3.当前Scheduler    
currentThread Scheduler和immediate Scheduler一样是同步的。但是，在我们的递归操作的情景中，它排队执行而不是立即执行。一个递归的操作符是一个它自己调度其他操作符的操作符。一个很好的例子就是repeat。repeat操作符，如果不给参数，一直重复之前的不定长的链中的Observable序列。

如果在使用即时Scheduler的运算符（如返回）上重复调用，则会遇到麻烦。 让我们通过重复值10来尝试这个，然后使用take来取代重复的第一个值。理想情况下，代码将打印10次，然后退出：
```
// Be careful: the code below will freeze your environment!
Rx.Observable.return(10).repeat().take(1) .subscribe(function(value) {
       console.log(value);
     });
```
> Error: Too much recursion  

此代码导致无限循环。订阅后，返回调用onNext（10），然后onCompleted，这使得重复订阅再次返回。由于返回在立即Scheduler上运行，因此此过程将重复，从而导致无限循环，并且无法执行。  
但是如果我们通过传递它作为第二个参数来调度currentThread Scheduler上的返回值，我们得到：   
```
var scheduler = Rx.Scheduler.currentThread;  
Rx.Observable.return(10, scheduler).repeat().take(1)  
.subscribe(function(value) { console.log(value);
});
```
> 10  

现在，当重复重新订阅到return，那个新的onNext调用将会排队，是因为之前的onCompleted还在发生。repeat就返回了一个一次性的对象给take。重复然后返回一个可取消的对象来执行，调用onCompleted并通过重复排列来取消重复，最终返回订阅的调用。    

作为经验法则，currentThread应该用于在大序列上迭代，当使用递归运算符（如repeat）时。
###### 何时使用

currentThread Scheduler对于涉及递归运算符（如repeat）的操作非常有用，通常用于包含嵌套运算符的迭代。  
 
## 2)动画Scheduling 
对于诸如画布或DOM动画的快速视觉更新，我们可以使用具有非常低的毫秒值的间隔运算符，或者我们可以使一个使用类似setTimeout的函数来调度通知的Scheduler。
   
但是这两种方法都不是理想的。 在这两种情况下，我们都会在浏览器中抛出所有这些更新，这些更新可能无法快速处理。这是因为浏览器试图渲染一个帧，然后它接收到渲染下一个帧的指令，所以它丢弃当前帧以保持速度。 结果是波动动画。 在web上有好多这种情况.
   
浏览器有一种本地的方法来处理动画，并且它们提供了一个API来使用它称为requestAnimationFrame。requestAnimationFrame允许浏览器通过在最适当的时间排列动画来优化性能，并帮助我们实现更平滑的动画。

### 1.为此有一个动画Scheduling
Rx DOM库附带一些额外的调度程序，其中一个是requestAnimationFrame调度程序。  
是的，你猜了 我们可以使用这个Scheduler来改进我们的宇宙飞船视频游戏。 在这个游戏中，我们建立了一个40ms的刷新速度 - 大约每秒25帧 -通过创建一个间隔可观察速度，然后使用combineLatest以间隔设置的速度更新整个游戏场景（因为它是最快更新的可观察 ）...但是谁知道浏览器通过使用这种技术丢弃了多少帧！ 我们将通过使用requestAnimationFrame获得更好的性能。  
   
我们创建一个Observable，它使用Rx.Scheduler.requestAnimationFrame作为其Scheduler。 请注意，它与间隔操作符的工作原理类似：
```
function animationLoop(scheduler) {  
return Rx.Observable.generate(0,
function() { return true; }, // Keep generating forever
function(x) { return x + 1; }, // Increment internal value
function(x) { return x; }, // Value to return on each notification Rx.Scheduler.requestAnimationFrame); // Schedule to requestAnimationFrame
}
```
现在，无论我们使用间隔，以25 FPS为动画制作图形，我们可以使用我们的animationLoop函数。 所以我们可以观察到画星星，看起来像这样：  
```
var StarStream = Rx.Observable.range(1, 250) .map(function() {
return {
x: parseInt(Math.random() * canvas.width),  
y: parseInt(Math.random() * canvas.height),  
size: Math.random() * 3 + 1
}; })
.toArray() .flatMap(function(arr) {
return Rx.Observable.interval(SPEED).map(function()   
{ return arr.map(function(star) {
if (star.y >= canvas.height) { star.y = 0;
}
star.y += 3; return star;
}); });
});
```
---Becomes this:
```
var StarStream = Rx.Observable.range(1, 250) .map(function() {
return {
x: parseInt(Math.random() * canvas.width),    
y: parseInt(Math.random() * canvas.height),   
size: Math.random() * 3 + 1
}; })
.toArray() .flatMap(function(arr) {
return animationLoop().map(function() {    
return arr.map(function(star) {
if (star.y >= canvas.height) { star.y = 0;
}
star.y += 3; return star;
}); });
});
```
这给我们一个更流畅的动画。 并且代码也更干净！  

# 二 、使用 Schedulers进行测试
测试可能是我们可以使用计划程序的最引人注目的场景之一。 到目前为止，这本书到目前为止我们一直在编码，没有对结果过多地考虑。但是在一个真实的软件项目中，我们将编写测试来确保我们的代码更健壮工作。  

测试异步代码很困难。我们通常碰到一下问题：
  
  
-   模拟异步事件是复杂和容易出错的。 测试的全部要点是避免bugs和errors，但如果您的测试本身有错误，那么它们没有任何帮助。
-   如果我们要准确测试基于时间的功能测试，自动测试变得非常慢。 例如，如果我们需要准确测试在尝试检索远程文件四秒钟后调用错误，则每次测试至少需要等待运行一段时间。 如果我们连续运行我们的测试，这会影响我们的开发时间。  

## 1）The TestScheduler
RxJS给了我们TestScheduler，一个旨在帮助测试的调度器。 TestScheduler允许我们在我们方便的时候模拟时间，并创建确定性测试，保证它们在100％的可重复性。 此外，它允许我们执行需要相当长的时间并将其压缩到瞬间的操作，同时保持测试的准确性。   
   
TestScheduler是VirtualTimeScheduler的专业化。 VirtualTimeSchedulers在“虚拟”时间而不是实时的执行动作。 计划的动作进入队列，并在虚拟时间内分配一小时。 然后，计划程序在其时钟进行时按顺序运行。 因为它是虚拟时间，一切都会立即运行，而不用等待指定的时间。 我们来看一个例子：
```
var onNext = Rx.ReactiveTest.onNext;   
QUnit.test("Test value order",  
function(assert) {
var scheduler = new Rx.TestScheduler();
var subject = scheduler.createColdObservable(
onNext(100, 'first'),  
onNext(200, 'second'),  
onNext(300, 'third')
);
var result = '';
subject.subscribe(function(value) { result = value });
scheduler.advanceBy(100);    
assert.equal(result, 'first');
scheduler.advanceBy(100);   
assert.equal(result, 'second');
scheduler.advanceBy(100);
assert.equal(result, 'third'); });
```  
在上面的代码中，我们测试了cold Observable按照正确顺序到达的某些值。为此，我们使用了TestScheduler中的createColdObservable的辅助方法来创建Observable，它在放回我们作为onNext参数传递的通知。在每个通知里面我们指定了通知将要发射的时间值。在这之后，我们订阅这个Observable，在createColdObservable中手工的让虚拟时间前进，检查它确定发射了那个期望的值。如果这个例子在正常的时间里运行，它将需要300ms，但是由于我们使用了TestScheduler来运行Observable，它将会立即执行，并遵循顺序。  

## 2)Writing a Real-World Test
没有更好的方法来了解如何使用虚拟时间来缩短时间，而不是在现实世界中为时间敏感的任务编写测试。在第76页：让我们回顾下地震例子中Buffering values的Observable：
```
quakes
.pluck('properties')
.map(makeRow)
.bufferWithTime(500)
.filter(function(rows) { return rows.length > 0; })  
.map(function(rows) {  
var fragment = document.createDocumentFragment();  
rows.forEach(function(row) {
      fragment.appendChild(row);
    });
return fragment; })
.subscribe(function(fragment) { table.appendChild(fragment);
});
```
为了使代码更可测试，我们将Observable封装在一个使用我们在bufferWithTime运算符中使用的Scheduler的函数中。 在Observables中进行参数化测试总是一个好主意。

```
function quakeBatches(scheduler) {   
return quakes.pluck('properties')
.bufferWithTime(500, null, scheduler || null)  
.filter(function(rows) {
return rows.length > 0; });
}
```
我们还通过采取一些措施来简化代码，但是保持本质.这些代码需要一个json对象的Observable，它包含一个properties属性，
将它们每500毫秒缓冲一次，并对过滤掉的批次进行过滤。   

我们要验证这个代码是否有效，但是我们绝对不想在每次运行测试时等待几秒钟，以确保我们的缓冲按预期工作。这正是虚拟时间和TestScheduler将会帮助我们的地方：
```
❶ var onNext = Rx.ReactiveTest.onNext;
var onCompleted = Rx.ReactiveTest.onCompleted;  
var subscribe = Rx.ReactiveTest.subscribe;
❷ var scheduler = new Rx.TestScheduler();
❸ var quakes = scheduler.createHotObservable(  
onNext(100, { properties: 1 }),  
onNext(300, { properties: 2 }),  
onNext(550, { properties: 3 }),   
onNext(750, { properties: 4 }),   
onNext(1000, { properties: 5 }),  
onCompleted(1100)
);
❹ QUnit.test("Test quake buffering", function(assert) { 
❺ var results = scheduler.startScheduler(function() {
return quakeBatches(scheduler) }, {
        created: 0,
        subscribed: 0,
        disposed: 1200
});
❻ var messages = results.messages;  
console.log(results.scheduler === scheduler);
❼ assert.equal( messages[0].toString(),
        onNext(501, [1, 2]).toString()
      );
      assert.equal(
        messages[1].toString(),
        onNext(1001, [3, 4, 5]).toString()
);
      assert.equal(
        messages[2].toString(),
        onCompleted(1100).toString()
); });

```  
让我们按步骤分析下：  
1. 我们首先从ReactiveTest加载一些帮助函数,这是一些在虚拟时间里的注册事件：onNext，onCompleted，subscribe. 
2. 我们创建一个新的TestScheduler将驱动整个测试.
3. 使用TestScheduler的createHotObservable方法，我们创建了一个假的Observable，它将会在虚拟时间的特定时刻模拟通知。尤其是，它将会在第一个秒内发射5个通知，1100毫秒完成。每次它发射一个特定properties属性的对象。
4. 我们可以使用任何测试框架来运行测试。对于我们的例子，我选择了QUnit。
5. 我们使用startScheduler方法创建一个Observable，它使用了一个 test Scheduler。第一个参数是一个函数，它创建了运行了我们SchedulerObservable。这种情况下，我们简单返回我们的quakeBatches函数，我们传递TestScheduler给它。第二个参数是一个包含了不同虚拟时间的对象，这是我们想创建Observable，订阅到它，并销毁它。对我们的例子来说，我们订阅从虚拟0时间开始并且我们销毁这个Observable在虚拟的1200毫秒之后。
6. startScheduler返回一个scheduler和message属性的对象。在message中我们可以Observable在虚拟时间发射的所有通知.
7. 我们的第一个断言测试是在501毫秒之后（恰好在第一个缓冲时间限制之后），我们的Observable产生值1和2。我们的第二个断言测试是在1001毫秒之后，我们的Observable产生剩余的值3,4和5.最后，我们的第三个断言检查序列是否完全在1100毫秒完成，正如我们在可观测的地震测试案例中所指定的那样。
 
该代码以非常可靠的方式有效地测试了我们的高度异步的Observables，而无需跳过环来模拟异步状态。 我们简单地指定我们希望我们的代码在虚拟时间做出反应的时间，我们使用test Scheduler来运行整个操作。
## 三、总结
Scheduler是RxJS的重要组成部分。 即使你没有明确使用它们也可以走很长的路程，它们是先进的概念，它将使您在程序中调和并发性的优势。 虚拟时间的概念对于RxJS是唯一的，对于测试异步代码等任务非常有用。

在下一章中，我们将使用Cycle.js，这是一种基于称为单向数据流的概念来创建令人惊叹的网络应用程序的反应性方法。 借助它，我们将使用现代技术创建一个快速的Web应用程序，可以显着改善传统的Web应用程序的方式。



























  
 


  
  

