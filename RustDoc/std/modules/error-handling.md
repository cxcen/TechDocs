# Rust 错误处理详解

## 1. 错误处理概览

Rust 使用 `Option` 和 `Result` 类型进行错误处理，无异常机制。

```mermaid
graph TB
    subgraph "Rust 错误处理"
        OPTION["Option<T><br/>可能缺失的值"]
        RESULT["Result<T, E><br/>可能失败的操作"]
        PANIC["panic!<br/>不可恢复错误"]
    end

    subgraph "处理方式"
        MATCH["match 模式匹配"]
        QUESTION["? 操作符传播"]
        COMBINATORS["组合子方法"]
        UNWRAP["unwrap/expect"]
    end

    OPTION --> MATCH
    RESULT --> MATCH
    OPTION --> QUESTION
    RESULT --> QUESTION

    style OPTION fill:#c8e6c9
    style RESULT fill:#bbdefb
    style PANIC fill:#ffccbc
```

---

## 2. Option<T>

Option 表示可能存在或缺失的值。

```mermaid
graph TB
    subgraph "Option<T> 定义"
        ENUM["enum Option<T> {<br/>    Some(T),<br/>    None,<br/>}"]
    end

    subgraph "常见场景"
        FIND["集合查找<br/>vec.get(index)"]
        PARSE["可选解析<br/>str.parse().ok()"]
        FIRST["首尾元素<br/>vec.first()"]
        FIELD["可选字段<br/>struct { name: Option<String> }"]
    end

    ENUM --> FIND
    ENUM --> PARSE
    ENUM --> FIRST
    ENUM --> FIELD

    style ENUM fill:#c8e6c9
```

### Option 方法

```mermaid
graph TB
    subgraph "检查与提取"
        IS_SOME["is_some() -> bool"]
        IS_NONE["is_none() -> bool"]
        UNWRAP["unwrap() -> T<br/>None 时 panic"]
        EXPECT["expect(msg) -> T<br/>None 时带消息 panic"]
        UNWRAP_OR["unwrap_or(default) -> T"]
        UNWRAP_OR_ELSE["unwrap_or_else(f) -> T"]
        UNWRAP_OR_DEFAULT["unwrap_or_default() -> T"]
    end

    subgraph "转换"
        MAP["map(f) -> Option<U>"]
        AND_THEN["and_then(f) -> Option<U>"]
        OR["or(opt) -> Option<T>"]
        OR_ELSE["or_else(f) -> Option<T>"]
        FILTER["filter(p) -> Option<T>"]
    end

    subgraph "与 Result 互转"
        OK_OR["ok_or(err) -> Result<T,E>"]
        OK_OR_ELSE["ok_or_else(f) -> Result<T,E>"]
    end

    style IS_SOME fill:#c8e6c9
    style MAP fill:#bbdefb
    style OK_OR fill:#fff9c4
```

### Option 链式处理

```mermaid
flowchart LR
    subgraph "链式调用示例"
        INPUT["Option<String>"]
        MAP1["map(|s| s.trim())"]
        FILTER["filter(|s| !s.is_empty())"]
        MAP2["map(|s| s.parse::<i32>())"]
        FLATTEN["flatten()"]
        OUTPUT["Option<i32>"]
    end

    INPUT --> MAP1 --> FILTER --> MAP2 --> FLATTEN --> OUTPUT

    style INPUT fill:#c8e6c9
    style OUTPUT fill:#bbdefb
```

---

## 3. Result<T, E>

Result 表示可能成功或失败的操作。

```mermaid
graph TB
    subgraph "Result<T, E> 定义"
        ENUM["enum Result<T, E> {<br/>    Ok(T),<br/>    Err(E),<br/>}"]
    end

    subgraph "常见场景"
        IO["I/O 操作<br/>File::open()"]
        PARSE["解析<br/>&quot;42&quot;.parse::<i32>()"]
        NET["网络操作<br/>TcpStream::connect()"]
        CUSTOM["自定义操作<br/>validate_input()"]
    end

    ENUM --> IO
    ENUM --> PARSE
    ENUM --> NET
    ENUM --> CUSTOM

    style ENUM fill:#c8e6c9
```

### Result 方法

```mermaid
graph TB
    subgraph "检查与提取"
        IS_OK["is_ok() -> bool"]
        IS_ERR["is_err() -> bool"]
        UNWRAP["unwrap() -> T<br/>Err 时 panic"]
        EXPECT["expect(msg) -> T"]
        UNWRAP_ERR["unwrap_err() -> E"]
        OK["ok() -> Option<T>"]
        ERR["err() -> Option<E>"]
    end

    subgraph "转换"
        MAP["map(f) -> Result<U, E>"]
        MAP_ERR["map_err(f) -> Result<T, F>"]
        AND_THEN["and_then(f) -> Result<U, E>"]
        OR["or(res) -> Result<T, E>"]
        OR_ELSE["or_else(f) -> Result<T, F>"]
    end

    subgraph "默认值"
        UNWRAP_OR["unwrap_or(default) -> T"]
        UNWRAP_OR_ELSE["unwrap_or_else(f) -> T"]
        UNWRAP_OR_DEFAULT["unwrap_or_default() -> T"]
    end

    style IS_OK fill:#c8e6c9
    style MAP fill:#bbdefb
    style UNWRAP_OR fill:#fff9c4
```

---

## 4. ? 操作符

? 操作符简化错误传播。

```mermaid
flowchart TD
    subgraph "? 操作符展开"
        QUESTION["expr?"]
        MATCH["等价于:<br/>match expr {<br/>    Ok(v) => v,<br/>    Err(e) => return Err(e.into()),<br/>}"]
    end

    QUESTION --> MATCH

    subgraph "自动转换"
        FROM["E1 实现 From<E2>"]
        INTO["则 Err(e2)? 自动转为 Err(E1)"]
    end

    MATCH --> FROM
    FROM --> INTO

    style QUESTION fill:#c8e6c9
    style FROM fill:#bbdefb
```

### ? 在不同上下文

```mermaid
graph TB
    subgraph "函数返回 Result"
        FN_RESULT["fn foo() -> Result<T, E> {<br/>    let x = may_fail()?;<br/>    Ok(x)<br/>}"]
    end

    subgraph "函数返回 Option"
        FN_OPTION["fn foo() -> Option<T> {<br/>    let x = may_none()?;<br/>    Some(x)<br/>}"]
    end

    subgraph "main 函数"
        MAIN["fn main() -> Result<(), Error> {<br/>    do_something()?;<br/>    Ok(())<br/>}"]
    end

    subgraph "try 块 (unstable)"
        TRY["let result: Result<_, _> = try {<br/>    let x = may_fail()?;<br/>    x + 1<br/>};"]
    end

    style FN_RESULT fill:#c8e6c9
    style MAIN fill:#bbdefb
```

---

## 5. 自定义错误类型

```mermaid
classDiagram
    class Error {
        <<trait>>
        +source(&self) Option~&dyn Error~
        +description(&self) &str deprecated
    }

    class Display {
        <<trait>>
        +fmt(&self, f: &mut Formatter) Result
    }

    class Debug {
        <<trait>>
        +fmt(&self, f: &mut Formatter) Result
    }

    class MyError {
        -kind: ErrorKind
        -message: String
        +new(kind, msg) Self
    }

    Error <|.. MyError : 实现
    Display <|.. MyError : 实现
    Debug <|.. MyError : 实现

    note for MyError "实现 Error 需要先实现<br/>Display 和 Debug"
```

### 错误类型设计

```mermaid
graph TB
    subgraph "简单枚举"
        SIMPLE["#[derive(Debug)]<br/>enum MyError {<br/>    NotFound,<br/>    InvalidInput,<br/>    IoError(io::Error),<br/>}"]
    end

    subgraph "结构体 + 枚举"
        COMPLEX["struct Error {<br/>    kind: ErrorKind,<br/>    message: String,<br/>    source: Option<Box<dyn Error>>,<br/>}<br/><br/>enum ErrorKind {<br/>    NotFound,<br/>    InvalidInput,<br/>    Io,<br/>}"]
    end

    subgraph "使用 thiserror"
        THISERROR["#[derive(thiserror::Error, Debug)]<br/>enum MyError {<br/>    #[error(&quot;not found: {0}&quot;)]<br/>    NotFound(String),<br/>    #[error(&quot;io error&quot;)]<br/>    Io(#[from] io::Error),<br/>}"]
    end

    SIMPLE --> COMPLEX
    COMPLEX --> THISERROR

    style SIMPLE fill:#c8e6c9
    style THISERROR fill:#bbdefb
```

---

## 6. 错误转换

```mermaid
graph TB
    subgraph "From trait"
        FROM["impl From<E2> for E1 {<br/>    fn from(err: E2) -> E1 {<br/>        // 转换逻辑<br/>    }<br/>}"]
    end

    subgraph "自动转换"
        AUTO["实现 From 后：<br/>• ? 操作符自动转换<br/>• .into() 可用"]
    end

    subgraph "常见转换"
        IO_TO_CUSTOM["io::Error -> MyError"]
        PARSE_TO_CUSTOM["ParseIntError -> MyError"]
    end

    FROM --> AUTO
    AUTO --> IO_TO_CUSTOM
    AUTO --> PARSE_TO_CUSTOM

    style FROM fill:#c8e6c9
```

### 错误链

```mermaid
sequenceDiagram
    participant App as 应用层
    participant Svc as 服务层
    participant IO as I/O 层

    IO->>IO: io::Error 发生
    IO->>Svc: 返回 Err(io_err)
    Svc->>Svc: 包装为 ServiceError
    Svc->>App: 返回 Err(svc_err)
    App->>App: 包装为 AppError

    Note over App: error.source() 可追溯错误链<br/>io_err -> svc_err -> app_err
```

---

## 7. panic! 与 catch_unwind

```mermaid
graph TB
    subgraph "panic! 行为"
        PANIC["panic!(&quot;message&quot;)"]
        UNWIND["默认: 栈展开 (unwind)"]
        ABORT["可选: 直接中止 (abort)"]
    end

    subgraph "触发 panic"
        EXPLICIT["显式: panic!()"]
        UNWRAP["unwrap() / expect()"]
        INDEX["越界访问 vec[999]"]
        ASSERT["assert! 失败"]
        OVERFLOW["算术溢出 (debug)"]
    end

    subgraph "处理 panic"
        CATCH["std::panic::catch_unwind()"]
        HOOK["std::panic::set_hook()"]
        RESUME["resume_unwind()"]
    end

    PANIC --> UNWIND
    PANIC --> ABORT
    EXPLICIT --> PANIC
    UNWRAP --> PANIC
    UNWIND --> CATCH

    style PANIC fill:#ffccbc
    style CATCH fill:#c8e6c9
```

### 何时使用 panic

```mermaid
flowchart TD
    Q1{错误类型?}

    Q1 -->|程序 bug| PANIC["使用 panic!<br/>• 不变量被违反<br/>• 不可能的状态<br/>• 编程错误"]

    Q1 -->|预期的失败| RESULT["使用 Result<br/>• 文件不存在<br/>• 网络超时<br/>• 无效输入"]

    Q1 -->|可选值| OPTION["使用 Option<br/>• 集合为空<br/>• 解析可选<br/>• 可选配置"]

    style PANIC fill:#ffccbc
    style RESULT fill:#c8e6c9
    style OPTION fill:#bbdefb
```

---

## 8. 错误处理最佳实践

```mermaid
mindmap
    root((错误处理))
        库代码
            返回 Result
            定义清晰的错误类型
            实现 Error trait
            提供错误转换
        应用代码
            在边界处理错误
            提供用户友好消息
            记录详细错误信息
            考虑使用 anyhow
        测试代码
            unwrap() 可接受
            使用 ? 在测试函数
            #[should_panic] 测试
```

### 常用 crate

```mermaid
graph TB
    subgraph "thiserror"
        THISERROR["• 派生 Error 实现<br/>• 适合库开发<br/>• 零运行时开销"]
    end

    subgraph "anyhow"
        ANYHOW["• 通用错误类型<br/>• 适合应用开发<br/>• 自动保留错误链"]
    end

    subgraph "eyre"
        EYRE["• anyhow 的替代<br/>• 更好的错误报告<br/>• 支持自定义报告"]
    end

    THISERROR -->|库开发| LIB["库 crate"]
    ANYHOW -->|应用开发| APP["应用 crate"]

    style THISERROR fill:#c8e6c9
    style ANYHOW fill:#bbdefb
```

---

## 9. 常见模式

### 组合多个 Result

```mermaid
graph TB
    subgraph "顺序执行"
        SEQ["let a = op1()?;<br/>let b = op2(a)?;<br/>let c = op3(b)?;<br/>Ok(c)"]
    end

    subgraph "and_then 链"
        CHAIN["op1()<br/>.and_then(op2)<br/>.and_then(op3)"]
    end

    subgraph "collect 收集"
        COLLECT["let results: Result<Vec<_>, _> =<br/>items.iter()<br/>.map(process)<br/>.collect();"]
    end

    SEQ -->|等价于| CHAIN
    COLLECT -->|短路| FIRST_ERR["第一个 Err 即返回"]

    style SEQ fill:#c8e6c9
    style COLLECT fill:#bbdefb
```

### 提供上下文

```rust
// 使用 map_err 添加上下文
fs::read_to_string(path)
    .map_err(|e| format!("failed to read {}: {}", path, e))?;

// 使用 anyhow context
use anyhow::Context;
fs::read_to_string(path)
    .context(format!("failed to read {}", path))?;
```

---

## 10. Result 与 Option 互转

```mermaid
graph LR
    subgraph "Option -> Result"
        O1["opt.ok_or(err)"]
        O2["opt.ok_or_else(|| err)"]
    end

    subgraph "Result -> Option"
        R1["res.ok()"]
        R2["res.err()"]
    end

    OPTION["Option<T>"] --> O1
    OPTION --> O2
    O1 --> RESULT["Result<T,E>"]
    O2 --> RESULT

    RESULT --> R1
    RESULT --> R2
    R1 --> OPTION2["Option<T>"]
    R2 --> OPTION3["Option<E>"]

    style OPTION fill:#c8e6c9
    style RESULT fill:#bbdefb
```
