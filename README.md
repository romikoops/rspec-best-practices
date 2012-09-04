# Rspec Best Practices
A collection of Rspec testing best practices

## Table of Contents

* [Introduction](#introduction)
* [Describe your methods](#describe-your-methods)
* [Keep your descriptions short](#keep-your-descriptions-short)
* [Single expectation test](#single-expectation-test)
* [Test valid, edge and invalid cases] (#test-valid-edge-and-invalid-cases)
* [Use subject] (#use-subject)
* [Use let] (#use-let)
* [To mock or not to mock] (#to-mock-or-not-to-mock)
* [Create data only when needed] (#create-data-only-when-needed)
* [Use factories and not fixtures] (#use-factories-and-not-fixtures)
* [Create a do_action method] (#create-a-do_action-method)
* [Easy to read matcher] (#easy-to-read-matcher)
* [Shared examples] (#shared-examples)
* [Run a single test spec] (#run-a-single-test-spec)
* [Other (relevant) suggestions] (#other-relevant-suggestions)
* [Literature] (#literature)
* [Libraries] (#libraries)
* [Styles Guides] (#styles-guides)
* [Fast Tests] (#fast-tests)
* [Credits] (#credits)

## Introduction

  RSpec is a great tool in the behavior driven design process of writing human readable specifications that direct and validate the development of your application. What follows are some guidelines taken from the literature, online resources, and from our experience.

## Describe your methods

  Keep clear the methods you are describing using "." as prefix for class methods and "#" as prefix for instance methods.

```ruby
# wrong
describe "the authenticate method for User" do
describe "if the user is an admin" do 

# correct
describe ".authenticate" do
```

  Contexts are a powerful method to make your tests clear and well organized. In the long term this practice will keep tests easy to read.

```ruby
# wrong
it "should have 200 status code if logged in" do
  response.should respond_with 200
end
it "should have 401 status code if not logged in" do
  response.should respond_with 401
end

# correct
context "when logged in" do
  it { should respond_with 200 }
end
context "when logged out" do
  it { should respond_with 401 }
end
```

## Keep your descriptions short

  A spec description should never be longer than 40 characters. If this happens, it suggests you should split it using a context (some exceptions are allowed).

```ruby
# wrong
it "should have 422 status code if an unexpected params will be added" do

# correct  
context "when not valid"
it { should respond_with 422 }
```

  As a side effect, in the example we removed the description related to the status code, which has been replaced by the expectation it { should respond_with 422 }. As confirm, if you run this test with the command rspec filename you will obtain an output similar to this.

```ruby
when not valid
  it should respond with 422
```

## Single expectation test

  The "one expectation" tip is more broadly expressed as "each test should make only one assertion. This helps you on finding possible errors, going directly to the failing test, and to make your code readable.
  
  Note: keep in mind that single expectation test does not mean single line test.

```ruby
# wrong
it "should create a resource" do
  response.should respond_with_content_type(:json)
  response.should assign_to(:resource)
end

# correct
it { should respond_with_content_type(:json) }
it { should assign_to(:resource) }
```

## Test valid, edge and invalid cases

  Testing is a good practice, but if you do not test the edge cases, it will not be so useful. For example, consider the following action:

```ruby
# extract destroy action
def destroy
  @resource = Resource.where(:id => params[:id])
  if @resource
    @resource.destroy
    head 204
  else
    render :template => "shared/404", :status => 404,
  end
end
```

  The error I usually see lies in testing only whether the resource has been removed. But, there is also an edge case where the resource is not found, which is also important to test. As a rule of thumb, try to think of all the possible inputs and states you can, especially the strangest ones.

```ruby
describe "#destroy" do
  context "when resource is found" do
    it "should respond with 204"
    it "should assign @resource"
  end
  context "when resource is not found" do
    it "should respond with 404"
    it "should not assign @resource"
  end
end
```

## Use subject

  When you have several tests related to the same "subject" you can use the subject{} method to DRY them up.
  
```ruby
# wrong
it { assigns("message").should match /The resource name is Genoveffa/ }
it { assigns("message").should match /it was born in Billyville/ }
it { assigns("message").creator.should match /Claudiano/ }

# correct
subject { assigns("message") }
it { should match /The resource name is Genoveffa/ }
it { should match /it was born in Billville/ }
its(:creator) { should match /Claudiano/ }
```

  However, you should not use heavily constructed subjects:

```ruby
# wrong
subject { Hero.first.equipment.equipped }
it { should include "sword" }
```

  Also, do not use an explicit subject within a specification unless it is a one-line specification, since it is difficult to remember what it refers to:

```ruby
# wrong
subject { Hero.first }
it "should be equipped with a sword" do
  subject.equipment.equipped.should include "sword"
end

# correct
subject { Hero.first }
it { should be_brave }
```

## Use let

  When you have to assign a variable to test, instead of using a before each block, use let. It will load only when the variable is firstly used in the test and get cached until that specific test is finished

```ruby
# wrong
describe "a house" do
  before do
    @house = Factory(:house)
  end# correct
context "when logged in" do
  it { should respond_with 200 }
end

  subject { @house }
  its(:size) { should == 10 }
end

# correct
describe "a house"
  let(:house) { Factory(:house) }
  ... # any example that does not use house will not call factory
  subject { house }
  its(:size) { should == 10 }
end
```

  The value will be cached across multiple calls in the same example but not across examples. This means that if you change the cached variable you could not see changes. In that case use the reload method.
  
## To mock or not to mock

  Do not (over)use mocks and test real behavior when possible. Anyway, sometimes they can be really useful, for example if you want to get back a "not found resource" (one of the few cases I use it).
  
```ruby
# simulate a not found resource
context "when not found"
  before(:each) do
    Resource.stub(:where).with(:created_from => params[:id]).and_return(false)
    ...
  end
  it { should respond_with 404 }
end
```

## Create data only when needed

  If you have ever worked in a medium size project (but also in a small ones), test suites can be heavy to run. To solve this problem, is important to not load more data than needed. Also if you think you need dozens of records, usually you are wrong. As Dmytro says, add a parameter to the method, which will limit the number of records to return. In this case you can create 3 records, and pass 2 as a parameter.

```ruby
# correct
describe "User"
  describe ".top" do
    before { 3.times { Factory(:user) } }
    it { User.top(2).should have(2).item }
  end 
end
```
    
## Use factories and not fixtures

  This is an old topic, but it's still good to remember. Do not use fixtures which are difficult to control -- instead, use factories/blueprints. Use them to reduce the verbosity on creating new data.

```ruby
# wrong
user = User.create( :name => "Genoveffa",
                    :surname => "Piccolina",
                    :city => "Billyville",
                    :birth => "17 Agoust 1982",
                    :active => true)

# correct (FactoryGirl)
user = Factory.create(:user)
```

  When defining a factory, start from a base valid one, which you can easily extend later on, into the code. If interested, the Rails Test prescriptions book face this problem in depth. It also discusses why you should not use fixtures in favour of factories.
  
## Create a do_action method

  While testing rails controllers, I encountered a common pattern on calling the actions. The code I was getting through was something like this.

```ruby
# wrong
post :create, :name   => "Resource Name",
              :scope  => "http://www.example.com/resource/scope",
              :type   => "http://www.example.com/resource/type",
              :format => "json"
```

  The point, is that I was repeating this on several tests, so I had to dry it. The solution was to create a do_action method that could accept some options, so that I could make the same call like this.
  
```ruby
# correct
do_action(:name => "Another name")

# do_action definition
def do_action(options = {})
  attributes = {:name   => "Resource Name",
                :scope  => "http://www.example.com/resource/scope",
                :type   => "http://www.example.com/resource/type",
                :format => "json" }
  attributes.merge!(options.symbolize_keys!)
  post :create, attributes
end
```

  Also if we wrote more code, we can easily have a default "do_action" method which you can use in all of your tests.
  
## Easy to read matcher

  This is taken directly from carbonfive article. Sometimes you feel the need of having readable matchers. Check out rspec matcher.
  
```ruby
# wrong
lambda { model.save! }.should raise_error(ActiveRecord::RecordNotFound)
collection.size.should == 4
 
# correct
expect { model.save! }.to raise_error(ActiveRecord::RecordNotFound)
collection.should have(4).items
```
    
## Shared examples

  Making tests is great and you get more confident day after day. But there will be a point where you will start to see code duplication coming up everywhere. RSpec offers shared examples to DRY out your test suite.

```ruby
# wrong
describe #show
  context "when own resources" do
    it "should have it" do 
      resource = Factory("user") do_get format: json   
      assigns(users).should include(resource)
    end  
  end
  context "when does not own resource" do
    it "should not have it" do 
      not_owned_resource = Factory("unknown") do_get     
      format: json assigns(users).should_not include(not_owned_resource)
    end
  end
end
```
  
  In this example we use the method it_should_behave_like which refers to the shared example below.
  
```ruby
# correct
describe "#show" do 
  it_should_behave_like "a secure resource"
end

shared_examples_for "a secure resource" do
  context "when own the resource" do 
    it "should have it" do
      resource = Factory("user") 
      do_get format: json 
      assigns(users).should include(resource)
    end 
  end
  context "when does not own resource" do 
    it "should not have it" do
      not_owned_resource = Factory("unknown") 
      do_get format: json 
      assigns(users).should_not include(not_owned_resource)
    end 
  end
end
```
    
  Read more on Jeff Pollard article.
  
## Run a single test spec

  Also if you can automatically run updated tests with solutions like guard, you could have the need to run a specific spec into your test suite. This is the secret line.
  
```sh
rake spec SPEC=spec/controllers/sessions_controller_spec.rb \
          SPEC_OPTS="-e \"should log in with cookie\""
```

```ruby
let(:vehicle)             { FactoryGirl.create(:vehicle, vehicle_attributes) }
let(:vehicle_attributes)  { {} }

describe "#to_s"
  subject { vehicle.to_s }
  let(:vehicle_attributes) { {name: "Carzilla"} }
  it { should == "Carzilla" }
end
```

## Other (relevant) suggestions

* When something in your application goes wrong, write a test that reproduces the error and then correct it. You will gain several hour of sleep and more serenity.
* Start writing dirty tests, with long descriptions, without contexts, making multiple expectations for test, but then refactor and next time follow the right way.
* Use solutions like guard (using guard-rspec) to automatically run all of your test, without thinking about it. Combining it with growl, it will become one of your best friends. Examples of other solutions are test_notifier, watchr and autotest.
* Use TimeCop to mock and test methods that relies on time.
* Use Webmock to mock HTTP calls to remote service that could not be available all the time and that you want to personalize.
* Use a good looking formatter to check if your test passed or failed. I use fuubar, which to me looks perfect.

## Literature

* The RSpec Book 
* Rails Test Prescriptions 
* Everyday rails spec
* Eggs on bread best practices
* Carbon Five best practices
* Dmytro best practices
* Andy Vanasse best practices
* Jeff Pollard best practices
* How to get Rails 3 and RSpec 2 running specs fast

## Libraries

* RSpec 2
* Factory girl
* Shoulda
* Timecop
* Webmock
* Fuubar
* Autotest, Watchr and Test Notifier

## Styles Guides

* MongoID

## Fast Tests

* Great presentation about model isolation

## Credits

The document has been started from Andrea Reginato for the Hack for School project. A special thanks to the Lelylan Team. This document is licensed under Creative Commons Attribution 3.0 Unported License.

## Notes on the go

Controller testing. In my personal experience I was making controller test in the beginning. I read the books and I applied what I’ve read. After the first project I dropped them in favour of acceptance tests. I’m aware of the fact that acceptance tests are way slower and that in the long term they can be cumbersome, but my mix is to split the rails app in different services, use Spec