**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON http://guides.rubyonrails.org.**

Active Record 遷移
========================

Migrations (遷移)，是 Active Record 的一種功能，可讓您與時俱進的管理資料庫綱要。只需用簡潔的 Ruby DSL 就能變更資料表，不需要寫純 SQL。

讀完本篇，您將了解：

* 可用來建立遷移的產生器。
* 由 Active Record 提供的資料庫操作方法。 
* 用來操作遷移與資料庫綱要的 bin/rails 指令。 
* 遷移與`schema.rb`的關係。

--------------------------------------------------------------------------------

1 綜覽
------------------

遷移是一種簡單、一致、方便[與時俱進管理資料庫](http://en.wikipedia.org/wiki/Schema_migration)的方法。遷移使用 Ruby DSL，不用自己寫 SQL，讓您的資料庫綱要與更改適用所有的資料庫。

您可以將每筆遷移想成是資料庫的新「版本」。從一開始的空白資料庫綱要中，遷移會逐漸增刪資料表、欄位、記錄等。Active Record 能依照時間順序更新資料庫綱要，不管從資料庫歷史的何處開始都能更新到最新版本。Active Record 也會更新 `db/schema.rb` 檔案，與最新的資料庫結構保持同步。


來看看遷移的範例：

```ruby
class CreateProducts < ActiveRecord::Migration[5.0]
  def change
    create_table :products do |t|
      t.string :name
      t.text :description

      t.timestamps
    end
  end
end
```

這筆遷移新增了一個名為`products`的資料表，有著`name`(類型為字串) 和`description`(類型為 text) 欄位。此外，還會自動新增一個隱藏的主鍵欄位`id`，是所有 Active Record models 的預設欄位，欄內的數字是用來辨識每筆資料的流水號。接著，`timestamps`巨集再新增了`created_at`和`updated_at`兩個欄位。Active Record 會自動處理特殊欄位，像是`id`和`timestamps`。

請注意，我們定義的 change 方法是隨著時間推進的。在執行此筆遷移前，是沒有資料表的；執行後，資料表才會出現。Active Record 也能倒回遷移：如果我們回滾此筆遷移，資料表將被移除。

某些資料庫中的 transactions (交易功能) 包含了能變更資料庫綱要的陳述式，遷移在此類的資料庫中會被包在 transaction 中執行。若資料庫不支援此種 transactions，則遷移失敗時，已經成功執行的部份將無法被回滾。若想要恢復已變更的部份，只能以手動的方式進行。

NOTE: 某些特定的查詢無法在 transaction 中執行。若您的連接器支援 DDL transactions，可以用`disable_ddl_transaction!`在單筆遷移裡停用 transaction。

若想在遷移中執行 Active Record 無法回滾的指令，可以用`reversible`手動回滾：


```ruby
class ChangeProductsPrice < ActiveRecord::Migration[5.0]
  def change
    reversible do |dir|
      change_table :products do |t|
        dir.up   { t.change :price, :string }
        dir.down { t.change :price, :integer }
      end
    end
  end
end
```

也可以用`up`和`down`來取代`change`:

```ruby
class ChangeProductsPrice < ActiveRecord::Migration[5.0]
  def up
    change_table :products do |t|
      t.change :price, :string
    end
  end

  def down
    change_table :products do |t|
      t.change :price, :integer
    end
  end
end
```

2 建立遷移
--------------------

### 2.1 新建獨立的遷移

遷移會以檔案的形式儲存在`db/migrate`目錄下，每筆遷移皆對應一個檔案。檔名會以`YYYYMMDDHHMMSS_create_products.rb`的形式命名：  
  
前面的`YYYYMMDDHHMMSS`是 UTC 格式的時間戳章，接下來是底線，底線後面的`create_products`是該筆遷移的名稱。遷移類別以 CamelCased (駝峰式) 來命名，會對應到時間戳章後面的 `create_products`。舉例來說，`20080906120000_create_products.rb`會定義出`CreateProducts`這樣的類別名稱；而
`20080906120001_add_details_to_products.rb`則會定義出
`AddDetailsToProducts`這樣的類別名稱。Rails 會依循時間戳章決定要執行哪筆遷移以及執行的順序，所以當您從其他應用程式複製遷移檔案過來，或是自己產生遷移時，要特別注意它們的順序。  
  
然而，要計算時間戳章並不容易，所以 Active Record 提供了能幫您處理好時間戳章的產生器：

```bash
$ bin/rails generate migration AddPartNumberToProducts
```

這會建立一個有正確命名格式的空白遷移檔案：
  
```ruby
class AddPartNumberToProducts < ActiveRecord::Migration[5.0]
  def change
  end
end
```

在命令列輸入的遷移名稱若是 AddXXXToYYY 或 RemoveXXXFromYYY，且後面接著一系列的欄位名稱與類型，則會自動產生`add_column`或`remove_column`：
  
```bash
$ bin/rails generate migration AddPartNumberToProducts part_number:string
```

上面的指令會產生：

```ruby
class AddPartNumberToProducts < ActiveRecord::Migration[5.0]
  def change
    add_column :products, :part_number, :string
  end
end
```

也可以替欄位加上索引 (index)，像這樣：


```bash
$ bin/rails generate migration AddPartNumberToProducts part_number:string:index
```

會產生：

```ruby
class AddPartNumberToProducts < ActiveRecord::Migration[5.0]
  def change
    add_column :products, :part_number, :string
    add_index :products, :part_number
  end
end
```

同樣地，可以移除欄位：


```bash
$ bin/rails generate migration RemovePartNumberFromProducts part_number:string
```
會產生：

```ruby
class RemovePartNumberFromProducts < ActiveRecord::Migration[5.0]
  def change
    remove_column :products, :part_number, :string
  end
end
```

也可以一次產生多個欄位：

```bash
$ bin/rails generate migration AddDetailsToProducts part_number:string price:decimal
```

會產生：

```ruby
class AddDetailsToProducts < ActiveRecord::Migration[5.0]
  def change
    add_column :products, :part_number, :string
    add_column :products, :price, :decimal
  end
end
```

剛剛已經看過兩種常見的遷移命名形式：AddXXXToYYY、RemoveXXXFromYYY。還有一種是 CreateXXX，後面接欄位名稱與類型，像這樣：

```bash
$ bin/rails generate migration CreateProducts name:string part_number:string
```
會新建 Products 資料表 (table)，有著 name 和 part_number 欄位：

```ruby
class CreateProducts < ActiveRecord::Migration[5.0]
  def change
    create_table :products do |t|
      t.string :name
      t.string :part_number
    end
  end
end
```

當然，產生遷移檔案不過是第一步，您還可以透過編輯`db/migrate/YYYYMMDDHHMMSS_add_details_to_products.rb`來新增或移除以符合需求。

產生器也支援`references`欄位類型 (等同於`belongs_to`)，例如：


```bash
$ bin/rails generate migration AddUserRefToProducts user:references
```

會產生：

```ruby
class AddUserRefToProducts < ActiveRecord::Migration[5.0]
  def change
    add_reference :products, :user, index: true, foreign_key: true
  end
end
```

這會給 Products 資料表新增一個`user_id`欄位並加上索引。若想知道更多`add_reference`的用法，可以至
[API 文件](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_reference)，有詳細的說明。

若遷移名稱中含有`JoinTable`，則會建立連接資料表 (join table)： 

```bash
$ bin/rails g migration CreateJoinTableCustomerProduct customer product
```

會產生：

```ruby
class CreateJoinTableCustomerProduct < ActiveRecord::Migration[5.0]
  def change
    create_join_table :customers, :products do |t|
      # t.index [:customer_id, :product_id]
      # t.index [:product_id, :customer_id]
    end
  end
end
```

### 2.2 Model 產生器

當 model (模型) 和 scaffold generator (鷹架產生器) 新增 model 時，也會建立遷移。這個遷移檔案會包含建立相關資料表的步驟。若進一步告訴 Rails 所需的欄位，欄位也會加入至遷移檔案裡。舉例來說，執行：


```bash
$ bin/rails generate model Product name:string description:text
```

會建立像這樣的遷移：

```ruby
class CreateProducts < ActiveRecord::Migration[5.0]
  def change
    create_table :products do |t|
      t.string :name
      t.text :description

      t.timestamps
    end
  end
end
```

您可以無上限的附加欄位名稱/類型。

### 2.3 傳入類型修飾符

某些常用的[類型修飾符](#column-modifiers)可以直接在命令列指定。這些修飾符在欄位類型之後指定，用大括號包起來：can be passed directly on
the command line. They are enclosed by curly braces and follow the field type:

舉例來說，執行：

```bash
$ bin/rails generate migration AddDetailsToProducts 'price:decimal{5,2}' supplier:references{polymorphic}
```

會建立這樣的遷移檔案：

```ruby
class AddDetailsToProducts < ActiveRecord::Migration[5.0]
  def change
    add_column :products, :price, :decimal, precision: 5, scale: 2
    add_reference :products, :supplier, polymorphic: true, index: true
  end
end
```

TIP: 可以看看產生器的說明文字來了解更多細節。

3 撰寫遷移
-------------------

使用產生器建立遷移檔案後，就可以開工了！

### 3.1 建立資料表

`create_table`是最基礎的方法之一，但通常只要使用 model 或是 scaffold 產生器便會自動產生出來。以下是個常見用法：  


```ruby
create_table :products do |t|
  t.string :name
end
```

會建立一張`products`資料表，有著 name 欄位 (以及隱藏的`id`欄位，下面會說明)

`create_table`預設會產生主鍵`id`，可以用`:primary_key`選項來修改主鍵的名稱 (記得更新對應的 model)。若不要主鍵，可以傳入`id: false`選項。若想要傳入資料庫特定選項，可以在`:options`選項中加入 SQL 片段。例如：


```ruby
create_table :products, options: "ENGINE=BLACKHOLE" do |t|
  t.string :name, null: false
end
```

會在用來建立資料表的 SQL 陳述式中，附上`ENGINE=BLACKHOLE`(使用 MySQL 或是 MariaDB 的預設是 `ENGINE=InnoDB`)。  
  
您還可以傳入`:comment`選項來加入對資料表的註解。這些註解會儲存在資料庫中，只要用資料庫管理工具就可以觀看，像是 MySQL Workbench 或 PgAdmin III。對於資料庫龐大的應用程式，強烈建議在遷移中加入註解，這樣能幫助其他人了解資料形式並建立程式文件。目前只有 MySQL 和 PostgreSQL 連接器有支援註解。


### 3.2 建立連接資料表

`create_join_table`方法會建立一張 HABTM (has and belongs to many，多對多繫結) 連接資料表。常見的用法如下：


```ruby
create_join_table :products, :categories
```

會建立一張`categories_products`資料表，包含`category_id`與`product_id`欄位。這些欄位的`:null`選項預設為`false`。可以用`:column_options`來修改預設值：


```ruby
create_join_table :products, :categories, column_options: { null: true }
```

連接資料表的預設名稱是由`create_join_table`的前面兩個引數組成的，按字母順序排列。若要修改名稱，可以用`:table_name`選項：


```ruby
create_join_table :products, :categories, table_name: :categorization
```

會建立`categorization`資料表。


`create_join_table`也支援區塊，可以加入索引 (預設中不會加)，或新增更多欄位： 

```ruby
create_join_table :products, :categories do |t|
  t.index :product_id
  t.index :category_id
end
```

### 3.3 修改資料表

`change_table`用來修改已存在的資料表。使用方式與`create_table`雷同，但傳入區塊的物件有更多方法可用。例如：

```ruby
change_table :products do |t|
  t.remove :description, :name
  t.string :part_number
  t.index :part_number
  t.rename :upccode, :upc_code
end
```

這會移除`description`與`name`欄位，新增`part_number`字串欄位並加入索引，最後替`upccode`欄位重新命名。


### 3.4 修改欄位


就像`remove_column`與`add_column`，Rails 也提供了`change_column`方法。

```ruby
change_column :products, :part_number, :text
```
這會將 products 資料表的`part_number`欄位改為`:text`類型。注意，`change_columb`指令是不可逆的。

除了`change_column`外，還有`change_column_null`與`change_column_default`方法，專門用來修改不被限制為 null 的欄位，以及欄位的預設值。

```ruby
change_column_null :products, :name, false
change_column_default :products, :approved, from: true, to: false
```

這會將 products 資料表的`:name`欄位設為`NOT NULL`，並將`:approved`欄位的預設值從 true 改為 false。


Note: 您也可以將`change_column_default`寫成
`change_column_default :products, :approved, false`，但不同的地方在於，這會使遷移不可逆。

### 3.5 欄位修飾符

欄位修飾符可在新建或修改欄位時使用：

* `limit`        設定`string/text/binary/integer`欄位的最大值。
* `precision`    定義`decimal`欄位的精度，含小數點可以有幾位。
* `scale`        定義`decimal`欄位的位數，小數點可以有幾位。
* `polymorphic`  給`belongs_to` associations (關連) 加入`type`欄位。
* `null`         欄位允許或不允許`NULL`值。
* `default`      允許設定欄位的預設值。注意，若使用了動態數值（譬如日期），預設值只會在第一次跑遷移時做計算（也就是跑遷移當下的日期）。
* `index`        給欄位加入索引。
* `comment`      給欄位加入註解。

某些連接器可能支援其他選項，請參考與連接器相關的 API 文件來了解更多資訊。

### 3.6 外鍵

雖然不是必須的，但若要[保證參照完整性](#active-record-and-referential-integrity)，則需加入外鍵約束。

```ruby
add_foreign_key :articles, :authors
```

上例會給`articles`資料表新增外鍵欄位`author_id`。外鍵會使用 `articles`資料表的主鍵`id`作為參照。若參照的欄位名稱不能從資料表名稱推測出來，可以使用`:column`與`primary_key`選項。


Railes 會幫每個外鍵命名，統一由`fk_rails_`開頭，接著在後面加上從`from_table`與`column`隨機產生的十個字元。若想要不同的名稱，可以用`:name`選項來修改。


NOTE: Active Record 只支援單欄外鍵。若要使用組合外鍵則需要`execute`和`structure.sql`。請參考
[導出資料庫綱要](#schema-dumping-and-you)。

要移除外鍵也很簡單：

```ruby
# let Active Record figure out the column name
remove_foreign_key :accounts, :branches

# remove foreign key for a specific column
remove_foreign_key :accounts, column: :owner_id

# remove foreign key by name
remove_foreign_key :accounts, name: :special_fk_name
```

### 3.7 輔助方法不夠用時怎麼辦

當 Active Record 提供的輔助方法不夠用時，可以使用`execute`方法來執行任何 SQL 語句：


```ruby
Product.connection.execute("UPDATE products SET price = 'free' WHERE 1=1")
```

若想知道各方法更詳細的說明與範例，可以查閱 API 文件。特別是：
  
[`ActiveRecord::ConnectionAdapters::SchemaStatements`](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html)
(在`change`, `up`與`down`中有哪些方法可以使用)  
  
[`ActiveRecord::ConnectionAdapters::TableDefinition`](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/TableDefinition.html)
(用`create_table`產生的物件有哪些方法可以使用)  
  
[`ActiveRecord::ConnectionAdapters::Table`](http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/Table.html)
(用`change_table`產生的物件有哪些方法可以使用)

### 3.8 使用`change`方法

在多數情況下，使用`change`方法的遷移 Active Record 都能倒回，故`change`方法是撰寫遷移檔案最主要的方式。以下是目前`change`方法裡支援的方法：

* add_column
* add_foreign_key
* add_index
* add_reference
* add_timestamps
* change_column_default (must supply a :from and :to option)
* change_column_null
* create_join_table
* create_table
* disable_extension
* drop_join_table
* drop_table (must supply a block)
* enable_extension
* remove_column (must supply a type)
* remove_foreign_key (must supply a second table)
* remove_index
* remove_reference
* remove_timestamps
* rename_column
* rename_index
* rename_table

`change_table`也是可逆的，只要傳給`change_table`的區塊沒有呼叫`change`、`change_default`或是`remove`即可。

`remove_column`只有將欄位類型放在第三個引數的時候才可以倒回。原本的欄位選項也要留著，否則 Rails 在回滾時會無法正確重建欄位： 

```ruby
remove_column :posts, :slug, :string, null: false, default: '', index: true
```

若需要用到其他方法，請使用`reverisble`或是撰寫`up`、`down`方法，而不是使用`change`方法。


### 3.9 使用`reversible`方法

若遇到 Active Record 無法倒回的複雜遷移時，可以使用`reversible`方法來指定該如何執行遷移和倒回，例如：


```ruby
class ExampleMigration < ActiveRecord::Migration[5.0]
  def change
    create_table :distributors do |t|
      t.string :zipcode
    end

    reversible do |dir|
      dir.up do
        # add a CHECK constraint
        execute <<-SQL
          ALTER TABLE distributors
            ADD CONSTRAINT zipchk
              CHECK (char_length(zipcode) = 5) NO INHERIT;
        SQL
      end
      dir.down do
        execute <<-SQL
          ALTER TABLE distributors
            DROP CONSTRAINT zipchk
        SQL
      end
    end

    add_column :users, :home_page_url, :string
    rename_column :users, :email, :email_address
  end
end
```

使用`reversible`也能確保指令有依照正確的順序執行。若倒回上述的遷移，`down`區塊會在`home_page_url`欄位移除後和`distributors`資料表移除前執行。

有些遷移的操作是完全不可逆的，像是刪除資料。舉例來說，在`down`區塊中加入`ActiveRecord::IrreversibleMigration`後，當別的開發者想要倒回遷移，就會跳出無法倒回的的錯誤訊息。


### 3.10 使用`up`/`down`方法

除了`change`方法外，也可以使用以前的`up`、`down`方法來撰寫遷移檔案。`up`方法用來改變資料庫綱要，`down`方法
`up`方法撰寫對資料庫綱要的變化 (遷移)，`down`方法則是撰寫取消`up`的操作 (回滾)。換句話說，`up`和`down`要可以互相抵消。例如，`up`建了一張資料表，則`down`要刪除該資料表。撰寫`down`最好的作法是完全依照`up`的相反順序來寫。在 3.9 中使用`reversible`的例子可以用 up＋down 來改寫：


```ruby
class ExampleMigration < ActiveRecord::Migration[5.0]
  def up
    create_table :distributors do |t|
      t.string :zipcode
    end

    # add a CHECK constraint
    execute <<-SQL
      ALTER TABLE distributors
        ADD CONSTRAINT zipchk
        CHECK (char_length(zipcode) = 5);
    SQL

    add_column :users, :home_page_url, :string
    rename_column :users, :email, :email_address
  end

  def down
    rename_column :users, :email_address, :email
    remove_column :users, :home_page_url

    execute <<-SQL
      ALTER TABLE distributors
        DROP CONSTRAINT zipchk
    SQL

    drop_table :distributors
  end
end
```

若遷移是不可逆的，請在`down`中加入`ActiveRecord::IrreversibleMigration`。這樣當別的開發者要倒回遷移時就會跳出無法倒回的錯誤訊息。


### 3.11 倒回之前的遷移

Active Record 提供了回滾遷移的方法：`revert`

```ruby
require_relative '20121212123456_example_migration'

class FixupExampleMigration < ActiveRecord::Migration[5.0]
  def change
    revert ExampleMigration

    create_table(:apples) do |t|
      t.string :variety
    end
  end
end
```

`revert`方法也可以倒回一個區塊的指令，這在倒回先前遷移檔案中的特定部分時非常好用。舉例來說，在執行`ExampleMigration`後，才發覺在驗證郵遞區號的`CHECK`約束部分應該要用 Active Record validations (Active Record 驗證) 比較好。


```ruby
class DontUseConstraintForZipcodeValidationMigration < ActiveRecord::Migration[5.0]
  def change
    revert do
      # copy-pasted code from ExampleMigration
      reversible do |dir|
        dir.up do
          # add a CHECK constraint
          execute <<-SQL
            ALTER TABLE distributors
              ADD CONSTRAINT zipchk
                CHECK (char_length(zipcode) = 5);
          SQL
        end
        dir.down do
          execute <<-SQL
            ALTER TABLE distributors
              DROP CONSTRAINT zipchk
          SQL
        end
      end

      # The rest of the migration was ok
    end
  end
end
```

不使用`revert`也可以做到相同的遷移，但會多一些步驟：  
1. 將`create_table`與`reversible`的順序對調。  
2. 用`drop_table`取代`create_table`。  
3. 將`up`、`down`的程式碼對調。  
上述的步驟`revert`都幫您處理好了。  


NOTE: 若要像上述的例子一樣加入 check 約束，必須要使用`structure.sql`作為倒出方法。[導出資料庫綱要](#schema-dumping-and-you).

4 執行遷移
------------------

Rails 提供了一系列的 bin/rails 指令來執行特定的遷移。


`rails db:migrate`大概是第一個會用到的遷移相關 bin/rails 指令。最基本的`rails db:migrate`形式會按照遷移的時間戳章執行所有遷移中尚未執行的`change`或是`up`方法，全都執行完畢後便跳出。


請注意，當執行`db:migrate`指令時，也會執行`db:schema:dump`任務來更新`db/schema.rb`檔案，以符合最新的資料庫結構。

如果在`db:migrate`後面指定版本，Active Record 會執行該版本前所有的遷移 (chagne, up, down)。版本的名稱是遷移檔名前的 UTC 時間戳章。舉例來說，要遷移到 20080906120000 版本：


```bash
$ bin/rails db:migrate VERSION=20080906120000
```

若 20080906120000 版本比現在的版本還大 (假設是向上遷移)，會執行所有遷移中的`change` (或`up`) 方法直到 20080906120000 版本 (包含 20080906120000)。若是向下遷移，則會從現在的版本向下執行所有遷移中的`down`方法直到 20080906120000 版本 (不包含 20080906120000)。


### 4.1 回滾

想要回滾上一次的遷移是很常遇到的情況。例如，當你做錯時想要更正，與其查找上一次遷移的版本名稱，可以直接執行：


```bash
$ bin/rails db:rollback
```

這會倒回`change`方法或是執行`down`方法來回滾上一次的遷移。若需要回復數個遷移，可以使用`STEP`參數：


```bash
$ bin/rails db:rollback STEP=3
```

像這樣就會回滾前三次執行的遷移。


執行`db:migrate:redo`後會先回滾，再遷移。跟`db:rollback`一樣，只要使用`STEP`參數就可以一次回滾數個版本，例如：


```bash
$ bin/rails db:migrate:redo STEP=3
```

以上雖然用`db:migrate`都辦得到，但卻更方便，因為不必額外指定要遷移或回滾的版本名稱。


### 4.2 設定資料庫

`rails db:setup`會新建資料庫、載入資料庫綱要、並用種子資料來初始化資料庫。


### 4.3 重置資料庫

`rails db:reset`會將資料庫移除並重新建立，等同於`rails db:drop db:setup`。

NOTE: 這跟執行所有的遷移是不一樣的。這只會使用目前的`db/schema.rb`或是`db/structure.sql`檔案。當有無法回滾的遷移時，`rails db:reset`就派不上用場了。想了解更多請查閱[導出資料庫綱要](#schema-dumping-and-you)。

### 4.4 執行特定的遷移

若需要執行特定的遷移，可以使用`db:migrate:up`與`db:migrate:down`。只要指定特定的版本，就會執行對應的`change`、`up`或`down`方法，例如：

```bash
$ bin/rails db:migrate:up VERSION=20080906120000
```

會執行 20080906120000 遷移中的`change`方法 (或是`up`方法)。Active Record 會判斷此遷移是否執行過了，若有，則不會執行。


### 4.5 在不同環境下執行遷移

在預設的情況下，`bin/rails db:migrate`會在`development`環境下執行。若要在不同環境下執行，可以使用`RAILS_ENV`。例如，想在`text`環境下執行遷移：


```bash
$ bin/rails db:migrate RAILS_ENV=test
```

### 4.6 修改執行遷移的輸出

在預設的情況下，遷移會告訴你它做了些什麼和花了多少時間。一個建立資料表和加入索引的遷移會有這樣的輸出：


```bash
==  CreateProducts: migrating =================================================
-- create_table(:products)
   -> 0.0028s
==  CreateProducts: migrated (0.0028s) ========================================
```

遷移提供了一些讓您控制輸出訊息的方法：


| 方法                 | 目的
| -------------------- | -------
| suppress_messages    | 接受區塊作為參數，區塊內指名的代碼不會產生輸出。
| say                  | 接受一個訊息字串，並輸出該字串。第二個參數可以用來指定要不要縮排。
| say_with_time        | 同上，但會附上區塊的執行時間。若區塊返回整數，會假定該整數是受影響列的數量。

例如，這筆遷移：

```ruby
class CreateProducts < ActiveRecord::Migration[5.0]
  def change
    suppress_messages do
      create_table :products do |t|
        t.string :name
        t.text :description
        t.timestamps
      end
    end

    say "Created a table"

    suppress_messages {add_index :products, :name}
    say "and an index!", true

    say_with_time 'Waiting for a while' do
      sleep 10
      250
    end
  end
end
```

會輸出下列訊息：

```bash
==  CreateProducts: migrating =================================================
-- Created a table
   -> and an index!
-- Waiting for a while
   -> 10.0013s
   -> 250 rows
==  CreateProducts: migrated (10.0054s) =======================================
```

若不想要 Active Record 輸出任何訊息，可以執行`rails db:migrate
VERBOSE=false`來隱藏所有訊息。

5 修改現有的遷移
----------------------------

撰寫遷移難免會有寫錯的時候。若您已經執行了該筆遷移，就不能直接修改再執行一次：Rails 會認為該筆遷移已經執行過了所以，所以輸入`rails db:migrate`也不會有任何動作。 您必須回滾遷移 (例如，使用`bin/rails db:rollback`)，修改後再用`rails db:migrate`執行正確的版本。

修改現有的遷移會替您跟同事們造成很多負擔，尤其是現有的遷移已經上線的話更是麻煩，所以通常不建議這麼做。正確的作法應該是要寫一個新的遷移來修正。修改一個尚未提交到版本管理的新遷移是相較無害的。

`revert`方法在撰寫新的遷移來復原整個或部分的先前遷移時，非常有幫助。請查閱[倒回之前的遷移](#reverting-previous-migrations)。

導出資料庫綱要
----------------------

### 6.1 資料庫綱要檔案有什麼用？

遷移是會變的，無法反映出當下的資料庫結構。`db/schema.rb`或是 Active Record 檢驗資料庫後產生的 SQL 檔案才能反映當下的資料庫結構。這兩個檔案都是不能修改的，負責表示資料庫的狀態。


為了部屬一個新的應用程式而重新執行所有的遷移，是不必要且容易出錯的。最簡單的作法是把目前的資料庫綱要載入資料庫。

舉例來說，來看看建立測試資料庫的過程：導出目前的開發資料庫 (導出成`db/schema.rb`或`db/structure.sql`)，接著載入到測試資料庫。 

資料庫綱要檔案也能幫助您了解 Active Record 物件有哪些屬性。屬性在 model 中是看不到的，且通常散佈在各個遷移檔案中，但是資料庫綱要檔案都幫您整理好了。若想在 model 中看到屬性資訊，
[annotate_models](https://github.com/ctran/annotate_models) gem 會自動在每個 model 檔案上方加入來自資料庫綱要的註解。

### 6.2 導出資料庫綱要的種類

要導出資料庫綱要有兩種方式。可以在`config/application.rb`檔案裡，使用`config.active_record.schema_format`來設定，值可以是 :sql 或 :ruby。


若選擇用`:ruby`，資料庫綱要會儲存在`db/schema.rb`，您會發覺它很像一個大型的遷移檔案：


```ruby
ActiveRecord::Schema.define(version: 20080906171750) do
  create_table "authors", force: true do |t|
    t.string   "name"
    t.datetime "created_at"
    t.datetime "updated_at"
  end

  create_table "products", force: true do |t|
    t.string   "name"
    t.text "description"
    t.datetime "created_at"
    t.datetime "updated_at"
    t.string "part_number"
  end
end
```

就某種程度來說這就是一個大型遷移檔案。這個檔案的建立是藉由檢視資料庫，並以`create_table`、`add_index`等方法來呈現資料庫結構。它擁有資料庫獨立的特性，也就是說只要任何 Active Record 支援的資料庫都能載入。這對於需要發佈應用程式到數種資料庫的時候非常有用。


但這樣的特性也有它的缺點：`db/schema.rb`無法呈現資料庫獨有的功能，像是觸發器 (triggers)、儲存過程 (stored procedures) 或是檢查約束 (check constraints)。那些在遷移中可以執行的自定 SQL 語句，是無法被資料庫綱要導出程式從資料庫中重建的。若要執行自定的 SQL，要將資料庫綱要的格式設成`:sql`。

與其使用 Active Record 提供的資料庫綱要導出程式，可以用特定資料庫的導出工具來導出資料庫結構（透過 db:structure:dump 指令來導出 db/structure.sql）。舉例來說，PostgreSQL 使用`pg_dump`這個工具來導出 SQL。至於 MySQL 和 MariaDB，資料庫綱要會包含數個資料表的`SHOW CREATE TABLE`輸出。

要載入這些資料庫綱要不過就是執行它們的 SQL 語句而已。也就是說，這會建立一個完整的資料庫結構複本。然而，使用`:sql`格式的資料庫綱要無法被其他的 RDBMS 資料庫載入。


### 6.3 導出資料庫綱要與版本管理

由於導出的資料庫綱要檔案是資料庫綱要最可靠的來源，所以強烈建議將它們加入版本管理。

`db/schema.rb`包含目前資料庫的版本號碼。這能夠應對合併兩個接觸到相同資料庫綱要的分支時產生的衝突。當衝突發生時，請手動保留最新的兩個版本來解決問題。

7 Active Record 與參照完整性
---------------------------------------

Active Record 認為事情要在 model 裡處理好，而不是在資料庫中。因此，不常使用像是觸發器或約束條件等需要在資料庫實作的功能。

像`validates :foreign_key, uniqueness: true`這樣的驗證，是 model 可以增強資料整合性的一種方法。`:dependent`選項讓 model 在刪除父物件後自動刪除子物件。像這樣在應用程式層級執行的操作無法保證參照的完整性，所以有些人認為應該要跟[外鍵約束](#foreign-keys) 一樣，放在資料庫解決。

雖然 Active Record 沒有提供所有能直接解決這類問題的工具，但可以用`execute`方法來執行任何 SQL 語句。


8 遷移與種子資料
------------------------

Rails 的遷移功能最主要的目的是以穩定的程序來修改資料庫綱要。遷移也可以用來新增和修改資料，這在無法被刪除或重建的現有資料庫中是很有用的，例如 production 資料庫：


```ruby
class AddInitialProducts < ActiveRecord::Migration[5.0]
  def up
    5.times do |i|
      Product.create(name: "Product ##{i}", description: "A product.")
    end
  end

  def down
    Product.delete_all
  end
end
```

使用 Rails 內建的 "seeds" 功能，可以輕鬆的替新建的資料庫加入初始資料。這在經常要重新載入資料庫的開發和測試階段非常有幫助。要使用這項功能非常容易，先在`db/seeds.rb`中寫一些 Ruby 程式碼，然後執行`rails db:seed`：


```ruby
5.times do |i|
  Product.create(name: "Product ##{i}", description: "A product.")
end
```

這樣的方式比用遷移來設定新應用程式的資料庫還要簡潔多了。

