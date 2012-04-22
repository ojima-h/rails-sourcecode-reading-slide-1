!SLIDE 

Rails SourceCode Reading \#1
==========

ActiveRecord
----------

!SLIDE commandline

例えば、次のようなコマンドを実行すると...

	$ rails generate Page title:string
	$ rake db:migrate
	
`Page`クラスに`title`というメソッドが追加される。

この辺の仕組みを見てみる

!SLIDE

app/models/page.rb を見てみると...

	@@@ ruby
	# app/models/page.rb
	class Page < ActiveRecord::Base
	end

!SLIDE commandline incremental

というわけで、とりあえず ActiveRecord のソースを取得。

	$ gem fetch activerecord
	$ gem unpack activerecord-<version>.gem
	
!SLIDE

activerecord の lib 以下を見てみるが、(当然) title らしきメソッドはみつからない。

ところが、lib/activerecord/base.rb を下の方まで見ていくと...

!SLIDE

	@@@ ruby
	# lib/activerecord/base.rb
	....
    include ActiveRecord::Persistence
    extend ActiveModel::Naming
    extend QueryCache::ClassMethods
    extend ActiveSupport::Benchmarkable
	....
	
てなかんじの 大量の `include` と `extend` が!!	
	
!SLIDE subsection

include と extend
----------

参考: [yamazのRails日記](http://rubyist.g.hatena.ne.jp/yamaz/20060722)

ざっくり言うと  

include と extend により "module" で定義されたメソッドをオブジェクトに追加できる。

extend はクラスメッソドとして、include は通常のインスタンスメソッドとして、追加する。

!SLIDE

includeをしらみつぶしにみていくと

	@@@ ruby
    include AttributeMethods
	
があやしい。。。

!SLIDE subsection

method_missing
----------

参考: [Rubyリファリンス](http://ref.xaio.jp/ruby/method_missing)

あるオブジェクトに`method_missing`というメソッドが定義されているとします。

このオブジェクトに対して、定義されていないメソッド `methodA` を呼び出したとき、method\_missing が呼び出されます。
第1引数に本来呼び出そうとしいたメソッド名、第2引数にそのときの引数が渡されます。

!SLIDE

さて、lib/activerecord/attribute\_methods.rb を見てみると、138行目に method\_missing が定義されています。

	@@@ ruby
	# module ActiveRecord::AttributeMethods
    def method_missing(method, *args, &block)
      unless self.class.attribute_methods_generated?
        self.class.define_attribute_methods

        if respond_to_without_attributes?(method)
          send(method, *args, &block)
        else
          super
        end
      else
        super
      end
    end

!SLIDE

`self.class.define_attributes_methods`に注目。

この場合、AttributeMethod を include しているのは ActiveRecord::Base。

したがって、`self` は ActiveRecord (の派生クラス) のインスタンス。

`self.class` は ActiveRecord (の派生クラス) 自身。

したがって、`self.class.define_attributes_method` はActiveRecord (の派生クラス) の特異メソッド。

!SLIDE

`define_attributes_method` を探していくと

同じファイル(lib/activerecord/attribute\_methods.rb) の38行目に発見。  

	@@@ ruby
	ActiveRecord::AttributeMethods::ClassMethods#define_attributes_method
	
ところが、`define_attributes_method` はサブモジュール ClassMethod のメソッドなので、直接はインクルードされないはず...

と思ってたら、こんなページを発見: <http://d.hatena.ne.jp/kitokitoki/20110301/p1>

!SLIDE subsection

ClassMethods
----------

[Ruby on Rails Documentation](http://api.rubyonrails.org/classes/ActiveSupport/Concern.html)

通常、モジュールを include しても、追加できるのはインスタンスメソッドだけです。

ところが、モジュールの定義において `ActiveSupport::Concern` を extend し、更にモジュール `ClassMethods` を
定義することで、 `ClassMethods` 内において定義されているメソッドをクラスメソッドとして追加できるようになります。

!SLIDE

というわけで、`ActiveRecord::AttributeMethods::ClassMethods`  
`#define_attributes_method` の定義を見ていく。

大事なのはこの辺:

	@@@ ruby
        @attribute_methods_mutex.synchronize do
          return if attribute_methods_generated?
          superclass.define_attribute_methods unless self == base_class
          super(column_names)
          @attribute_methods_generated = true
        end

!SLIDE 

まずは、base_classの定義を探してみる。

	@@@ ruby
	# base.rb:696
	    include Inheritance
		
    # inheritance.rb:42
	# ActiveRecord::Inheritance::ClassMethods
      def base_class
        class_of_active_record_descendant(self)
      end

	# inheritance.rb:86
      def class_of_active_record_descendant(klass)
        if klass == Base || klass.superclass == Base || klass.superclass.abstract_class?
          klass
        elsif klass.superclass.nil?
          raise ActiveRecordError, "#{name} doesn't belong in a hierarchy descending from ActiveRecord"
        else
          class_of_active_record_descendant(klass.superclass)
        end
      end
	
!SLIDE subsection

super
----------

[Rubyリファレンスマニュアル](http://doc.ruby-lang.org/ja/1.9.2/doc/spec=2fcall.html)

メソッドの定義内で super が呼ばれると、そのメソッドが上書きしようとしているメソッドを呼び出します。

!SLIDE

もう1度 attribute_methods.rb を見ていくと

	@@@ ruby
    module ClassMethods
      def define_attribute_methods

のちょっと前にこんな記述が...

    @@@ ruby
    include ActiveModel::AttributeMethods

!SLIDE

ActiveModel のソースをとってきて中を見てみると、

activemodel-3.2.3/lib/active\_model/attribute\_methods.rb の255行目に `define_attribute_methods` の定義発見。

	@@@ ruby
	# ActiveRecord::AttributeMethods::ClassMethods#define_attributes_method
      def define_attribute_methods(attr_names)
        attr_names.each { |attr_name| define_attribute_method(attr_name) }
      end

!SLIDE

	@@@ ruby
      def define_attribute_method(attr_name)
        attribute_method_matchers.each do |matcher|
          method_name = matcher.method_name(attr_name)
          unless instance_method_already_implemented?(method_name)
            generate_method = "define_method_#{matcher.method_missing_target}"

            if respond_to?(generate_method, true)
              send(generate_method, attr_name)
            else
              define_optimized_call generated_attribute_methods, method_name, matcher.method_missing_target, attr_name.to_s
            end
          end
        end
        attribute_method_matchers_cache.clear
      end
	
!SLIDE

以下、素直に追っていくと、`ActiveRecord::MethodAttributes` の定義内の

	@@@ ruby
    included do
      include Read
      include Write
      include Query
      include TimeZoneConversion

が肝であることがわかる。

!SLIDE

例えば、lib/activerecord/method\_attributes/read.rb を見てみると、

	@@@ ruby
          def define_method_attribute(attr_name)
            cast_code = attribute_cast_code(attr_name)

            generated_attribute_methods.module_eval <<-STR, __FILE__, __LINE__ + 1
              def __temp__
                #{internal_attribute_access_code(attr_name, cast_code)}
              end
              alias_method '#{attr_name}', :__temp__
              undef_method :__temp__
            STR

            generated_external_attribute_methods.module_eval <<-STR, __FILE__, __LINE__ + 1
              def __temp__(v, attributes, attributes_cache, attr_name)
                #{external_attribute_access_code(attr_name, cast_code)}
              end
              alias_method '#{attr_name}', :__temp__
              undef_method :__temp__
            STR
          end

!SLIDE

`attribute_cast_code` は 「attr\_name に相当するデータベースのコラムからデータをとりだし、適当な型に変換するための__コード__」を返す。

	@@@ ruby
        case type
        when :string, :text        then var_name
        when :integer              then "(#{var_name}.to_i rescue #{var_name} ? 1 : 0)"
        when :float                then "#{var_name}.to_f"
		....
		
!SLIDE

`generated_attribute_methods` は__モジュール__オブジェクト。このモジュールの中に、各attributeにアクセスするためのメソッドを定義する

	@@@ ruby
	# activemodel/lib/active-model/attribute_methods.rb:285
      def generated_attribute_methods #:nodoc:
        @generated_attribute_methods ||= begin
          mod = Module.new
          include mod
          mod
        end
      end

即座に include されているので、このモジュールの中に定義されたメソッドも普通のメソッドと同様にアクセスできる。

!SLIDE

`internal_attribute_access_code` は実際に定義しようとしているメソッドの中身を記述する__コード__を返しす。

activerecord/lib/active\_record/attribute\_methods/read.rb:94 で定義されています。

例えば次のような文字列を返します。

    attr_name = '#{attr_name}';
	@attributes_cache[attr_name] ||=
	  ((v=@attributes[attr_name]) && (v.to_i rescue #{var_name} ? 1 : 0)

!SLIDE

`generated_external_attribute_methods` は `generated_attribute_methods` と同様、モジュールです。

しかし、`generated_external_attribute_methods` はパブリックメッソドとして定義されており、ActiveRecord内から使われることはありません。

!SLIDE subsection

class_eval, module_eval
----------

[Ruby リファレンスマニュアル](http://doc.ruby-lang.org/ja/1.9.2/class/Module.html)

`Module#module_eval`, `Module#class_eval` は、文字列を受け取り、その文字列がモジュール/クラスの定義文中にあるものとみなして、
その文字列を解釈・実行する。

ただし、ローカル変数だけは `module_eval` が呼ばれた場所で評価される。

!SLIDE

というわけで、自分で定義してないメソッドが呼べるのは `module_eval` のおかげでした!!

`method_missing` を利用して、そのメソッドが初めて呼ばれたタイミングで定義しているというわけでした。
