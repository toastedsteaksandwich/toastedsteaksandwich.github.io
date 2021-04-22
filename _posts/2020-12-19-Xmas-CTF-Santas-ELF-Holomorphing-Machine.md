---
title: Xmas CTF 2020 - Santa's ELF Holomorphing Machine Writeup
author: Ashiq Amien
date: 2020-12-19 14:00:00 +0200
categories: [CTF, Programming]
tags: ctf
math: true
---

## Brief


Santa's ELF holomorphing machine was a programming challenge described by the following brief:

![Challenge brief](/assets/img/sample/santas-holomorphing-elf-machine-brief.png)

We're also given a text file with about 800 functions and complex numbers, as follows:

```
u = -3 * x + 95 * y; z = -0.12652202789462033 + 0.006530883329643569 * i
v = -65 * x + 5 * y; z = -0.16588235294117648 + 0.04352941176470588 * i
u = 55 * x + -93 * y; z = 0.09379818399862944 + 0.023213979784135686 * i
u = 71 * x + -26 * y; z = 0.09060696169319574 + 0.09358054923911142 * i
v = 78 * x + -38 * y; z = -0.01487778958554729 + -0.056854410201912856 * i
...
```

## Objective

After reading the brief, it looks like we need to find some function $$ f_i $$ for each given complex number $$z_i$$, and then plot the points on a normal x-y plane. At this point I wasn't sure what a [holomorphic function](https://en.wikipedia.org/wiki/Holomorphic_function) was, so I did some reading and found that it has to do with complex differentiation - which wasn't very helpful. However, all holomorphic functions have the following property:


If a complex function $$ f(x + iy) = u(x ,y) + iv(x, y)$$ is holomorphic, then $$u$$ and $$v$$ have first partial derivatives with respect to $$x$$ and $$y$$, and satisfy the Cauchy-Riemann equation:

\begin{equation}
\frac{\partial u}{\partial x} = \frac{\partial v}{\partial y},    \frac{\partial u}{\partial y} = -\frac{\partial v}{\partial x}
\end{equation}

Since we're provided with either $$u$$ or $$v$$, we can use the above equation to work out the other corresponding part, and join it to have $$ f_i(x + iy) = u(x ,y) + iv(x, y)$$ for each $$z_i$$. As an example, $$u = -3 * x + 95 * y;$$ has the following partial derivatives:

\begin{equation}
\frac{\partial u}{\partial x} = -3, \frac{\partial u}{\partial y} = 95,
\end{equation}

So by the Cauchy-Riemann equation, we have:

\begin{equation}
\frac{\partial v}{\partial y} = -3, -\frac{\partial v}{\partial x} = 95,
\end{equation}

## Piecing it together

We can piece this together to find that $$ v = (-95) * x + (-3) * y$$. We'd then send $$z_i$$ to the new $$ f_i(x + iy) = u(x ,y) + iv(x, y)$$ for each $$i$$ and plot the corresponding (x,y) points. I wrote the following code to do this:

```python
#!/usr/bin/python3
from pylab import *
xpoints = []
ypoints = []

lines = open("holoinput.txt").read().split('\n')
for line in lines:
	if line.startswith("u"):
		u = line.split(';')[0] #get just the u = ax + by
		ua = u.split(' ')[2] #get just a
		ub = u.split(' ')[6] #get just b
		va = float(ub)*(-1) #v = -bx + ay
		vb = float(ua) #v = -bx + ay

		#now we need to send in z into u and v, to get f(z) 
		z = line.split(';')[1] #just get z = a + bi
		za = z.split(' ')[3] #we just need the a from z = a + bi
		zb = z.split(' ')[5] #we just need the b from z = a + bi
		
		#work out u(za,zb)
		uz = float(ua) * float(za) + float(ub) * float(zb)
		
		#work out v(za,zb)
		vz = float(va) * float(za) + float(vb) * float(zb)
		
		#the affix is (u(za,zb), v(za,zb))
		print("(" + str(uz) + ", " + str(vz) + ")")
		xpoints.append(uz)
		ypoints.append(-vz) #need to flip all points about the y-axis

	#do the same as above, but with the v functions
	if line.startswith("v"):
		v = line.split(';')[0]
		va = v.split(' ')[2]
		vb = v.split(' ')[6]
		ua = float(vb)
		ub = float(va)*(-1)
		z = line.split(';')[1]
		za = z.split(' ')[3]
		zb = z.split(' ')[5]
		uz = float(ua) * float(za) + float(ub) * float(zb)
		vz = float(va) * float(za) + float(vb) * float(zb)
		print("(" + str(uz) + ", " + str(vz) + ")")
		xpoints.append(uz)
		ypoints.append(-vz) #need to flip all points about the y-axis

scatter(xpoints, ypoints, marker='.')
show()
```

After running the program, I found that we needed to flip the resulting scatter plot about the y-axis, which is why we have `ypoints.append(-vz)`. The resulting scatter plot is the flag:


![The flag](/assets/img/sample/santas-holomorphing-elf-machine-flag.png)


Thanks to [@HTSP](https://twitter.com/htspctf) and the author for a fun challenge :)
