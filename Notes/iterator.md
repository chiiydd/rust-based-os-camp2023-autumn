# 初始化和遍历

## next()

**fn next(&mut self) -> Option\<Self::item\>**

- next方法返回的是Option类型，有值的时候返回Some(元素类型)、无值的时候返回None

- 手动迭代必须将迭代器声明为 `mut` 可变，因为调用 `next` 会改变迭代器其中的状态数据（当前遍历的位置等），而 `for` 循环去迭代则无需标注 `mut`，因为它会帮我们自动完成

- next 方法对迭代器的遍历是消耗性的，每次消耗它一个元素，最终迭代器中将没有任何元素，只能返回 None

  ```rust
  let values = vec![1, 2, 3];
  
  {
      let result = match IntoIterator::into_iter(values) {
          mut iter => loop {
              match iter.next() {
                  Some(x) => { println!("{}", x); },
                  None => break,
              }
          },
      };
      result
  }
  
  ```

  

## [into_iter, iter, iter_mut](https://course.rs/advance/functional-programing/iterator.html#into_iter-iter-iter_mut)

- `into_iter` 会夺走所有权
- `iter` 是借用
- `iter_mut` 是可变借用

# 消费者与适配器

## 消费者适配器

消费者是迭代器上的方法，它会消费掉迭代器中的元素，然后返回其类型的值，这些消费者都有一个共同的特点：在它们的定义中，都依赖 `next` 方法来消费元素，因此这也是为什么迭代器要实现 `Iterator` 特征，而该特征必须要实现 `next` 方法的原因。

如代码注释中所说明的：在使用 `sum` 方法后，我们将无法再使用 `v1_iter`，因为 `sum` 拿走了该迭代器的所有权：

```rust
fn sum<S>(self) -> S
    where
        Self: Sized,
        S: Sum<Self::Item>,
    {
        Sum::sum(self)
    }
```

从 `sum` 源码中也可以清晰看出，`self` 类型的方法参数拿走了所有权。

除了sum、还有collect()、fold等

## 迭代器适配器

既然消费者适配器是消费掉迭代器，然后返回一个值。那么迭代器适配器，顾名思义，会返回一个新的迭代器，这是实现链式方法调用的关键：`v.iter().map().filter()...`。

与消费者适配器不同，迭代器适配器是惰性的，意味着你**需要一个消费者适配器来收尾，最终将迭代器转换成一个具体的值**

迭代器适配器是惰性的，只有被消费者适配器调用时，才会真正的执行。

```rust
let v1: Vec<i32> = vec![1, 2, 3];

v1.iter().map(|x| x + 1);
```

运行后输出:

```rust
warning: unused `Map` that must be used
 --> src/main.rs:4:5
  |
4 |     v1.iter().map(|x| x + 1);
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_must_use)]` on by default
  = note: iterators are lazy and do nothing unless consumed // 迭代器 map 是惰性的，这里不产生任何
```

- `map`：转换数据。接受一个闭包并为迭代器中的每个元素调用该闭包，然后返回一个新的迭代器，其中包含闭包返回的值。

```rust
let v = vec![1, 2, 3];
let v_squared: Vec<i32> = v.iter().map(|x| x * x).collect();
```

- `filter`：过滤数据。接受一个闭包并为迭代器中的每个元素调用该闭包。如果闭包返回true，则元素将包含在新的迭代器中。

```rust
let v = vec![1, 2, 3];
let v_even: Vec<&i32> = v.iter().filter(|x| *x % 2 == 0).collect();
```



迭代器是 Rust 的 **零成本抽象**（zero-cost abstractions）之一，意味着如果没有使用它，并不会带来运行时的开销。
