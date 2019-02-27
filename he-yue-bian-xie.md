超脑链使用类Javascript的语言来编写智能合约，这个类Javascript的语言以Typescript为原型，通过扩展的数据类型标志符，来达到强类型语言的编程语法.

#### 系统内置的方法

function NAME\(str: string\): u64 : 方法\*\*NAME\(\)\*\*用来将一个string转成一个account\_name类型.`str`的字符长度不超过12个字符, 内容只能包括以下字符\(不能以`.`结尾\):`.012345abcdefghijklmnopqrstuvwxyz`

function RNAME\(account: account\_name\): string : 方法\*\*RNAME\(\)**用来将一个account\_name类型转为string类型, 它是**NAME\(\)\*\*方法的反向方法.

function ACTION\(str: string\): Action : 方法\*\*ACTION\(\)\*\*将一个string类型转为Action类型.`str`的长度不超过21个字符, 内容只能包括以下字符\(不能包含`.`\):`._0-9a-zA-Z`. Action类封装了action相关的信息.

Action.sender : 当前transaction的发起者, account\_name类型.

Action.receiver : 当前transaction的接收者, 即合约帐户, account\_name类型.

Block.number : head block的块高。

Block.id : head block的id，sha256的hash值。

Block.timestamp : head block的时间戳，从EPOCH开始的秒数。

#### 编写第一个合约Hello world

```
import { NAME, RNAME } from "ultrain-ts-lib/src/account";
import { Log } from "ultrain-ts-lib/src/log";
import { Contract } from "ultrain-ts-lib/lib/contract";

class HelloWorld extends Contract {

    @action
    hi(name: account_name, age: u32, msg: string): void {
        Log.s("hi: name = ").s(RNAME(name)).s(" age = ").i(age, 10).s(" msg = ").s(msg).flush();
    }
}
```

我们以上代码做如下说明：

1. import: 用来引入其它文件中定义的类和方法，详细用法可参考
   [typescript](https://www.tutorialspoint.com/typescript/index.htm)
   的说明。
2. extends Contract: 合约都需要派生自Contract，而且一个项目中**只能有一个Contract**
3. @action: 申明一个合约方法。只有@action标志的方法，才能被调用。
4. Log: 打印Log。需要在config.ini文件中配置
   `contracts-console = true`
   才能打印到终端。

#### 在Action中Return信息

为了便于在调用方与节点中传递部分执行状态信息，引入Return模块.  
Return模块返回的数据会附加在http的response中， 调用方可以通过分析response得到Return的信息。  
需要强调的是，**Return的信息仅仅是在一个节点\(host\_url \)上预执行的结果，并非区块链网络共识的结果。也就是说, Return返回的结果, 并不是最终交易执行的结果.**  
**Return的信息只供参考，它可能与区块链网络共识结果不一致。**

要Return信息，可以在action调用中，通过`Return`,`ReturnArray`方法来完成。Return信息有以下需要注意的点：

> NOTICE

1. Return的message是有长度限制的，默认的message长度为128个character。（int型数据会转成对应的string）。如果是在侧链中使用，可以在config.ini文件中配置
   `contract-return-string-length`
   来扩展长度限制。
2. 只支持Return基本数据类型int和string， 以及int\[\]和string\[\]。
3. 可以调用Return或ReturnArray多次，信息将被concat。
4. 超出长度限制的信息，会直接丢弃，不会抛出异常。

###### Return信息的示例

```
class HelloContract extends Contract{
    @action
    on_hi(name: u64, age: u32, msg: string): void {
        Return<string>("call hi() succeed.");
        ReturnArray<u8>([1,2,3]);
    }
}
```

执行正常的情况下，Return的结果是`call hi() succeed.123`

#### 资产查询和转移

在合约中，可以查询一个帐号在ultrainio.token合约中的资产，即ultrain平台资产。查询资产使用`Asset.balanceOf(who: account_name): Asset`方法。 转移ultrain平台资产，可以使用`Asset.transfer(from: account_name, to: account_name, val: Asset, memo: string): void`方法。

使用详情请参考[示例balance](https://github.com/ultrain-os/ultrain-ts-lib/blob/master/example/balance/balance.ts)。

```
import "allocator/arena";
import { Contract } from "ultrain-ts-lib/src/contract";
import { Asset } from "ultrain-ts-lib/src/asset";
import { ultrain_assert } from "ultrain-ts-lib/src/utils";
class BalanceContract extends Contract {

    @action
    transfer(from: account_name, to: account_name, bet: Asset): void {

        let balance = Asset.balanceOf(from);
        ultrain_assert(balance.gte(bet), "your balance is not enough.");

        balance.prints("banalce from: ");

        Asset.transfer(from, to, bet, "this is a transfer test");
    }
}
```

> NOTICE 使用Asset.transfer命令转移资产时，需要保证`from`的权限已经授权给了`utrio.code`，在使用命令行的情况下，可以通过以下命令来授权:
>
> `clutrain set account permission $from active '{"threshold": 1, "keys":[{"key":"$PubKey_of_from", "wieght": 1}], "accounts": [{"permission": {"actor": "$from", "permission": "utrio.code"}, "weight": 1]}' owner -p $from`
>
> `$from`是需要授权的帐号。

#### 持久化存储

Ultrain的智能合约提供了DBManager来存储合约数据到数据库中。不同于以太坊会自动保存数据，Ultrain需要明确的调用API来保存、读取数据。

##### Serializable接口

Serializable是一个Interface， 定义以下三个方法：

```
import {DataStream} from "ultrain-ts-lib/src/datastream";
export interface Serializable {
    deserialize(ds: DataStream): void;
    serialize(ds : DataStream) : void;
    primaryKey(): u64;
}
```

`deserialize(ds: DataStream): void;`

* 方法用来做反序列化工作，从DataStream的字节流中读取数据进行初始化工作。
* `serialize(ds: DataStream): void;`
  方法用来做序列化工作，将class的数据写入到字节流中。
* `primaryKey(): u64;`
  标志一个primary key。 如果这个class将作为一条独立的记录写入数据库，那primaryKey\(\)返回的数据将成为数据库中的primary key.

> _NOTICE_

1. 一个实现了ISerialzable接口的class，编译器将自动实现以上三个方法，并将class中的成员变量都序列化/反序列化。如果需要单独override某一个/全部方法，则可以手动实现对应的方法。
2. 如果要排除某个成员变量，以避免序列化和反序列化，可以使用
   `@ignore`
   注解；
3. 如果要指定某个成员变量为primaryKey，可以使用
   `@primaryid`
   注解。需要注意的是，被注解为@primaryid的变量必须是u64类型，如果没有变量被注解为@primaryid，则primaryKey\(\)方法默认使用
   `0`
   作为返回值。
4. 如果使用了@注解，同时又override了serialize\(\)、deserialize\(\)、primaryKey\(\)方法中的某一个（或全部），编译器将优先使用override的方法。

对于Serializable接口的使用，举例如下:

```
class Person implements Serializable{
    name: string;
    age: u32;
    sex: string;
    salary: u32;
    @ignore
    address: string; // 被忽略，不序列化和反序列化
    
    constructor() {
        this.name = "xx";
        //...
    }
    // 重写primaryKey()方法，返回Person的id
    primaryKey(): u64 {
        return NAME(this.name);
    }
}
```

##### 可序列化存储的数据

存储到数据库中的数据，必须是能够序列化和反序列化。可以序列化存储的数据有以下几类：

1. 内置基本数据类型： u8/i8, u16/i16, u32/i32, u64/i64, boolean, string。 有一些类型其实也是基本数据类型的别名，如account\_name。
2. 基本数据类型的一维数组： u8\[\], i8\[\], ..., string\[\]
3. 实现了Serializable接口的类， 如上的Person。
4. 实现了Serializable接口的类的一维数组，如Person\[\]。

##### 声明合约中DB的table信息

如果合约中需要使用到DB进行数据存取，则需要在具体的Contract类中注解说明table的信息。 如下简单的一份伪代码：

```
class Person implements Serializable {
   name: string;
   sex: string;
}

class Car implements Serializable {
   model: string;
   power: u32;
   color: string;
}

@database(Person, "persons")
@database(Car, "cars")
// @database() if any more
clas MyContract extends Contract {
    //...
    // your logic here
}    
```

上述代码将会生成两张表格： "persons"和"cars"。 需要注意的是，@database注解中的Person和Car两个类，**必须实现Serializable接口**。

##### 数据库读写

Contract中数据存取要通过DBManager来管理。

###### DBManager的定义：

```
export class DBManager<T extends Serializable> {
    constructor(tblname: u64, owner: u64, scope: u64) {}
    public cursor(): Cursor<T> {}
    public emplace(payer: u64, obj: T): void {}
    public modify(payer: u64, newobj: T): void {}
    public exists(primary: u64): boolean {}
    public get(primary: u64, out: T): boolean { }
    public erase(obj: T): void {}
} 
```

constructor\(\)方法接收三个参数，

* `tblname: u64`表示表名；
  `owner：u64`表示这个表在哪个合约中，一般的，owner和该合约的receiver是一样的。
  `scope: u64`表示表中的一个上下文。
* cursor\(\)方法读取数据表中的所有记录。
* emplace\(\)方法向表中加入一条记录。
  `payer`表示这个帐号将为数据存储付费，
  `obj`是一个Serializable的对象，将数据存入DB。
* modify\(\)方法更新表中的数据。
  `payer`表示这条记录的创建者、付费方；
  `newobj`是更新后的数据，newobj的primaryKey对应的对象会被更新。
* exists\(\)方法判断一个primaryKey是否存在。
* get\(\)方法从DB中读取primary对应的记录，并反序列化到out中。
* erase\(\)方法用来删除一条记录，obj的primaryKey对应的记录如果存在，将被删掉。

> NOTICE table没有方法可以显式删除，只有当table中的记录都删掉时，table会自动被删除。

##### 使用Cursor遍历所有记录

我们提供了cursor来遍历所有的记录，但是必须明白，这个操作非常非常低效，因为在当调用cursor\(\)方法时，会将所有的表中的数据都加载到内存里面。如果表中的数据很多的话，那这个交易将会被cursor方法阻塞，从而导致交易超时失败。 如下示例演示了怎样使用cursor

```
let cursor = this.db.cursor();
Log.s("cursor.count =").i(cursor.count).flush();

while(cursor.hasNext()) {
    let p: Person = cursor.get();
    p.prints();
    cursor.next();
}
```

##### table里面scope和primary key的关系

table中的数据，可以按scope来分类，也可以通过primary key来分类。尽管它们都可以达到分类数据的效果，但是在table中，scope和primary key是两个不同的维度，它们之间的关系，大概可用下面的结构来表示：

```
 |--table
 |----scope1
 |--------primaryKey_1
 |--------primaryKey_2
 |--------........
 |----scope2
 |--------primaryKey_x
 |--------primaryKey_y
 |--------.......
```

在不同的scope下面，primary key可以取相同的值。

##### 使用示例

DB的读写操作，请参考[示例Person](https://github.com/ultrain-os/ultrain-ts-lib/blob/master/example/person/person.ts)。

```
import "allocator/arena";
import { Contract } from "ultrain-ts-lib/src/contract";
import { Log } from "ultrain-ts-lib/src/log";
import { ultrain_assert } from "ultrain-ts-lib/src/utils";
import { DBManager } from "ultrain-ts-lib/src/dbmanager";
import { NAME } from "ultrain-ts-lib/src/account";

class Person implements Serializable {
    // name: string;
    name: string
    age: u32;
    salary: u32;

    primaryKey(): u64 { return NAME(this.name); }

    prints(): void {
        Log.s("name = ").s(this.name).s(", age = ").i(this.age).s(", salary = ").i(this.salary).flush();
    }
}

const tblname = "humans";
const scope = "dept.sales";

@database(Person, "humans")
// @database(SomeMoreRecordStruct, "other_table")
class PersonContract extends Contract {

    db: DBManager<Person>;

    public onInit(): void {
        this.db = new DBManager<Person>(NAME(tblname), this.receiver, NAME(scope));
    }


    public onStop(): void {

    }

    constructor(code: u64) {
        super(code);
        this._receiver = code;

        this.onInit();
    }

    @action
    add(name: string, age: u32, salary: u32): void {
        let p = new Person();
        p.name = name;
        p.age = age;
        p.salary = salary;

        let existing = this.db.exists(NAME(name));
        ultrain_assert(!existing, "this person has existed in db yet.");
        p.prints();
        this.db.emplace(this.receiver, p);
    }

    @action
    modify(name: string, salary: u32): void {
        let p = new Person();
        let existing = this.db.get(NAME(name), p);
        ultrain_assert(existing, "the person does not exist.");

        p.salary = salary;

        this.db.modify(this.receiver, p);
    }

    @action
    remove(name: string): void {
        Log.s("start to remove: ").s(name).flush();
        this.db.erase(NAME(name));
    }
}
```



