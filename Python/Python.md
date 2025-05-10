python官方教程，最权威的学习地址：https://docs.python.org/zh-cn/3.13/tutorial/index.html

[[python包]]

[输入输出格式化、文件读写、json](https://docs.python.org/zh-cn/3.13/tutorial/inputoutput.html#formatted-string-literals)

[异常处理](https://docs.python.org/zh-cn/3.13/tutorial/errors.html)
```python
try:
  raise NameError('HiThere') #BaseException是所有异常的共同基类,它的一个子类， Exception ，是所有非致命异常的基类
except NameError:
  print('An exception flew by!')
  raise #不打算处理该异常，则可以使用更简单的 [`raise`](https://docs.python.org/zh-cn/3.13/reference/simple_stmts.html#raise) 语句重新触发异常
else:
  print() # 适用于try子句没有引发异常但又必须要执行的代码
finally:
  print('finally') # 最后执行，没有异常情况下，即先执行else
```

[[asyncio]]
