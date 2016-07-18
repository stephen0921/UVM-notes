# 后门访问，可以通过read/write （指定参数是back-door）或者peek/poke。
## 如果使用read/write，寄存器的动作将尽可能的模拟出来，比如wc，rc。但使用peek/poke，寄存器的动作不会理会，相当于直接force。

# set需要后面加上update。
