# 1. 阅读随手记
#### 第一章 整洁代码
LeBlanc lawer: Later equals never。我们写代码时总是会说下次改，有时候还加上TODO，但大部分都没有下次。如果当下可以做的事情，为什么要拖到下次？

破窗理论：这个是我曾经或者一直在犯的错误，如果已经看到代码不合理的地方，在改动这些代码时往往会给自己放纵的理由：反正这块现在由问题，不过是把问题给重复了一次，下次修改的时候一起优化就好了。导致整个项目的代码质量越来越差。

整洁的代码只做好一件事。

代码读和写：读与写的时间比例超过10:1。在写新代码时我们一直在读老代码，在写一个函数时，我们一直在读它的调用方，它的前提，它的后置等等。

编写的代码的难度取决于读周边代码的难度。

__让营地比你来时更干净__, __commit的代码比check out时更整洁__