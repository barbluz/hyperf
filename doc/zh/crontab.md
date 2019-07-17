# 计划任务

通常来说，执行计划任务会通过 Linux 的 `crontab` 命令来实现，但现实情况下，并不是所有开发人员都能够拥有生产环境的服务器去设置计划任务的，这里 [hyperf/crontab](https://github.com/hyperf-cloud/crontab) 组件为您提供了一个 `秒级` 计划任务功能，只需通过简单的定义即可完成一个计划任务的定义。 

# 安装

```bash
composer require hyperf/crontab
```

# 使用

## 启动任务调度器进程

在使用计划任务组件之前，需要先在 `config/autoload/processes.php` 内注册一下 `Hyperf\Crontab\Process\CrontabDispatcherProcess` 自定义进程，如下：

```php
<?php
// config/autoload/processes.php
return [
    Hyperf\Crontab\Process\CrontabDispatcherProcess::class,
];
```

这样服务启动时会启动一个自定义进程，用于对计划任务的解析和调度分发。如果您没有注册该进程，亦可理解为您关闭了计划任务功能。

## 定义计划任务

### 通过配置文件定义

您可于 `config/autoload/crontab.php` 的配置文件内配置您所有的计划任务，文件返回一个 `Hyperf\Crontab\Crontab[]` 结构的数组，如配置文件不存在可自行创建：

```php
<?php
// config/autoload/crontab.php
use Hyperf\Crontab\Crontab;
return [
    (new Crontab())->setName('Foo')->setRule('* * * * *')->setCallback([App\Task\FooTask::class, 'execute'])->setMemo('这是一个示例的计划任务'),
];
```

### 通过注解定义

通过 `@Crontab` 注解可以快速完成对一个任务的定义，以下的定义示例与配置文件定义所达到的目的都是一样的。定义一个名为 `Foo` 每分钟执行一次 `App\Task\FooTask::execute()` 的计划任务。

```php
<?php
namespace App\Task;

use Hyperf\Contract\StdoutLoggerInterface;
use Hyperf\Crontab\Annotation\Crontab;

/**
 * @Crontab(name="Foo", rule="* * * * *", callback="execute", memo="这是一个示例的计划任务")
 */
class FooTask
{

    /**
     * @Inject()
     * @var \Hyperf\Contract\StdoutLoggerInterface
     */
    private $logger;

    public function execute()
    {
        $this->logger->info(date('Y-m-d H:i:s', time()));
    }
}
```

### 任务属性

#### name

计划任务的名称，可以为任意字符串，各个计划任务之间的名称要唯一。

#### rule

计划任务的执行规则，在分钟级的定义时，与 Linux 的 `crontab` 命令的规则一致，在秒级的定义时，规则长度从 5 位变成 6 位，在规则的前面增加了对应秒级的节点，如 `*/5 * * * * *` 则代表每 5 秒执行一次。注意在注解定义时，规则存在 `\` 符号时，需要进行转义处理，即填写 `*\/5 * * * * *`。

#### callback

计划任务的执行回调，即计划任务实际执行的代码，在通过配置文件定义时，这里需要传递一个 `[$class, $method]` 的数组，`$class` 为一个类的全称，`$method` 为 `$class` 内的一个 `public` 方法。当通过注解定义时，只需要提供一个当前类内的 `public` 方法的方法名即可，如果当前类只有一个 `public` 方法，您甚至可以不提供该属性。

#### memo

计划任务的备注，该属性为可选属性，没有任何逻辑上的意义，仅供开发人员查阅帮助对该计划任务的理解。

### 调度分发策略

计划任务在设计上允许通过不同的策略来调度分发执行任务，目前仅提供了 `多进程执行策略`、`协程执行策略` 两种策略，默认为 `多进程执行策略`，后面的迭代会增加更多更强的策略。   

#### 更改调度分发策略

通过在 `config/dependencies.php` 更改 `Hyperf\Crontab\Strategy\StrategyInterface` 接口类所对应的实例来更改目前所使用的策略，默认情况下使用 `多进程执行策略`，对应的类为 `Hyperf\Crontab\Strategy\ProcessStrategy`，如我们希望更改策略为一个新的策略，比如为 `App\Crontab\Strategy\FooStrategy`，那么如下：

```php
<?php
return [
    'dependencies' => [
        \Hyperf\Crontab\Strategy\StrategyInterface::class => \App\Crontab\Strategy\FooStrategy::class,
    ],
];
```

##### 多进程执行策略

策略类：`Hyperf\Crontab\Strategy\ProcessStrategy`   

默认情况下使用此策略，即为 `CrontabDispatcherProcess` 进程解析计划任务，并通过进程间通讯轮训传递执行任务到各个 `Worker` 进程中，由各个 `Worker` 进程以协程来实际运行执行任务。

##### 协程执行策略

策略类：`Hyperf\Crontab\Strategy\CoroutineStrategy`   

默认情况下使用此策略，即为 `CrontabDispatcherProcess` 进程解析计划任务，并在进程内为每个执行任务创建一个协程来运行。