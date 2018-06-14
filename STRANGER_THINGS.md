# mattr_accessor
```ruby
module CarColors
  mattr_accessor :colors do # provide block to set default values
    [:blue, :red, :green]
  end
end

class Sedan
  include CarColors
end

sedan = Sedan.new
sedan.colors # [:blue, :red, :green]
Sedan.class_variable_get "@@colors" # [:blue, :red, :green]
```
# How ruby store Constants in Module
```ruby
module Colors
  RED = '0xff0000'
end
```
First, when the module keyword is processed, the interpreter creates a new entry in the constant table of the class object stored in the Object constant. Said entry associates the name "Colors" to a newly created module object. Furthermore, the interpreter sets the name of the new module object to be the string "Colors".

Later, when the body of the module definition is interpreted, a new entry is created in the constant table of the module object stored in the Colors constant. That entry maps the name "RED" to the string "0xff0000".

In particular, Colors::RED is totally unrelated to any other RED constant that may live in any other class or module object. If there were any, they would have separate entries in their respective constant tables.

Pay special attention in the previous paragraphs to the distinction between class and module objects, constant names, and value objects associated to them in constant tables.
