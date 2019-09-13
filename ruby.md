# Ruby tips & tricks

## Benchmark

Benchmarking is very useful but I always forget the syntax and I have to google it, so here is a cool template:

```ruby
require 'benchmark'
require 'benchmark/ips'
require 'memory_profiler'

N = 50000

PROCS = [
  [
    "Description 1",
    -> { 'Insert code here' }
  ],
  [
    "Description 2",
    -> { 'Insert code here' }
  ]
]

puts("Benchmarking #{N} times\n\n")

Benchmark.bm do |x|
  PROCS.each do |(description, proc)|
    x.report(description) do
      N.times { proc.call }
    end
  end
end

Benchmark.ips do |x|
  PROCS.each do |(description, proc)|
    x.report(description) do
      N.times { proc.call }
    end
  end
end

PROCS.each do |(description, proc)|
  puts "\nMemory usage of #{description} ".ljust(50, '-')
  report = MemoryProfiler.report do
    proc.call
  end
  puts "\tTotal allocated: #{report.total_allocated_memsize} bytes (#{report.total_allocated} objects)"
  puts "\tTotal retained: #{report.total_retained_memsize} bytes (#{report.total_retained} objects)"
end
```

## Cache file contents

If you have an application that reads a limited amount of files several times, it can be useful to cache the contents to avoid IO operations (one use case can be reading fixture files in your test suite).
This implementation allows you to use Redis (if installed) or stores the contents in a hash in memory:

```ruby
require 'singleton'
require 'redis'

class FileCache
  include Singleton

  PREFIX = 'FileCache:'.freeze

  def initialize
    if defined? Redis
      @database = Redis.new
      @driver = :redis
    else
      @database = {}
      @driver = :hash
    end
  end

  def read(path)
    return get_file_from_redis(path) if redis?
    get_file_from_hash(path)
  end

  def self.read(path)
    instance.read(path)
  end

  def flush_all
    if redis?
      @database.flushall
    else
      @database = {}
    end
    :ok
  end

  def self.flush_all
    instance.flush_all
  end

  def clear
    if redis?
      @database.del(@database.keys.select { |key| key =~ Regexp.new(PREFIX) })
    else
      @database = {}
    end
    :ok
  end

  def self.clear
    instance.clear
  end

  def redis?
    @driver == :redis
  end

  private

  def get_file_from_redis(path)
    content = @database.get("#{PREFIX}#{path}")
    return content unless content.nil?
    content = read_content(path)
    @database.set("#{PREFIX}#{path}", content)
    content
  end

  def get_file_from_hash(path)
    return @database["#{PREFIX}#{path}"] if @database.key?(path)
    content = read_content(path)
    @database["#{PREFIX}#{path}"] = content
    content
  end

  def read_content(path)
    File.read(path)
  end
end
```

With this class defined you can use it this way:

```ruby
FileCache.read('path/to/my/file')
```

## Core extensions

Here are some core extensions to built-in classes that can be useful:

```ruby
class Float
  # Round a number up
  #
  #   12.34.round_up(1)  # => 12.4
  #   12.34.round_up(2)  # => 12.35
  #   12.34.round_up(3)  # => 12.341
  def round_up(decimals)
    (self + 10**(-1 * decimals)).round(decimals)
  end
end

# https://github.com/rails/rails/blob/master/activesupport/lib/active_support/core_ext/hash/compact.rb
class Hash
  unless Hash.instance_methods(false).include?(:compact)
    # Returns a hash with non +nil+ values.
    #
    #   hash = { a: true, b: false, c: nil }
    #   hash.compact        # => { a: true, b: false }
    #   hash                # => { a: true, b: false, c: nil }
    #   { c: nil }.compact  # => {}
    #   { c: true }.compact # => { c: true }
    def compact
      select { |_, value| !value.nil? }
    end
  end

  unless Hash.instance_methods(false).include?(:compact!)
    # Replaces current hash with non +nil+ values.
    # Returns +nil+ if no changes were made, otherwise returns the hash.
    #
    #   hash = { a: true, b: false, c: nil }
    #   hash.compact!        # => { a: true, b: false }
    #   hash                 # => { a: true, b: false }
    #   { c: true }.compact! # => nil
    def compact!
      reject! { |_, value| value.nil? }
    end
    
    def contains?(other_hash)
      return false if other_hash.nil?

      merge(other_hash) == self
    end
  end
end

class String
  BOOLEANS = {
      'true' => true, 'True' => true
  }.freeze

  def to_bool
    BOOLEANS.fetch(self, false)
  end
end

class String
  def colorize(color_code)
    "\e[#{color_code}m#{self}\e[0m"
  end

  def red
    colorize(31)
  end

  def green
    colorize(32)
  end

  def yellow
    colorize(33)
  end

  def blue
    colorize(34)
  end
end
```

## Create hash with empty (or not) values

Let's say you have an array of keys and you want to create a hash with those values as the keys with empty values (or some other value).

```ruby
keys = %i[one two three four]

Hash[
  keys.zip(
    Array.new(keys.size)
  )
]
#=> {:one=>nil, :two=>nil, :three=>nil, :four=>nil}

# Passing another parameter to the Array builder assigns other values

Hash[
  keys.zip(
    Array.new(keys.size) { [] }
  )
]
#=> {:one=>[], :two=>[], :three=>[], :four=>[]}
```

## Deep dup

The built-in `dup` method does a shallow copy of objects:

```ruby
a = { one: { two: []} }
#=> {:one=>{:two=>[]}}
b = a.dup
#=> {:one=>{:two=>[]}}
b[:one][:two] << 1
#=> [1]
b
#=> {:one=>{:two=>[1]}}
a
#=> {:one=>{:two=>[1]}}
```

So, let's create a `deep_dup` method that creates a deep copy of objects:

```ruby
def deep_dup(obj)
  Marshal.load(Marshal.dump(obj))
end

a = { one: { two: []} }
#=> {:one=>{:two=>[]}}
b = deep_clone(a)
#=> {:one=>{:two=>[]}}
b[:one][:two] << 1
#=> [1]
a
#=> {:one=>{:two=>[]}}
b
#=> {:one=>{:two=>[1]}}
```

## Pass arguments to the shorthand way of calling map

Ruby has a short way to call map, making this two equivalent:

```ruby
["1", "2", "3"].map { |number| number.to_i } #=> [1, 2, 3]

["1", "2", "3"].map(&:to_i) #=> [1, 2, 3]
```

The problem is that we can't use the shorthand syntax for methods that receive arguments:

```ruby
[["Jane", 1], ["Doe", 2]].map { |pair| pair.join(': ') }.join("\n") #=> "Jane: 1\nDoe: 2"

[["Jane", 1], ["Doe", 2]].map(&:join(': ')).join("\n")
# SyntaxError ((irb):1: syntax error, unexpected '(', expecting ')')
# ...], ["Doe", 2]].map(&:join(': ')).join("\n")
# ...                              ^

[["Jane", 1], ["Doe", 2]].map(&:join, ': ').join("\n")
# SyntaxError ((irb):1: syntax error, unexpected ',', expecting ')')
# ...e", 1], ["Doe", 2]].map(&:join, ': ').join("\n")
# ...                              ^
```

One way to achieve this would be by monkey patching the symbol class:

```ruby
class Symbol
  def with(*args, &block)
    ->(caller, *rest) { caller.send(self, *rest, *args, &block) }
  end
end
```

This way we can do:

```ruby
[["Jane", 1], ["Doe", 2]].map(&:join.with(': ')).join("\n") #=> "Jane: 1\nDoe: 2"
```
