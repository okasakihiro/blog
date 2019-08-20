---
title: Laravel对MongoDb数据的表单验证
date: 2019/8/13 15:03:28
categories: 后端
tag: [PHP, Laravel]
toc: true
---

> 这两天的开发过程真的是无比的兴奋，在很短的时间内就接触到了很多新的知识，同时也感谢朋友小红对我的指导。

# 无处不在的表单验证

说实话，我最开始学习PHP的时候就对表单验证产生过很多迷惑：
 * 把所有参数接过来再筛选？
 * 怎么限制参数类型比较好？
 * 怎么只获取我想要的参数？
 * 前后端都要有验证的必要性？

等等诸如此类的问题，我已经考虑过不知道多少回了。

这次再开发的时候，依旧存在上述的疑问。

不过好在这次请教了一下朋友，结合 `Laravel` 的文档，整理出了一套新的表单验证方法。

# 眼是懒汉手是好汉

表单验证的另一个问题，就是麻烦。

但是再麻烦，表单验证也必须要做（随便相信用户传过来的数据可不行哦）。

这其实是一个不可避免的问题，你简单了，规则就简单了，简单的规则又能起到多大程度的验证效果呢？

首先从思路上来讲，主要是一下两个步骤：

1. 只取我想要的参数，不管用户传递了什么参数。
2. 对参数类型进行限制，只有符合条件的参数才会被传进来。

一般情况在经过上面两步的过滤后，基本就可以拿到符合我们想要的参数了。

# 具体实现过程

首先我们先看第一部分，`只取我想要的参数，不管用户传递了什么参数`。

这里我们用最传统的方法先来描述一下：
```PHP
  if(array_key_exists($_POST['id'])) {
    $id = $_POST['id']; 
  }
```
这个算是比较原始获取请求参数的方法了，这里通过数组的 `key` 可以只获取我想要的参数。

接下来问题就出现了，如果参数是个数组怎么办？总不能一个一个写吧？

这里可以用到 `Laravel Eloquent ORM` 的一个特性： `Mutators` (修改器)

这里我们先看一下修改器(Mutator)的概念：

> 若要定义一个修改器，则需在模型上面定义 setFooAttribute 方法。要访问的 Foo 字段使用「驼峰式」命名。让我们再来定义一个 first_name 属性的修改器。当我们尝试在模式上在设置 first_name 属性值时，该修改器将被自动调用。

同时结合 `Laravel` 的 `collect`  (集合) 来实现请求参数的过滤：

下面下一个例子：

```PHP
  //controller
  $userDescribe = [
    'age' => 26,
    'is_vip' => 0,
    'name' => 'xxa',
  ];
  $userModel = new User();
  $userModel->user_describe = $userDescribe;

  //model
  //该修改器意味着在向 userModel 中存入 user_describe 时，会对这个变量进行相关的操作
  public function setUserDescribeAttribute($value) 
  {
    //将参数转化为集合进行操作
    $collection = collect($value);
    //only方法意味着只获取only参数中的参数内容
    $filtered   = $collection->only([
      'age',
      'is_vip',
    ]);
    //结果会将name这个参数过滤掉
    $this->attributes['evaluation_detail'] = $filtered->toArray();
  }
```

通过上面的组合，可以有效控制类似数组等类似集合的数据的字段，可以节省大量判断代码。

第一步已经完成，那第二步我们`需要对变量的类型进行判断`。

当然继续沿用上面的修改器也可以实现，但 `Laravel` 为我们封装了更简单的方法： `Validation` （表单验证）

> Laravel 提供了几种不同的方法来验证传入应用程序的数据。默认情况下，`Laravel` 的控制器基类使用 `ValidatesRequests Trait`，它提供了一种方便的方法去使用各种强大的验证规则来验证传入的 `HTTP` 请求。

表单验证有很多中实现方法，为了使程序结构更加规整，这里使用了`验证表单请求`的验证方法:

> 面对更复杂的验证情境中，你可以创建一个「表单请求」来处理更为复杂的逻辑。表单请求是包含验证逻辑的自定义请求类。可使用 `Artisan` 命令 `make:request` 来创建表单请求类：
> 
> php artisan make:request Supplier

每一个表单请求验证都有一些继承的方法：
```PHP
/**
 * 获取适用于请求的验证规则。
 * 此部分内容为重要部分。
 *
 * @return array
 */
public function rules()
{
    return [
        'title' => 'required|unique:posts|max:255', //验证规则
        'body' => 'required',
    ];
}
```

在 `rules()` 方法中，针对每个字段进行相关的条件约束，便可以实现表单验证功能。

之后在控制器中，修改相关的请求实例，并调用验证方法即可：

```PHP
//controller
public function add(Supplier $request)
{
  //使用Supplier的实例来验证其rules中定义的规则
  $request->validated();
  //...
}
```

这样就实现了验证表单请求的过程。

通过以上两步，基本可以是实现对请求参数的控制。

# 说在最后

虽然本身在保存数据时是有数据库约束的，但上面的方法适用于如：MongoDb等数据库（本次原因）。不过即便是存在数据库约束，在开发过程中验证用户提交的数据，终究是有必要的。

整个过程描述的比较简略，具体实现可以参考 `Laravel` 文档。

注：前后端都要验证！