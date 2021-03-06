# 面向流实现者的API

* [stream.Readable类](#class_Readable)
  - [new stream.Readable([options])](#new_Readable)
  - [readable._read(size)](#_read)
  - [readable.push(chunk[, encoding])](#push)
  - [例子：一种计数流](#a_counting_stream)
  - [例子：简单的协议 V1（次优）](#simpleProtocol_v1)
* [stream.Writable类](#class_Writable)
  - [new stream.Writable([options])](#new_Writable)
  - [writable._write(chunk, encoding, callback)](#_write)
  - [writable._writev(chunks, callback)](#_writev)
* [stream.Duplex类](#class_Duplex)
  - [new stream.Duplex(options)](#new_Duplex)
* [stream.Transform类](#class_Transform)
  - [new stream.Transform([options])](#new_Transform)
  - ['finish'和'end'事件](#event_finish_and_end)
  - [transform._transform(chunk, encoding, callback)](#_transform)
  - [transform._flush(callback)](#_flush)
  - [例子：简单的协议分析器 V2](#simpleProtocol_parser_v2)
* [stream.PassThrough类](#class_PassThrough)

--------------------------------------------------


无论实现任何形式的流，模式都是一样的：

1. 在你的子类中扩展合适的父类。（[util.inherits()](../util/util.md#inherits) 方法对此很有帮助。）

2. 在你的构造函数中调用合适的父类的构造函数，以确保内部机制设置正确。

3. 实现一个或多个特定的方法，如下详述。

类扩充和实现方法取决于你要编写哪种流类：

| 使用情景                  | 类                            |  要实现的方法                  |
| ----------------------- | ----------------------------- | ---------------------------- |
| 只读                     | [Readable](#class_Readable)   | `_read`                      |
| 只写                     | [Writable](#class_Writable)   | `_write`, `_writev`          |
| 读写                     | [Duplex](#class_Duplex)       | `_read`, `_write`, `_writev` |
| 操作写入的数据，然后读取结果  | [Transform](#class_Transform) | `_transform`, `_flush`       |

在你的实现代码中，需要强调的是绝对不要调用[面向流消费者的 API ](./api_for_stream_consumers.md#)中描述的方法。否则，可能会导致在消费你的流接口的过程中产生不良副作用。


<div id="class_Readable" class="anchor"></div>
## stream.Readable类

`stream.Readable` 是一个被设计为拓展底层实现 [stream._read(size)](#_read) 方法的抽象类。

如何在你的程序中消费流，请参阅[面向流消费者的 API ](./api_for_stream_consumers.md#)。下文解释了如何在你的程序中实现可读流。


<div id="new_Readable" class="anchor"></div>
#### new stream.Readable([options])

- `options` {Object}

  - `highWaterMark` {Number} 停止从底层资源读取前能够存储在内部缓冲区的最大字节数。默认：`16384`（16kb）或 `objectMode` 流中的 `16`
  
  - `encoding` {String} 如果指定，则缓冲区将使用指定的编码字符串解码。默认：`null`
  
  - `objectMode` {Boolean} 此流是否应该表现为对象流。意味着 [stream.read(n)](./api_for_stream_consumers.md#read) 返回单个值用于替代一个 n 大小的 Buffer 。默认：`false`
  
  - `read` {Function} 实现 [stream._read()](#_read) 的方法
  
在拓展自可读（Readable）类的类中，请确保调用 Readable 构造函数，以便缓冲设置可以被正确地初始化。


<div id="_read" class="anchor"></div>
#### readable._read(size)

- `size` {Number} 异步读取的字节数

注意：**请实现这个方法，但不要直接调用它**。

这是一个带有下划线前缀的方法，因为它是在类内部定义的，应该仅被 Readable 类内部的方法的调用。所有的 Readable 流实现都必须提供一个 _read 方法用于从底层资源获取数据。

当调用 `_read()` 时，如果从资源获取的数据可用，`_read()` 的实现应该通过调用 [this.push(dataChunk)](##push) 开始将数据推入到读取队列中。`_read()` 应该继续从资源处读取并推送数据知道推送返回 `false`，此时应停止从资源处读取。仅当 `_read()` 在停止后被再次调用时，它应该开始从资源处读取更多的数据，并将数据推送到队列中。

注意：一旦调用 `_read()` 方法，直到 [stream.push()](#push) 方法被调用前将不能被再次调用。

`size` 参数仅做参考。实现中 "read" 是一个单一的回调，返回的数据可以用这个来知道有多少数据获取。实现中不相关的，如 TCP 或 TLS ，可能会忽略此参数，并且简单的提供可用的数据。没有必要在调用 [stream.push(chunk)](#push) 前，比如 "wait" 直到 `size` 字节的数据可用。


<div id="push" class="anchor"></div>
#### readable.push(chunk[, encoding])

- `chunk` {Buffer} | {Null} | {String} 推入读队列的数据块

- `encoding` {String} 字符串块的编码。必须是一个有效的 Buffer 编码，比如 `'utf8'` 或 `'ascii'` 。

- 返回 {Boolean} 是否应该执行更多的推入

注意：**该方法应该在 Readable 流的实现者中调用，而不是 Readable 流的消费者**。

如果传了一个非 `null` 值，`push()` 方法会将数据块放入到流处理器随后消费的队列里。如果传了 `null` ，它标志着已到达流的末尾（EOF），之后没有更多的数据可以被写入。

当触发 ['readable'](./api_for_stream_consumers.md#readable_event_readable) 时间时，用 `push()`  添加的数据可以通过调用 [stream.read()](./api_for_stream_consumers.md#read) 方法拉取。

这个 API 被设计成尽可能地灵活。例如，你可以包裹有某种暂停/恢复机制的低级资源和一个数据回调。在这种情况下，你可以通过这样做来包裹低级资源对象：

```javascript
// source is an object with readStop() and readStart() methods,
// and an `ondata` member that gets called when it has data, and
// an `onend` member that gets called when the data is over.

util.inherits(SourceWrapper, Readable);

function SourceWrapper(options) {
    Readable.call(this, options);

    this._source = getLowlevelSourceObject();

    // Every time there's data, we push it into the internal buffer.
    this._source.ondata = (chunk) => {
        // if push() returns false, then we need to stop reading from source
        if (!this.push(chunk))
            this._source.readStop();
    };

    // When the source ends, we push the EOF-signaling `null` chunk
    this._source.onend = () => {
        this.push(null);
    };
}

// _read will be called when the stream wants to pull more data in
// the advisory size argument is ignored in this case.
SourceWrapper.prototype._read = function (size) {
    this._source.readStart();
};
```


<div id="a_counting_stream" class="anchor"></div>
#### 例子：一种计数流

这是一个可读（Readable）流的基本示例。它触发从 1 到 1,000,000 的升序操作，然后结束。

```javascript
const Readable = require('stream').Readable;
const util = require('util');
util.inherits(Counter, Readable);

function Counter(opt) {
    Readable.call(this, opt);
    this._max = 1000000;
    this._index = 1;
}

Counter.prototype._read = function () {
    var i = this._index++;
    if (i > this._max)
        this.push(null);
    else {
        var str = '' + i;
        var buf = new Buffer(str, 'ascii');
        this.push(buf);
    }
};
```


<div id="simpleProtocol_v1" class="anchor"></div>
#### 例子：简单的协议 V1（次优）

这类似于[此处](./api_for_stream_consumers.md#unshift)所描述的 `parseHeader` 函数，但作为一个自定义的流来实现。另请注意，这种实现不会将输入的数据转换为字符串。

而然，最好通过转换（[Transform](#class_Transform)）流来实现。参阅[简单的协议分析器 V2](#simpleProtocol_parser_v2)，一个更好的实现。

```javascript
// A parser for a simple data protocol.
// The "header" is a JSON object, followed by 2 \n characters, and
// then a message body.
//
// NOTE: This can be done more simply as a Transform stream!
// Using Readable directly for this is sub-optimal. See the
// alternative example below under the Transform section.

const Readable = require('stream').Readable;
const util = require('util');

util.inherits(SimpleProtocol, Readable);

function SimpleProtocol(source, options) {
    if (!(this instanceof SimpleProtocol))
        return new SimpleProtocol(source, options);

    Readable.call(this, options);
    this._inBody = false;
    this._sawFirstCr = false;

    // source is a readable stream, such as a socket or file
    this._source = source;

    source.on('end', () => {
        this.push(null);
    });

    // give it a kick whenever the source is readable
    // read(0) will not consume any bytes
    source.on('readable', () => {
        this.read(0);
    });

    this._rawHeader = [];
    this.header = null;
}

SimpleProtocol.prototype._read = function (n) {
    if (!this._inBody) {
        var chunk = this._source.read();

        // if the source doesn't have data, we don't have data yet.
        if (chunk === null)
            return this.push('');

        // check if the chunk has a \n\n
        var split = -1;
        for (var i = 0; i < chunk.length; i++) {
            if (chunk[i] === 10) { // '\n'
                if (this._sawFirstCr) {
                    split = i;
                    break;
                } else {
                    this._sawFirstCr = true;
                }
            } else {
                this._sawFirstCr = false;
            }
        }

        if (split === -1) {
            // still waiting for the \n\n
            // stash the chunk, and try again.
            this._rawHeader.push(chunk);
            this.push('');
        } else {
            this._inBody = true;
            var h = chunk.slice(0, split);
            this._rawHeader.push(h);
            var header = Buffer.concat(this._rawHeader).toString();
            try {
                this.header = JSON.parse(header);
            } catch (er) {
                this.emit('error', new Error('invalid simple protocol data'));
                return;
            }
            // now, because we got some extra data, unshift the rest
            // back into the read queue so that our consumer will see it.
            var b = chunk.slice(split);
            this.unshift(b);
            // calling unshift by itself does not reset the reading state
            // of the stream; since we're inside _read, doing an additional
            // push('') will reset the state appropriately.
            this.push('');

            // and let them know that we are done parsing the header.
            this.emit('header', this.header);
        }
    } else {
        // from there on, just provide the data to our consumer.
        // careful not to push(null), since that would indicate EOF.
        var chunk = this._source.read();
        if (chunk) this.push(chunk);
    }
};

// Usage:
// var parser = new SimpleProtocol(source);
// Now parser is a readable stream that will emit 'header'
// with the parsed header data.
```


<div id="class_Writable" class="anchor"></div>
## stream.Writable类

`stream.Writable` 是一个被设计为拓展底层实现 [stream._write(chunk, encoding, callback)](#_write) 方法的抽象类。

如何在你的程序中消费可写流，请参阅[面向流消费者的 API ](./api_for_stream_consumers.md#)。下文解释了如何在你的程序中实现可写流。


<div id="new_Writable" class="anchor"></div>
#### new stream.Writable([options])

- `options` {Object}

  - `highWaterMark` {Number} 当 [stream.write()](./api_for_stream_consumers.md#write) 开始返回 `false` 时的 Buffer 等级。默认：`16384`（16kb）或 `objectMode` 流中的 `16`
  
  - `decodeStrings` {Boolean} 在传递给 [stream._write()](#_write) 前是否解码字符串到缓冲区。默认：`true`
  
  - `objectMode` {Boolean} [stream.write(anyObj)](./api_for_stream_consumers.md#write) 是否是一个有效的操作。如果设置，你可以写入任意的数据，而不只是 `Buffer` / `String` 数据。默认：`false`
  
  - `write` {Function} [stream._write()](#_write) 方法的实现
  
  - `writev` {Function} [stream._writev()](#_writev) 方法的实现
  
在拓展自可写（Writable）类的类中，请确保调用 Writable 构造函数，以便缓冲设置可以被正确地初始化。


<div id="_write" class="anchor"></div>
#### writable._write(chunk, encoding, callback)

- `chunk` {Buffer} | {String} 被写入的（数据）块。**总会是**一个 buffer ，除非 `decodeStrings` 参数被设置成 `false` 。

- `encoding` {String} 如果该块是一个字符串，那么这是编码类型。如果该块是一个 buffer ，那么这是一个指定的值 -  'buffer'，在这种情况下请忽略它。

- `callback` {Function} 当你处理完成所提供的块时调用此函数（有一个可选的 error 参数）。

所有的 Writable 流实现都必须提供一个 _write 方法用于向底层资源发送数据。

注意：**此函数禁止被直接调用**。它应该由子类来实现，并且仅被可写（Writable）类的内部方法调用。

该回调函数采用标准的 `callback(error)` 模式来表明写入成功完成还是遇到错误。

如果构造函数选项中设定了 `decodeStrings` 标志，则 `chunk` 可能会是字符串而不是 Buffer，并且 `encoding` 表明了字符串的格式。这种设计是为了支持对某些字符串数据编码提供优化处理的实现。如果您没有明确地将 `decodeStrings` 选项设定为 `false` ，那么您可以安全地忽略 `encoding` 参数，并假定 `chunk` 总是一个 Buffer。

这是一个带有下划线前缀的方法，因为它是在类内部定义的，并且不应该由用户程序直接调用。但是，我们**希望**你在自己的扩展类中重写此方法。


<div id="_writev" class="anchor"></div>
#### writable._writev(chunks, callback)

- `chunks` {Array} 被写入的块。每个块都是以下格式：`{ chunk: ..., encoding: ... }`。

- `callback` {Function} 当你处理完成所提供的块时调用此函数（有一个可选的 error 参数）。

注意：**此函数禁止被直接调用**。它可能是由子类实现，并且仅被可写（Writable）类的内部方法调用。

此函数是完全可选的实现。在大多数情况下，它是不必要的。如果实现，它将被所有滞留在写入队列中的数据块调用。


<div id="class_Duplex" class="anchor"></div>
## stream.Duplex类

“双工”（duplex）流兼具可读和可写特性，比如一个 TCP 嵌套字连接。

值得注意的是，`stream.Duplex` 是一个被设计为拓展底层实现 [stream._read(size)](#_read) 和 [stream._write(chunk, encoding, callback)](#_write) 方法的抽象类，就像你实现可读（Readable）或可写（Writable）类所做的那样。

由于 JavaScript 并不具备多原型继承能力，这个类实际上继承自 Readable，并寄生自 Writable。从而让用户在双工（Duplex）类的拓展中能同时实现低级的 [stream._read(n)](#_read) 和 [stream._write(chunk, encoding, callback)](#_write) 方法。


<div id="new_Duplex" class="anchor"></div>
#### new stream.Duplex(options)

- `options` {Object} 同时传入可读（Readable）和可写（Writable）构造函数。同时有以下字段：
  
  - `allowHalfOpen` {Boolean} 默认：`true`。如果设置为 `false`，那么当写入端结束后流将自动结束读取端，反之亦然。
  
  - `readableObjectMode` {Boolean} 默认：`false`。设置流读取端的 `objectMode`。如果 `objectMode` 为 `true` 也没有影响。
  
  - `writableObjectMode` {Boolean} 默认：`false`。设置流读取端的 `objectMode`。如果 `objectMode` 为 `true` 也没有影响。
  
在拓展自双工（Duplex）类的类中，请确保调用其构造函数，以便缓冲设置可以被正确地初始化。


<div id="class_Transform" class="anchor"></div>
## stream.Transform类

“转换”（transform）流实际上是一个输出与输入存在因果关系的双工（duplex）流，比如 [zlib](../zlib/) 流或 [crypto](../crypto/) 流。

它并不要求输入和输出需要相同大小、相同块数或同时到达。例如，一个 Hash 流只会在输入结束时产生一个数据块的输出；一个 zlib 流会产生比输入小得多或大得多的输出。

转换（Transform）类必须实现 [stream._transform()](#_transform) 方法，而不是 [stream._read()](#_read) 和 [stream._write()](#_write) 方法，同时也可以选择性地实现 [stream._flush()](#_flush) 方法。（详见下文。）


<div id="new_Transform" class="anchor"></div>
#### new stream.Transform([options])

- `options` {Object}

  - `transform` {Function} 实现 [stream._transform()](#_transform) 方法
  
  - `flush` {Function} 实现 [stream._flush()](#_flush) 方法
  
在拓展自转换（Transform）类的类中，请确保调用其构造函数，以便缓冲设置可以被正确地初始化。


<div id="event_finish_and_end" class="anchor"></div>
#### 'finish'和'end'事件

['finish'](./api_for_stream_consumers.md#writable_event_finish) 和 ['end'](./api_for_stream_consumers.md#readable_event_end) 事件分别来自可写（Writable）父类和可读（Readable）父类。`'finish'` 事件在调用 [stream.end()](./api_for_stream_consumers.md#end) 和所有的块都已被 [stream._transform()](#_transform) 处理后触发。`'end'` 在调用回调函数 [stream._flush()](#_flush) 输出所有数据后触发。


<div id="_transform" class="anchor"></div>
#### transform._transform(chunk, encoding, callback)

- `chunk` {Buffer} | {String} 需要被转换的数据块。**总会是**一个 buffer，除非 `decodeStrings` 选项被设置成 `false`。

- `encoding` {String} 如果该块是一个字符串，那么这是编码类型。如果该块是一个 buffer ，那么这是一个指定的值 -  'buffer'，在这种情况下请忽略它。

- `callback` {Function} 当你处理完成所提供的块时调用此函数（有一个可选的 error 参数）

注意：**此函数禁止被直接调用**。它可能是由子类实现，并且仅被转换（Transform）类的内部方法调用。

所有的转换（Transform）流的实现都必须提供一个 `_transform()` 方法用于接受输入并产生输出。

`_transform()` 应当承担特定的 Transform 类中所有处理被写入的字节、并将它们丢给接口的可读部分的职责，进行异步 I/O，处理其它事情等。

调用 `transform.push(outputChunk)` 零次或多次来从输入块生成输出，这取决于你有多少数据块要作为结果输出。

仅在当前块被完全消费后才调用回调函数。需要注意的是，输出可能会也可能不会作为任何特定的输入块的结果。如果提供了回调函数的第二个参数，它会被传递给 push 方法。换言之，以下是等效的：

```javascript
transform.prototype._transform = function (data, encoding, callback) {
    this.push(data);
    callback();
};

transform.prototype._transform = function (data, encoding, callback) {
    callback(null, data);
};
```

这是一个带有下划线前缀的方法，因为它是在类内部定义的，并且不应该由用户程序直接调用。但是，我们**希望**你在自己的扩展类中重写此方法。


<div id="_flush" class="anchor"></div>
#### transform._flush(callback)

- `callback` {Function} 当你强制刷新任何剩余数据时调用此函数（有一个可选的 error 参数）

注意：**此函数禁止被直接调用**。它可能是由子类实现，如果是这样的话，它仅被转换（Transform）类的内部方法调用。

在一些情景中，你的转换操作可能需要在流的末尾多发一些数据。例如，一个 `Zlib` 压缩流会储存一些内部状态以便更好地压缩输出，但在最后它需要尽可能好地处理剩下的东西以使数据完整。

在这些情况下，你可以实现一个 `_flush()` 方法，它会在所有写入数据被消费后，并且在标志着可读端到达末尾的 ['end'](./api_for_stream_consumers.md#readable_event_end) 事件触发前的最后一刻被调用。和 [stream._transform()](#_transform) 一样，只需在 flush 操作完成时适当地调用 `transform.push(chunk)` 零或多次。

这是一个带有下划线前缀的方法，因为它是在类内部定义的，并且不应该由用户程序直接调用。但是，我们**希望**你在自己的扩展类中重写此方法。


<div id="simpleProtocol_parser_v2" class="anchor"></div>
#### 例子：简单的协议分析器 V2

[这里](#simpleProtocol_v1)的简易协议解析器例子能够很简单地使用高级的 Transform 流类实现，类似于 `parseHeader` 和 `SimpleProtocol v1` 示例。

在这个示例中，输入会被导流到解析器中，而不是作为一个输入的参数提供。这种做法更符合 Node.js 流的惯例。

```javascript
const util = require('util');
const Transform = require('stream').Transform;
util.inherits(SimpleProtocol, Transform);

function SimpleProtocol(options) {
    if (!(this instanceof SimpleProtocol))
        return new SimpleProtocol(options);

    Transform.call(this, options);
    this._inBody = false;
    this._sawFirstCr = false;
    this._rawHeader = [];
    this.header = null;
}

SimpleProtocol.prototype._transform = function (chunk, encoding, done) {
    if (!this._inBody) {
        // check if the chunk has a \n\n
        var split = -1;
        for (var i = 0; i < chunk.length; i++) {
            if (chunk[i] === 10) { // '\n'
                if (this._sawFirstCr) {
                    split = i;
                    break;
                } else {
                    this._sawFirstCr = true;
                }
            } else {
                this._sawFirstCr = false;
            }
        }

        if (split === -1) {
            // still waiting for the \n\n
            // stash the chunk, and try again.
            this._rawHeader.push(chunk);
        } else {
            this._inBody = true;
            var h = chunk.slice(0, split);
            this._rawHeader.push(h);
            var header = Buffer.concat(this._rawHeader).toString();
            try {
                this.header = JSON.parse(header);
            } catch (er) {
                this.emit('error', new Error('invalid simple protocol data'));
                return;
            }
            // and let them know that we are done parsing the header.
            this.emit('header', this.header);

            // now, because we got some extra data, emit this first.
            this.push(chunk.slice(split));
        }
    } else {
        // from there on, just provide the data to our consumer as-is.
        this.push(chunk);
    }
    done();
};

// Usage:
// var parser = new SimpleProtocol();
// source.pipe(parser)
// Now parser is a readable stream that will emit 'header'
// with the parsed header data.
```


<div id="class_PassThrough" class="anchor"></div>
## stream.PassThrough类

这是转换（[Transform](#class_Transform)）流的一个简单实现，将输入的字节简单地传递给输出。它的主要用途是演示和测试，但偶尔也能在构建某种特殊流时派上用场。