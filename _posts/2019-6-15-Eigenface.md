---
layout: post
title:  "人脸识别之特征脸方法Eigenface及python实现"
date:   2019-06-16 1:46:00 +0700
categories: zijin update
---

## 读入图片并降维
首先，调用cv.imread函数读入待降维图片，考虑到源图片虽大小不一但总体相差无几，故以读入的第一个图片大小为标准（static_shape）通过cv.resize函数统一所有图片的大小。然后调用numpy库的reshape函数将图像矩阵降为一维，最后把降维后的函数加入存储结果的list中。以上步骤重复数次，直至所有图片都被读入。最终得到M\*N^2矩阵.

## 求均值人脸
这一工作由numpy库中的mean函数实现。在本例中，它接收三个参数，第一个为输入矩阵，第二个用于判断是行求平均还是列求平均（缺省表示总平均）。第三个为期望输出的数据类型.

## 求更新后人脸
这一工作由numpy库中的subtract函数实现，其功能是输出同等大小矩阵对应位置相减后的矩阵。将步骤1中获得的矩阵各行同均值人脸依次相减，所得结果存入数组，就获得了更新后的人脸矩阵。按照书本上的要求，该矩阵记为A.

## 求特征向量
在求特征向量之前，首先使用A.T方法求得该矩阵的转置矩阵，然后使用numpy库的dot() 函数求得AAT（注意：这里的和教材中相反，因此M*M矩阵是通过AAT求得的）。最后调用numpy库linalg模块中的eig函数（返回一个方阵的特征值和特征向量（元组方式），支持虚数计算，非方阵会报错）获取AAT的特征向量并存入数组中。求特征人脸AT和前六个特征向量依次相乘，得到每个特征人脸的 1\*N^2$向量。复原特征人脸并输出。将上一步得到的一维特征人脸按照标准大小reshape，重新变成二维矩阵后以图像形式输出.

## Python 代码

```Python
'''

Numpy library: Provides a way to handle matrices

Linaig module: used to calculate eigenvalues and eigenvectors

opencv2.0: graphics library

'''



from numpy import *
from numpy import linalg as la
import cv2 as cv



'''

 function: img2vector

 usage: temp_array = img2vector(temp_img)

 This function calls the reshape function 

 in numpy to reduce the two-dimensional matrix

 to a one-dimensional vector

'''





def img2vector(img):
    rows, cols = img.shape
    imgVector = zeros((1,rows*cols)) 
    imgVector = reshape(img,(1,rows*cols))
    return imgVector



'''

 function: average

 usage: array_average = average(array1)

 Calculate the integer mean value 

 of the row vector of the matrix

'''



def average(array):
    return mean(array,0,int)



'''

 function: refine

 usage: array_refine - refine(array_average, array1)

 Calculate the mean face

'''



def refine(array_average, array1):
    refine1 = []
    for member in array1:
    temp_refine = subtract(member, array_average)
    refine1.append(temp_refine)
    return refine1



####################### global scope #######################



# read image



static_shape = cv.imread("face1.jpg",0).shape



# The 36 images to be processed are dimensionally reduced



array1 = []

for i in range(0, 36):
    temp_img = cv.imread("face" + str(i + 1) + ".jpg",0) 
    # Since the image sizes are not the same, 
    # resize all the images against the size of the first image
    temp_img = cv.resize(temp_img,static_shape)
    temp_array = img2vector(temp_img)
    array1.append(temp_array[0])



# Find the updated matrix



array_average = average(array1)
array_refine = refine(array_average, array1)



# Matrix transpose

array_refine = array(array_refine)
array_tran = array_refine.T



# Dot matrix multiplication



array_multi = dot(array_refine, array_tran)



# calculate eigenvalues and eigenvectors



u,v= linalg.eig(array_multi)



# Using eigenvector and original matrix to calculate characteristic face



result1 = []

for i in range(0, 6):
    temp_m = dot(array_tran, v[i])
    result1.append(temp_m)



# Change the data type to int



for i in range(0, 6):
    for j in range(0, static_shape[1] * static_shape[0]):
        result1[i][j] = int(result1[i][j])



# Reshape the two-dimensional matrix and save it as a picture



for i in range(0, 6):
    temp_r = reshape(result1[i],static_shape)
    cv.imwrite("result"+str(i)+".jpg", temp_r)


```

.END