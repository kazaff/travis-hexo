title: 去年的今天就是你的Deadline
date: 2016-12-11 09:37:12
tags:
- momentjs
- 去年同日
categories: 前端
---

我最近有个感悟：时间，最复杂的存在。

像我这样一个从没有跨过时区的孩子，是感受不到时间算法的复杂度的。直到有一天，我们的系统需要面对多时区的场景，我才恍然大悟。简简单单的一个时区问题都已经让系统很多基于时间的逻辑变得复杂，更不要提分布式时序问题了（我们今天并不讨论这个问题，我也讲不好）。

时区什么的并不是我们今天的讨论重点，其实我想说的是“去年同日”这个问题。本来觉得这个不是啥问题，只需要将给定的日期减去一年即可得呀：

```javascript
console.log(moment('2016-12-11').subtract(1, 'years').format('YYYY-MM-DD'));
// or
console.log(moment('2016-12-11').subtract(366, 'days').format('YYYY-MM-DD'));
```

That's All! 这里要安利一个很强大很主流的时间库：[Moment](http://momentjs.com/)，本文重度依赖该库。

可现实总是残酷的，从业务角度来思考的话，这种“去年的今天”意义是不大的。举个例子，通常运营人员希望得到的时间关系是：今天是12月的第一个星期三，那去年12月的第一个星期三呢？

这种对应关系中，线索包含：年份，月份，星期几，第几个。打开日历，我们观察一下每个月，会发现每月包含的周个数是不同的，有包含5周的，有包含六周的，极端情况下也会有包含4周的（2026-02）。这就导致一个问题，如果今年的当月包含五周，而去年同月只有四周，那么回答“去年X月的最后一个星期三是几号？”就需要特殊对待了，因为描述中使用的“最后一个”，很口语化，但却直接影响了排序方向。

而涉及到第几个星期几，就更复杂了。这里面还有一个隐藏的条件：每周起始天是星期几？我们国家都是星期一是每周的第一天，而美国是按照星期天是第一天哟~~

小弟我写了一个Moment的扩展，方便来计算下面几个问题：

1. 指定月份包含的总周数，例如：2015年8月份总共包含6周
2. 指定日期属于当月的第几周，例如：当日属于9月份的第三个周
3. 指定日期在当月中是第几个礼拜几，例如：当日为9月份的第二个礼拜三
4. 指定月份中指定序号的指定星期的日期，例如：9月份第二个礼拜三的日期

代码如下：

```javascript
(function () {
  var moment = (typeof require !== 'undefined' && require !== null) && !require.amd ? require('moment') : this.moment;
	var firstDay = 0;

	// 设置每周起始日是星期几，num范围：0~6，0代表星期日，1代表星期一，...，6代表星期六
	moment.setFirstDay = function (num){
		firstDay = num;
	};

	// 返回指定月份包含的总周数，例如：2015年8月份总共包含6周
  moment.totalOfWeekOfMonth = function (date) {
		var origin = moment(date);
		var dd = moment(date);
		dd.date(1);

		var totalOfWeekOfMonth = 0;
		var step = 1;
		var plus = dd.day() !== firstDay ? 2 : 1;	// 若1号不是周起始日，相当于第一周，碰见第一个礼拜日时，就需要+2，之后再碰见周日只需要+1
    while(dd.month() === origin.month()){
        if(dd.day() === firstDay){
            totalOfWeekOfMonth += plus;
						plus = 1;
						step = 7;
        }
        dd.add(step, 'd');
    }
    return totalOfWeekOfMonth;
  };

	// 返回指定日期属于当月的第几周，例如：当日属于9月份的第三个周
	moment.weekOfMonth = function (date){
		var origin = moment(date);
    var dd = moment(date);
    dd.date(1);

    var weekOfMonth = dd.day() === firstDay ? 0 : 1;	// 若该月第一天是周起始日，为了避免下面循环中重复累加，初始值应该为0
    while(dd.date() <= origin.date() && dd.month() === origin.month()){
        if(dd.day() === firstDay){
            weekOfMonth++;
        }
        dd.add(1, 'd');
    }
    return weekOfMonth;
	}

	// 返回指定日期在当月中是第几个礼拜几，例如：当日为9月份的第二个礼拜三
	moment.numOfDayOfWeek = function (date){
		var origin = moment(date);
    var dd = moment(date);
    dd.date(1);

    var numOfDayOfWeek = 0;
    while(dd.date() <= origin.date() && dd.month() === origin.month()){
        if(dd.day() === origin.day()){
            numOfDayOfWeek++;
        }
        dd.add(1, 'd');
    }
    return numOfDayOfWeek;
	}

	// 返回指定月份中指定序号的指定星期的日期，例如：9月份第二个礼拜三的日期
	moment.dateOfNumOfDayOfWeek = function (date, dayOfWeek, num){
    var dd = moment(date);
    dd.date(1);
		var month = dd.month();
		var times = 1;

    while(dd.month() === month){
        if(dd.day() === dayOfWeek){
					if(times === num){
						return dd.format('YYYY-MM-DD');
					}
					times++;
        }
        dd.add(1, 'd');
    }
    return undefined;
	}

  if (typeof module !== "undefined" && module !== null) {
    module.exports = moment;
  } else {
    this.moment = moment;
  }
}).call(this);
```

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8">
		<script src="//cdn.bootcss.com/moment.js/2.17.1/moment.min.js"></script>
		<script src="./moment-sameday.js" charset="utf-8"></script>
		<title></title>
	</head>
	<body>
		<script type="text/javascript">
			moment.setFirstDay(1);
			console.log("当月总周数：", moment.totalOfWeekOfMonth('2026-02-01'));
			console.log("当日为第几周：", moment.weekOfMonth('2026-02-02'));
			console.log("当日为第几个星期几：", moment.numOfDayOfWeek('2026-02-02'));
			console.log("当月第几个星期几是几号：", moment.dateOfNumOfDayOfWeek('2026-02', 0, 1));
		</script>
	</body>
</html>
```

至于上面提到的“最后一个”问题，在你的业务中到底如何计算，我相信你可以通过上面4个方法的组合应用来得到你要的答案！我能帮的就这么多了，好自为之哟~~

我的项目，获取去年同日的最终逻辑：

```javascript
// 假设dd为当日，targetDate为去年同日
// ...
var targetDate = null;
// 计算当日是本月内的第N个星期X
var day = dd.day();	//星期几
var num = moment.numOfDayOfWeek(dd);	//第几个

// 计算去年当月的第N-1, 第N和第N+1个星期X的日期
var tmpDd = moment(dd).subtract(1, 'years');
var dates = [];
dates.push(num > 1 ? moment.dateOfNumOfDayOfWeek (tmpDd, day, num - 1): undefined);
dates.push(moment.dateOfNumOfDayOfWeek (tmpDd, day, num));
dates.push(dates[1] ? moment.dateOfNumOfDayOfWeek (tmpDd, day, num + 1): undefined);

// 计算去年当月三个目标日期与今年当日的距离（日期距离）

for(var i = 0, max = dates.length, duration = 31; i < max; i++){
	if(dates[i] !== undefined){
		var tmpDuration = Math.abs(moment(dates[i]).date() - dd.date());
		if(tmpDuration <= duration){
			targetDate = dates[i];
			duration = tmpDuration;
		}
	}
}

// targetDate已经计算出来了
```
