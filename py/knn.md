---
title: 基于K-近邻算法（KNN）的验证码识别
tags:
  - python
  - 算法
category: 'python'
keywords: 'python,验证码,KNN'
date: 2019-03-24 00:35:05
---

# 基于K-近邻算法（KNN）的验证码识别

看《机器学习实战》第2章的时候觉得KNN可以把用来做验证码识别，然后查了看了下确实可以，然后就拿来练练手，本文代码有简单的封装，具体可以在[[代码地址](https://github.com/XuChaoChi/KNN)]([http://coding.net](https://github.com/XuChaoChi/KNN))上查看

KNN的优点：精确度高，对异常值不敏感，无数据输入假定(毕竟是取前K个点中间错1一个什么的差距不大)
KNN的缺点：计算复杂度高、空间复杂度高(因为要和每一个样本比较，如果有很多样本，空间和计算时间都不会理想)

## 原始的验证码 

![原始验证码](/blog/img/knn/1.png)

首先分析下处理过程
1. 颜色过多->灰度，然后二值化
2. 有噪点->8邻域降噪
3. 位置和大小固定->直接按照像素分割
4. 创建数据集
5. 使用KNN验证

<!--more-->

## 灰度与二值化

我选择的是skimage模块来处理灰度，因为色彩差距不大二值化的时候阈值选择了200，用126这个大众阈值效果不是很好

    def Conversion(self, imgPath):
          image = Image.open(imgPath)
          img_gray = image.convert('L')
          pixdata = img_gray.load()
          w, h = img_gray.size
          # 遍历所有像素，大于阈值的为黑色
          for y in range(h):
              for x in range(w):
                  if pixdata[x, y] < 200: #阈值200差不多
                      pixdata[x, y] = 0
                  else:
                      pixdata[x, y] = 255
          return img_gray

![灰度与二值化](/blog/img/knn/2.png)

## 8邻域降噪

因为噪点大多是孤立的所有可以遍历所有像素点，当被遍历的像素点周围（8个领域）小于n个黑色像素点的时候就可以把这个像素置为白

    def ClearNoise(self, imgData):
            w, h = imgData.size
            pixdata = imgData.load()
            for y in range(1, h-1):
                for x in range(1, w-1):
                    if pixdata[x, y] == 0:
                        points = []
                        count = 0
                        points.append(pixdata[x,y+1])
                        points.append(pixdata[x,y-1])
                        points.append(pixdata[x+1,y+1])
                        points.append(pixdata[x+1,y-1])
                        points.append(pixdata[x+1,y])
                        points.append(pixdata[x-1,y+1])
                        points.append(pixdata[x-1,y-1])
                        points.append(pixdata[x-1,y])
                        
                        for k in points:
                            if k == 0:
                                count += 1
                        if count < 4:
                            pixdata[x,y] = 255
            return imgData

![降噪结果](/blog/img/knn/3.png)

## 分割验证码

因为这次的验证码位置不变所以直接分割就行了，使用matplotlib打开可以看到高30，宽120。

    def CutImg(self, imgData):         
        for i, n in enumerate(self.cuts):
            tempImg = imgData.crop(n)
            tempImg.save("cut%s.png" % self.index)
            self.index += 1

![分割图片](/blog/img/knn/4.png)

## 创建数据集

本人取得1~9的数字样本后分别放到了1~9的文件夹中如图：

![测试样本](/blog/img/knn/5.png)

    def CreateDataSet():
      labelNames = fp.GetLabelsName()
      paths = fp.GetLabelsPath()
      group = np.zeros((0, 600))
      labels = []
      for i, path in enumerate(paths):
          filePaths = fp.GetFilesPath(path)
          for filePath in filePaths:
              labels.append(labelNames[i])
              img = plt.imread(filePath).reshape(-1)
              group = np.append(group, [img], axis=0)      
      return group, labels

## 使用KNN验证 



    #分类器
    def classify(inX, dataSet, labels, k):
      dataSetSize = dataSet.shape[0] #获取dataSet的大小（有几组数）
      diffMat = np.tile(inX, (dataSetSize, 1)) - dataSet #  欧氏距离 
      sqDiffmat = diffMat ** 2 
      sqDistances = sqDiffmat.sum(axis=1) 
      distances = sqDistances ** 0.5 
      sortedDistance = distances.argsort()#将array的元素排序

      classCount = {}
      for i in range(k):
          targetLebal = labels[sortedDistance[i]]#取对应label的数值
          classCount[targetLebal] = classCount.get(targetLebal, 0) + 1 #dict.get() 当targetLebal不存在时候返回0,然后+1
      sortedClassCnt = sorted(classCount.items(), key = operator.itemgetter(1), reverse=True) # 根据dict的value排序，即找到出现比较多的label
      return sortedClassCnt[0][0]


    def GetValidateCode(path, group, labels):
        imgHelp = ImgHelp()
        imgs = imgHelp.GetCutImg(path)
        strRet = ""
        for img in imgs:
            strRet += str(classify(np.asarray(img).reshape(-1), group, labels, 3))
        return strRet

## 正确率测试

测试了10组正确9组

    code:22592, result:True
    code:24879, result:True
    code:28155, result:True
    code:33858, result:True
    code:34764, result:False
    code:42547, result:True
    code:56299, result:True
    code:61865, result:True
    code:76238, result:True
    code:89975, result:True