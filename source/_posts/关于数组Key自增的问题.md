---
title: 关于数组Key自增的问题
date: 2019/7/11 22:53:36
categories: 后端
tag: [PHP, 数据结构]
---

昨晚刷LeetCode，遇到了需要实现 **栈** 的方式，脑中浮现出两个方法：
使用数组来实现出栈，具体方法如下：
 1. 入栈
    ```PHP
        array_push();           //向数组末尾押入数据
        $array[] = $variable;   //同上，但此方法比调用函数更快
    ```
 2. 出栈
    1. 使用自带函数：
    *  ```PHP
        array_pop(); // 弹出数组最后一个元素
        //或
        reset();end(); //操作数组指针
        ```
    2. 手动操作key进行弹出
    *  ```PHP
        $currentValue = $array[count($array) - 1];
        unset($array[count($array) - 1]);
        ```

使用自带函数很轻松的就实现了想要的结果，当我尝试用第二种方法实现的时候却得到了意外的结果：
```PHP
class Solution{
    /**
     * 简单举例模拟进出栈
     * 栈为空时，进栈，如果栈不为空，先出栈相加再进栈
     */
    public function stack() :void
    {
        $number = [1, 2, 3, 4];  //初始数组
        $stack = [];             //空栈
        for ($i = 0;$i < count($number); $i++) {
            if(empty($stack)) {
                //栈为空，数字进栈
                $stack[] = $number[$i];
            } else {
                //栈不为空，数字出栈相加再进栈
                $temp = $stack[count($stack) - 1]; //强行出栈，不做'+='
                unset($stack[count($stack) - 1]);  //完成出栈模拟
                echo $temp . PHP_EOL;
                $stack[] = $number[$i] + $temp;    //注意此处入栈
            }
        }
    }
}
```
此时运行这段代码，PHP会抛出一个 `Notice` 级别的错
误：
> PHP Notice: Undefined offset: 0 

原因是在 `temp = $stack[count($stack) - 1];` 这一行代码中，未找到数组 `key` 对应的 `value` 值，也就是说，没找到 `$stack[0]` 的值？

讲到这里其实有的人可能已经反应过来了，不过接着往后讲：
首先在 `$temp` 这个变量前添加 `@` 符号来抑制警告输出，此时的输出结果：
```PHP
1
3
```

嗯？正常的输出应该是 ‘1，3，6，10’ 啊，后面的 ‘6’ 和 ‘10’ 为什么不见了？

答案就在于，如果使用一开始表示的模拟入栈方法，那么数组的 `key` 是会自增的。

也就意味着，在循环第二次，也就是第一次出栈的时候，我们 `unset()` 的是 `$stack[0]`。

但是在第二次出栈时，插入的 `$temp` 对应在 `$stack` 变量内， 已经是 `$stack[1]` 了，但我们获取的仍旧是 `$stack[0]` 原因是 `$stack[count($stack) - 1]`），所以在不显示通知提示时，从 `$stack[0]` 获取的值是 `NULL`，所以得到了上面的结果。

对于这种问题，可以使用一个变量来充当指针，根据入栈次数进行自增，这样在模拟出栈的时候就不会出现取到不存在的 `key` 的情况了。当然也可以使用自带函数来实现。

