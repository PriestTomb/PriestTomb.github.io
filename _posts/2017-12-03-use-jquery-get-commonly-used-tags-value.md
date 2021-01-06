---
layout: post
title: 使用jQuery获取常用标签中的值
date: 2017-12-03
categories:
- 乱七八糟的前端
tags: [jQuery]
status: publish
type: post
published: true
author:
  login: PriestTomb
  email: mxingzh@163.com
  display_name: PriestTomb
---

## 写在前面

最近在东奔西跑的面试，作为做 java ee 的程序猿，难免要会一些前端的东西，比如 jQuery，面试中也被考到不少 jQuery 的内容，其中就有使用 jQuery 获取某些标签中的值的问题，因为平时不够重视，所以吃了些亏，今天抽了点时间把几个常用的标签写了一遍，一方面巩固一下这些小知识点，一方面也方便以后查询【哈哈

## 代码

#### 1. html代码

```html
<body>
	<h3>&lt;div&gt;</h3>
	<div id="testDiv">
		这是一个 div 标签
	</div>
	<input type="button" onclick="getDivVal();" value="获取"/>
	<hr>

	<h3>&lt;a&gt;</h3>
	<a id="testA" href="https://priesttomb.github.io">
		这是a标签
	</a><br/>
	<input type="button" onclick="getAVal();" value="获取1"/>
	<input type="button" onclick="getAHref();" value="获取2"/>
	<hr>

	<h3>&lt;span&gt;</h3>
	<span id="testSpan">
		这是 span 标签
	</span><br/>
	<input type="button" onclick="getSpanVal();" value="获取"/>
	<hr>

	<h3>&lt;label&gt;</h3>
	<label id="testLabel">
		这是 label 标签
	</label><br/>
	<input type="button" onclick="getLabelVal();" value="获取"/>
	<hr>

	<h3>&lt;input&gt;</h3>
	<input id="testInput" type="text" value="input标签" /><br/>
	<input type="button" onclick="getInputVal();" value="获取"/>
	<hr>

	<h3>&lt;li&gt;</h3>
	<ul class="testUl">
		<li>111</li>
		<li>222</li>
		<li>333</li>
	</ul>
	<hr>

	<h3>radio</h3>
	<label><input type="radio" name="testRadio" value="A" />A</label>
	<label><input type="radio" name="testRadio" value="B" />B</label><br/>
	<input type="button" onclick="getRadioVal();" value="获取"/>
	<hr>

	<h3>&lt;select&gt;</h3>
	<select id="testSelect">
		<option>select1</option>
		<option>select2</option>
		<option>select3</option>
	</select><br/>
	<input type="button" onclick="getSelectVal();" value="获取"/>
	<hr>

	<h3>checkbox</h3>
	<label><input type="checkbox" value="X" name="testCheckbox" />X</label>
	<label><input type="checkbox" value="Y" name="testCheckbox" />Y</label>
	<label><input type="checkbox" value="Z" name="testCheckbox" />Z</label><br/>
	<input type="button" onclick="getCheckboxVal();" value="获取"/>
	<hr>
</body>
```

#### 2. js代码

```javascript
$(function(){
	//获取选中li的内容
	$('.testUl').find('li').click(function(){
		alert($(this).text().trim());
	});

	//获取选中li内容的另一种方法
//	$('.testUl li').each(function(){
//		$(this).click(function(){
//			alert($(this).text().trim());
//		});
//	});
});

function getDivVal(){
	alert($('#testDiv').text().trim());
	//alert($('#testDiv').html().trim());
}

function getAVal() {
	alert($('#testA').html().trim());
}

function getAHref() {
	alert($('#testA').attr('href'));
}

function getSpanVal() {
	alert($('#testSpan').text().trim());
	//alert($('#testSpan').html().trim());
}

function getInputVal() {
	alert($('#testInput').val().trim());
}

function getRadioVal() {
	//alert($("input:radio:checked").val());
	//alert($("input[type='radio']:checked").val());
	alert($("input[name='testRadio']:checked").val());
}

function getSelectVal() {
	alert($("#testSelect option:selected").val().trim());
	//alert($("#testSelect").find("option:selected").val().trim());
}

function getCheckboxVal() {
//	$("input:checkbox:checked").each(function(){
//		alert($(this).val());
//	});

//	$("input[type='checkbox']:checked").each(function(){
//		alert($(this).val());
//	});

	$("input[name='testCheckbox']:checked").each(function(){
		alert($(this).val());
	});
}

function getLabelVal() {
	//alert($("#testLabel").html().trim());
	alert($("#testLabel").text().trim());
}
```

## 效果图

![效果图.png](/images/blog_img/20171203/效果图.png)
