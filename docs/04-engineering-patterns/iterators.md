# 迭代器链

> **函数式思想** - `map` → `filter` → `collect` → `flatten`

## 什么是迭代器？

迭代器是 Rust 中处理集合数据的强大工具。它提供了：
- **惰性求值** - 只在需要时计算
- **零开销抽象** - 编译后和手写循环一样高效
- **链式调用** - 优雅地组合多个操作

## 基本用法

### 创建迭代器

```rust
let v1 = vec![1, 2, 3];
let v1_iter = v1.iter();  // 创建迭代器
```

### 使用迭代器

```rust
let v1 = vec![1, 2, 3];
let v1_iter = v1.iter();

for val in v1_iter {
    println!("得到: {}", val);
}
```

### 迭代器适配器

```rust
let v1: Vec<i32> = vec![1, 2, 3];
let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();

assert_eq!(v2, vec![2, 3, 4]);
```

## 常用迭代器方法

### map - 转换每个元素

```rust
let numbers = vec![1, 2, 3, 4, 5];
let doubled: Vec<i32> = numbers.iter()
    .map(|x| x * 2)
    .collect();

println!("{:?}", doubled);  // [2, 4, 6, 8, 10]
```

### filter - 过滤元素

```rust
let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
let evens: Vec<i32> = numbers.into_iter()
    .filter(|x| x % 2 == 0)
    .collect();

println!("{:?}", evens);  // [2, 4, 6, 8, 10]
```

### filter_map - 过滤并转换

```rust
let strings = vec!["1", "2", "three", "4", "five"];
let numbers: Vec<i32> = strings.iter()
    .filter_map(|s| s.parse().ok())
    .collect();

println!("{:?}", numbers);  // [1, 2, 4]
```

### collect - 收集结果

```rust
let numbers = vec![1, 2, 3];
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
let sum: i32 = numbers.iter().sum();
let count: usize = numbers.iter().count();
```

### fold - 累积操作

```rust
let numbers = vec![1, 2, 3, 4, 5];
let sum = numbers.iter().fold(0, |acc, x| acc + x);
println!("{}", sum);  // 15
```

### reduce - 归约操作

```rust
let numbers = vec![1, 2, 3, 4, 5];
let max = numbers.iter().reduce(|acc, x| if x > acc { x } else { acc });
println!("{:?}", max);  // Some(5)
```

### take - 取前 n 个元素

```rust
let numbers = vec![1, 2, 3, 4, 5];
let first_three: Vec<i32> = numbers.iter()
    .take(3)
    .cloned()
    .collect();

println!("{:?}", first_three);  // [1, 2, 3]
```

### skip - 跳过前 n 个元素

```rust
let numbers = vec![1, 2, 3, 4, 5];
let rest: Vec<i32> = numbers.iter()
    .skip(2)
    .cloned()
    .collect();

println!("{:?}", rest);  // [3, 4, 5]
```

### zip - 组合两个迭代器

```rust
let numbers = vec![1, 2, 3];
let letters = vec!['a', 'b', 'c'];
let zipped: Vec<_> = numbers.iter()
    .zip(letters.iter())
    .collect();

println!("{:?}", zipped);  // [(1, 'a'), (2, 'b'), (3, 'c')]
```

### enumerate - 添加索引

```rust
let items = vec!["a", "b", "c"];
for (index, item) in items.iter().enumerate() {
    println!("{}: {}", index, item);
}
```

### chain - 连接迭代器

```rust
let v1 = vec![1, 2, 3];
let v2 = vec![4, 5, 6];
let chained: Vec<i32> = v1.iter()
    .chain(v2.iter())
    .cloned()
    .collect();

println!("{:?}", chained);  // [1, 2, 3, 4, 5, 6]
```

### flatten - 展平嵌套结构

```rust
let nested = vec![vec![1, 2], vec![3, 4], vec![5, 6]];
let flattened: Vec<i32> = nested.iter()
    .flatten()
    .cloned()
    .collect();

println!("{:?}", flattened);  // [1, 2, 3, 4, 5, 6]
```

### flat_map - 映射并展平

```rust
let words = vec!["hello", "world"];
let chars: Vec<char> = words.iter()
    .flat_map(|s| s.chars())
    .collect();

println!("{:?}", chars);  // ['h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd']
```

## 链式调用示例

### 示例 1：处理数字列表

```rust
let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

let result: Vec<i32> = numbers.iter()
    .filter(|&x| x % 2 == 0)      // 过滤偶数
    .map(|x| x * x)                // 平方
    .filter(|&x| x > 10)           // 过滤大于 10 的
    .collect();

println!("{:?}", result);  // [16, 36, 64, 100]
```

### 示例 2：处理字符串

```rust
let text = "hello world rust programming";

let words: Vec<&str> = text.split_whitespace()
    .filter(|word| word.len() > 4)
    .map(|word| word.to_uppercase())
    .collect();

println!("{:?}", words);  // ["HELLO", "WORLD", "RUST", "PROGRAMMING"]
```

### 示例 3：处理用户数据

```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}

let users = vec![
    User { name: "Alice".to_string(), age: 30, active: true },
    User { name: "Bob".to_string(), age: 25, active: false },
    User { name: "Charlie".to_string(), age: 35, active: true },
];

let active_user_names: Vec<String> = users.iter()
    .filter(|user| user.active)
    .map(|user| user.name.clone())
    .collect();

println!("{:?}", active_user_names);  // ["Alice", "Charlie"]
```

## 惰性求值

迭代器是惰性的，只有在调用消费适配器时才会执行：

```rust
let v1: Vec<i32> = vec![1, 2, 3];
let v1_iter = v1.iter().map(|x| {
    println!("处理: {}", x);
    x + 1
});

// 此时还没有执行 map 操作
// 只有在调用 collect() 或其他消费适配器时才会执行

let v2: Vec<i32> = v1_iter.collect();
// 现在才会打印和处理
```

## 消费适配器 vs 迭代器适配器

### 消费适配器（Consuming Adapters）

消费适配器会消耗迭代器并返回一个值：

```rust
let numbers = vec![1, 2, 3, 4, 5];

// sum - 求和
let sum: i32 = numbers.iter().sum();

// count - 计数
let count = numbers.iter().count();

// collect - 收集
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();

// any - 检查是否有元素满足条件
let has_even = numbers.iter().any(|x| x % 2 == 0);

// all - 检查是否所有元素都满足条件
let all_positive = numbers.iter().all(|x| *x > 0);

// find - 查找第一个满足条件的元素
let first_even = numbers.iter().find(|x| x % 2 == 0);

// position - 查找第一个满足条件的元素的位置
let first_even_pos = numbers.iter().position(|x| x % 2 == 0);
```

### 迭代器适配器（Iterator Adapters）

迭代器适配器返回新的迭代器：

```rust
let numbers = vec![1, 2, 3, 4, 5];

// map, filter, take, skip, zip, chain, flatten 等
let doubled = numbers.iter().map(|x| x * 2);
let evens = numbers.iter().filter(|x| x % 2 == 0);
```

## 自定义迭代器

### 实现 Iterator trait

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

fn main() {
    let mut counter = Counter::new();

    assert_eq!(counter.next(), Some(1));
    assert_eq!(counter.next(), Some(2));
    assert_eq!(counter.next(), Some(3));
    assert_eq!(counter.next(), Some(4));
    assert_eq!(counter.next(), Some(5));
    assert_eq!(counter.next(), None);
}
```

### 使用自定义迭代器

```rust
let sum: u32 = Counter::new()
    .zip(Counter::new().skip(1))
    .map(|(a, b)| a * b)
    .filter(|x| x % 3 == 0)
    .sum();

assert_eq!(18, sum);
```

## 性能考虑

### 迭代器 vs 循环

```rust
// 使用循环
let mut result = Vec::new();
for item in items {
    if condition(item) {
        result.push(transform(item));
    }
}

// 使用迭代器（性能相同，但更优雅）
let result: Vec<_> = items.iter()
    .filter(|item| condition(item))
    .map(|item| transform(item))
    .collect();
```

**重要：** Rust 的迭代器在编译时会优化，性能与手写循环相同！

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 使用循环
std::vector<int> result;
for (int x : numbers) {
    if (x % 2 == 0) {
        result.push_back(x * 2);
    }
}
```

```rust
// Rust - 使用迭代器（更简洁，性能相同）
let result: Vec<i32> = numbers.iter()
    .filter(|x| x % 2 == 0)
    .map(|x| x * 2)
    .collect();
```

### Go 视角

```go
// Go - 使用循环
var result []int
for _, x := range numbers {
    if x % 2 == 0 {
        result = append(result, x * 2)
    }
}
```

```rust
// Rust - 使用迭代器（更函数式，性能更好）
let result: Vec<i32> = numbers.iter()
    .filter(|x| x % 2 == 0)
    .map(|x| x * 2)
    .collect();
```

## 实际应用示例

### 示例 1：处理 CSV 数据

```rust
let csv_data = "name,age,city\nAlice,30,NYC\nBob,25,LA";

let users: Vec<_> = csv_data.lines()
    .skip(1)  // 跳过标题行
    .map(|line| {
        let fields: Vec<&str> = line.split(',').collect();
        (fields[0], fields[1].parse::<u32>().unwrap(), fields[2])
    })
    .filter(|(_, age, _)| *age > 25)
    .collect();
```

### 示例 2：处理文件路径

```rust
use std::path::Path;

let paths = vec!["/home/user/file.txt", "/tmp/data.log", "/var/log/app.log"];

let log_files: Vec<&str> = paths.iter()
    .filter(|path| path.ends_with(".log"))
    .map(|path| Path::new(path).file_name().unwrap().to_str().unwrap())
    .collect();
```

### 示例 3：处理 API 响应

```rust
#[derive(Deserialize)]
struct ApiResponse {
    items: Vec<Item>,
}

struct Item {
    id: u32,
    name: String,
    active: bool,
}

let active_item_ids: Vec<u32> = response.items.iter()
    .filter(|item| item.active)
    .map(|item| item.id)
    .collect();
```

## 常见错误与解决方案

### 错误 1：忘记调用 collect()

```rust
let numbers = vec![1, 2, 3];
let doubled = numbers.iter().map(|x| x * 2);  // ❌ 这是迭代器，不是 Vec
// doubled 的类型是 Map<...>，不是 Vec<i32>
```

**解决方案：** 调用 `collect()`：

```rust
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
```

### 错误 2：多次使用迭代器

```rust
let numbers = vec![1, 2, 3];
let iter = numbers.iter();

let sum: i32 = iter.sum();
let count = iter.count();  // ❌ 迭代器已被消费
```

**解决方案：** 重新创建迭代器或使用 `by_ref()`：

```rust
let numbers = vec![1, 2, 3];
let mut iter = numbers.iter();

let sum: i32 = iter.by_ref().sum();
let count = iter.count();  // ✅
```

### 错误 3：所有权问题

```rust
let numbers = vec![1, 2, 3];
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
// numbers 仍然有效，因为 iter() 借用
```

## 实践建议

1. **优先使用迭代器** - 比循环更安全、更清晰
2. **利用链式调用** - 组合多个操作
3. **理解惰性求值** - 只在需要时计算
4. **使用类型注解** - 帮助编译器推断类型
5. **阅读标准库文档** - 了解更多迭代器方法

## 扩展练习

1. **实现一个简单的查询构建器** - 使用迭代器链
2. **处理大型数据集** - 使用迭代器避免加载所有数据到内存
3. **实现流式处理** - 使用迭代器处理流数据
4. **优化现有代码** - 将循环改为迭代器

## 下一步

掌握了迭代器后，接下来学习：
- **[错误处理](./error-handling.md)** - Rust 的错误处理哲学

---

**记住：迭代器是 Rust 函数式编程的核心，它让你写出既优雅又高效的代码！** 🦀
