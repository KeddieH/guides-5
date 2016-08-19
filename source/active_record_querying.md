**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON http://guides.rubyonrails.org.**

Active Record 查詢
=============================

本篇詳細介紹使用 Active Record 從資料庫取出資料的各種方法。

讀完本篇，您將了解：

* 如何使用各種方法和條件來查找資料庫記錄 (record)。
* 如何替找出的資料庫記錄指定排序方式、要取出的屬性、分組、與其他特性。
* 如何使用 eager loading 來減少取出資料時的資料庫查詢次數。
* 如何使用動態查詢方法。
* 如何使用方法鏈 (method chaining) 來同時使用多個 Active Record 方法。
* 如何檢查特定的資料庫記錄是否存在。
* 如何在 Active Record model 中做各種運算。
* 如何在 relation 中使用 EXPLAIN。

--------------------------------------------------------------------------------

若您習慣使用純 SQL 來查找資料庫記錄，就會發現在 Rails 中可以用更好的方式來完成相同的操作。Active Record 讓您在多數情況下都不需要使用 SQL。

本篇的例子都會用到下列的 model 來講解：


TIP: 除非有特別說明，不然下列的 model 都用 id 做為主鍵。

```ruby
class Client < ApplicationRecord
  has_one :address
  has_many :orders
  has_and_belongs_to_many :roles
end
```

```ruby
class Address < ApplicationRecord
  belongs_to :client
end
```

```ruby
class Order < ApplicationRecord
  belongs_to :client, counter_cache: true
end
```

```ruby
class Role < ApplicationRecord
  has_and_belongs_to_many :clients
end
```

Active Record 會幫您查詢資料庫，且相容多數的資料庫系統，包括 MySQL、MariaDB、PostgreSQL、與 SQLite。不管使用哪種資料庫，Active Record 的方法格式都是一樣的。


1 取出資料
------------------------------------

Active Record 提供了多種查找方法 (finder method) 來從資料庫取出物件。每個查找方法都可以傳入引數來對資料庫執行不同的查詢，不用寫純 SQL。

查找方法有：

* `find`
* `create_with`
* `distinct`
* `eager_load`
* `extending`
* `from`
* `group`
* `having`
* `includes`
* `joins`
* `left_outer_joins`
* `limit`
* `lock`
* `none`
* `offset`
* `order`
* `preload`
* `readonly`
* `references`
* `reorder`
* `reverse_order`
* `select`
* `distinct`
* `where`

以上方法皆會回傳一個`ActiveRecord::Relation`實體。

`Model.find(options)`的主要操作可以總結如下：

* 將傳入的參數轉換成相對應的 SQL 查詢語句。
* 執行 SQL 語句，從資料庫中取出對應的結果。
* 根據每個查詢結果，從適當的 model 實體化出 Ruby 物件。
* 若有`after_find`與`after_initialize`回呼的話，會執行它們。 

### 取出單一物件

Active Record 提供了不同的方式來取出單一物件。


#### 1.1.1`find`

使用`find`方法，傳入的參數會對應到物件的主鍵 (primary key)，並取出該物件。例如：

```ruby
# Find the client with primary key (id) 10.
client = Client.find(10)
# => #<Client id: 10, first_name: "Ryan">
```

等同於以下的 SQL：


```sql
SELECT * FROM clients WHERE (clients.id = 10) LIMIT 1
```

若沒有找到符合的記錄，`find`會拋出`ActiveRecord::RecordNotFound`異常。

在使用`find`時，也可以傳入一個主鍵陣列來同時查找多個物件。這會回傳一個陣列，包含所有符合主鍵的記錄。


```ruby
# Find the clients with primary keys 1 and 10.
client = Client.find([1, 10]) # Or even Client.find(1, 10)
# => [#<Client id: 1, first_name: "Lifo">, #<Client id: 10, first_name: "Ryan">]
```

等同於以下的 SQL：

```sql
SELECT * FROM clients WHERE (clients.id IN (1,10))
```

WARNING: 除非**所有**傳入的主鍵都有相符的物件，否則`find`方法會拋出`ActiveRecord::RecordNotFound`異常。

#### 1.1.2`take`

`take`方法不會照順序來取出記錄。例如：


```ruby
client = Client.take
# => #<Client id: 1, first_name: "Lifo">
```

等同於以下的 SQL：

```sql
SELECT * FROM clients LIMIT 1
```

若沒有找到記錄，`take`會回傳`nil`，且不會拋出異常。

您可以傳入一個數值引數給`take`，會回傳多筆結果 (不超過該數值)。例如：

```ruby
client = Client.take(2)
# => [
#   #<Client id: 1, first_name: "Lifo">,
#   #<Client id: 220, first_name: "Sara">
# ]
```

等同於以下的 SQL：

```sql
SELECT * FROM clients LIMIT 2
```

`take!`方法跟`take`一模一樣，只是當沒有符合的記錄時，會拋出`ActiveRecord::RecordNotFound`異常。

TIP: 用`take`取出的紀錄會依不同的資料庫引擎而有所不同。

#### 1.1.3`first`

`first`方法在預設的情況下，會依照主鍵的排序來取出第一筆記錄。例如：


```ruby
client = Client.first
# => #<Client id: 1, first_name: "Lifo">
```

等同於以下的 SQL：


```sql
SELECT * FROM clients ORDER BY clients.id ASC LIMIT 1
```

若沒有找到符合的記錄，`first`會回傳`nil`，且不會拋出異常。


若您的[預設環境](active_record_querying.html#applying-a-default-scope) 有設定排序方法，`first`會根據該排序方法來回傳第一筆記錄。 

您可以傳入一個數值引數給`first`來取出多筆記錄。例如：傳入3，會回傳第一到第三筆記錄。

```ruby
client = Client.first(3)
# => [
#   #<Client id: 1, first_name: "Lifo">,
#   #<Client id: 2, first_name: "Fifo">,
#   #<Client id: 3, first_name: "Filo">
# ]
```

等同於以下的 SQL：


```sql
SELECT * FROM clients ORDER BY clients.id ASC LIMIT 3
```

若經過`order`方法排序後，`first`會回傳依照該`order`排序的第一筆記錄。


```ruby
client = Client.order(:first_name).first
# => #<Client id: 2, first_name: "Fifo">
```

等同於以下的 SQL：


```sql
SELECT * FROM clients ORDER BY clients.first_name ASC LIMIT 1
```

`first!`方法跟`first`一模一樣，只是當沒有符合的記錄時，會拋出`ActiveRecord::RecordNotFound`異常。


#### 1.1.4`last`

`last`方法在預設的情況下，會依照主鍵的排序來取出最後一筆記錄。例如：


```ruby
client = Client.last
# => #<Client id: 221, first_name: "Russel">
```
等同於以下的 SQL：

```sql
SELECT * FROM clients ORDER BY clients.id DESC LIMIT 1
```

若沒有找到符合的記錄，`last`會回傳`nil`，且不會拋出異常。


若您的[預設環境](active_record_querying.html#applying-a-default-scope) 有設定排序方法，`last`會根據該排序方法來回傳最後一筆記錄。 

您可以傳入一個數值引數給`last`來取出多筆記錄。例如：傳入3，會回傳三筆資料，從最後一筆往前推。


```ruby
client = Client.last(3)
# => [
#   #<Client id: 219, first_name: "James">,
#   #<Client id: 220, first_name: "Sara">,
#   #<Client id: 221, first_name: "Russel">
# ]
```

等同於以下的 SQL：


```sql
SELECT * FROM clients ORDER BY clients.id DESC LIMIT 3
```

若經過`order`方法排序後，`last`會回傳依照該`order`排序的最後一筆記錄。


```ruby
client = Client.order(:first_name).last
# => #<Client id: 220, first_name: "Sara">
```

等同於以下的 SQL：


```sql
SELECT * FROM clients ORDER BY clients.first_name DESC LIMIT 1
```

`last!`方法跟`last`一模一樣，只是當沒有符合的記錄時，會拋出`ActiveRecord::RecordNotFound`異常。


#### 1.1.5`find_by`

`find_by`方法會取出符合條件的第一筆記錄。例如：

```ruby
Client.find_by first_name: 'Lifo'
# => #<Client id: 1, first_name: "Lifo">

Client.find_by first_name: 'Jon'
# => nil
```

等同於：


```ruby
Client.where(first_name: 'Lifo').take
```

等同於以下的 SQL：


```sql
SELECT * FROM clients WHERE (clients.first_name = 'Lifo') LIMIT 1
```

`find_by!`方法跟`find_by`一模一樣，只是當沒有符合的記錄時，會拋出`ActiveRecord::RecordNotFound`異常。例如：


```ruby
Client.find_by! first_name: 'does not exist'
# => ActiveRecord::RecordNotFound
```

等同於：


```ruby
Client.where(first_name: 'does not exist').take!
```

### 1.2 批次取出多筆記錄

很多時候，我們需要對大量的記錄做同樣的事，像是寄信給大量的使用者或是輸出資料。

這也許是最直接的作法：

```ruby
# 若遇到龐大的資料表，這會佔用過多的記憶體
User.all.each do |user|
  NewsMailer.weekly(user).deliver_now
end
```
當資料表越大，這樣的方式就越不可行。`User.all.each`會告訴 Active Record 一次把 _整張資料表_ 抓出來，接著從每一列建出物件，最後將所有物件放在記憶體中。可想而知，若資料庫有龐大數量的紀錄，記憶體就不夠用了。

為了解決這個問題，Rails 提供了兩種方法來將記錄分批成適合記憶體的大小來處理。第一種方法是`find_each`，會取出一批記錄，分別傳入 _每一筆_ 記錄到區塊中。第二種方法是`find_in_batches`，會取出一批記錄，然後以陣列的方式傳入 _整批_ 記錄到區塊中。

TIP: `find_each`與`find_in_batches`方法專門用來批次處理記憶體無法負荷的龐大資料量。若只有一千筆資料，使用平常的查詢方法就夠了。 

#### 1.2.1`find_each`

`find_each`方法會取出一批記錄，分別傳入 _每一筆_ 記錄到區塊中。在下面的例子中，`find_each`一次會取出 1000 筆 user 記錄，並一個一個傳入區塊中：

```ruby
User.find_each do |user|
  NewsMailer.weekly(user).deliver_now
end
```

一批處理完後，會再取出下一批，直到所有記錄都處理完畢。

如上例所示，`find_each`可以用在 model 類別上。此外，也可以用在 relation：

```ruby
User.where(weekly_subscriber: true).find_each do |user|
  NewsMailer.weekly(user).deliver_now
end
```

由於批次處理會強制使用固定的排序，故無法自行加上排序方式。

若有排序，當`config.active_record.error_on_ignored_order`為 true 時，會拋出`ArgumentError`，否則在預設 (false) 的情況下，會發出警告並直接忽略排序。預設值可以用 `:error_on_ignore`選項來更改，以下會說明。


##### 1.2.1.1`find_each`可用的選項

**`:batch_size`**

`:batch_size`選項可以指定每一批要取出多少筆記錄。例如，一次取出 5000 筆：

```ruby
User.find_each(batch_size: 5000) do |user|
  NewsMailer.weekly(user).deliver_now
end
```

**`:start`**

在預設的情況下，會依照主鍵以遞增的順序來抓取記錄 (主鍵的型別必須為整數)。當您不想要從最小的 ID 開始時，`:start`選項可以設定要從哪個 ID 開始取出。這對某些情況很有幫助，像是當批次處理的過程被打斷了，只要有記住最後一個被處理過的 ID，就可以從那繼續開始。
   
舉例來說，只寄信給 ID 2000 以後的使用者：


```ruby
User.find_each(start: 2000) do |user|
  NewsMailer.weekly(user).deliver_now
end
```

**`:finish`**

類似`:start`選項，`:finish`能讓您設定要取到哪個 ID 為止。這在想要針對特定記錄做處理時很有幫助。
  
例如，只寄信給 ID 2000 到 10000 的使用者時：


```ruby
User.find_each(start: 2000, finish: 10000) do |user|
  NewsMailer.weekly(user).deliver_now
end
```

另一個例子是，如果您要一次讓很多人負責同一組記錄，可以分別設定`:start`與`:finish`來讓他們各別處理 10000 筆記錄。


**`:error_on_ignore`**

可以指定當關連中有排序時是否要拋出錯誤。


####1.2.2`find_in_batches`

`find_in_batches`跟`find_each`很像，兩個都會批次取出紀錄。唯一的不同在於`find_in_batches`會以陣列的方式傳入 _整批_ 記錄到區塊中，而不是個別傳入。下面的例子會一次傳入最多 1000 筆帳單到區塊處理，並在下個區塊處理剩下的帳單：

```ruby
# Give add_invoices an array of 1000 invoices at a time.
Invoice.find_in_batches do |invoices|
  export.add_invoices(invoices)
end
```

如上所示，`find_in_batches`可以用在 model 類別上。此外，也可以用在 relation：

```ruby
Invoice.pending.find_in_batches do |invoice|
  pending_invoices_export.add_invoices(invoices)
end
```

由於批次處理會強制使用固定的排序，故無法自行加上排序方式。


#####1.2.2.1 `find_in_batches`可用的選項

`find_in_batches`可以使用的選項跟跟`find_each`一樣。

2 條件
----------

`where`方法可以用來取出符合條件的紀錄，即代表 SQL 語句中的`WHERE`部分。條件可以是字串、陣列、或是 hash。


### 2.1 純字串條件

若想替查詢加入條件，可以用字串的形式直接傳入 where。像是`Client.where("orders_count = '2'")`，會回傳所有`orders_count`是 2 的客戶。

WARNING: 使用純字串條件會有 SQL injection (SQL 資料隱碼攻擊) 的風險。例如，`Client.where("first_name LIKE '%#{params[:first_name]}%'")`是不安全的。請看下節來了解如何用陣列來處理條件。

### 2.2 陣列條件

如果要找的 orders_count 是不固定的數字，像是要放入從某處來的引數該怎麼辦？可以這樣寫：

```ruby
Client.where("orders_count = ?", params[:orders])
```

Active Record 會將第一個引數視為條件字串，其他的引數則會取代其中的`?`。上例會將`?`換成`params[:orders]`來查詢。

也可以一次宣告多個條件：

```ruby
Client.where("orders_count = ? AND locked = ?", params[:orders], false)
```

上述例子中，第一個`?`會換成`params[:orders]`，第二個`?`則會換成 SQL 裡的`false`(根據不同的 adapter 而異)。

這樣寫...

```ruby
Client.where("orders_count = ?", params[:orders])
```

比下面這種寫法好多了

```ruby
Client.where("orders_count = #{params[:orders]}")
```

前者的寫法比較好，因為較安全。將變數直接放進條件字串裡， _不管變數是什麼_ 都會直接存入資料庫。這表示，惡意的使用者可以直接將變數存入資料庫。這樣做等於是把整個資料庫放在風險中，因為一旦有使用者發現他可以存入任何變數時，就可以對資料庫做任何事了。所以，千萬不要把變數直接放入條件字串裡。

TIP: 想了解更多關於 SQL injection 的資訊，請參閱[Ruby on Rails 安全指南](security.html#sql-injection).

#### 2.2.1 佔位符

類似`(?)`的取代方式，您也可以在條件字串中宣告鍵 (key) 和相對應的 hash 鍵值對 (key/value)：


```ruby
Client.where("created_at >= :start_date AND created_at <= :end_date",
  {start_date: params[:start_date], end_date: params[:end_date]})
```

當有很多條件時，這樣的方法能夠增加可讀性。


### 2.3 Hash 條件

Active Record 也接受 hash 條件，可以增加條件式的可讀性。使用 hash 條件時，鍵是要查詢的欄位，值是查詢的方式：


NOTE: 只有 equality、range、與 subset 可以用來寫 hash 條件。

#### 2.3.1 Equality 條件

```ruby
Client.where(locked: true)
```

這會產生下面的 SQL：

```sql
SELECT * FROM clients WHERE (clients.locked = 1)
```

欄位名稱也可以是字串：

```ruby
Client.where('locked' => true)
```

在 belongs_to 的關係中，若 Active Record 物件被當做值來使用，關聯鍵也可以用來查詢。polymorphic (多型性) 關係也可以用這樣的方式。


```ruby
Article.where(author: author)
Author.joins(:articles).where(articles: { author: author })
```

NOTE: 條件的值不能用符號。舉例來說，`Client.where(status: :active)`是不行的。

#### 2.3.2 Range 條件

```ruby
Client.where(created_at: (Time.now.midnight - 1.day)..Time.now.midnight)
```

這會用 SQL 的`BETWEEN`來找出所有昨天建立的客戶：


```sql
SELECT * FROM clients WHERE (clients.created_at BETWEEN '2008-12-21 00:00:00' AND '2008-12-22 00:00:00')
```

這種寫法示範了如何簡化[陣列條件](#array-conditions)中舉的例子。

#### 2.3.3 Subset 條件

若想用 SQL 的`IN`來查詢，可以在條件 hash 中傳入陣列：


```ruby
Client.where(orders_count: [1,3,5])
```

這會產生下列的 SQL：


```sql
SELECT * FROM clients WHERE (clients.orders_count IN (1,3,5))
```

### 2.4 NOT 條件

SQL 的`NOT`可以使用`where.not`來寫：


```ruby
Client.where.not(locked: true)
```
換句話說，可以先呼叫不帶引數的`where`，緊接著加上`not`傳入`where`條件。這會產生如下的 SQL：


```sql
SELECT * FROM clients WHERE (clients.locked != 1)
```

3 排序
--------

使用`order`方法，可以按特定的順序取出記錄。

例如有一組記錄，想要按照`created_at`欄位做遞增排列：


```ruby
Client.order(:created_at)
# OR
Client.order("created_at")
```

也可以用`ASC`(遞增排列)或`DESC`(遞減排列)： 


```ruby
Client.order(created_at: :desc)
# OR
Client.order(created_at: :asc)
# OR
Client.order("created_at DESC")
# OR
Client.order("created_at ASC")
```

也可以按不同欄位排序：


```ruby
Client.order(orders_count: :asc, created_at: :desc)
# OR
Client.order(:orders_count, created_at: :desc)
# OR
Client.order("orders_count ASC, created_at DESC")
# OR
Client.order("orders_count ASC", "created_at DESC")
```

如果想要一次呼叫很多`order`，後面的`order`會附在第一個`order`之後。


```ruby
Client.order("orders_count ASC").order("created_at DESC")
# SELECT * FROM clients ORDER BY orders_count ASC, created_at DESC
```

4 選出特定欄位
-------------------------

在預設的情況下，`Model.find`會使用`select *`來取出所有欄位。


若只想取出特定的欄位，可以用`select`方法來宣告。

例如，只想要`viewable_by`與`locked`欄位：

```ruby
Client.select("viewable_by, locked")
```

這會產生如下的 SQL 語句：

```sql
SELECT viewable_by, locked FROM clients
```

注意，使用`select`代表只從您選取的欄位來實體化物件。若試圖存取不存在的欄位，會得到`ActiveModel::MissingAttributeError`異常：


```bash
ActiveModel::MissingAttributeError: missing attribute: <attribute>
```

上面的`<attribute>`是您要取出的欄位。`id`方法不會拋出`ActiveRecord::MissingAttributeError`異常，所以在關聯裡使用的時候要特別小心，因為它們需要`id`方法才能正常運作。

若想找出一個欄位中所有不同的值，可以使用`distinct`，每個值只會回傳一筆記錄，比如說：

```ruby
Client.select(:name)
# =>  可能會回傳兩筆相同名字的紀錄
```

```ruby
Client.select(:name).distinct
# => 每個不同的名字都只回傳一筆記錄，就算有兩個同樣的名字也只回傳一筆
```

這會產生如下的 SQL 語句：


```sql
SELECT DISTINCT name FROM clients
```

也可以移除唯一值約束 (uniqueness constraint)：

```ruby
query = Client.select(:name).distinct
# => 回傳所有不同的名字

query.distinct(false)
# => 回傳所有的名字，就算有相同的名字
```

5 Limit 與 Offset
----------------

想要對`Model.find`產生的 SQL 使用`LIMIT`的話，可以使用`limit`與`offset`。

使用`limit`可以宣告要取出幾筆記錄，`offset`則是宣告在取出記錄前要跳過幾筆記錄。例如：


```ruby
Client.limit(5)
```

這會回傳最多 5 筆客戶記錄。由於沒有宣告`offset`，所以會回傳資料表的最前面五筆記錄。產生的 SQL 語句如下：


```sql
SELECT * FROM clients LIMIT 5
```

若加入`offset`的話：

```ruby
Client.limit(5).offset(30)
```

則會從第 31 筆記錄開始回傳，SQL 語句如下：


```sql
SELECT * FROM clients LIMIT 5 OFFSET 30
```

6 Group
--------

要在查詢中使用 SQL 的`GROUP BY`，可以用`group`方法。 

例如，想找出下訂單的日期：
For example, if you want to find a collection of the dates on which orders were created:

```ruby
Order.select("date(created_at) as ordered_date, sum(price) as total_price").group("date(created_at)")
```

每個有訂單的日期都會回傳一個`Order`物件。

產生的 SQL 語句如下：

```sql
SELECT date(created_at) as ordered_date, sum(price) as total_price
FROM orders
GROUP BY date(created_at)
```

### 6.1 Group 裡項目的總數

若想知道使用`group`查詢結果的總數，可以在`group`後呼叫`count`。

```ruby
Order.group(:status).count
# => { 'awaiting_approval' => 7, 'paid' => 12 }
```

會產生如下的 SQL：


```sql
SELECT COUNT (*) AS count_all, status AS status
FROM "orders"
GROUP BY status
```

7 Having
---------


在 SQL 裡，可以使用`HAVING`子句來對`GROUP BY`欄位下條件。您可以在`Model.find`加入`having`方法來使用 SQL 的`HAVING`。例如：


```ruby
Order.select("date(created_at) as ordered_date, sum(price) as total_price").
  group("date(created_at)").having("sum(price) > ?", 100)
```

會執行如下的 SQL：


```sql
SELECT date(created_at) as ordered_date, sum(price) as total_price
FROM orders
GROUP BY date(created_at)
HAVING sum(price) > 100
```

This returns the date and total price for each order object, grouped by the day they were ordered and where the price is more than $100.

8 覆蓋條件
---------------------

### 8.1`unscope`

使用`unscope`可以移除特定的條件，例如：


```ruby
Article.where('id > 10').limit(20).order('id asc').unscope(:order)
```

會執行如下的 SQL：

```sql
SELECT * FROM articles WHERE id > 10 LIMIT 20

# Original query without `unscope`
SELECT * FROM articles WHERE id > 10 ORDER BY id asc LIMIT 20

```

也可以`unscope`特定的`where`子句，例如：

```ruby
Article.where(id: 10, trashed: false).unscope(where: :id)
# SELECT "articles".* FROM "articles" WHERE trashed = 0
```

使用了`unscope`的 relation 會影響所有與其合併的 relation：
A relation which has used `unscope` will affect any relation into which it is merged:

```ruby
Article.order('id asc').merge(Article.unscope(:order))
# SELECT "articles".* FROM "articles"
```

### 8.2`only`

使用`only`可以留下特定的條件，例如：

```ruby
Article.where('id > 10').limit(20).order('id desc').only(:order, :where)
```

會執行如下的 SQL：

```sql
SELECT * FROM articles WHERE id > 10 ORDER BY id DESC

# Original query without `only`
SELECT "articles".* FROM "articles" WHERE (id > 10) ORDER BY id desc LIMIT 20

```

### 8.3`reorder`


使用`reorder`可以覆蓋預設的排序方式，例如：

```ruby
class Article < ApplicationRecord
  has_many :comments, -> { order('posted_at DESC') }
end

Article.find(10).comments.reorder('name')
```

會執行如下的 SQL：

```sql
SELECT * FROM articles WHERE id = 10
SELECT * FROM comments WHERE article_id = 10 ORDER BY name
```

如果沒有使用`reorder`的話，會執行如下的 SQL：

```sql
SELECT * FROM articles WHERE id = 10
SELECT * FROM comments WHERE article_id = 10 ORDER BY posted_at DESC
```

### 8.4`reverse_order`

使用`reverse_order`可以反轉已宣告的排序方式。

```ruby
Client.where("orders_count > 10").order(:name).reverse_order
```

會執行如下的 SQL：

```sql
SELECT * FROM clients WHERE orders_count > 10 ORDER BY name DESC
```

如果查詢裡沒有宣告排序方式，則會依照主鍵的相反順序來排序：

```ruby
Client.where("orders_count > 10").reverse_order
```

會執行如下的 SQL：

```sql
SELECT * FROM clients WHERE orders_count > 10 ORDER BY clients.id DESC
```

`reverse_order`**不接受參數**。


### 8.5`rewhere`

使用`rewhere`可以覆蓋既有的`where`條件，例如：


```ruby
Article.where(trashed: true).rewhere(trashed: false)
```

會執行如下的 SQL：

```sql
SELECT * FROM articles WHERE `trashed` = 0
```

若沒有使用`rewhere`：

```ruby
Article.where(trashed: true).where(trashed: false)
```

則會執行如下的 SQL：

```sql
SELECT * FROM articles WHERE `trashed` = 1 AND `trashed` = 0
```

9 空 Relation
-------------

使用`none`方法會回傳一個空的、可連鎖使用的 relation。接在回傳的 relation 後的所有條件都會回傳空的 relation。當回傳的 relation 可能是空的，又需要可以連鎖使用時，就可以用這個方法。


```ruby
Article.none # returns an empty Relation and fires no queries.
```

```ruby
# The visible_articles method below is expected to return a Relation.
@articles = current_user.visible_articles.where(name: params[:name])

def visible_articles
  case role
  when 'Country Manager'
    Article.where(country: country)
  when 'Reviewer'
    Article.published
  when 'Bad User'
    Article.none # => returning [] or nil breaks the caller code in this case
  end
end
```

10 唯讀物件
----------------

Active Record 提供了`readonly`方法來禁止修改回傳的物件。只要試圖更改唯獨的紀錄，就會拋出`ActiveRecord::ReadOnlyRecord`異常。

```ruby
client = Client.readonly.first
client.visits += 1
client.save
```

由於`client`已經設成唯獨物件，所以上例程式碼進行到`client.save`的時候就會拋出`ActiveRecord::ReadOnlyRecord`異常，因為 _visits_ 的植被改變了。


11 更新時鎖定記錄 
--------------------------

鎖定能避免更新記錄時的 race condition (競態條件) 並確保更新是原子性的 (atomic)。

Active Record 提供了兩種鎖定機制：

* 樂觀鎖定 (Optimistic Locking)
* 悲觀鎖定 (Pessimistic Locking)

### 11.1 樂觀鎖定

透過檢查記錄從資料庫取出後，是否有其他程序修改此記錄，樂觀鎖定可讓多個使用者編輯相同的紀錄，並假設發生資料衝突的可能性最小。若有人同時修改此記錄，會拋出`ActiveRecord::StaleObjectError`異常，並忽略更新。


**樂觀鎖定欄位**

要使用樂觀鎖定，需要在資料表加一欄叫做`lock_version`的整數欄位。每次更新記錄，Active Record 就會增加`lock_version`欄位。如果要更新一筆記錄，它的`lock_version`值比現在資料庫中的`lock_version`值還要小的話，會無法更新且拋出`ActiveRecord::StaleObjectError`異常。例如：


```ruby
c1 = Client.find(1)
c2 = Client.find(1)

c1.first_name = "Michael"
c1.save

c2.name = "should fail"
c2.save # Raises an ActiveRecord::StaleObjectError
```

接著您就要負責處理衝突，解決異常。看是要回滾、合併、或是運用商務邏輯來解決衝突。

這個行為可以透過設定`ActiveRecord::Base.lock_optimistically = false`來關掉。

`lock_version`欄位名稱可以透過`ActiveRecord::Base`提供的類別屬性`locking_column`來覆蓋：

```ruby
class Client < ApplicationRecord
  self.locking_column = :lock_client_column
end
```

### 11.2 悲觀鎖定

悲觀鎖定使用資料庫提供的鎖定機制。在建立 relation 的時候使用`lock`，可以將選取的列做互斥鎖定。使用`lock`的 relation 通常會包在 transaction 中，以避免死鎖的情況發生。

例如：

```ruby
Item.transaction do
  i = Item.lock.first
  i.name = 'Jones'
  i.save!
end
```

上述在 MySQL 會產生如下的 SQL：

```sql
SQL (0.2ms)   BEGIN
Item Load (0.3ms)   SELECT * FROM `items` LIMIT 1 FOR UPDATE
Item Update (0.4ms)   UPDATE `items` SET `updated_at` = '2009-02-07 18:05:56', `name` = 'Jones' WHERE `id` = 1
SQL (0.8ms)   COMMIT
```

您也可以在`lock`方法傳入純 SQL 來使用不同種類的鎖定。例如，MySQL 的`LOCK IN SHARE MODE`可以鎖定一筆記錄但同時允許其他查詢來讀取。直接傳入 lock 就可以了：


```ruby
Item.transaction do
  i = Item.lock("LOCK IN SHARE MODE").find(1)
  i.increment!(:views)
end
```

若已經有 model 的實體，可以用以下的寫法來將操作包在 transaction 裡並同時鎖定：

```ruby
item = Item.first
item.with_lock do
  # This block is called within a transaction,
  # item is already locked.
  item.increment!(:views)
end
```

12 連接資料表
--------------

Active Record 提供了`joins`與`left_outer_joins`兩種方法，來對 SQL 指定`JOIN`子句。`joins`要用在`INNER JOIN`或自訂的查詢，`left_outer_joins`則是用在有使用`LEFT OUTER JOIN`的查詢。


### 12.1`joins`

`joins`方法有很多種使用方式。

#### 12.1.1 使用字串形式的 SQL 片段 

可以直接在`joins`中傳入純 SQL 來指定`JOIN`：

```ruby
Author.joins("INNER JOIN posts ON posts.author_id = author.id AND posts.published = 't'")
```

這會產生如下的 SQL：

```sql
SELECT clients.* FROM clients INNER JOIN posts ON posts.author_id = author.id AND posts.published = 't'
```

#### 12.1.2 使用關聯名稱的陣列或 Hash 形式

Active Record 讓您使用 model 中定義的[關聯](association_basics.html)名稱，作為使用`joins`時替關聯指定`JOIN`子句的捷徑。

例如，以下有`Category`、`Article`、`Comment`、`Guest`以及`Tag`models：

```ruby
class Category < ApplicationRecord
  has_many :articles
end

class Article < ApplicationRecord
  belongs_to :category
  has_many :comments
  has_many :tags
end

class Comment < ApplicationRecord
  belongs_to :article
  has_one :guest
end

class Guest < ApplicationRecord
  belongs_to :comment
end

class Tag < ApplicationRecord
  belongs_to :article
end
```

接下來的方法都會使用`INNER JOIN`來產生連接查詢 (join query)：

##### 12.1.2.1 連接單個關聯

```ruby
Category.joins(:articles)
```

這會產生：

```sql
SELECT categories.* FROM categories
  INNER JOIN articles ON articles.category_id = categories.id
```

用白話來說就是：「依文章分類來回傳分類物件」。注意，若不同的文章有相同的類別，則會看到重複的分類物件。若想要去掉重複結果，可以使用`Category.joins(:articles).distinct`。


#### 12.1.3 連接多個關聯

```ruby
Article.joins(:category, :comments)
```

這會產生：

```sql
SELECT articles.* FROM articles
  INNER JOIN categories ON articles.category_id = categories.id
  INNER JOIN comments ON comments.article_id = articles.id
```

也就是：「依分類來回傳文章物件，且至少有一則評論」。再次注意，若文章有數則評論則會重複出現。


##### 12.1.3.1 連接單層嵌套關聯 (nested association) 

```ruby
Article.joins(comments: :guest)
```

這會產生：

```sql
SELECT articles.* FROM articles
  INNER JOIN comments ON comments.article_id = articles.id
  INNER JOIN guests ON guests.comment_id = comments.id
```

也就是：「回傳所有有訪客評論的文章」。


##### 12.1.3.2 連接多層嵌套關聯

```ruby
Category.joins(articles: [{ comments: :guest }, :tags])
```

這會產生：

```sql
SELECT categories.* FROM categories
  INNER JOIN articles ON articles.category_id = categories.id
  INNER JOIN comments ON comments.article_id = articles.id
  INNER JOIN guests ON guests.comment_id = comments.id
  INNER JOIN tags ON tags.article_id = articles.id
```

也就是：「回傳所有具備下列條件的分類：有文章，且文章有訪客評論和標籤。」


#### 12.1.4 對連接的資料表指定條件


您可以使用一般的[陣列](#array-conditions)與[字串](#pure-string-conditions) 條件來對連接的資料表指定條件。[Hash 條件](#hash-conditions)則提供了特殊的語法來指定： 

```ruby
time_range = (Time.now.midnight - 1.day)..Time.now.midnight
Client.joins(:orders).where('orders.created_at' => time_range)
```

使用嵌套的 hash 條件可以寫得更簡潔：


```ruby
time_range = (Time.now.midnight - 1.day)..Time.now.midnight
Client.joins(:orders).where(orders: { created_at: time_range })
```

這會用 SQL 的`BETWEEN`來找到所有昨天下訂單的客戶。


### 12.2`left_outer_joins`

若想要選取一組不論有無關聯的記錄，可以用`left_outer_joins`方法。

```ruby
Author.left_outer_joins(:posts).distinct.select('authors.*, COUNT(posts.*) AS posts_count').group('authors.id')
```

這會產生：

```sql
SELECT DISTINCT authors.*, COUNT(posts.*) AS posts_count FROM "authors"
LEFT OUTER JOIN posts ON posts.author_id = authors.id GROUP BY authors.id
```

也就是：「連帶發文數量回傳所有的作者，不論他們到底有沒有發文。」


13 Eager Loading 關聯
--------------------------

Eager Loading 是一種機制，能用最少的查詢次數來載入`Model.find`回傳物件的關聯記錄。


**N + 1 查詢問題**

下列的程式碼會找到 10 個客戶並印出他們的郵遞區號：


```ruby
clients = Client.limit(10)

clients.each do |client|
  puts client.address.postcode
end
```

乍看之下可能沒什麼問題，然而，問題出在查詢的執行次數。上面的程式碼會執行 1 次查詢 (找到 10 個客戶) + 10 次查詢 (從每個用戶載入地址) = 總共 **11** 次查詢。


**N + 1 查詢問題解決辦法**

Active Record 可以讓您在事前指定所有要載入的關聯，只要在呼叫`Model.find`時宣告`includes`方法就可以了。有了`includes`，Active Record 會確保用最少的查詢次數來載入所有指定的關聯。

回到剛剛的例子，我們可以把`Client.limit(10)`用 eager load 改寫：

```ruby
clients = Client.includes(:address).limit(10)

clients.each do |client|
  puts client.address.postcode
end
```

上述程式碼只會執行 **2** 次查詢，而不像之前會執行 **11** 次：


```sql
SELECT * FROM clients LIMIT 10
SELECT addresses.* FROM addresses
  WHERE (addresses.client_id IN (1,2,3,4,5,6,7,8,9,10))
```

### 13.1 Eager Loading 多個關聯

Active Record 讓您在`includes`方法中使用陣列、hash、或嵌套 hash (內有陣列、hash) ，只需呼叫一次`Model.find`就可以載入任意數量的關聯。

#### 13.1.1 有多個關聯的陣列

```ruby
Article.includes(:category, :comments)
```

這會載入所有的文章，以及每篇文章的類別與評論。


#### 13.1.2 嵌套關聯 Hash

```ruby
Category.includes(articles: [{ comments: :guest }, :tags]).find(1)
```

這會找到 id 為 1 的類別，並載入所有關聯的文章、文章的標籤與評論、以及每則評論的關聯訪客。


### 13.2 對 Eager Loaded 關聯下條件

雖然 Active Record 允許您像`joins`那樣來對 eager loaded 關聯下條件，但仍建議使用[連接資料表](#joining-tables)。

若堅持要這麼做，可以像平常那樣使用`where`：


```ruby
Article.includes(:comments).where(comments: { visible: true })
```

這會產生有`LEFT OUTER JOIN`語句的查詢，`joins`方法則是產生`INNER JOIN`。


```ruby
  SELECT "articles"."id" AS t0_r0, ... "comments"."updated_at" AS t1_r5 FROM "articles" LEFT OUTER JOIN "comments" ON "comments"."article_id" = "articles"."id" WHERE (comments.visible = 1)
```

若沒有`where`條件，則會像平常一樣產生兩組查詢。


NOTE: 像這樣使用`where`只有在傳入 Hash 時才有用。若要傳入 SQL 片段，則需要用`refrences`來強制連接資料表：

```ruby
Article.includes(:comments).where("comments.visible = true").references(:comments)
```

上例的`includes`查詢，就算文章都沒有評論，仍會載入所有文章。若是使用`joins`(INNER JOIN)，則**必須**要符合連接條件，否則不會回傳任何記錄。




14 作用域
--------

作用域 (scope) 可以讓您將常用的查詢定義成關聯物件或 model 的方法。有了作用域，就可以使用之前提過的每一種方法，像是`where`、`joins`與`includes`。所有的作用域方法都會回傳一個`ActiveRecord::Relation`物件，來允許呼叫更多的方法 (像是其他作用域)。

現在就來定義一個簡單的作用域。在類別中使用`scope`方法，並傳入呼叫此作用域時想要執行的查詢：


```ruby
class Article < ApplicationRecord
  scope :published, -> { where(published: true) }
end
```

這與定義一個類別方法完全相同，至於要用哪個就看個人喜好了：


```ruby
class Article < ApplicationRecord
  def self.published
    where(published: true)
  end
end
```

作用域也可以與其他作用域連鎖使用：

```ruby
class Article < ApplicationRecord
  scope :published,               -> { where(published: true) }
  scope :published_and_commented, -> { published.where("comments_count > 0") }
end
```

要呼叫`published`作用域，可以在類別上呼叫：

```ruby
Article.published # => [published articles]
```

也可以對由`Article`物件組成的關聯使用：

```ruby
category = Category.first
category.articles.published # => [published articles belonging to this category]
```

### 14.1 傳入參數 

作用域可以傳入參數：

```ruby
class Article < ApplicationRecord
  scope :created_before, ->(time) { where("created_at < ?", time) }
end
```

可以像呼叫類別方法那樣呼叫作用域：

```ruby
Article.created_before(Time.zone.now)
```

然而，這只是重複做了類別方法可以做到的事。


```ruby
class Article < ApplicationRecord
  def self.created_before(time)
    where("created_at < ?", time)
  end
end
```

要讓作用域接受參數的話，建議使用類別方法。這些方法仍可以在關聯物件上使用：


```ruby
category.articles.created_before(time)
```

### 14.2 使用條件式

作用域可以使用條件式：

```ruby
class Article < ApplicationRecord
  scope :created_before, ->(time) { where("created_at < ?", time) if time.present? }
end
```

如同其他的例子，這跟類別方法很類似。

```ruby
class Article < ApplicationRecord
  def self.created_before(time)
    where("created_at < ?", time) if time.present?
  end
end
```

然而，有很重要的一點要注意：作用域總是會回傳一個`ActiveRecord::Relation`物件，就算條件式判斷為`false`也一樣，而類別方法則會回傳`nil`。這在使用條件式串接類別方法時，若條件式回傳`false`，會造成`NoMethodError`。


### 14.3 使用預設的作用域 Applying a default scope

若想要對 model 中所有的查詢使用同一個作用域，可以在 model 中使用`default_scope`方法。
If we wish for a scope to be applied across all queries to the model we can use the
`default_scope` method within the model itself.

```ruby
class Client < ApplicationRecord
  default_scope { where("removed_at IS NULL") }
end
```

當在這個 model 上執行查詢，SQL 語句看起來會是：
When queries are executed on this model, the SQL query will now look something like
this:

```sql
SELECT * FROM clients WHERE removed_at IS NULL
```

若想要預設的作用域做更多複雜的事，可以將它定義為類別方法：


```ruby
class Client < ApplicationRecord
  def self.default_scope
    # Should return an ActiveRecord::Relation.
  end
end
```

NOTE: `default_scope`在建立記錄時也會執行，但更新時不會，像是： 

```ruby
class Client < ApplicationRecord
  default_scope { where(active: true) }
end

Client.new          # => #<Client id: nil, active: true>
Client.unscoped.new # => #<Client id: nil, active: nil>
```

### 14.4 合併作用域 

就像`where`一樣，作用域可以用 SQL 的`AND`來合併。

```ruby
class User < ApplicationRecord
  scope :active, -> { where state: 'active' }
  scope :inactive, -> { where state: 'inactive' }
end

User.active.inactive
# SELECT "users".* FROM "users" WHERE "users"."state" = 'active' AND "users"."state" = 'inactive'
```

`scope`作用域與`where`條件可以混合使用，最終的 SQL 會用`AND`來把所有條件接起來。

```ruby
User.active.where(state: 'finished')
# SELECT "users".* FROM "users" WHERE "users"."state" = 'active' AND "users"."state" = 'finished'
```
如果要讓最後一個`where`條件覆蓋先前的，可以使用`Relation#merge`。

```ruby
User.active.merge(User.inactive)
# SELECT "users".* FROM "users" WHERE "users"."state" = 'inactive'
```

請注意，`default_scope`會被`scope`作用域和`where`條件覆蓋掉。

```ruby
class User < ApplicationRecord
  default_scope { where state: 'pending' }
  scope :active, -> { where state: 'active' }
  scope :inactive, -> { where state: 'inactive' }
end

User.all
# SELECT "users".* FROM "users" WHERE "users"."state" = 'pending'

User.active
# SELECT "users".* FROM "users" WHERE "users"."state" = 'pending' AND "users"."state" = 'active'

User.where(state: 'inactive')
# SELECT "users".* FROM "users" WHERE "users"."state" = 'pending' AND "users"."state" = 'inactive'
```

從上面可以看到，`default_scope`被`scope`與`where`覆蓋掉了。


### 14.5 移除所有作用域

若想要移除作用域，可以用`unscoped`方法。這在特定的查詢不需要用到`default_scope`的時候很有用。


```ruby
Client.unscoped.load
```

`unscoped`方法會移除所有作用域，接著對資料表做普通的查詢。


```ruby
Client.unscoped.all
# SELECT "clients".* FROM "clients"

Client.where(published: false).unscoped.all
# SELECT "clients".* FROM "clients"
```

`unscoped`也接受區塊。

```ruby
Client.unscoped {
  Client.created_before(Time.zone.now)
}
```

15 動態查詢方法
---------------

Active Record 對每個資料表裡定義的欄位（又稱屬性），都提供了一個 finder 方法。假設您的`Client`model 中有個欄位叫做`first_name`，Active Record 就會提供一個`find_by_first_name`方法。若`Client`model 有`locked`欄位，就有`find_by_locked`可以用。

若在動態查詢方法名稱的最後加上驚嘆號（`!`），當它們沒有回傳任何記錄時，就會拋出`ActiveRecord::RecordNotFound`異常，像是`Client.find_by_name!("Ryan")`。

若想同時查找 name 與 locked 欄位，可以在方法中間使用`and`來連接。例如，`Client.find_by_first_name_and_locked("Ryan", true)`。

16 Enums
---------

`enum`巨集會將一欄整數 map 成一組值。
The `enum` macro maps an integer column to a set of possible values.

```ruby
class Book < ApplicationRecord
  enum availability: [:available, :unavailable]
end
```

這會自動產生對應的[作用域](#scopes)來查詢 model。也加入了在狀態和查詢目前的狀態間轉換的方法。

```ruby
# Both examples below query just available books.
Book.available
# or
Book.where(availability: :available)

book = Book.new(availability: :available)
book.available?   # => true
book.unavailable! # => true
book.available?   # => false
```

在[Rails API 文件](http://api.rubyonrails.org/classes/ActiveRecord/Enum.html)有 enums 完整的說明。

17 了解方法鏈
--------------

Active Record 提供了[方法鏈](http://en.wikipedia.org/wiki/Method_chaining)的形式，可以輕易的同時使用多個 Active Record 方法。

要能夠連接方法，前一個呼叫的方法必須回傳一個`ActiveRecord::Relation`，像是
`all`、`where`、與`joins`。只回傳單一物件的方法（請參閱[取出單一物件](#retrieving-a-single-object)）必須放在最後面。


接著會舉一些例子。這篇指南不會列出所有可能的組合，只會舉一些來當範例。當呼叫一個 Active Record 方法時，除非需要資料，不然不會馬上產生查詢。所以下列的例子都會產生一個單一查詢。


### 17.1 從數個資料表取出篩選過的資料 Retrieving filtered data from multiple tables

```ruby
Person
  .select('people.id, people.name, comments.text')
  .joins(:comments)
  .where('comments.created_at > ?', 1.week.ago)
```

這會產生如下的結果：

```sql
SELECT people.id, people.name, comments.text
FROM people
INNER JOIN comments
  ON comments.person_id = people.id
WHERE comments.created_at = '2015-01-01'
```

### 17.2 從數個資料表取出特定的資料 
```ruby
Person
  .select('people.id, people.name, companies.name')
  .joins(:company)
  .find_by('people.name' => 'John') # this should be the last
```

這會產生：

```sql
SELECT people.id, people.name, companies.name
FROM people
INNER JOIN companies
  ON companies.person_id = people.id
WHERE people.name = 'John'
LIMIT 1
```

NOTE: 請注意，若有數比資料符合查詢，`find_by`只會取出第一筆並忽略其他筆資料（請看上面的`LIMIT 1`敘述）。

18 查找或新建物件
-----------------

想要查找一個物件，並且要在找不到時新建一個物件，是很常見的情況，這時可以用`find_or_create_by`與`find_or_create_by!`方法。


### 18.1 `find_or_create_by`

`find_or_create_by`方法會檢查有指定屬性的紀錄是否存在。若不存在，就會呼叫`create`。來看看下面的例子吧。

假設現在要找一位名稱為 Andy 的客戶，且找不到的話，就要建立一個。這時可以這樣寫：


```ruby
Client.find_or_create_by(first_name: 'Andy')
# => #<Client id: 1, first_name: "Andy", orders_count: 0, locked: true, created_at: "2011-08-30 06:09:27", updated_at: "2011-08-30 06:09:27">
```

這個方法會產生如下的 SQL：

```sql
SELECT * FROM clients WHERE (clients.first_name = 'Andy') LIMIT 1
BEGIN
INSERT INTO clients (created_at, first_name, locked, orders_count, updated_at) VALUES ('2011-08-30 05:22:57', 'Andy', 1, NULL, '2011-08-30 05:22:57')
COMMIT
```

`find_or_create_by`會回傳已經存在的紀錄或是新建的紀錄。在剛剛的例子裡，由於名稱為 Andy 的客戶不存在，所以會新建該記錄並回傳。

新建的紀錄是否會被存到資料庫裡，取決於是否通過驗證（就跟`create`一樣）。

假設現在想在新建記錄時將`locked`屬性設為`false`，但又不想包含在查詢裡。也就是說，想要找一位名為 Andy 的客戶，如果他不存在，就新建一位未鎖定的 Andy 客戶。

有兩種方法可以達成上述的目的。第一種是`create_with`：

```ruby
Client.create_with(locked: false).find_or_create_by(first_name: 'Andy')
```

第二種是使用區塊：

```ruby
Client.find_or_create_by(first_name: 'Andy') do |c|
  c.locked = false
end
```

上面的區塊只有在新建客戶的時候才會執行。若客戶已經存在，區塊會被忽略。


### 18.2 `find_or_create_by!`

您也可以使用`find_or_create_by!`，這會在新建的紀錄沒有通過驗證時拋出異常。這篇的內容並沒有涵蓋到驗證，但先假設您加了這一行到`Client`model：

```ruby
validates :orders_count, presence: true
```

這時，若在建立新客戶時沒有傳入`orders_count`，就會拋出`ActiveRecord::RecordInvalid`異常：


```ruby
Client.find_or_create_by!(first_name: 'Andy')
# => ActiveRecord::RecordInvalid: Validation failed: Orders count can't be blank
```

### 18.3 `find_or_initialize_by`


`find_or_initialize_by`方法的運作跟`find_or_create_by`相同，但它在找不到記錄時會呼叫`new`而不是`create`。也就是說，新建的 model 物件會放在記憶體裡，而不是存進資料庫。沿用`find_or_create_by`的例子，現在要找一位名為 Nick 的客戶：


```ruby
nick = Client.find_or_initialize_by(first_name: 'Nick')
# => #<Client id: nil, first_name: "Nick", orders_count: 0, locked: true, created_at: "2011-08-30 06:09:27", updated_at: "2011-08-30 06:09:27">

nick.persisted?
# => false

nick.new_record?
# => true
```

由於這個物件還沒存進資料庫，所以產生的 SQL 會像這樣：


```sql
SELECT * FROM clients WHERE (clients.first_name = 'Nick') LIMIT 1
```

若要存進資料庫，可以呼叫`save`：

```ruby
nick.save
# => true
```

19 用 SQL 查詢
--------------

若想要用 SQL 來查詢，可以使用`find_by_sql`。這個方法會將查詢到的物件放在陣列裡回傳，就算只找到一筆記錄。舉例來說：


```ruby
Client.find_by_sql("SELECT * FROM clients
  INNER JOIN orders ON clients.id = orders.client_id
  ORDER BY clients.created_at desc")
# =>  [
#   #<Client id: 1, first_name: "Lucas" >,
#   #<Client id: 2, first_name: "Jan" >,
#   ...
# ]
```

`find_by_sql`能讓您輕易的自訂查詢方式，並取出實體化的物件。


### 19.1 `select_all`

`find_by_sql`有一個很類似的方法：`connection#select_all`。`select_all`
會用自訂的 SQL 從資料庫取出物件，但不會實體化；它會回傳一個 hash 陣列，每個 hash 都表示一筆記錄。

```ruby
Client.connection.select_all("SELECT first_name, created_at FROM clients WHERE id = '1'")
# => [
#   {"first_name"=>"Rafael", "created_at"=>"2012-11-10 23:23:45.281189"},
#   {"first_name"=>"Eileen", "created_at"=>"2013-12-09 11:22:35.221282"}
# ]
```

### 19.2 `pluck`

`pluck`可以用來查詢資料表的一個或多個欄位。`pluck`接受欄位名稱作為參數，並回傳由指定欄位的值所組成的陣列。

```ruby
Client.where(active: true).pluck(:id)
# SELECT id FROM clients WHERE active = 1
# => [1, 2, 3]

Client.distinct.pluck(:role)
# SELECT DISTINCT role FROM clients
# => ['admin', 'member', 'guest']

Client.pluck(:id, :name)
# SELECT clients.id, clients.name FROM clients
# => [[1, 'David'], [2, 'Jeremy'], [3, 'Jose']]
```

下面這段程式碼：

```ruby
Client.select(:id).map { |c| c.id }
# or
Client.select(:id).map(&:id)
# or
Client.select(:id, :name).map { |c| [c.id, c.name] }
```

可以用`pluck`取代：

```ruby
Client.pluck(:id)
# or
Client.pluck(:id, :name)
```

與`select`不同，`pluck`會直接將資料庫查詢的結果轉成 Ruby 的陣列，而不會建出`ActiveRecord`物件。這樣可以提昇龐大的、或是需要經常使用的查詢的執行效能。然而，所有 model 可用的方法就不能用了，例如：


```ruby
class Client < ApplicationRecord
  def name
    "I am #{super}"
  end
end

Client.select(:name).map &:name
# => ["I am David", "I am Jeremy", "I am Jose"]

Client.pluck(:name)
# => ["David", "Jeremy", "Jose"]
```

此外，不像`select`與其他`Relation`作用域，`pluck`會馬上啟動查詢，所以只能與已經建立的作用域連鎖使用，而無法與之後的作用域連鎖使用。

```ruby
Client.pluck(:name).limit(1)
# => NoMethodError: undefined method `limit' for #<Array:0x007ff34d3ad6d8>

Client.limit(1).pluck(:name)
# => ["David"]
```

### 19.3 `ids`

`ids`可以藉由資料表的主鍵來取得 relation 所有的 ID。


```ruby
Person.ids
# SELECT id FROM people
```

```ruby
class Person < ApplicationRecord
  self.primary_key = "person_id"
end

Person.ids
# SELECT person_id FROM people
```

20 物件存在性 
---------------

若只是想單純的檢查物件是否存在，可以使用`exists?`。這個方法會使用跟`find`一樣的查詢，但會回傳`true`或`false`，而不是回傳物件。


```ruby
Client.exists?(1)
```

`exists?`方法可以接受多個值，但要注意的是，只要有任何一筆記錄存在，就會回傳`true`。


```ruby
Client.exists?(id: [1,2,3])
# or
Client.exists?(name: ['John', 'Sergei'])
```

`exists?`就算不傳入任何參數也可以使用。


```ruby
Client.where(first_name: 'Ryan').exists?
```

上面的例子中，若有至少一位客戶的`first_name`是 Ryan，就會回傳`true`，否則回傳`false`。


```ruby
Client.exists?
```

若`Client`資料表是空的，這會回傳`false`，若不是則回傳`true`。


也可以在 model 或 relation 上用`any?`與`many?`來檢查記錄是否存在。


```ruby
# 透過 model
Article.any?
Article.many?

# 透過作用域名稱
Article.recent.any?
Article.recent.many?

# 透過 relation
Article.where(published: true).any?
Article.where(published: true).many?

# 透過關聯
Article.first.categories.any?
Article.first.categories.many?
```

21 計算
---------

這節會先用 count 當例子，但 count 適用的選項在下面所有子章節也適用。

所有計算方法都可以直接在 model 上呼叫：

```ruby
Client.count
# SELECT count(*) AS count_all FROM clients
```

或在 relation 呼叫：

```ruby
Client.where(first_name: 'Ryan').count
# SELECT count(*) AS count_all FROM clients WHERE (first_name = 'Ryan')
```

也可以對 relation 使用各種查詢方法來執行複雜的計算：


```ruby
Client.includes("orders").where(first_name: 'Ryan', orders: { status: 'received' }).count
```

這會執行下面的 SQL：

```sql
SELECT count(DISTINCT clients.id) AS count_all FROM clients
  LEFT OUTER JOIN orders ON orders.client_id = client.id WHERE
  (clients.first_name = 'Ryan' AND orders.status = 'received')
```

### 21.1 Count（計數）

想知道 model 的資料表裡有多少筆記錄，可以呼叫`Client.count`，會回傳記錄的數量。若要針對特定欄位來查詢，例如查詢所有有年齡的客戶數量，可以用`Client.count(:age)`。

可用的選項請參考[計算](#calculations)一節。

### 22.2 Average（平均）

若要找出資料表特定欄位的平均值，可以用`average`：


```ruby
Client.average("orders_count")
```

這會回傳該欄位的平均值（可能是浮點數，例如3.14159265）。

可用的選項請參考[計算](#calculations)一節。


### 22.3 Minimum（最小值）

若要找出資料表特定欄位的最小值，可以用`minimum`：


```ruby
Client.minimum("age")
```

可用的選項請參考[計算](#calculations)一節。


### 22.4 Maximum（最大值）

若要找出資料表特定欄位的最大值，可以用`maximum`：


```ruby
Client.maximum("age")
```

可用的選項請參考[計算](#calculations)一節。


### 22.5 Sum（總和）

若要找出資料表特定欄位所有記錄的總和，可以用`sum`：

```ruby
Client.sum("orders_count")
```

可用的選項請參考[計算](#calculations)一節。


23 執行 EXPLAIN 
---------------

您可以在由 relation 觸發的查詢上執行 EXPLAIN。例如：


```ruby
User.where(id: 1).joins(:articles).explain
```

在 MySQL 與 MariaDB 下會輸出：

```
EXPLAIN for: SELECT `users`.* FROM `users` INNER JOIN `articles` ON `articles`.`user_id` = `users`.`id` WHERE `users`.`id` = 1
+----+-------------+----------+-------+---------------+
| id | select_type | table    | type  | possible_keys |
+----+-------------+----------+-------+---------------+
|  1 | SIMPLE      | users    | const | PRIMARY       |
|  1 | SIMPLE      | articles | ALL   | NULL          |
+----+-------------+----------+-------+---------------+
+---------+---------+-------+------+-------------+
| key     | key_len | ref   | rows | Extra       |
+---------+---------+-------+------+-------------+
| PRIMARY | 4       | const |    1 |             |
| NULL    | NULL    | NULL  |    1 | Using where |
+---------+---------+-------+------+-------------+

2 rows in set (0.00 sec)
```


Active Record 會根據所使用的資料庫的 shell 來印出。所以，相同的查詢在 PostegreSQL 會輸出：


```
EXPLAIN for: SELECT "users".* FROM "users" INNER JOIN "articles" ON "articles"."user_id" = "users"."id" WHERE "users"."id" = 1
                                  QUERY PLAN
------------------------------------------------------------------------------
 Nested Loop Left Join  (cost=0.00..37.24 rows=8 width=0)
   Join Filter: (articles.user_id = users.id)
   ->  Index Scan using users_pkey on users  (cost=0.00..8.27 rows=1 width=4)
         Index Cond: (id = 1)
   ->  Seq Scan on articles  (cost=0.00..28.88 rows=8 width=4)
         Filter: (articles.user_id = 1)
(6 rows)
```

Eager loading 可能會觸發多筆查詢，而某些查詢會需要先前查詢的結果。因此，`explain`會實際執行該查詢，並詢問要查詢哪一個，例如：


```ruby
User.where(id: 1).includes(:articles).explain
```

在 MySQL 與 MariaDB 環境下會輸出：

```
EXPLAIN for: SELECT `users`.* FROM `users`  WHERE `users`.`id` = 1
+----+-------------+-------+-------+---------------+
| id | select_type | table | type  | possible_keys |
+----+-------------+-------+-------+---------------+
|  1 | SIMPLE      | users | const | PRIMARY       |
+----+-------------+-------+-------+---------------+
+---------+---------+-------+------+-------+
| key     | key_len | ref   | rows | Extra |
+---------+---------+-------+------+-------+
| PRIMARY | 4       | const |    1 |       |
+---------+---------+-------+------+-------+

1 row in set (0.00 sec)

EXPLAIN for: SELECT `articles`.* FROM `articles`  WHERE `articles`.`user_id` IN (1)
+----+-------------+----------+------+---------------+
| id | select_type | table    | type | possible_keys |
+----+-------------+----------+------+---------------+
|  1 | SIMPLE      | articles | ALL  | NULL          |
+----+-------------+----------+------+---------------+
+------+---------+------+------+-------------+
| key  | key_len | ref  | rows | Extra       |
+------+---------+------+------+-------------+
| NULL | NULL    | NULL |    1 | Using where |
+------+---------+------+------+-------------+


1 row in set (0.00 sec)
```


### 23.1 解讀 EXPLAIN

解讀 EXPLAIN 的輸出不在此篇的範疇內，若想更深入了解請查閱以下幾篇：

* SQLite3: [EXPLAIN QUERY PLAN](http://www.sqlite.org/eqp.html)

* MySQL: [EXPLAIN Output Format](http://dev.mysql.com/doc/refman/5.7/en/explain-output.html)

* MariaDB: [EXPLAIN](https://mariadb.com/kb/en/mariadb/explain/)

* PostgreSQL: [Using EXPLAIN](http://www.postgresql.org/docs/current/static/using-explain.html)
