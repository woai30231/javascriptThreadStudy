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


* 记住一点，能用setInterval实现的操作，一定能用setTimeout来实现，如下面的例子：


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

>> setInterval

 ![setInterval](https://github.com/woai30231/javascriptThreadStudy/blob/master/images/demo_3.png)


>> 由此可见，他们的区别了！






