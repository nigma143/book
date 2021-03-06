## Обрабатываемые ошибки и `Result`

Множество ошибок не являются настолько критичными, чтобы останавливать выполнение
программы. Весьма часто необходима просто правильная их обработка. К примеру, при
открытии файла может произойти ошибка из-за отсутствия файла. Решения могут быть
разные: от игнорирования до создания нового файла.

Надеюсь, что вы ещё помните содержание главы 2, где мы рассматривали перечисление
`Result`. Оно имеет два значения `Ok` и `Err`.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`T` и `E` параметры перечисления. `T`  - это тип, которые будет возвращён, при
успехе, а `E` при ошибке.

Давайте вызовем функцию, которая возвращает значение `Result`, потому что эта функция
может потерпеть неудачу. В листинге 9-2 мы пытаемся открыть файл.

<span class="filename">Filename: src/main.rs</span>

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");
}
```

<span class="caption">Listing 9-2: Открытие файла</span>

Интересно, как узнать, какой тип возвращает метод `File::open`. Это просто. Надо
поставить тип  данных, который точно не подойдёт и увидим тип данных в описании
ошибки:

```rust,ignore,does_not_compile
let f: u32 = File::open("hello.txt");
```

Информационное сообщение:

```text
error[E0308]: mismatched types
 --> src/main.rs:4:18
  |
4 |     let f: u32 = File::open("hello.txt");
  |                  ^^^^^^^^^^^^^^^^^^^^^^^ expected u32, found enum
`std::result::Result`
  |
  = note: expected type `u32`
  = note:    found type `std::result::Result<std::fs::File, std::io::Error>`
```

Всё, я думаю, ясно из описания.

Для обработки исключительной ситуации необходимо добавить следующий код:

Когда вызов `File::open` успешен, значение в переменной `f` будет экземпляром `Ok` внутри которого содержится дескриптор файла. Если вызов не успешный, значением переменной `f` будет экземпляр `Err` который содержит больше информации про то какая ошибка произошла.

Мы добавим код из листинга 9-3 чтобы предпринять разные действия, в зависимости от значения, которое вернёт вызов `File::open`. Листинг 9-4 показывает один из способов обработки `Result` пользуясь базовыми инструментами языка, таким как выражение `match` рассмотренное в Главе 6.

<span class="filename">Filename: src/main.rs</span>

```rust,should_panic
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => {
            panic!("There was a problem opening the file: {:?}", error)
        },
    };
}
```

<span class="caption">Listing 9-3: Использование выражения <code>match</code> для обработки
<code>Result</code></span>

Обратите внимание, что перечисление `Result`, также как `Option`, входит в состав
экспорта по умолчанию.

Здесь мы сообщаем, что значение `Ok` содержит значение `file` типа `File`.
Другое значение может хранить значение типа `Err`. В этом примере мы используем
вызов макроса `panic!`. Если нет файла с именем *hello.txt*, будет выполнен этот код.
Следовательно, будет выведено следующее сообщение:

Другая ветвь `match` обрабатывает случай, где мы получаем значение `Err` вызова `File::open`. В этом примере мы используем вызов макроса `panic!`, если в нашей текущей директории нет файла с именем *hello.txt*, где будет выполнен этот код. Мы увидим следующее сообщение макроса `panic!` :

```text
thread 'main' panicked at 'There was a problem opening the file: Error { repr:
Os { code: 2, message: "No such file or directory" } }', src/main.rs:8
```

Как обычно, данное сообщение точно говорит, что пошло не правильно.

### Обработка различных ошибок с помощью match

Пример создание нового файла при отсутствии запрашиваемого файла:

<span class="filename">Filename: src/main.rs</span>

<!-- ignore this test because otherwise it creates hello.txt which causes other
tests to fail lol -->

```rust,ignore
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(ref error) if error.kind() == ErrorKind::NotFound => match File::create("hello.txt") {
            Ok(fc) => fc,
            Err(e) => panic!("Tried to create file but there was a problem: {:?}", e),
        },
        Err(error) => panic!("There was a problem opening the file: {:?}", error),
    };
    print!("{:#?}",f);
}
```

<span class="caption">Listing 9-4: Обработка различных ошибок несколькими способами</span>

Типом значения возвращаемого функцией `File::open` внутри `Err` варианта является `io::Error`, который представляет из себя структуру предоставляемую стандартной библиотекой. Данная структура  имеет метод  `kind`, который можно вызвать для получения значение `io::ErrorKind`. Перечисление `io::ErrorKind` предоставлено стандартной библиотеке и имеет варианты представляющие различные типы ошибок, которые могут появиться в операциях `ввода/вывода`. Вариант, который мы хотим использовать это `ErrorKind::NotFound`. Он даёт информацию, что данный файл ещё не существует. Так что мы попадаем на match ветвь `f`, но также у нас есть внутренний match для `error.kind()`.

Условие, которое мы проверяем на внутреннем match - это является ли возвращённое значение `error.kind()` вариантом  `NotFound` для перечисления `ErrorKind`. Если является, то пробуем создать новый файл с помощью `File::create`. Те не менее, так как `File::create` также может завершиться не успешно, то нам необходима вторая ветка внутреннего `match` выражения. Когда файл не может быть создан, то печатаются разные сообщения. Вторая ветвь внешнего кода `match` остаётся той же самой, так что программа "паникует" на любой другой ошибке кроме отсутствующего файла.

Достаточно про `match`! Код с `match` выражением является очень удобным, но также достаточно примитивным. В главе 13, вы узнаете про замыкания (closures), а у типа `Result<T, E>` есть много методов, которые принимают замыкание и реализованы с помощью выражения`match`. Использование данных методов сделает ваш код более лаконичным. Более опытные разработчики могут написать это как в листинге 9-5:

```rust,ignore
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound {
            File::create("hello.txt").unwrap_or_else(|error| {
                panic!("Problem creating the file: {:?}", error);
            })
        } else {
            panic!("Problem opening the file: {:?}", error);
        }
    });
}
```

Не смотря на то, что данный код имеет такое же поведение как в листинге 9-5, он не содержит ни одного выражения `match` и более чист для чтения. Вернёмся к этому примеру после чтения главы 13 и рассмотрим метод `unwrap_or_else` из стандартной библиотеки. Многие их этих методов могут очистить код от больших, вложенных выражений `match` при обработке ошибок.

### Сокращённые макросы обработки ошибок `unwrap` и `expect`

Метод `unwrap` - это оболочка выражения `match`, которая возвращает `Ok` или `Err`.

<span class="filename">Если мы выполним код без наличия файла <em>hello.txt</em>, будет выведена ошибка:</span>

```rust,should_panic
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").unwrap();
    print!("{:#?}", f);
}
```

Есть ещё один метод похожий на `unwrap`. Используя `expect` вместо `unwrap` и
предоставляющий хорошие информативные описания ошибок::

```text
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Error {
repr: Os { code: 2, message: "No such file or directory" } }',
/stable-dist-rustc/build/src/libcore/result.rs:868
```

Другой метод похожий на `unwrap` это метод `expect`, позволяющий выбрать сообщение для ошибки макроса `panic!`. Использование `expect` вместо `unwrap` с предоставлением хорошего сообщения об ошибке, передаёт ваше намерение и делает отслеживание источника ошибки макроса `panic!` более простым. Синтаксис метода `expect` выглядит так:

<span class="filename">Название файла: src/main.rs</span>

```rust,should_panic
use std::fs::File;

fn main() {
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
    print!("{:?}", f);
}
```

Мы используем `expect` таким же образом, каким и `unwrap`: возвращаем ссылку на файл или
вызов макроса `panic!`.

```text
thread 'main' panicked at 'Failed to open hello.txt: Error { repr: Os { code:
2, message: "No such file or directory" } }', src/libcore/result.rs:906:4
```

Так как сообщение об ошибке начинается с нашего пользовательского текста: `Failed to open hello.txt`, то потом будет проще найти из какого места в коде данное сообщение происходит. Если использовать `unwrap` во множестве мест, то придётся потратить время для выяснения какой именно вызов `unwrap` вызывает "панику", так как все вызовы  `unwrap` генерируют одинаковое сообщение.

### Распространение генерируемых ошибок

Когда вы пишите функцию, в результате работы которой может произойти непредвиденная
ошибка, вместо того, чтобы обрабатывать эту ошибку вы можете создать подробное
описание этой и передать ошибку по цепочке на верхний уровень обработки кода.

Например, код программы 9-5 читает имя пользователя из файла. Если файл не существует
или не может быть прочтён, то функция возвращает эту ошибку в код, который вызвал
эту функцию:

<span class="filename">Название файла: src/main.rs</span>

```rust
use std::io;
use std::io::Read;
use std::fs::File;

fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");

    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut s = String::new();

    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}
```

<span class="caption">Listing 9-5: Функция, которая возвращает ошибки в вызывающий код, 
используя выражение <code>match</code></span>

Данную функцию можно записать гораздо короче, но чтобы изучить обработку ошибок, мы собираемся сделать многое вручную. А в конце будет показан более короткий способ. Давайте, сначала рассмотрим тип возвращаемого значения `Result<String, io::Error>`. 
Здесь есть возвращаемое значение функции типа `Result<T, E>` где шаблонный параметр `T` был заполнен конкретным типом `String` и шаблонный параметр `E` был заполнен конкретным типом `io::Error`. Если эта функция будет выполнена успешно, будет возвращено `Ok`, содержащее значение типа `String`, т.е. имя пользователя прочитанное функцией из файла. Если же при чтении файла будут какие-либо проблемы, то вызывающий код получит значение `Err` с экземпляром типа `io::Error`, которое содержит больше информации о проблеме. Тип `io::Error` для возвращаемого типа выбран потому что он является типом ошибки возвращаемой  из обоих операций, вызываемых в теле функции, которые могут не завершиться успешно. Это функции `File::open` и `read_to_string`.

Тело функции начинается с вызова `File::open`. Затем обрабатывается значение `Result` возвращаемое с помощью `match` аналогично коду `match` листинга 9-4, но только вместо вызова `panic!` для случая `Err`, делается ранний возврат из данной функции и передача значения ошибки из `File::open` обратно в вызвавший код, как значение ошибки уже текущей функции. Если `File::open` будет успешен, сохраняется дескриптор файла в переменной `f` и выполнение продолжается далее.

Тело этой функции начинается с вызову функции `File::open`. Далее, мы получаем результат
анализа результата чтения файла функцией `match`. Если функция `File::open` сработала
успешно, мы сохраняет ссылку на файл в переменную `f` и программа продолжает свою
работу.

Далее, мы создаём строковую переменную `s` и вызываем метод файла `read_to_string`,
который читает содержание файла, как строковые данные, в переменную `s`. Результатом
работы этой функции будет значение перечисления `Result`: `Ok` или `io::Error`.

Этого же результата можно достичь с помощью сокращённого написания (с помощью использования
символа `?`).

#### Сокращённое описание для оператора распространения ошибки `?`

Код программы 9-6 показывает реализацию функции `read_username_from_file`, функционал
которой аналогичен коду программы 9-5, но имеет сокращённое описание:

<span class="filename">Название файла: src/main.rs</span>

```rust
use std::io;
use std::io::Read;
use std::fs::File;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

<span class="caption">Код программы 9-6: Пример функции, которая возвращает ошибку,
используя символ <code>?</code></span>

Благодаря использованию символа `?` сокращается запись кода (код, написанный в
предыдущем примере создаётся компилятором самостоятельно).

Есть разница между тем, что делает выражение `match` листинга 9-6 и оператор `?`. Ошибочные значения при выполнении методов с оператором `?` возвращены из функции `from`, определённой в типаже `From` стандартной библиотеки. Данный типаж используется для конвертирования ошибок одного типа в другой тип ошибок. Когда оператор `?` вызывает функцию `from`, то полученный тип ошибки конвертируется в тип ошибки, который определён для возврата в текущей функции. Это удобно, когда функция возвращает один тип ошибки для представления всех возможных вариантов из-за которых она может не завершиться успешно, даже если части кода функции могут прервать выполнение по разным причинам. Так как каждый стандартный тип ошибки реализует функцию `from` определяя, как конвертировать себя в возвращаемый тип ошибки, то оператор `?` позаботится о такой конвертации автоматически.

В коде примера 9-6 в первой строке функция `File::open` возвращает содержимое значения
перечисления `Ok` в переменную `f`. Если же в при работе этой функции происходит
ошибка, будет возвращён экземпляр структуры `Err`. Те же самые действия произойдут
при чтении текстовых данных из файла с помощью функции `read_to_string`.

Использование сокращённых конструкций позволят уменьшить количество строк кода и
место потенциальных ошибок. Написанный в предыдущем примере сокращённый код можно
сделать ещё меньше с помощью сокращения промежуточных переменных и конвейерного вызова
методов:

<span class="filename">Название файла: src/main.rs</span>

```rust
use std::io;
use std::io::Read;
use std::fs::File;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();

    File::open("hello.txt")?.read_to_string(&mut s)?;

    Ok(s)
}
```

<span class="caption">Listing 9-8: Конвейерное комбинирование вызовов методов после оператора <code>?</code></span>

Мы перенесли создание экземпляра структуры `String` в начало функции. Вместо того,
чтобы создавать переменную `f` мы последовательно вызываем методы экземпляров
выходные данных.

Обсуждая разные способы записи данной функции, на листинге 9-9 показан способ сделать его ещё короче.

<span class="filename">Название файла: src/main.rs</span>

```rust
use std::io;
use std::fs;

fn read_username_from_file() -> Result<String, io::Error> {
    fs::read_to_string("hello.txt")
}
```

<span class="caption">Листинг 9-9: Использование <code>fs::read_to_string</code> вместо открытия и чтения файла</span>

Чтение файла в строку довольно распространённая операция, так что Rust предоставляет удобную функцию `fs::read_to_string` открытия файла, создание новой `String`, чтения содержимого файла, размещение содержимого в эту `String` и возврат её. Конечно, использование функции `fs::read_to_string` не даёт возможности объяснить все обработки ошибок, поэтому мы сначала изучили длинный способ.

#### Оператор `?` можно использовать для функций возвращающих `Result`

Сокращённую запись с помощью символа `?` можно использовать в функциях, которые возвращают значение перечисления `Result`. Соответственно, если функция не возвращает значение перечисления `Result`, а в коде написано обратное - компилятор сгенерирует
ошибку. Пример:

Посмотрим что происходит, если использовать оператор `?` в коде функции `main`, которая как вы помните имеет возвращаемый тип `()`:

```rust,ignore,does_not_compile
use std::fs::File;

fn main() {
    let f = File::open("hello.txt")?;
}
```

Описание ошибки:

```text
error[E0277]: the `?` operator can only be used in a function that returns
`Result` or `Option` (or another type that implements `std::ops::Try`)
 --> src/main.rs:4:13
  |
4 |     let f = File::open("hello.txt")?;
  |             ^^^^^^^^^^^^^^^^^^^^^^^^ cannot use the `?` operator in a
  function that returns `()`
  |
  = help: the trait `std::ops::Try` is not implemented for `()`
  = note: required by `std::ops::Try::from_error`
```

Данная ошибка указывает, что можно использовать оператор `?` в функции, возвращающей `Result<T, E>`. Когда вы пишите код в функции, которая не возвращает `Result<T, E>` и хотите использовать оператор `?`, при вызове других функций, которые возвращают `Result<T, E>`, то у вас есть два способ исправить проблему. Изменить возвращаемый тип вашей функции на `Result<T, E>`, если ничего не ограничивает это сделать. Другая техника это использование `match` или одного из методов типа  `Result<T, E>` для обработки `Result<T, E>` любым подходящим способом.

Функция `main` является специальной и есть ограничение, какой возвращаемый тип у неё должен быть. Один из допустимых типов для main это `()`, другой допустимый возвращаемый тип  `Result<T, E>`, как в примере:

```rust,ignore
error[E0308]: mismatched types
 -->
  |
3 |     let f = File::open("hello.txt")?;
  |             ^^^^^^^^^^^^^^^^^^^^^^^^^ expected (), found enum
`std::result::Result`
  |
  = note: expected type `()`
  = note:    found type `std::result::Result<_, _>`
```

В описании ошибки сообщается, что функция `main` должна возвращать кортеж, а вместо
этого - функция возвращает `Result`.

В следующем разделе будет рассказано об особенностях вызова макроса `panic!`, приведены
рекомендации при выборе конструкции для отслеживания ошибок.


