# 构建完整的Web应用

本章我们将在前后端都使用RxJS来构建一个典型的web应用。在应用中我们会对文档对象模型（DOM）转换，并在Node.js服务器中使用WebSockets进行客户端-服务器通信。

应用的用户界面，我们将使用RxJS-DOM库，这是一个由制作RxJS的团队提供的库，它提供了方便的操作符来处理DOM和浏览器相关的东西，使我们的写代码更轻松。而服务器部分，我们将使用两个成熟的node库，并使用Observables包装它们的一些API，以在我们的应用程序中使用它们。

学完本章后，你就能够使用RxJS构建用户界面，使用我们前面学到的技术声明界面构建和对DOM的操作。也可以在Node.js项目中使用RxJS，或者使用响应式编程和RxJS开发任何项目。

## 构建一个实时的地震数据仪表盘
回过头来看下本书29页我们开始学习的时候接触的实时可视化地震仪表盘项目，现在我们用RxJS来重写它的前后端代码。我们使用Node.js开发服务端部分，并改进应用程序使其具有更好的交互性和更丰富的内容。

下图就是完成后的仪表盘截图：
![](http://ojo0vdkcl.bkt.clouddn.com/earthquick.jpeg)

继续完善本书29页的实时地震仪表盘项目代码：

```
var quakes = Rx.Observable 
	.interval(5000) 
	.flatMap(function() {
		return Rx.DOM.jsonpRequest({
		url: QUAKE_URL,
		jsonpCallback: 'eqfeed_callback' }).retry(3);
	}) 
	.flatMap(function(result) {
		return Rx.Observable.from(result.response.features); 
	})
	.distinct(function(quake) { return quake.properties.code; });
	
quakes.subscribe(function(quake) {
	var coords = quake.geometry.coordinates; 
	var size = quake.properties.mag * 10000;
  L.circle([coords[1], coords[0]], size).addTo(map);
});
```
这个代码有个潜在的错误：它可以在DOM准备好之前执行，这就导致每当我们执行的代码中需要用的DOM元素的时候就会抛出错误。因此，我们想要的是在DOMContentLoaded事件触发后再执行我们的代码，这时浏览器就知道页面上所有DOM元素了。

为此RxJS-DOM提供了Rx.DOM.ready()的Observable，每当DOMContentLoaded事件触发是它就会触发一次。所以我们把代码一个初始化函数中，订阅Rx.DOM.ready()，当其触发时执行初始化：
```
function initialize() {
	var quakes = Rx.Observable
		.interval(5000) 
		.flatMap(function() {
			return Rx.DOM.jsonpRequest({
				url: QUAKE_URL,
				jsonpCallback: 'eqfeed_callback'
			}); 
		})
		.flatMap(function(result) {
			return Rx.Observable.from(result.response.features);
		})
		.distinct(function(quake) { return quake.properties.code; });
	quakes.subscribe(function(quake) {
		var coords = quake.geometry.coordinates; 
		var size = quake.properties.mag * 10000;
    L.circle([coords[1], coords[0]], size).addTo(map);
  });
}
Rx.DOM.ready().subscribe(initialize);
```
接下来，我们在HTML中添加一个空表，用来填充接下来的地震数据：
```
<table>
  <thead>
		<tr> 
			<th>Location</th> 
			<th>Magnitude</th> 
			<th>Time</th>
    </tr>
  </thead>
	<tbody id="quakes_info">
  </tbody>
</table>
```
准备好这些，我们就可以开始写新的仪表盘代码了。

## 添加一个地震列表
新仪表盘的第一个功能是显示实时的地震列表，包括其位置、震幅、日期等信息，这些数据与USGS网站上的地图数据相同。首先我们创建一个返回一行元素的函数，这个函数接收一个porps的对象作为参数：
```
function makeRow(props) {
	var row = document.createElement('tr'); 
	row.id = props.net + props.code;
	var date = new Date(props.time);
	var time = date.toString();
	[props.place, props.mag, time].forEach(function(text) {
		var cell = document.createElement('td'); 
		cell.textContent = text; 
		row.appendChild(cell);
	});
	return row; 
}
```
