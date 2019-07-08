---
title: 关于函数 array_key_exists() 和 isset() 判断数组中 key 的区别
date: 2019/6/13 13:50:12
categories: 后端
tag: PHP
toc: true
---

这几天在开发过程中频繁使用到了 `array_key_exists()` 函数。
虽然以前也在用，但是今天写着写着突然想到，如果判断存不存在的话，那用 `isset()` 岂不是也可以？
但在实际替换之后却发现了下面的问题：

```php
<?php
    //初始化一个数组
    $array = [
        'name' => 'avatar',
        'hash' => null
    ];
    var_dump(array_key_exists('name', $array));             //true
    var_dump(isset($array['name']));                        //true

    var_dump(array_key_exists('hash', $array));             //true
    var_dump(isset($array['hash']));                        //false

    var_dump(array_key_exists('not exist key', $array));    //false
    var_dump(isset($array['not exist key']));               //false
    die;
```
可见问题一目了然，当使用 `isset()` 检测数组中 `key` 是否存在时，
如果被检测的 `key` 对应的值为 `null` ，那么 `isset()` 会返回 `false` ，
这和我们的想要的结果肯定是不一致的。
⬇具体原因在 [PHP手册-isset()函数](https://www.php.net/manual/zh/function.isset.php) 中有详细的解释。
**isset() --- 检测变量是否已设置并且非 `NULL` 。**

> 总结：判断数组中的 `key` 是否存在，还是应该乖乖的使用 `array_key_exists()` 函数。
