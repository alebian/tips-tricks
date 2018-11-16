# Ruby tips & tricks

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
