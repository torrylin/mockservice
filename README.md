mockservice
===========

> 构造数据服务

---------------------------

## 安装与配置


### 选择npm安装

> npm install mockservice

### gitub安装 (需要支持git协议)

> npm install git://github.com/fcfe/mockservice.git

### 配置edp服务器 

/// (如果开发依赖不是是edp环境，绕道[mock-cli](#mock-cli))

> 配置方法: 在edp-webserver-config文件中添加如下代码

```js
    exports.port = 8848;

    var ms = require('mockservice');
    ms.config({
        // dir 相当于定义了mock的basedir 及require的baseUrl
        dir: __dirname + '/phoenix/debug'
    });

    // edp 通过getLocations 路由请求处理器
    exports.getLocations = function () {
        return [
            { // 将特定请求转向代理
                location: /path=GET\/nikon/,
                handler: ms.proxy({
                        replace: { // 对url的替换规则
                            source: '/nirvana-workspace',
                            target: ''
                        },
                        host: 'dev.liandong.org',
                        port: 8848
                    });
            },
            { // 其它请求转为本地mock
                location: /^\/request.ajax/, 
                handler: ms.request()
            }
        ];
    };
```

- 启动服务器

> edp ws start

- 使用edp时建议mocksrvice安装在项目代码上一级目录, 如`F://fengchao/node_modules`

### mock-cli

- 如使用独立服务器; 需要全局安装 `npm install mocksrvice -g`; 

    // 进入工作路径
    cd workspace
    
    // 启动mock服务器
    mock 8848

### ms-config.js配置

> ms-config.js 文件是可选文件，配置是为了使用更方便
> ms-config.js 文件可以放到目标文件夹下；只影响当前文件夹或子文件夹下的模块

> 详细配置参考 [config 详细说明](https://github.com/linkwisdom/mockservice/blob/master/docs/config.md);

```js
    module.exports = {
        // mock接口文件是否缓存
        cache: false,
        
        // 接口匹配规则
        pathRegs: [/\w+_\w+/, 'scookie', 'zebra'],
        
        // 以下配置只在baseDir中有效
        // packages 定义了基于basedir的寻址方式
        packages: {
            'lib': './lib',
            'tpl': './template'
        },
        // 如果不写logError，则错误信息不显示输出
        logError: {
            // 如果不指定logFile，则将错误信息输出到控制台
            logFile: 'ms-erorr.log'
        }
    };
```

----------------------------

### 使用说明

 > 启动程序后ms自动扫描目录下所有index.js文件及符合特定规则的文件（如果需要，规则可由ms-config.js文件配置）;
 
 > mock文件不会立即加载；只有请求触发时会加载；且不进行缓存；
 
 > 测试：启动edp或mock程序；浏览器中测试，或发curl请求
 
     http://localhost:8848/request.ajax?path=GET/auth&param={}
     
-----------------------

## 构造mock数据规范

- 请求规范

> 前端代码发送真实请求；请求路径符合request.ajax?path=XXX形式;
> 参数param可以是POST或GET参数
param符合严格规范的json格式

- 构造数据代码规范

> 所有响应request.ajax?path={pathname}&param={object}请求每个pathname对应一个mock文件；

> mock文件名`/`替换为`_`；

> 独立mock文件命名为{pkgname}/{pathname}.js;

-- 对应每个接口应该指定一个响应函数；响应函数有固定参数列表(path, param)

-- index.js 文件可以定义多个接口的响应函数 (但是不建议写到index文件)

-- {pathname}.js 文件只能定义对应pathname的响应函数


————————————————————————

## mock文件示例

```js
// mock 的用法参考./test的文件

module.exports = function (path, param) {

    // tpl在packages定义了路径
    var tpl = require('tpl/hospital');
    
    // lib/mendb是基于menset概念设计(未完全实现）
    var db = require('lib/mendb');
    
    /**
     * include 是mockservice内置的包导入函数；用于加载node模块
     * moment, random, template 为支持mockservice内置的组件
     */
    var moment = include('moment');  // 时间格式化组件
    var random = include('random');  // 随机数据产生器
    var template = include('template'); // 基于etpl的模板解析引擎 
    
    /**
     * 因为采用的是menset设计，因此直接赋值相当于改变了数据集
     * db 支持数据集的增删改查
     */
    var hospital = db.hospital.find({id: param.hospitalId});
    // 修改hospital的访问量
    hospital.visitCount++;
    
    // 业务数据
    var data = {
        timestamp: random.timestamp(),
        title: tpl.title(hospital),
        creative: tpl.creative({
            id: hospital.id,
            name: hospital.name,
            city: random.words(hospital.cities), // 随机选择国内城市
            section: random.words(hospital.sections) // 随机选择医院科室
        });
     };
    
    // 返回数据
    return {
        status: 200, // 业务status，与http状态无关
        _status: 300, // 指定http状态； 不输出
        _timeout: 1000, // 延迟发送毫秒时间；不输出
        data: data
    };
};

// 注意db相关的功能尚在完善与测试中；暂不要在业务中使用
```

## 扩展modules 说明
> 扩展的modules是为了更好的支持mock数据的生成；

> 所有的modules通过include(modulename)既可以获取到；

> mockservice; 内置了以下通用mock支持组件

- random : 产生随机数据 
> [random 说明文档](https://github.com/linkwisdom/mockservice/blob/master/docs/random.md)

- storage : 数据增删改查操作支持
> [storage 说明文档](https://github.com/linkwisdom/mockservice/blob/master/docs/storage.md)

- template : 采用的是etpl解析引擎，方便模板化产生数据
> [template 说明文档](https://github.com/linkwisdom/mockservice/blob/master/docs/template.md)

- moment : 基于moment.js时间格式化组件（无多国语言包）
> [moment 中文](http://momentjs.cn/docs/)

------------------------

### ISSUE todo

- 构造数据支持简单物料读写逻辑

- 构造数据模版

- 构造数据工具方法

- 构造数据规范
