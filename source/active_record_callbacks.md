**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON http://guides.rubyonrails.org.**

Active Record 回呼
=======================

本篇講解了如何掛載是見到 Active Record 物件的生命週期。

讀完本篇，您將了解：

* Active Record 物件的生命週期。
* 如何建立回呼方法 (callback method) 來回應物件生命週期中的事件。
* 如何建立特殊的類別來封裝回呼常見的行為。

--------------------------------------------------------------------------------

1 物件的生命週期
---------------------

在操作 Rails 應用程式時，經常會新建、更新、或是刪除物件。Active Record 提供了掛載事件到物件生命週期的機制，讓您控制應用程式與資料。


回呼可以讓您在更改物件狀態的前後，觸發特定的程式碼。


2 回呼綜覽
------------------

回呼是一種方法，在物件生命週期中的特定時間點被呼叫。使用回呼，就可以在 Active Record 物件被新建、儲存、更新、刪除、驗證、或從資料庫中載入的時候來執行特定的程式碼。


### 2.1 註冊回呼

在使用回呼前要先註冊才行。您可以將回呼看作是一般方法來寫，然後使用巨集風格類別方法 (macro-style class method) 來註冊為回呼：


```ruby
class User < ApplicationRecord
  validates :login, :email, presence: true

  before_validation :ensure_login_has_a_value

  protected
    def ensure_login_has_a_value
      if login.nil?
        self.login = email unless email.blank?
      end
    end
end
```

巨集風格類別方法也接受區塊，若回呼程式碼只有一行那麼短，就可以使用區塊形式：


```ruby
class User < ApplicationRecord
  validates :login, :email, presence: true

  before_create do
    self.name = login.capitalize if name.blank?
  end
end
```

回呼也可以針對特定的生命週期事件來觸發：


```ruby
class User < ApplicationRecord
  before_validation :normalize_name, on: :create

  # :on takes an array as well
  after_validation :set_location, on: [ :create, :update ]

  protected
    def normalize_name
      self.name = name.downcase.titleize
    end

    def set_location
      self.location = LocationService.query(self)
    end
end
```

一般會建議將回呼方法宣告為 protected 或 private。若宣告為 public 方法，就可以從 model 外呼叫它們，這違反了物件封裝的精神。 


3 可用的回呼
-------------------

以下是所有可用的 Active Record 回呼，依照每一次執行中被呼叫的順序來排序：


### 3.1 新建物件

* `before_validation`
* `after_validation`
* `before_save`
* `around_save`
* `before_create`
* `around_create`
* `after_create`
* `after_save`
* `after_commit/after_rollback`

### 3.2 更新物件

* `before_validation`
* `after_validation`
* `before_save`
* `around_save`
* `before_update`
* `around_update`
* `after_update`
* `after_save`
* `after_commit/after_rollback`

### 3.3 刪除物件

* `before_destroy`
* `around_destroy`
* `after_destroy`
* `after_commit/after_rollback`

WARNING. `after_save`在新建與更新中都會執行，但不論回呼註冊的順序為何，一定會在`after_create`與`after_update`兩個特定的回呼後執行。 

### 3.4`after_initialize`與`after_find`

當實體化 Active Record 物件時 (不論是用`new`或是從資料庫中載入記錄)，會呼叫`after_initialize`。這比覆蓋 Active Record 的`initialize`方法還要好多了。

當 Active Record 從資料庫中載入記錄時，會呼叫`after_find`。若同時使用`after_find`與`after_initialize`，會先呼叫`after_find`。



`after_initialize`與`after_find`回呼都沒有對應的`before_*`，但它們的註冊方式跟一般回呼相同。

```ruby
class User < ApplicationRecord
  after_initialize do |user|
    puts "You have initialized an object!"
  end

  after_find do |user|
    puts "You have found an object!"
  end
end

>> User.new
You have initialized an object!
=> #<User id: nil>

>> User.first
You have found an object!
You have initialized an object!
=> #<User id: 1>
```

### 3.5 `after_touch`

當 Active Record 物件被 touch 後，就會呼叫`after_touch`。


```ruby
class User < ApplicationRecord
  after_touch do |user|
    puts "You have touched an object"
  end
end

>> u = User.create(name: 'Kuldeep')
=> #<User id: 1, name: "Kuldeep", created_at: "2013-11-25 12:17:49", updated_at: "2013-11-25 12:17:49">

>> u.touch
You have touched an object
=> true
```

可以搭配`belongs_to`使用：


```ruby
class Employee < ApplicationRecord
  belongs_to :company, touch: true
  after_touch do
    puts 'An Employee was touched'
  end
end

class Company < ApplicationRecord
  has_many :employees
  after_touch :log_when_employees_or_company_touched

  private
  def log_when_employees_or_company_touched
    puts 'Employee/Company was touched'
  end
end

>> @employee = Employee.last
=> #<Employee id: 1, company_id: 1, created_at: "2013-11-25 17:04:22", updated_at: "2013-11-25 17:05:05">

# triggers @employee.company.touch
>> @employee.touch
Employee/Company was touched
An Employee was touched
=> true
```

4 執行回呼
-----------------

以下方法會觸發回呼：

* `create`
* `create!`
* `decrement!`
* `destroy`
* `destroy!`
* `destroy_all`
* `increment!`
* `save`
* `save!`
* `save(validate: false)`
* `toggle!`
* `update_attribute`
* `update`
* `update!`
* `valid?`

另外，`after_find`回呼會由以下查詢方法來觸發：

* `all`
* `first`
* `find`
* `find_by`
* `find_by_*`
* `find_by_*!`
* `find_by_sql`
* `last`

每當初始化新的物件後，就會觸發`after_initialize`回呼。

NOTE: `find_by_*`與`find_by_*!`是 Active Record 替每個屬性自動產生的動耐查詢方法，想了解更多請參閱[動態查詢方法](active_record_querying.html#dynamic-finders)章節。

5 略過回呼
------------------

跟驗證一樣，回呼是可以略過的。以下是可以略過回呼的方法：

* `decrement`
* `decrement_counter`
* `delete`
* `delete_all`
* `increment`
* `increment_counter`
* `toggle`
* `touch`
* `update_column`
* `update_columns`
* `update_all`
* `update_counters`

然而，要使用這些方法時請小心。回呼中有重要的商業規則和應用程式邏輯，若沒有真正了解其中的含意而略過它們，可能會導致無效的資料。


6 終止執行
-----------------

當您替 model 註冊新的回呼時，它們會被加入佇列裡等待執行。這個佇列包含了所有 model 中要執行的驗證、已註冊回呼、以及資料庫操作。


整調回呼鏈 (callback chain) 會被包在一筆交易 (transaction) 中。只要有任一 _before_ 回呼方法回傳`false`或拋出異常，執行鏈就會終止並回滾； _after_ 回呼則是要拋出異常執行鏈才會終止並回滾。


WARNING. 任何非`ActiveRecord::Rollback`或`ActiveRecord::RecordInvalid`的異常會在回呼鏈終止後被 Rails 重複拋出。拋出非`ActiveRecord::Rollback`或`ActiveRecord::RecordInvalid`的異常，可能會破壞沒有預期`save`與`update_attributes`(通常會回傳`true`或`false`)這類方法的程式碼來拋出異常。

7 關聯回呼
--------------------

回呼可以在 model 間運作，甚至可以用 model 間的關聯來定義。舉個例子，假設使用者有很多文章，照理說那些文章在使用者被刪除後也要被刪除，這時可以在與`User`model 相關聯的`Article`model 加入`after_destroy`回呼：


```ruby
class User < ApplicationRecord
  has_many :articles, dependent: :destroy
end

class Article < ApplicationRecord
  after_destroy :log_destroy_action

  def log_destroy_action
    puts 'Article destroyed'
  end
end

>> user = User.first
=> #<User id: 1>
>> user.articles.create!
=> #<Article id: 1, user_id: 1>
>> user.destroy
Article destroyed
=> #<User id: 1>
```

8 條件式回呼
---------------------

就像驗證一樣，我們也可以讓回呼在滿足特定條件時才執行。條件可以透過`:if`與`:unless`來指定，可以傳入`Symbol`、`String`、`Proc`或`Array`。若要回呼在滿足條件時執行，請用`:if`；若要在不滿足條件時執行，請用`:unless`。


### 8.1 使用`Symbol`指定`:if`與`:unless`

`:if`與`:unless`可以傳入`symbol`。`symbol`會對應到執行回呼前，所要呼叫的先決條件方法的名稱。使用`:if`時，若先決條件方法回傳`false`，就不會執行回呼；若是用`:unless`，則是回傳`true`才不會執行。使用`symbol`是最常見的，這樣的方式也可以同時註冊很多先決條件來判斷是否該執行回呼。


```ruby
class Order < ApplicationRecord
  before_save :normalize_card_number, if: :paid_with_card?
end
```

### 8.2 使用字串指定`:if`與`:unless`

您也可以傳入字串 (string)。傳入的字串將會用`eval`來求值，所以必須使用有效的 Ruby 程式碼。只有在條件夠簡短時才能使用這個方式：

```ruby
class Order < ApplicationRecord
  before_save :normalize_card_number, if: "paid_with_card?"
end
```

### 8.3 使用`Proc`指定`:if`與`:unless`

最後，也可以傳入`Proc`物件。這樣的方式最適合在撰寫簡短的驗證方法 (通常只有一行) 時使用 ：


```ruby
class Order < ApplicationRecord
  before_save :normalize_card_number,
    if: Proc.new { |order| order.paid_with_card? }
end
```

### 8.4 多重條件回呼

撰寫條件式回呼時，也可以在一個回呼中同時使用`:if`與`:unless`：


```ruby
class Comment < ApplicationRecord
  after_create :send_email_to_author, if: :author_wants_emails?,
    unless: Proc.new { |comment| comment.article.ignore_comments? }
end
```

9 回呼類別
----------------

Active Record 可以將回呼方法封裝成類別，這時若要讓其他 model 來重複使用一個回呼，就變得非常容易。

來看看這個例子，我們替`PictureFile`model 建立一個有`after_destroy`回呼的類別：


```ruby
class PictureFileCallbacks
  def after_destroy(picture_file)
    if File.exist?(picture_file.filepath)
      File.delete(picture_file.filepath)
    end
  end
end
```

如上，在類別裡宣告回呼時，回呼方法會將接收的 model 物件視為一個參數。現在就可以在這個 model 中使用回呼類別了：


```ruby
class PictureFile < ApplicationRecord
  after_destroy PictureFileCallbacks.new
end
```

注意，由於我們將回呼宣告為實體方法，所以要實體化一個新的`PictureFileCallbacks`物件。這在回呼用到實體物件的情況下特別好用。然而，通常將回呼宣告成類別方法會更合理：


```ruby
class PictureFileCallbacks
  def self.after_destroy(picture_file)
    if File.exist?(picture_file.filepath)
      File.delete(picture_file.filepath)
    end
  end
end
```

若使用這樣的方式宣告回呼，就不需要實體化一個`PictureFileCallbacks`物件了。

```ruby
class PictureFile < ApplicationRecord
  after_destroy PictureFileCallbacks
end
```

在回呼類別中，可以不限數量的宣告回呼。

10 交易回呼
---------------------

當完成資料庫交易時，會觸發兩個額外的回呼：`after_commit`與`after_rollback`。它們與`after_save`回呼非常類似，只差在它們會等到資料庫的變更被提交或回滾後才執行。當 Active Record model 需要與資料庫交易外的外部系統互動時，這兩個回呼非常有用。


比如說在上例中，`PictureFile`model 需要在刪除某個記錄後，刪除其對應的檔案。若執行了`after_destroy`回呼後拋出了異常，則交易回滾。但此時檔案已經被刪除了，造成 model 處在不一致的狀態。舉例來說，以下的`picture_file_2`不是有效的檔案，`save!`方法會拋出錯誤。


```ruby
PictureFile.transaction do
  picture_file_1.destroy
  picture_file_2.save!
end
```

若使用`after_commit`回呼，就可以解決這個問題。


```ruby
class PictureFile < ApplicationRecord
  after_commit :delete_picture_file_from_disk, on: :destroy

  def delete_picture_file_from_disk
    if File.exist?(filepath)
      File.delete(filepath)
    end
  end
end
```

NOTE: `:on`選項可以指定回呼何時被觸發。若不指定，則每個 action 都會觸發。

由於只在新建、更新、或刪除時使用`after_commit`回呼時是很常見的，故提供了這些操作的別名：

* `after_create_commit`
* `after_update_commit`
* `after_destroy_commit`

```ruby
class PictureFile < ApplicationRecord
  after_destroy_commit :delete_picture_file_from_disk

  def delete_picture_file_from_disk
    if File.exist?(filepath)
      File.delete(filepath)
    end
  end
end
```

WARNING. `after_commit`與`after_rollback`回呼在交易區塊中新建、更新、或刪除 model 時是一定會執行的。這時若回呼拋出任何異常都會被忽略，來確保彼此不會互相干擾。因此，當回呼拋出異常時，記得自己救援並在回呼中做適當的處理。 
