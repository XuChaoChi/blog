---
title: QML ListView实现过滤、排序、查找等
tags: qml
category: QML相关
abbrlink: aed67fdb
date: 2018-07-06 14:51:52
keywords: qml,listview
---

主要利用DelegateModel， 这里的难点是怎么去遍历ListView的每一行
### 1、首先利用DelegateModel来建立一个简单的ListView ###
	 import QtQuick 2.0
	 import QtQml.Models 2.2
	 
	 Rectangle {
	     width: 200; height: 200
	     DelegateModel {
	         id: visualModel
	         model: ListModel {
	             id:listModel;
	             ListElement { value: "1" }
	             ListElement { value: "2" }
	             ListElement { value: "5" }
	             ListElement { value: "6" }
	             ListElement { value: "4" }
	             ListElement { value: "3" }
	         }
	 
	         delegate: Rectangle {
	             id: item
	             height: 25
	             width: 200
	             Text {
	                 text: value
	             }
	         }
	     }
	 
	     ListView {
	         id:listView;
	         anchors.fill: parent
	         model: visualModel
	     }
	 }
<!--more-->
### 2、接下来的的事情就是如何去遍历他 ###

这个时候首先通过ListModel的count去获取model的数量
	count : int
The number of data entries in the model.
然后通过ListModel中的get获取对象，然后进行对比
	object get(int index)
将过程放在DelegateModel中

	Component.onCompleted: {
         var rowCount = listModel.count;
         for( var i = 0;i < rowCount;i++ ) {
             var model = listModel.get(i);
             console.log(model.value)
         }
     }
同样的原理实现删除、查找、排序等等

### 3、完整代码 ###
	 import QtQuick 2.0
	 import QtQml.Models 2.2
	 
	 Rectangle {
	     width: 200; height: 200
	 
	 
	     DelegateModel {
	         id: visualModel
	 
	         model: ListModel {
	             id:listModel;
	             ListElement { value: "1"; value2:"one"}
	             ListElement { value: "2"; value2:"two"}
	             ListElement { value: "5"; value2:"five"}
	             ListElement { value: "6"; value2:"six"}
	             ListElement { value: "4"; value2:"four"}
	             ListElement { value: "3"; value2:"three"}
	 
	         }
	          //删除col1中和itemData相等的值
	         function remove(itemData) {
	             var rowCount = listModel.count;
	             for( var i = 0;i < rowCount;i++ ) {
	                 var data = listModel.get(i);
	                 if(data.value == itemData) {
	                     listModel.remove(i, 1);
	                 }
	             }
	         }
	         //排序(这边我只用冒泡排了序号，后面的value2懒得一起排，也可以用ListModel.move来排序，这样可以一步到位)
	         function sort(){
	             var rowCount = listModel.count;
	             for(var i = 0; i < rowCount; i++)
	                 {
	                     for(var j = 0; i + j < rowCount - 1; j++)
	                     {
	                         if(listModel.get(j).value > listModel.get(j+1).value)
	                         {
	                             var temp = listModel.get(j).value;
	                             listModel.get(j).value = listModel.get(j+1).value;
	                             listModel.get(j+1).value = temp;
	                         }
	                     }
	                 }
	         }
	         //通过value查找value2
	         function find(value1)
	         {
	             var rowCount = listModel.count;
	             for( var i = 0;i < rowCount;i++ ) {
	                 var data = listModel.get(i);
	                 if(data.value == value1) {
	                     console.log(data.value2)
	                 }
	             }
	         }
		 
	         delegate: Rectangle {
	             id: item
	             height: 25
	             width: 200
	             Row{
	                 Text {
	                     text: value
	                 }
	                 Text {
	                     text: value2
	                 }
	             }	 
	             MouseArea {
	                 anchors.fill: parent
	                 onClicked: {
	                     listView.currentIndex = index;
	                     //点击
	                     //visualModel.remove(value)
	                     //visualModel.sort();
	                     visualModel.find(value);
	                 }
	             }
	         }
	     }
	 
	     ListView {
	         id:listView;
	         anchors.fill: parent
	         model: visualModel
	     }	 
	 }

### 4、如果只是想从View上暂时的屏蔽过滤，或者分类可以使用DelegateModelGroup ###
