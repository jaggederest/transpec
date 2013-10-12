[![Gem Version](https://badge.fury.io/rb/transpec.png)](http://badge.fury.io/rb/transpec) [![Dependency Status](https://gemnasium.com/yujinakayama/transpec.png)](https://gemnasium.com/yujinakayama/transpec) [![Build Status](https://travis-ci.org/yujinakayama/transpec.png?branch=master)](https://travis-ci.org/yujinakayama/transpec) [![Coverage Status](https://coveralls.io/repos/yujinakayama/transpec/badge.png)](https://coveralls.io/r/yujinakayama/transpec) [![Code Climate](https://codeclimate.com/github/yujinakayama/transpec.png)](https://codeclimate.com/github/yujinakayama/transpec)

# Transpec

**Transpec** automatically converts your specs into latest [RSpec](http://rspec.info/) syntax with static analysis.

This aims to facilitate smooth transition to RSpec 3.

See the following pages for the new RSpec syntax and the plan for RSpec 3:

* [Myron Marston » RSpec's New Expectation Syntax](http://myronmars.to/n/dev-blog/2012/06/rspecs-new-expectation-syntax)
* [RSpec's new message expectation syntax - Tea is awesome.](http://teaisaweso.me/blog/2013/05/27/rspecs-new-message-expectation-syntax/)
* [Myron Marston » The Plan for RSpec 3](http://myronmars.to/n/dev-blog/2013/07/the-plan-for-rspec-3)

Note that Transpec does not yet support all conversions for the RSpec changes,
and also the changes for RSpec 3 is not fixed and may vary in the future.
So it's recommended to follow updates of both RSpec and Transpec.

## Examples

Here's an example spec:

```ruby
describe Account do
  subject(:account) { Account.new(logger) }
  let(:logger) { mock('logger') }

  describe '#balance' do
    context 'initially' do
      it 'is zero' do
        account.balance.should == 0
      end
    end
  end

  describe '#close' do
    it 'logs an account closed message' do
      logger.should_receive(:account_closed).with(account)
      account.close
    end
  end

  describe '#renew' do
    context 'when the account is renewable and not closed' do
      before do
        account.stub(:renewable? => true, :closed? => false)
      end

      it 'does not raise error' do
        lambda { account.renew }.should_not raise_error(Account::RenewalError)
      end
    end
  end
end
```

Transpec would convert it to the following form:

```ruby
describe Account do
  subject(:account) { Account.new(logger) }
  let(:logger) { double('logger') }

  describe '#balance' do
    context 'initially' do
      it 'is zero' do
        expect(account.balance).to eq(0)
      end
    end
  end

  describe '#close' do
    it 'logs an account closed message' do
      expect(logger).to receive(:account_closed).with(account)
      account.close
    end
  end

  describe '#renew' do
    context 'when the account is renewable and not closed' do
      before do
        allow(account).to receive(:renewable?).and_return(true)
        allow(account).to receive(:closed?).and_return(false)
      end

      it 'does not raise error' do
        expect { account.renew }.not_to raise_error
      end
    end
  end
end
```

### Real Examples

You can see real conversion examples below:

* https://github.com/yujinakayama/guard/commit/transpec-demo
* https://github.com/yujinakayama/mail/commit/transpec-demo
* https://github.com/yujinakayama/twitter/commit/transpec-demo

## Installation

```bash
$ gem install transpec
```

## Basic Usage

Before converting your specs:

* Make sure your project has `rspec` gem dependency `2.14` or later. If not, change your `*.gemspec` or `Gemfile` to do so.
* Run `rspec` and check if all the specs pass.
* Ensure the Git repository is clean. (You don't want to mix up your changes and Transpec's changes, right?)

Then, run `transpec` (using `--commit-message` is recommended) in the project root directory:

```bash
$ cd some-project
$ transpec --commit-message
Processing spec/spec_helper.rb
Processing spec/spec_spec.rb
Processing spec/support/file_helper.rb
Processing spec/support/shared_context.rb
Processing spec/transpec/ast/scanner_spec.rb
Processing spec/transpec/ast/scope_stack_spec.rb
```

This will convert and overwrite all spec files in the `spec` directory.

After the conversion, run `rspec` again and check whether all pass:

```bash
$ bundle exec rspec
# ...
843 examples, 0 failures
```

If all pass, commit the changes with auto-generated message:

```bash
$ git add -u
$ git commit -eF .git/COMMIT_EDITMSG
```

And you are done!

## Options

### `-f/--force`

Force processing even if the current Git repository is not clean.

```bash
$ git status --short
 M spec/spec_helper.rb
$ transpec
The current Git repository is not clean. Aborting.
$ transpec --force
Processing spec/spec_helper.rb
Processing spec/spec_spec.rb
Processing spec/support/file_helper.rb
```

### `-m/--commit-message`

Generate commit message that describes conversion summary.
Currently only Git is supported.

When you commit, you need to run the following command to use the generated message:

```bash
$ git commit -eF .git/COMMIT_EDITMSG
```

### `-d/--disable`

Disable specific conversions.

```bash
$ transpec --disable expect_to_receive,allow_to_receive
```

#### Available conversion types

Conversion Type     | Target Syntax                    | Converted Syntax
--------------------|----------------------------------|----------------------------
`expect_to_matcher` | `obj.should matcher`             | `expect(obj).to matcher`
`expect_to_receive` | `obj.should_receive`             | `expect(obj).to receive`
`allow_to_receive`  | `obj.stub`                       | `allow(obj).to receive`
`deprecated`        | `obj.stub!`, `mock('foo')`, etc. | `obj.stub`, `double('foo')`

### `-n/--negative-form`

Specify negative form of `to` that is used in `expect` syntax.
Either `not_to` or `to_not`.
`not_to` is used by default.

```bash
$ transpec --negative-form to_not
```

### `-p/--no-parentheses-matcher-arg`

Suppress parenthesizing argument of matcher when converting
`should` with operator matcher to `expect` with non-operator matcher
(`expect` syntax does not directly support the operator matchers).
Note that it will be parenthesized even if this option is specified
when parentheses are necessary to keep the meaning of the expression.

```ruby
describe 'original spec' do
  it 'is an example' do
    1.should == 1
    2.should > 1
    'string'.should =~ /^str/
    [1, 2, 3].should =~ [2, 1, 3]
    { key: value }.should == { key: value }
  end
end

describe 'converted spec' do
  it 'is an example' do
    expect(1).to eq(1)
    expect(2).to be > 1
    expect('string').to match(/^str/)
    expect([1, 2, 3]).to match_array([2, 1, 3])
    expect({ key: value }).to eq({ key: value })
  end
end

describe 'converted spec with -p/--no-parentheses-matcher-arg option' do
  it 'is an example' do
    expect(1).to eq 1
    expect(2).to be > 1
    expect('string').to match /^str/
    expect([1, 2, 3]).to match_array [2, 1, 3]
    # With non-operator method, the parentheses are always required
    # to prevent the hash from being interpreted as a block.
    expect({ key: value }).to eq({ key: value })
  end
end
```

## Troubleshooting

You might see the following warning while conversion:

```
Cannot convert #should into #expect since #expect is not available in the context.
spec/awesome_spec.rb:4:      1.should == 1
```

This message would be shown with specs like this:

```ruby
describe '#should that cannot be converted to #expect' do
  class MyAwesomeTestRunner
    def run
      1.should == 1
    end
  end

  it 'is 1' do
    test_runner = MyAwesomeTestRunner.new
    test_runner.run
  end
end
```

### Reason

* `should` is defined on `Kernel` (included by `Object`), so you can use `should` almost everywhere.
* `expect` is defined on `RSpec::Matchers` (included by `RSpec::Core::ExampleGroup`), so you can use `expect` only where `self` is an instance of `RSpec::Core::ExampleGroup`.

With the above example, in the context of `1.should == 1`, the `self` is an instance of `MyAwesomeTestRunner`.
So Transpec tracks contexts and skips conversion if the target syntax cannot be converted in a case like this.

### Solution

You need to rewrite the spec by yourself.

## Supported Conversions

### Standard expectations

```ruby
# Targets
obj.should matcher
obj.should_not matcher

# Converted
expect(obj).to matcher
expect(obj).not_to matcher
expect(obj).to_not matcher # with `--negative-form to_not`
```

* Disabled by: `--disable expect_to_matcher`
* Related Information: [Myron Marston » RSpec's New Expectation Syntax](http://myronmars.to/n/dev-blog/2012/06/rspecs-new-expectation-syntax)

### Operator matchers

```ruby
# Targets
1.should == 1
1.should < 2
Integer.should === 1
'string'.should =~ /^str/
[1, 2, 3].should =~ [2, 1, 3]

# Converted
expect(1).to eq(1)
expect(1).to be < 2
expect(Integer).to be === 1
expect('string').to match(/^str/)
expect([1, 2, 3]).to match_array([2, 1, 3])
```

* Related Information: [Myron Marston » RSpec's New Expectation Syntax](http://myronmars.to/n/dev-blog/2012/06/rspecs-new-expectation-syntax#almost_all_matchers_are_supported)

### `be_close` matcher

```ruby
# Targets
(1.0 / 3.0).should be_close(0.333, 0.001)

# Converted
(1.0 / 3.0).should be_within(0.001).of(0.333)
```

* Disabled by: `--disable deprecated`
* Related Information: [New be within matcher and RSpec.deprecate fix · rspec/rspec-expectations](https://github.com/rspec/rspec-expectations/pull/32)

### Expectations on Proc

```ruby
# Targets
lambda { do_something }.should raise_error
proc { do_something }.should raise_error
-> { do_something }.should raise_error

# Converted
expect { do_something }.to raise_error
```

* Disabled by: `--disable expect_to_matcher`
* Related Information: [Myron Marston » RSpec's New Expectation Syntax](http://myronmars.to/n/dev-blog/2012/06/rspecs-new-expectation-syntax#unification_of_block_vs_value_syntaxes)

### Negative error expectations with specific error

```ruby
# Targets
expect { do_something }.not_to raise_error(SomeErrorClass)
expect { do_something }.not_to raise_error('message')
expect { do_something }.not_to raise_error(SomeErrorClass, 'message')
lambda { do_something }.should_not raise_error(SomeErrorClass)

# Converted
expect { do_something }.not_to raise_error
lambda { do_something }.should_not raise_error # with `--disable expect_to_matcher`
```

* Disabled by: `--disable deprecated`
* Related Information: [Consider deprecating `expect { }.not_to raise_error(SpecificErrorClass)` · rspec/rspec-expectations](https://github.com/rspec/rspec-expectations/issues/231)

### Message expectations

```ruby
# Targets
obj.should_receive(:foo)
SomeClass.any_instance.should_receive(:foo)

# Converted
expect(obj).to receive(:foo)
expect_any_instance_of(SomeClass).to receive(:foo)
```

* Disabled by: `--disable expect_to_receive`
* Related Information: [RSpec's new message expectation syntax - Tea is awesome.](http://teaisaweso.me/blog/2013/05/27/rspecs-new-message-expectation-syntax/)

### Message expectations that are actually method stubs

```ruby
# Targets
obj.should_receive(:foo).any_number_of_times
obj.should_receive(:foo).at_least(0)

SomeClass.any_instance.should_receive(:foo).any_number_of_times
SomeClass.any_instance.should_receive(:foo).at_least(0)

# Converted
allow(obj).to receive(:foo)
obj.stub(:foo) # with `--disable allow_to_receive`

allow_any_instance_of(SomeClass).to receive(:foo)
SomeClass.any_instance.stub(:foo) # with `--disable allow_to_receive`
```

* Disabled by: `--disable deprecated`
* Related Information: [Don't allow at_least(0) · rspec/rspec-mocks](https://github.com/rspec/rspec-mocks/issues/133)

### Method stubs

```ruby
# Targets
obj.stub(:foo)

obj.stub!(:foo)

obj.stub(:foo => 1, :bar => 2)

SomeClass.any_instance.stub(:foo)

# Converted
allow(obj).to receive(:foo)

allow(obj).to receive(:foo)

allow(obj).to receive(:foo).and_return(1)
allow(obj).to receive(:bar).and_return(2)

allow_any_instance_of(SomeClass).to receive(:foo)
```

* Disabled by: `--disable allow_to_receive`
* Related Information: [RSpec's new message expectation syntax - Tea is awesome.](http://teaisaweso.me/blog/2013/05/27/rspecs-new-message-expectation-syntax/)

### Deprecated method stub aliases

```ruby
# Targets
obj.stub!(:foo)
obj.unstub!(:foo)

# Converted
obj.stub(:foo) # with `--disable allow_to_receive`
obj.unstub(:foo)
```

* Disabled by: `--disable deprecated`
* Related Information: [Consider deprecating and/or removing #stub! and #unstub! at some point · rspec/rspec-mocks](https://github.com/rspec/rspec-mocks/issues/122)

### Method stubs with deprecated specification of number of times

```ruby
# Targets
obj.stub(:foo).any_number_of_times
obj.stub(:foo).at_least(0)

# Converted
allow(obj).to receive(:foo)
obj.stub(:foo) # with `--disable allow_to_receive`
```

* Disabled by: `--disable deprecated`
* Related Information: [Don't allow at_least(0) · rspec/rspec-mocks](https://github.com/rspec/rspec-mocks/issues/133)

### Deprecated test double aliases

```ruby
# Targets
stub('something')
mock('something')

# Converted
double('something')
```

* Disabled by: `--disable deprecated`
* Related Information: [Deprecate "stub" for doubles · rspec/rspec-mocks](https://github.com/rspec/rspec-mocks/issues/214)

## Compatibility

Tested on MRI 1.9, MRI 2.0 and JRuby in 1.9 mode.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## License

Copyright (c) 2013 Yuji Nakayama

See the [LICENSE.txt](LICENSE.txt) for details.
