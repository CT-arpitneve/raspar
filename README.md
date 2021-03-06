## Raspar - scraping library

Raspar is a html scraping library which help to map html elements to ruby object using 'css' or 'xpath' selector.Using this library user can define multiple parser for different websites, raspar can then select parser according to page url.

[![Build Status](https://travis-ci.org/jiren/raspar.png?branch=master)](https://travis-ci.org/jiren/raspar)
 [![Coverage Status](https://coveralls.io/repos/jiren/raspar/badge.png?branch=master)](https://coveralls.io/r/jiren/raspar?branch=master)


## Installation

Add this line to your application's Gemfile:

    gem 'raspar', :git => 'git://github.com/jiren/raspar.git'

And then execute:

    $ bundle

## Usage

```ruby
  
  result = Raspar.parse(url, html) #This will return parsed result object hash.

  #Result
  { :products => [
    #<Raspar::Result:0x007ffc91e4d640
      @attrs={:name=>"Test1", :price=>"10", :image=>"1", :desc=>"Description"},
      @domain="example.com",
      @name=:products>,
    #<Raspar::Result:0x007ffc91e57be0
      @attrs={:name=>"Test2", :price=>"20", :image=>"2", :desc=>"Description"},
      @domain="example.com",
      @name=:products>
    ] 
   }

```

## Example

### Sample HTML

```html
<!DOCTYPE html>
<html>
    <body>
    <span class="desc">Description</span>
    <divi class="item">
      <img src="1">
      <span>Test1</span>
      <span class="price">10</span>
    </div>

    <div class="item">
      <img src="2">
      <span>Test2</span>
      <span class="price">20</span>
    </div>

    <span class="second">
      <img src="2">
      <span>Test2</span>
      <span class="price">20</span>
    </span>

    <div class="offer">
      <span class="name">First Offer</span>
      <span class="percentage">10% off</span>
    </div>

  </body>
</html>
```


#### Parser for above HTML 

```ruby
class SampleParser
  include Raspar

  domain 'http://sample.com'

  attr :desc, '.desc', :eval => :format_desc

  collection :product, '.item,span.second' do
    attr :image_url, 'img', :prop => 'src', :eval => :make_image_url
    attr :name,  'span:first'
    attr :price, 'span.price', :eval => Proc.new{|price, ele| price.to_i} 
    attr :price_map do |text, ele|
      val = ele.search('span').collect{|s| s.content.strip}
      {val[0] => val[1].to_f}
    end
  end

  collection :offer, '.offer' do
    attr :name, '.name'
    attr :discount, '.discount' do |text, ele|
      test.split('%').first.to_f
    end
  end

  def name_price(val, ele)
    val = ele.search('span').collect{|s| s.content.strip}
    {val[0] => val[1].to_f}
  end

  def make_image_url(path, ele)
    URI(@domain_url).merge(path).to_s
  end

  def format_desc(text, ele)
    "Description: #{text}"
  end

end
```

- 'domain' method registers parser for given domain, so raspar can differentiate parsers at runtime.
- Define 'attr' which is going to parse. First argument is 'css' or 'xpath' selector. Second argument contain options.
  - Valid options are :field, :eval.
  - :prop selects particular property/attribute for html element. In example for image, select image url using :prop => 'src'
  - :eval is used to post process attr value. It can be proc, method or block. Each method, proc or block used for eval has two argument, first is html element text and second is html element as a Nokogiri doc.  
  - if :eval is not define then parser will return text of selected html element.
- If your page has multiple type of objects or collections then define using 'collection' block. In above example '.item' and 'span.second' are products while '.offer' element contain offer detail.
- In html page some of attributes are common which do not reside under any particular collection, such attribute values will be added to each parsed object.

### Add Parser in different way

It takes only one argument domain url and block.

```ruby

Raspar.add('http://example.com') do
  attr :desc, '.desc', :eval => :format_desc

  collection :product, '.item,span.second' do
    attr :image_url, 'img', :prop => 'src'
    attr :name,  'span:first'
    attr :price, 'span.price', :eval => Proc.new{|price, ele| price.to_i} 
    attr :price_map do |text, ele|
      val = ele.search('span').collect{|s| s.content.strip}
      {val[0] => val[1].to_f}
    end
  end
  
  def format_desc(text, ele)
    "Desc: #{text.downcase}"
  end
  
end


```


### Dynamically add Parser

```ruby
  
domain  = 'http://www.sample.com'
selector_map = {
  :common_attrs => {
    :desc => {:select => '.desc'}
  },
  :collections =>{
    :item => {
      :select => 'div, span.second', 
      :attrs => {
        :name =>  { :select => 'span:first'},
        :price =>  { :select => 'span.price', :eval => :parse_price},
        :image => { :select => 'img', :prop => 'src'}
      }
    }
  }
}

module ParserHelper
  def parse_price(val, ele)
    val.gsub(/[ ,]/, ' ' => '', ',' => '.').to_f
  end
end

Raspar.add(domain, selector_map, ParserHelper) //Add parser

```

For post processing user can add parser helper, but it is not mandatory.


## Contributing

Please send me a pull request so that this can be improved.

## License

This is released under the MIT license.
