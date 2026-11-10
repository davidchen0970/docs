# C++ 編碼風格與慣例

## 整體理念

C++ 是一門龐大而複雜的語言，因此對於什麼是「好」或「壞」的 C++ 程式碼，往往存在不同的看法。Bjarne Stroustrup 曾說過：「在 C++ 之中，藏著一門更小、更簡單、更安全的語言，正努力掙脫而出。」我們致力於以這種 C++ 變體來撰寫程式碼，因此遵循 Bjarne 與 Herb Sutter 在 CppCon 2015 上提出的「C++ Core Guidelines」。

除了那些幫助界定「好」與「壞」C++ 的規則之外，我們還有一些通用原則，幫助我們將所開發的軟體與 OpenBMC 所要解決的問題領域中的限制保持一致。這些原則是：

1. 程式碼應該清晰簡潔。
2. 程式碼應該以現代化的慣例來撰寫。
3. 程式碼應該具備效能。

### 程式碼應該清晰簡潔

> 簡潔是機智的靈魂。

重要的是，程式碼應該為了審查者與維護者而最佳化，而不是為了撰寫者。解決方案應該避免那些會損害審查與理解清晰度的詭計。

現代化的慣例讓 C++ 得以成為一門富表現力卻又簡潔的語言。我們傾向於選擇那些以盡可能少的行數來精簡表達問題的解決方案。

當清晰度與簡潔性之間產生衝突時，應該以清晰度為優先。

### 程式碼應該以現代化的慣例來撰寫

我們致力於讓程式碼符合並運用最新的 C++ 標準。以今日而言，這代表所有的 C++ 程式碼都應該以 C++23 的編譯器設定來編譯。隨著 C++26 標準定案且編譯器開始支援它，我們也會跟著移轉過去。

我們同樣致力於讓程式碼庫與語言設計者所建議的最新慣例保持同步。這反映在我們選擇遵循 C++ Core Guidelines 上。

最後，我們希望在可能的情況下，盡量讓電腦替我們思考。這代表每個 repository 都應該有 Continuous Integration 測試，以便在合併之前快速找出 regression。這也代表要盡量讓本文件中更多的內容由工具來強制執行，例如 clang-format 與 clang-tidy。

對於那些從 pre-C++11 環境來到本專案的人，我們強烈推薦《Effective Modern C++》這本書，作為快速掌握 C++98/03 與 C++11/14/17/20 之間差異的方式。

### 程式碼應該具備效能

OpenBMC 的目標是嵌入式處理器，它們通常只有 32-64MB 的 flash，處理能力相當於 2016 年一般的智慧手錶。這代表有時候我們必須限制函式庫的選擇，及／或編碼技巧，以彌補這個限制。由於目前的技術，效能評估按照 { 程式碼大小、CPU 使用率、記憶體大小 } 的順序進行。

從巨觀最佳化的角度來看，我們期望所有的解決方案都針對當前問題具備適當的演算法複雜度。因此，即使某個 `O(n^3)` 的演算法清晰度良好，只要存在 `O(n*lg(n))` 的解決方案，它仍可能被拒絕。

## 全域準則與慣例

請遵循 C++ Core Guidelines (CCG) 所建立的準則。

<https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md>

[[最後審查的修訂版本為 53bc78f]]

例外情況：

### Guideline Support Library (GSL)

我們目前不使用 CCG 所提供的 Guideline Support Library。此刻可以忽略 CCG 中任何關於 GSL 慣例的建議。

### 風格建議

以下幾項不予遵循：

- NL.10 Avoid CamelCase
- NL.17 Use K&R-derived layout

## 函式庫與特性相關細節

關於特定語言特性或函式庫，OpenBMC 專案內的額外建議。

### 例外狀況

我們確實使用 exceptions 作為 OpenBMC 內錯誤處理的基礎。

### Boost

在以下情況下允許使用 boost：

1. 以僅頭檔案（header-only）函式庫的形式使用。這能讓未使用的函式與方法在連結時期被編譯器移除，避免增加大量的額外負擔與 flash 用量。
2. 實作者應該只包含解決當前問題所需的最低階頭檔案。這讓這些用法能夠在新標準可用時被找出並向前推進，同時也能縮短編譯時間，並降低意外使用的可能性。（例如 `#include <boost/container/flat_map.hpp>`，而不是 `#include <boost/container.hpp>`）
3. 所使用的模組在 std namespace 中不應該有符合相同實作或清晰度需求的對應物。例如，應該優先使用 std::chrono 而非 boost::chrono；std::array 而非 boost::array。
4. 使用上不應與本編碼標準的任何核心原則（清晰度、現代化慣例或效能）相衝突。

### iostream

iostream 使用 `operator<<` 的慣例會導致相對於 printf 風格操作增加程式碼大小，因為每個附加的值都需要個別的函式呼叫。因此，我們不使用 iostream 或 iostream 風格的 API 來做記錄（logging）。

有些情況下，使用 iostream 工具（例如 sstream）反而能產生更清晰且大小相近的程式碼。在那些情況下可以使用 iostream。

## 編碼風格

縮排、命名慣例等等。

### 一般

- 行長度應該限制在 80 個字元。
- 縮排應該使用 4 個空格字元。
- 檔案應該使用 Unix 風格的換行字元 (\n)。

### Clang 格式化

各個 OpenBMC repository 可以視需要使用
[clang-format](https://clang.llvm.org/docs/ClangFormat.html)。如果在 repository 的根目錄中找到 .clang_format 檔案，OpenBMC CI 基礎設施會在程式碼簽入（check-in）時自動驗證程式碼格式。這使得程式碼格式能在簽入時自動驗證。

如果需要使用自訂設定，例如對 C 與 C++ 檔案使用不同的 clang 格式，可以建立一個 format-code.sh 腳本，例如根據檔案類型使用不同的 .clang\* 檔案作為輸入。如果在 repository 的根目錄中找到 format-code.sh 腳本，它會作為 CI 的一部分執行，並檢查執行它之後是否沒有檔案被修改（與執行 clang 時的檢查相同）。

OpenBMC 需要 6.0 或更高版本的 clang-format。關於如何對 repository 中的所有程式碼執行 clang-format 的範例，可以參考 CI 所使用的
[tool](https://github.com/openbmc/openbmc-build-scripts/blob/master/scripts/format-code.sh)。

[Example .clang-format](https://www.github.com/openbmc/docs/blob/master/style/cpp/.clang-format)

### 括號風格

- 使用「Allman」風格的括號。括號應該獨立成一行，並與建立作用域（scope）的敘述處於相同的縮排層級。

```cpp
if (condition)
{
    ...
}
```

```cpp
void foo()
{
    ...
}
```

- 即使是單行的條件與迴圈敘述也應該加上括號。

```cpp
/// Wrong.
if (condition)
    do_something;

/// Correct
if (condition)
{
    do_something;
}
```

### 縮排

- namespace 內的內容應該與 namespace 本身處於相同的縮排層級。

```cpp
namespace foo
{

content

}
```

- class / struct 內的內容應該縮排。

```cpp
class Foo
{
    public:
        Foo();
}
```

- function / 條件 / 迴圈內的內容應該縮排。

```cpp
void foo()
{
    while (1)
    {
        if (bar())
        {
            ...
        }
    }
}
```

- Switch / case 敘述應該縮排。

```cpp
switch (foo)
{
    case bar:
    {
        bar();
        break;
    }

    case baz:
    {
        baz();
        break;
    }
}
```

- 標籤（label）應該縮排到比目前縮排少一層的位置，而不是貼齊左邊。（這並不是說偏好或應該經常使用 goto 與標籤，而只是說當使用它們時，應該以這種方式使用。）

```cpp
void foo()
{
    if (bar)
    {
        do
        {
            if (baz)
            {
                goto exit;
            }

        } while(1);

    exit:
        cleanup();
    }
}
```

### 命名慣例

- 我們大致上不在名稱上使用任何前綴或後綴。
- 首字母縮略字（acronym）應該 throughout 保持一致的大小寫，並遵循其對應章節中的要求。

```cpp
/// Correct.
SomeBMCType someBMCVariable = bmcFunction();

/// Wrong: type and variable are mixed-case, function isn't lowerCamelCase.
SomeBmcType someBmcVariable = BMCFunction();
```

### 頭檔案順序

頭檔案的 include 順序：

```text
local headers (e.g. "daemon_sys.hpp")
c-libraries
cpp-libraries (including openbmc libraries)
```

來源檔案（source file）的 include 順序：

```text
source.hpp (if applicable)
local headers
c-libraries
cpp-libraries
```

全部以字母順序排序。

#### 檔案

- C++ 頭檔案應該以「.hpp」結尾。C 頭檔案應該以「.h」結尾。
- C++ 檔案應該以 lower_snake_case 命名。

#### 型別

- 對於型別別名，優先使用 'using' 而非 'typedef'。
- Struct、class、enum 以及帶型別的範本參數都應該使用 UpperCamelCase。
- 優先使用 namespace 作用域，而非帶有前綴的長名稱。
- struct / class 內的單字型別別名可以使用小寫以符合 STL 慣例（`using type = T`），而多字型別別名則應該使用 UpperCamelCase（`using ArrayOfT = std::array<T, N>`）。
- 例外情況：函式庫 API 可以使用 lower_snake_case，以符合它所抽象化的 STL 或底層 C 函式庫的慣例。Application API 都應該使用 UpperCamelCase。
- 例外情況：為了方便而為範本類別所建立的範本型別別名，可以使用 `_t` 結尾，以符合 STL 的慣例。

```cpp
template <typename T>
class Foo
{
    using type = std::decay_t<T>;
};

template <typename T> using foo_t = Foo<T>::type;
```

#### 變數

- 所有變數都應該使用 lowerCamelCase，包括 class 成員在內，且不使用底線。

#### 函式

- 所有函式都應該使用 lowerCamelCase。
- 例外情況：函式庫 API 可以使用 lower_snake-case，以符合它所抽象化的 STL 或底層 C 函式庫的慣例。Application API 都應該使用 lowerCamelCase。

#### 常數

- 常數與 enum 成員應該像變數一樣以 lowerCamelCase 命名。

#### Namespace

- Namespace 應該使用 lower_snake_case。
- 最頂層的 namespace 應該根據其所在的 repository 來命名。
- 偏好使用名為 'details' 或 'internal' 的 namespace，以表示在頭檔案中相當於「private」的 namespace，以及在 C++ 檔案中的匿名 namespace。

### 頭檔案保護

優先使用 '#pragma once' 頭檔案保護，而非 '#ifndef' 風格。

### 額外空白

- 遵循 NL.18：Use C++-style declarator layout。

```cpp
foo(T& bar, const S* baz); /// Correct.
foo(T &bar, const S *baz); /// Incorrect.
```

- 遵循 NL.15：Use spaces sparingly。

- 在條件之後、括號之前插入空白。

```cpp
if (...)
while (...)
for (...)
```

- 為了可讀性，在二元運算子兩側插入空白。

```cpp
foo((a-1)/b,c-2); /// Incorrect.
foo((a - 1) / b, c - 2); /// Correct.
```

- 不要在單元運算子兩側插入空白。

```cpp
a = * b;  /// Incorrect.
a = & b;  /// Incorrect.
a = b -> c;  /// Incorrect.
if (! a)  /// Incorrect.
```

- 不要在括號內部，或在函式呼叫與參數之間插入空白。

```cpp
foo(x, y); /// Correct.
foo ( x , y ); /// Incorrect.

do (...)
{
} while(0); /// 'while' here is structured like a function call.
```

- 偏好將換行放在運算子之後，以表示接續（continuation）。

```cpp
if (this1 == that1 &&
    this2 == that2) /// Correct.

if (this1 == that1
    && this2 == that2) /// Incorrect.
```

- 長行應該讓接續部分對齊到與括號相同的層級，或讓括號內的所有項目都縮排兩層。

```cpp
reallyLongFunctionCall(foo,
                       bar,
                       baz); // Correct.

reallyLongFunctionCall(
        foo,
        bar,
        baz); // Also correct.

reallyLongFunctionCall(
        foo, bar, baz); // Similarly correct.

reallyLongFunctionCall(foo,
        bar,
        baz); // Incorrect.
```

### 其他準則

- 對於表示大小、數量等的東西，一律使用 `size_t` 或 `ssize_t`。當 size_t 就足夠時，使用帶有固定寬度的型別（例如 `uint8_t`）需要有強力的理由。

- 當大小因為與硬體互動或某些外部定義的 API/ABI（例如 system call 或函式庫介面）而重要時，使用 `uint8_t`、`int16_t`、`uint32_t`、`int64_t` 等型別。當不涉及這類互動時，不要在沒有充分理由的情況下使用它們；請優先使用 size_t 或 int。
