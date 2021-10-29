## 1、业务逻辑

新建某个模型之后，利用 Observer 模型事件 Created 推入异步短信发送队列

```App\Http\Controllers\UsersController```

```php
    public function store(User $user)
    {
        \DB::beginTransaction();
        
        try{
            $input = request()->validated();
            $user->fill($input);
            $user->save();
            //do something......
            //其他数据表操作
        
        	\DB::commit();
        } catch ($e \Exception) {
            \DB::rollBack();
        }
        
    }
```

```App\Observers\UsersObserver``` 

```php
class UsersObserver
{
    public function created (User $user)
    {
        dispatch(new SmsQueue($user));
    }
}
```

## 2、发现异常

业务部门反馈偶尔有用户收取不到短信通知，我便查看日志发现偶尔有错误异常：No query results for model [App\Models\User]. 表示找不到对应的模型

我敲不应该啊，我是在创建模型之后再进行队列调用……，遂对业务代码再进行仔细核查猜测应该是受事务影响。

**验证猜想：**

```php
    public function store(User $user)
    {
        \DB::beginTransaction();
        
        try{
            $input = request()->validated();
            $user->fill($input);
            $user->save();
            //do something......
            //其他数据表操作
        
            sleep(3); //三秒之后再提交事务            
        	\DB::commit();
        } catch ($e \Exception) {
            \DB::rollBack();
        }
        
    }
```

果然在等待三秒之后提交队列异常已是 100%  触发。

## 3、原因分析

1.  $user->save() 这个方法创建数据成功，便会一并触发调度器，将模型事件一一执行。
2. 在事件中推送模型至队列中，而队列进程在不间断消费队列中数据。
3. 在大部分情况下 do something 处理速度正常的话，队列进程将会照常运行。
4. 如果在 do something 阶段偶尔出现延迟，造成事务还未 commit 而队列已经开始消费新模型；故引发上述错误。

然后我在搜索 Github Issues 记录时，发现此问题在 2015 年的一个 [Issue](https://github.com/laravel/framework/issues/8627) 已经有人提出，而在 Laravel 8.X 中终于新增了对事务模型事件的支持；[文档地址](https://laravel.com/docs/8.x/eloquent#observers-and-database-transactions)，在社区文档似乎并没有找到相关说明~

由于我的版本是 6.x 所以用不了这个新特性[哭唧唧]~~

## 4、解决异常



## 4、解决异常

## 4、解决异常

## 4、解决异常

### 1. 修改 MySQL 事务隔离级别（不推荐）

这里涉及到 MySQL 的事务隔离级别，InnoDB 引擎的默认隔离级别是 REPEATABLE READ，关于各个级别的区别可以在[官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)找到。

将隔离级别切换到 READ UNCOMMITTED 即可解决此问题，但是为了防止出现更大的问题我劝你别用这种方式~

### 2. 增加事件监听

查看源码得知在事务完成之后，会调用对应的事件，所以只需增加对事件的监听即可。

![image-20211029160303394](https://cdn.lidong-myself.cn/typora/image-20211029160303394.png)

1. 新增类 ```App\Handlers\TransactionHandler``` 

   ```php
   class TransactionHandler
   {
       public array $handlers;
   
       public function __construct()
       {
           $this->handlers = [];
       }
   
       public function add(\Closure $handler)
       {
           $this->handlers[] = $handler;
       }
   
       public function run()
       {
           foreach ($this->handlers as $handler) {
               $handler();
           }
       }
   }
   ```

2. 创建辅助函数  ```app/helpers.php```

   ```php
   
   if (! function_exists('after_transaction')) {
       /*
        * 事务结束之后再进行操作
        * */
       function after_transaction(Closure $job)
       {
           app()->singletonIf(\App\Handlers\TransactionHandler::class, function (){
               return new \App\Handlers\TransactionHandler();
           });
           app(\App\Handlers\TransactionHandler::class)->add($job);
       }
   }
   
   ```

3. 创建监听  ```App\Listeners\TransactionListener```

   ```php
   
   namespace App\Listeners;
   
   use App\Handlers\TransactionHandler;
   
   class TransactionListener
   {
       public function handle()
       {
           app(TransactionHandler::class)->run();
       }
   }
   ```

4. 绑定监听 ```App\Providers\EventServiceProvider```

   ```php
   
   namespace App\Providers;
   
   use App\Listeners\TransactionListener;
   use Illuminate\Database\Events\TransactionCommitted;
   use Illuminate\Database\Events\TransactionRolledBack;
   use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;;
   
   class EventServiceProvider extends ServiceProvider
   {
       /**
        * The event listener mappings for the application.
        *
        * @var array
        */
       protected $listen = [
           TransactionCommitted::class => [
               TransactionListener::class
           ],
           TransactionRolledBack::class => [
               TransactionListener::class
           ]
       ];
   }
   
   ```

5. 更改调用方式  ```App\Observers\UsersObserver``` 

```php
class UsersObserver
{
    public function created (User $user)
    {
        after_transaction(function(){
            dispatch(new SmsQueue($user));
        });
    }
}
```

OK，一个优雅的解决方式就完成了~~