## 阵列编程

如果看 Vectorization 的 wiki 页面，有好多种解释，我们来看第一个：

- Array programming, a style of computer programming where operations are applied to whole arrays instead of individual elements

这个中文又叫做阵列编程：

> 在计算机科学中，阵列编程指称允许向作为整体的一组数值同时应用运算操作的一种解决方案。这种方案经常用于科学和工程上的各种场合（settings）。

看它举的例子，比如如果我们要加两个 array，在一些语言中，我们需要：

```
for (i = 0; i < n; i++)
    for (j = 0; j < n; j++)
        a[i][j] += b[i][j];
```

但是在一些 array based languages，我们可以：

```
a = a + b
```

或者

```
a(:,:) = a(:,:) + b(:,:)
```


以上的语法让我们想起了 Numpy，的确， Numpy 可以让我们做到类似的操作，这也就是 Python Vectorization 主要使用的工具.

这篇文章的标题~lol [Look Ma, No For-Loops: Array Programming With NumPy](https://realpython.com/numpy-array-programming/#reader-comments)


## 例子

这里我来写一个自己使用过程中的例子，比如我有一堆离散点 pt[0], pt[1], ..., pt[n-1]，我想求这堆点的曲率。


首先先来看离散点的曲率，因为三点总能构成一个圆，所以我就用 [Menger curvature](https://en.wikipedia.org/wiki/Menger_curvature) 好了，三个点分别为 x,y,z,其中 A 是三角形的面积：


$$ 
c(x,y,z)={\frac {1}{R}}={\frac {4A}{|x-y||y-z||z-x|}}
$$

所以我可以写出三点的曲率的函数，然后 pt[0],...pt[n-1] 这 n 个点可以得到 n-2 个曲率值：


```python
def menger_curvature(x, y, z):
    '''
    https://en.wikipedia.org/wiki/Menger_curvature
    Given:
        x, y, z : 3 points
    Return:
        menger curvature    
    '''
    x = np.asarray(x)
    y = np.asarray(y)
    z = np.asarray(z)
    
    a = np.linalg.norm(x - y)
    b = np.linalg.norm(y - z)
    c = np.linalg.norm(z - x)
    
    area = np.linalg.norm( np.cross(x - y, z - x) )
    
    if area < 1e-20: return 0
    return 2 * area / ( a * b * c)


def curvature_along_curve( samples ):
    '''
    Given:
        samples: An array of many points
    Returns:
        curvature_along_curve: An array of curvature along the curve. 
        It will have length two smaller.
    '''
    
    ### 1. compute curvatures along the curve
    ### 2. result[i] = curvature[i+1] - curvature[i]
   
    curvature = []
    for i in range(len(samples)-2):
        curvature.append( menger_curvature(samples[i], samples[i+1], samples[i+2]) )
    
    return curvature

```

也就是每连续的三个点我们来计算 menger curvature，然后再把它填到 list 中，这里我还是用之前的 clothoid 函数来取一堆点，然后来计算：


```python
import timeit
start = timeit.default_timer()

curvature = curvature_along_curve(points)

elapsed = (timeit.default_timer() - start)
print('elapsed', elapsed)
```


取了1000个点，计算用了 0.06735321700011809s，看起来还行。让我们一起来看看 Vectorization 的魅力吧。

首先，我们每次都是取连续的三个点，也就是我们的 x，y，z 总是连续着的三个点，三角形的三个边是 a = x - y, b = y - z, c = x - z. 其次，对于面积的计算我们可以用叉积，当然，对于面积极小的三角形我们做特殊处理，直接认为它们处于同一直线，所以曲率返回为0.

按照上面的思路写下：


```python
def menger_curvature_everywhere(pts):
    '''
    Given:
        pts: A polyline of 2D points
    Returns:
        The menger curvature at pts[1], pts[2], ..., pts[n-1]
    '''
    
    pts = np.asarray(pts)
    
    xy = (pts[:-1] - pts[1:])[:-1]
    yz = (pts[:-1] - pts[1:])[1:]
    zx = pts[2:] - pts[:-2]
    
    a = ( xy**2 ).sum( 1 )
    b = ( yz**2 ).sum( 1 )
    c = ( zx**2 ).sum( 1 )
    
    
    area = np.cross( xy, zx ) ** 2 
    result = 2 * np.sqrt( area / ( a * b * c) )
    result[ area < 1e-20 ] = 0.
    
    return result
```

计算：

```python
import timeit
start = timeit.default_timer()

v_curvature = menger_curvature_everywhere(points)

elapsed = (timeit.default_timer() - start)
print('elapsed', elapsed)
```


花的时间仅为 0.00071410199994752475s，算是区别很大了。同时，我也可以验证一下 v_curvature 和 curvature 相同。

我可以直接用 allclose：


```python
np.allclose(v_curvature, curvature) # → True
```

给我了相等的答案，如果我还有怀疑和疑惑，我也可以回到 for loop 验证，o(╯□╰)o


```python
for i in range(len(curvature)):
    print(curvature[i] - v_curvature[i])
```



可以看到一堆 0 和 1e-17, 1e-16的数量级，用下面的 for loop 直接验证它们求得的每个数之间都相差很小。


```python
for i in range(len(curvature)):
    if (curvature[i] - v_curvature[i]) > 1e-15:
        print('ith curvature differs greater than 1e-15')
```

当然我们还可以继续 vectorize 求曲率的变化，比如使用 for loop 我们需要：


```python
def variation_of_curvature( curvature ):
    '''
    Given:
        curvature: 
    Returns:
        variation_of_curvature: An array of variation-of-curvature values computed along the curve. 
        It will have length one smaller.
    '''

    result = []
    for i in range(len(curvature)-1):
        result.append(curvature[i+1] - curvature[i])
    return result
```

使用 for loop 需要 0.0002464679998865904s。

如果 vectorize 的话，我们直接就用 


```python
v_result = v_curvature[1:] - v_curvature[:-1]
```

只需要 8.216299966079532e-05s



```python
np.allclose(result, v_result) # -> still True
```


Cool!

