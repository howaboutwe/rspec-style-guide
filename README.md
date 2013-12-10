# The RSpec Style Guide

This RSpec style guide recommends best practices so that real-world Ruby
programmers can write code that can be maintained by other real-world Ruby
programmers. A style guide that reflects real-world usage gets used, and a
style guide that holds to an ideal that has been rejected by the people it is
supposed to help risks not getting used at all &ndash; no matter how good it is.

The guide is separated into several sections of related rules. I've
tried to add the rationale behind the rules (if it's omitted I've
assumed that is pretty obvious).

The guide is still a work in progress - some rules are lacking
examples, some rules don't have examples that illustrate them clearly
enough. In due time these issues will be addressed - just keep them in
mind for now.

You can generate a PDF or an HTML copy of this guide using
[Transmuter](https://github.com/TechnoGate/transmuter).

## Table of Contents

* [RSpec](#rspec)
* [Views](#views)
* [Controllers](#controllers)
* [Models](#models)
* [Mailers](#mailers)
* [Rake Tasks](#rake-tasks)
* [Miscellaneous](#miscellaneous)
* [Contributing](#contributing)
* [Spread the Word](#spread-the-word)

## RSpec

* Try to use just one expectation per example, within reason.

    ```Ruby
    # bad
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns new article and renders the new article template' do
          get :new
          assigns[:article].should be_a_new Article
          response.should render_template :new
        end
      end

      # ...
    end

    # good
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns a new article' do
          get :new
          assigns[:article].should be_a_new Article
        end

        it 'renders the new article template' do
          get :new
          response.should render_template :new
        end
      end

    end
    ```

* Keep the full spec name (concatentation of the nested descriptions)
  grammatically correct.
  * Top level: use `describe` with a constant name: `describe User ...`
  * 2nd level: use `describe` with a method name: `describe "#awesome?"`
  * Inner blocks: use a `context` that starts with `when`: `context "when user is unsubscribed"`
  * Example describes the expectation: `it "is false"`.
  * Full spec name: "User#awesome? when user is unsubscribed is false"
 
* Do not use "should" in our example names.

  ```Ruby
  # good
  it "returns true"

  # bad
  it "should return true"

* Write expectations at a high level, removed from logic and implementation details.

  ```Ruby
  # bad
  it "calls more_results if i=0" do
    # ...
  end

  # good
  context "no results are returned by the initial search" do
    it "attempts to find more results" do
      # ...
    end
  end
  ```

* Use `describe` for concepts which don't in themselves vary (e.g. "callbacks, validations, <method_name>"). Typically these are nouns.

* Name `describe` blocks as follows:
  * use "description" for non-methods
  * use pound "#method" for instance methods
  * use dot ".method" for class methods

    ```Ruby
    class Article
      def summary
        #...
      end

      def self.latest
        #...
      end
    end

    # the spec...
    describe Article
      describe '#summary'
        #...
      end

      describe '.latest'
        #...
      end
    end
    ```

* Do not write iterators to generate tests.

  ```Ruby
  # bad
  [:new, :show, :index].each do |action|
    "it returns 200" do
      get action
      response.should be_ok
    end
  end
  ```

* Use Factory Girl to create test objects.

* Try to avoid mocking and stubbing, favoring test parameters or
  attributes instead. When resorting to mocking and stubbing, only
  mock against a small, stable, obvious (or documented) API, so stubs
  are likely to represent reality after future refactoring.

* Valid reasons to use stubs/mocks:
  * Performance: To prevent running a slow, unrelated task.
  * Determinism: To ensure the test gives the same result each
    time. e.g. Time.now, Kernel#rand, external web services.
  * Vendoring: When relying on 3rd party code used as a "black box",
    which wasn't written with testability in mind.
  * Legacy: Stubbing old code that requires complex setup. (New code
    should not require complex setup!)
  * BDD: To remove the dependence on code that does not yet exist.
  * Controller / Functional tests:

    > In a controller spec, we don't care about how our data objects are created or what data they contain; we are writing expectations for the functional behavior of that controller, and that controller only. Mocks and stubs are used to decouple from the model layer and stay focused on the task of specing the controller.
    >
    > joahking, RailsForum: http://railsforum.com/viewtopic.php?pid=68311#p68311

* Use `let` blocks instead of `before(:each)` blocks to create data for
  the spec examples. `let` blocks get lazily evaluated.

    ```Ruby
    # use this:
    let(:article) { FactoryGirl.create(:article) }

    # ... instead of this:
    before(:each) { @article = FactoryGirl.create(:article) }
    ```

### Views

* The directory structure of the view specs `spec/views` matches the
  one in `app/views`. For example the specs for the views in
  `app/views/users` are placed in `spec/views/users`.
* The naming convention for the view specs is adding `_spec.rb` to the
  view name, for example the view `_form.html.haml` has a
  corresponding spec `_form.html.haml_spec.rb`.
* `spec_helper.rb` need to be required in each view spec file.
* The outer `describe` block uses the path to the view without the
  `app/views` part. This is used by the `render` method when it is
  called without arguments.

    ```Ruby
    # spec/views/articles/new.html.haml_spec.rb
    require 'spec_helper'

    describe 'articles/new.html.haml' do
      # ...
    end
    ```

* The method `assign` supplies the instance variables which the view
  uses and are supplied by the controller.

    ```Ruby
    # spec/views/articles/edit.html.haml_spec.rb
    describe 'articles/edit.html.haml' do
    it 'renders the form for a new article creation' do
      assign(
        :article,
        mock_model(Article).as_new_record.as_null_object
      )
      render
      rendered.should have_selector('form',
        method: 'post',
        action: articles_path
      ) do |form|
        form.should have_selector('input', type: 'submit')
      end
    end
    ```

* The helpers specs are separated from the view specs in the `spec/helpers` directory.

### Controllers

* Mock the models and stub their methods. Testing the controller should not depend on the model creation.
* Test only the behaviour the controller should be responsible about:
  * Execution of particular methods
  * Data returned from the action - assigns, etc.
  * Result from the action - template render, redirect, etc.

        ```Ruby
        # Example of a commonly used controller spec
        # spec/controllers/articles_controller_spec.rb
        # We are interested only in the actions the controller should perform
        # So we are mocking the model creation and stubbing its methods
        # And we concentrate only on the things the controller should do

        describe ArticlesController do
          # The model will be used in the specs for all methods of the controller
          let(:article) { mock_model(Article) }

          describe 'POST create' do
            before { Article.stub(:new).and_return(article) }

            it 'creates a new article with the given attributes' do
              Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
              post :create, message: { title: 'The New Article Title' }
            end

            it 'saves the article' do
              article.should_receive(:save)
              post :create
            end

            it 'redirects to the Articles index' do
              article.stub(:save)
              post :create
              response.should redirect_to(action: 'index')
            end
          end
        end
        ```

* Use context when the controller action has different behaviour depending on the received params.

    ```Ruby
    # A classic example for use of contexts in a controller spec is creation or update when the object saves successfully or not.

    describe ArticlesController do
      let(:article) { mock_model(Article) }

      describe 'POST create' do
        before { Article.stub(:new).and_return(article) }

        it 'creates a new article with the given attributes' do
          Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
          post :create, article: { title: 'The New Article Title' }
        end

        it 'saves the article' do
          article.should_receive(:save)
          post :create
        end

        context 'when the article saves successfully' do
          before { article.stub(:save).and_return(true) }

          it 'sets a flash[:notice] message' do
            post :create
            flash[:notice].should eq('The article was saved successfully.')
          end

          it 'redirects to the Articles index' do
            post :create
            response.should redirect_to(action: 'index')
          end
        end

        context 'when the article fails to save' do
          before { article.stub(:save).and_return(false) }

          it 'assigns @article' do
            post :create
            assigns[:article].should be_eql(article)
          end

          it 're-renders the "new" template' do
            post :create
            response.should render_template('new')
          end
        end
      end
    end
    ```

### Models

* Do not mock the models in their own specs.
* Use Factory Girl or Mechanize to make real objects.
* It is acceptable to mock other models or child objects.
* Create the model for all examples in the spec to avoid duplication.
* Avoid unnecessary database calls.

    ```Ruby
    describe Article
      let(:article) { FactoryGirl.build(:article) }
    end
    ```

* Add an example ensuring that the fabricated model is valid.

    ```Ruby
    describe Article
      it 'is valid with valid attributes' do
        article.should be_valid
      end
    end
    ```

* When testing validations, use `have(x).errors_on` to specify the attibute
which should be validated. Using `be_valid` does not guarantee that the problem
 is in the intended attribute.

    ```Ruby
    # bad
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should_not be_valid
      end
    end

    # preferred
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should have(1).error_on(:title)
      end
    end
    ```

### Mailers

* The mailer spec should verify that:
  * the subject is correct
  * the receiver e-mail is correct
  * the e-mail is sent to the right e-mail address
  * testing the email body should be done in a view spec

     ```Ruby
     describe SubscriberMailer
       let(:subscriber) { FactoryGirl.build(:subscription, email: 'johndoe@test.com', name: 'John Doe') }

       describe 'successful registration email'
         subject { SubscriptionMailer.successful_registration_email(subscriber) }

         its(:subject) { should == 'Successful Registration!' }
         its(:from) { should == ['info@your_site.com'] }
         its(:to) { should == [subscriber.email] }
       end
     end
     ```

### Rake Tasks

* Rake tasks should be dumb (one-liners ideally), and call out to a model or
  library class which is tested like any other.

### Miscellaneous

* Avoid incidental state as much as possible.
    ```Ruby
    describe Article do
      let(:article) { FactoryGirl.create(:article) }
      let(:another_article) { FactoryGirl.create(:article) }

      describe "#publish" do
        # bad
        it "publishes the article" do
          article.publish
          
          # Creating another shared Article test object above would cause this
          # test to break
          Article.count.should == 2
        end

        # good
        it "publishes the article" do
          -> { article.publish }.should change(Article, :count).by(1)
        end
      end
    end
    ```
* Shared examples are helpful for tidying up repetitive expectations,
but should be written to be composable if possible to allow for situations where
many but not all of the shared expectations are required.

* Be careful not to focus on being "DRY" by moving repeated expectations
into a shared environment too early, as this can lead to brittle tests
that rely too much on one other.

# Contributing

Feel free to open tickets or send pull requests with improvements. Thanks in
advance for your help!

# Spread the Word

A community-driven style guide is of little use to a community that
doesn't know about its existence. Tweet about the guide, share it with
your friends and colleagues. Every comment, suggestion or opinion we
get makes the guide just a little bit better. And we want to have the
best possible guide, don't we?
