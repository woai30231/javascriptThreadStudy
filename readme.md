# javascript线程及与线程有关的性能优化

## 声明

这些内容都是按我自己的理解来组织和写的，可能术语什么的有些不是很严谨，所以有些概念模糊的应也专业的术语为准，<span style="color:#ccc;">这里介绍的有些“术语”并不等同于你已经知道的那些“术语”，所以不要硬套概念在这里去理解！</span>当然了，我也会经常复习查看这里的文档，对一些错误额观点会及时更正，尽量保证严谨性！

## javascript线程

我觉得在开始描述相关问题之前，需要理解一下javascript里面的线程概念，首先需要知道：

*  javascript是单线程的，也就是说，一段代码，js执行的时候是从上往下一句一句的执行，前面的代码永远要先于后面的代码执行，如：

``` javascript
	var a = 15;
	var b = 16;
	//这里js代码在运行的时候，肯定先执行把15赋值给a的操作，再来执行把16赋值给b的操作
```

* 同步操作、异步操作

+ 首先得知道什么是同步操作！就好比两个人去食堂排队打饭，排在前面的人打完之后才轮到后面的人打饭！这就是同步操作，大家按先来后到的顺序做事！同步的好处就是简单有规则，所以调试起来相对轻松，因为大家都是按“规则”办事的，不会出现“插队”的情况，所以要“调查谁”,只要找到“它前面的相关人”，就能“逮住他”。同样，同步也是有不好的地方的，比如资源不能充分利用，因为“排队”的时候不能做其他事情！只能等待，不能合理安排自己的任务等！

+ 异步操作，同样以去食堂打饭来说明！有一群人去食堂打饭，小明发现在他前面有好多好多人在排队，可是刚好他现在有一件急需要做的事情去处理，还好，他有一个很好的朋友在食堂吃饭，于是他跑过去跟他朋友说道：“哥们，我现在有很重要的事需要做，你能不能在人少的时候给我打电话告诉我一下，我再过来打饭！”于是，小明就去做他自己的事情去了，等没有人排队的时候，他的朋友打电话告诉他，可以过来打饭了！于是小明就很舒服地去打饭了。其实可以把这个过程就叫做异步，我们可以看到，异步很亮眼的一个好处就是，小明可以打饭和做其他事情两不误，所以能合理利用资源！当然了，这是需要付出代价的，至少在代码实现上肯定比同步难！异步的不好的地方也有很多，很难调试和断言，比如下面的代码：

``` javascript
	

	var a = '';
	getData();//前面里面包含一个异步操作，实现对a = 15的赋值操作
	console.log(a);//我们发现在这个地方打印a是个空字符串，因为在这个地方，异步操作并没有执行
	//解决方法就是使用回调，callback，如
	getData(function(){
		console.log(a);//print  15
	});

```

+ 按我的理解来说，javascript只是“同步”的，没有“异步”一说！只不过因为javascript代码借助了代码所在的宿主环境，由宿主来管理这些“异步”的代码，从而让javascript得以实现“异步”一说！那么宿主是怎么管理“异步代码”的呢？简单来说就是通过一种排队机制实现的！可以这样子来理解：假设当前有一段代码正在执行，而且大概需要执行20ms,当执行到10ms时候突然触发了一个点击事件，这里如果是多线程的话，那么不用等待，监听器直接触发，可是js单线程的，所以事件监听器不能执行，那怎么办呢？此时，宿主的管理作用就出来了，宿主并没有让事件监听器立即执行，而是把监听器的代码用排队的方式放在当前执行代码的后面，当当前代码在20ms之后执行完成之后，再来执行事件监听器代码！可以用一张图片把这个过程描述如下：

![图片](https://github.com/woai30231/javascriptThreadStudy/blob/master/images/demo_1.png)

## setTimeout和setInterval


* setTimeout定时器

setTimeout描述的操作就是程序在多少时间之后再执行某操作，如：

``` javascript
	
	var a = 1;
	function fun(){
		a += 1;
		console.log(a);
	};

	setTimeout(fun,5000);
	//5秒之后打印2
```


>> setTimeout API

``` javascript

  	var id = setTimeout(fn,timer);
  	//fn是签名函数
  	//timer间隔时间
  	//返回一个id值，在fn未触发之前，可以通过clearTimeout(id)清除，从而不执行fn
  	clearTimeout(id);

```


* setInterval 间隔定时器

setInterval描述的是每隔多少时间执行某操作，如：

``` javascript
	var cc = 1;
	function fn(){
		cc += 1;
		console.log(cc);
	};


	setInterval(fn,1000);
```

>> setInterval API


``` javascript
	
	var id = setInterval(fn,timer);
	//fn是要执行签名名字，
	//timer是间隔时间
	//返回一个id，用于将来某个时间用clearInterval清除间隔定时器
	clearInterval(id);


```


###　setTimeout和setInterval的区别

* 首先从概念上来说明，setTimeout多少时间之后执行某操作，只执行一次，而setInterval每隔多少时间之后执行某操作，如果不用clearInterval清除的话，将会一直执行下去。其实两个方法都返回一个id值，用于清除定时器，分别是clearTimeout和clearInterval，还有说明一下这两个操作都是异步的，其实这也是javascript在浏览器中最最最简单的异步操作了！

* 再次从性能上来说，setTimeout的性能是要优于setInterval的，这一点将会在后面的文档中说明，需要联系上面所说的排队机制！


* setTimeout和setInterval都不能保证到了时间点一定会执行,如：setTimeout(fn,5000),并不能保证5s之后一定能执行fn。这得取决于当前js线程队列里面还有没有其他待处理队列，如果刚好没有的话，那么就能刚好执行，如果当前线程里面已经有了其它待处理队列正在执行，那么需要排队，等到javascript线程空闲的时候才会执行定时器！还有需要记住一点，能用setInterval实现的操作，一定能用setTimeout来实现，如下面的例子：


``` javascript
	
	//实现对一个数字定时加1操作 
	//setTimeout
	(function(){
		var a = 0;
		setTimeout(function fun(){
			a += 1;
			console.log(a);
			setTimeout(fun,1000);
		},1000);
	})();

	//setInterval

	(function(){
		var a = 0;
		setInterval(function(){
			a += 1;
			console.log(a);
		},1000);
	})();


```


* setTimeout 和 setInterval最重要的区别就是：如果用setTimeout和setInterval来实现一个重复的操作，切记！setTimeout是等待循环的操作执行完成之后，才继续在间隔时间之后再把循环操作添加到javascript的线程里面，而setInterval是不等待的，它从来不管放在线程里面循环操作有没有执行完成，反正到点就会把循环操作添加到javascript线程队列里面。但是这里有一点需要说明一下，js线程不会维护setInterval里面已经过期的了的循环操作，所以同一个setInterval在线程里面只会有一个轮次。理解这一点很重要，这是setTimeout性能优于setInterval的根源！现在用一张草图说明一下这个过程，如下：

>> setTimeout

 ![setTimeout](https://github.com/woai30231/javascriptThreadStudy/blob/master/images/demo_2.png)


 _注意：上面的图实际上有点不准确，正常情况应该是在10ms处时才添加第一个队列，然后在30ms处添加第二个队列，以此类推！这里只是为方便说明，所以图片上是在0ms时添加了第一个队列，望注意！_



>> setInterval

 ![setInterval](https://github.com/woai30231/javascriptThreadStudy/blob/master/images/demo_3.png)


>> 由此可见，setTimeout可以让浏览器喘口气，因为setTimeout是等他添加的队列执行完成之后才在间隔时间后添加队列，而setInterval是不管浏览器死活的，它自己爽了就好，它定时就添加队列，但是严重影响性能！至于为什么这样会影响性能，后面的文档会仔细说明！（**合理的利用setTimeout，能把一个耗时大的操作，变成一些耗时短小的操作，从而提升画面交互体验，比如页面卡顿什么的！**）



## 耗时大的操作影响交互和性能

* 为了说明这个问题，我们需要一个实例来说明一下，下面是实例的节选代码，全部代码可到[demo1.html](https://github.com/woai30231/javascriptThreadStudy/blob/master/html/demo1.html)!我们这里实现一个操作：用js实现向页面添加20000*6的一个表格，并且每个单元格需要显示当前的序号，我们知道反复对html进行dom操作、渲染是一个很影响性能的过程，查看页面就知道很卡，而且还可能死机等情况！话不多说，代码如下：


``` html
	<table>
		<tbody></tbody>
	</table>



	 <script type="text/javascript">
	 	
	 window.onload = function(){
	 	(function(){

	 		var table = document.getElementsByTagName('table')[0];
		 	var tbody = table.getElementsByTagName('tbody')[0];
		 	var num = 0
		 	for(var i = 0,len = 20000;i<len;i++){
		 		var tr = document.createElement("tr");
		 		for(var j = 0,len1 = 6;j<len1;j++){
		 			var td = document.createElement('td');
		 			num += 1;
		 			var txt = document.createTextNode(num);
		 			td.appendChild(txt);
		 			tr.appendChild(td);
		 		};
		 		tbody.appendChild(tr);
		 	};


	 	})();
	 };



	 </script>
	
```


我们发现上面的页面加载的时候空白了一段时间，虽然这里性能损耗还不足以让浏览器死机。但现在改进一下js代码，是可以让这个空白时间缩短的，好的，代码如下(查看[全部代码](https://github.com/woai30231/javascriptThreadStudy/blob/master/html/demo2.html))：



``` html


	<table>
		<tbody></tbody>
	</table>
	
	<script type="text/javascript">
	 	
	 window.onload = function(){
	 	(function(){
	 		/*这里我们把原本一步完成的事情，在这里分成5小步，从而达到把耗时大的代码划分为耗时小的代码
	 		有利于html页面快速构建*/
	 		var table = document.getElementsByTagName('table')[0];
	 		var tbody = table.getElementsByTagName('tbody')[0];
	 		var stepNum = 4000;
	 		var isComplete = false;//表格是否渲染完成
	 		var num = 0;//单元格序号
	 		var timeoutId = setTimeout(function fn(){
	 			if(isComplete){
	 				clearTimeout(timeoutId);
	 				return;
	 			};
	 			for(var i = 0,len = 4000;i<len;i++){
	 				var tr = document.createElement('tr');
	 				for(var j = 0,len1 = 6;j<len1;j++){
	 					var td = document.createElement('td');
	 					num += 1;
	 					var currentNum = num;//因为i是从零开始的，所以需要加1
	 					td.appendChild(document.createTextNode(currentNum));
	 					tr.appendChild(td);
	 				};
	 				tbody.appendChild(tr);
	 			};
	 			stepNum += 4000;
	 			if(stepNum > 20000){
	 				isComplete = true;//说明已经超过20000行了
	 			};
	 			setTimeout(fn,0);//0ms之后继续调用fn
	 			//这里说明一下，setTimeout和setInterval并不能准确保证短时粒度的执行
	 			//也就是说，这里虽然要求是0ms之后把代码推送到事件队列里面
	 			//但是可能实际上是真正执行的是在比0ms长的时间之后推送到时间队列里面
	 			//关于这一点可以再开一个单元来说明
	 		},0);
	 	})();
	 };



	 </script>


```

**我们发现使用了setTimeout来的代码打开页面会快了许多，当然了可能视觉上看不是很明显，原因也是有的，其一就是我们这里的代码量还算在合理量之间，其二，可能跟浏览器的性能什么的有一些关系。但这的确是加快了页面响应时间的，不信，我们可以在代码中加一些东西，来看看当页面刚记载的时候到页面有内容呈现花了多少时间，所以对以上代码分别做如下更改**

>> 未用setTimeout版，点[这里](https://github.com/woai30231/javascriptThreadStudy/blob/master/html/demo3.html)查看全部代码

``` javascript
	

	<table>
		<tbody></tbody>
	</table>




	<script type="text/javascript">
	 	
	 window.onload = function(){
	 	var startTime = new Date().getTime();
	 	(function(){

	 		var table = document.getElementsByTagName('table')[0];
		 	var tbody = table.getElementsByTagName('tbody')[0];
		 	var num = 0
		 	for(var i = 0,len = 20000;i<len;i++){
		 		var tr = document.createElement("tr");
		 		for(var j = 0,len1 = 6;j<len1;j++){
		 			var td = document.createElement('td');
		 			num += 1;
		 			var txt = document.createTextNode(num);
		 			td.appendChild(txt);
		 			tr.appendChild(td);
		 		};
		 		tbody.appendChild(tr);
		 	};


	 	})();
	 	var endTime = new Date().getTime();
	 	var diffTime = endTime - startTime;
	 	console.log("页面渲染这个表格花费了"+diffTime+"毫秒");
	 };



	 </script>

```

_浏览器控制台的截图(chrome浏览器)_

![](https://github.com/woai30231/javascriptThreadStudy/blob/master/images/demo_4.png)


>> 使用setTimeout版，点[这里](https://github.com/woai30231/javascriptThreadStudy/blob/master/html/demo4.html)查看全部代码


``` html

	 <table>
	 	<tbody></tbody>	
	 </table>




	 <script type="text/javascript">
	 	
	 window.onload = function(){
	 	var startTime = new Date().getTime();
	 	(function(){
	 		/*这里我们把原本一步完成的事情，在这里分成5小步，从而达到把耗时大的代码划分为耗时小的代码
	 		有利于html页面快速构建*/
	 		var table = document.getElementsByTagName('table')[0];
	 		var tbody = table.getElementsByTagName('tbody')[0];
	 		var stepNum = 4000;
	 		var isComplete = false;//表格是否渲染完成
	 		var num = 0;//单元格序号
	 		var isDisplayTime = true;//是否打印时间
	 		var timeoutId = setTimeout(function fn(){
	 			if(isComplete){
	 				clearTimeout(timeoutId);
	 				return;
	 			};
	 			for(var i = 0,len = 4000;i<len;i++){
	 				var tr = document.createElement('tr');
	 				for(var j = 0,len1 = 6;j<len1;j++){
	 					var td = document.createElement('td');
	 					num += 1;
	 					var currentNum = num;//因为i是从零开始的，所以需要加1
	 					td.appendChild(document.createTextNode(currentNum));
	 					tr.appendChild(td);
	 				};
	 				tbody.appendChild(tr);
	 			};
	 			stepNum += 4000;
	 			if(stepNum > 20000){
	 				isComplete = true;//说明已经超过20000行了
	 			};
	 			if(isDisplayTime){
	 				isDisplayTime = false;
	 				var endTime = new Date().getTime();
	 				var diffTime = endTime - startTime;
	 				console.log("渲染这个表格共花了"+diffTime+"毫秒");
	 			};
	 			setTimeout(fn,0);//0ms之后继续调用fn
	 			//这里说明一下，setTimeout和setInterval并不能准确保证短时粒度的执行
	 			//也就是说，这里虽然要求是0ms之后把代码推送到事件队列里面
	 			//但是可能实际上是真正执行的是在比0ms长的时间之后推送到时间队列里面
	 			//关于这一点可以再开一个单元来说明
	 		},0);
	 	})();
	 };



	 </script>

```

_浏览器控制台的截图（chrome浏览器）_

![](https://github.com/woai30231/javascriptThreadStudy/blob/master/images/demo_5.png)


## setTimeout是怎么提升页面响应时间的？

实际上这得归功于浏览器的内部渲染机制，这里不做过多介绍，因为要讲明白这些东西，完全是就是写一个长篇大论了，奈何自己能力有限，有些知识的掌握程度还欠火候，所以不能在这里乱说一些，只能把自己所能掌握的说明一下！

其实浏览器有一个机制，那就是如果某段代码的执行时间过长，那么就会造成页面卡顿，因为在某段代码执行的过程中，它不能做其它事情，不能渲染页面。甚至有些代码的执行时间实在过长，浏览器会直接死机，当然了有的浏览器对执行时间大于某个阀值的，会直接给出弹出提示，并拒绝代码的执行！

setTimeout的奥妙就是把一个执行时间很长的代码分成执行时间很小的代码段，这样浏览器就能逐步渲染页面了，从而解决了页面迟迟显示不出来的问题，以及因为代码执行时间过长浏览器死机的问题。




## 事件轮询

**这部分内容待完善**


## setTimeout和setInterval间隔时间粒度讨论（仅作讨论，以说明在小粒度的时候误差很大）

目前来说，鉴于各大浏览器的js引擎等原因，这两种定时器都很难实现时间间隔粒度精确到1ms或比这个时间更小的时间粒度的处理，当然了，浏览器各大厂商正在努力想这个方向靠拢！我们来做一个测试，代码如下:

>> **setTimeout版** 点[这里](https://github.com/woai30231/javascriptThreadStudy/blob/master/html/demo5.html)查看全部代码


``` javascript
		

		var startTime = new Date().getTime();
		for(var i = 0;i<100;i++){
			setTimeout(function fn(){
				var endTime = new Date().getTime();
				var diffTime = endTime - startTime;
				console.log("中间相差了"+diffTime+"毫秒");
				startTime = endTime;//结束时间作开始时间
			},1);
		};

```

- _浏览器控制台截图(firefox浏览器)_

![](https://github.com/woai30231/javascriptThreadStudy/blob/master/images/demo_6.png)


>> **setInterval**版，点[这里](https://github.com/woai30231/javascriptThreadStudy/blob/master/html/demo6.html)


``` javascript

	var startTime = new Date().getTime();
	var num = 0;
	var id = setInterval(function fn(){			
		if(num>=100){
			clearInterval(id);
			return;
		};
		var endTime = new Date().getTime();
		var diffTime = endTime - startTime;
		startTime = endTime;//结束时间赋值给开始时间
		console.log("间隔了"+diffTime+"毫秒");
		num += 1;
	},1);

```

![](https://github.com/woai30231/javascriptThreadStudy/blob/master/images/demo_7.png)





















