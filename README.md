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

## RSpec

* Use just one expectation per example.

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

* Make heavy use of `describe` and `context`, but do not use a `context` for a single test.
* Name the `describe` blocks as follows:
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

* Use Factory Girl or Mechanize to create test objects (follow the
  project's conventions)
* Make heavy use of mocks and stubs

    ```Ruby
    # mocking a model
    article = mock_model(Article)

    # stubbing a method
    Article.stub(:find).with(article.id).and_return(article)
    ```

* When mocking a model, use the `as_null_object` method. It tells the
  output to listen only for messages we expect and ignore any other
  messages.

    ```Ruby
    article = mock_model(Article).as_null_object
    ```

* Use `let` blocks instead of `before(:each)` blocks to create data for
  the spec examples. `let` blocks get lazily evaluated.

    ```Ruby
    # use this:
    let(:article) { Fabricate(:article) }

    # ... instead of this:
    before(:each) { @article = Fabricate(:article) }
    ```

* Use `subject` when possible

    ```Ruby
    describe Article do
      subject { Fabricate(:article) }

      it 'is not published on creation' do
        subject.should_not be_published
      end
    end
    ```

* Use `specify` if possible. It is a synonym of `it` but is more readable when there is no docstring.

    ```Ruby
    # bad
    describe Article do
      before { @article = Fabricate(:article) }

      it 'is not published on creation' do
        @article.should_not be_published
      end
    end

    # good
    describe Article do
      let(:article) { Fabricate(:article) }
      specify { article.should_not be_published }
    end
    ```

* Use `its` when possible

    ```Ruby
    # bad
    describe Article do
      subject { Fabricate(:article) }

      it 'has the current date as creation date' do
        subject.creation_date.should == Date.today
      end
    end

    # good
    describe Article do
      subject { Fabricate(:article) }
      its(:creation_date) { should == Date.today }
    end
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

* Always mock the models in the view specs. The purpose of the view is
  only to display information.
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

* When a view uses helper methods, these methods need to be
  stubbed. Stubbing the helper methods is done on the `template`
  object:

    ```Ruby
    # app/helpers/articles_helper.rb
    class ArticlesHelper
      def formatted_date(date)
        # ...
      end
    end

    # app/views/articles/show.html.haml
    = "Published at: #{formatted_date(@article.published_at)}"

    # spec/views/articles/show.html.haml_spec.rb
    describe 'articles/show.html.haml' do
      it 'displays the formatted date of article publishing'
        article = mock_model(Article, published_at: Date.new(2012, 01, 01))
        assign(:article, article)

        template.stub(:formatted_date).with(article.published_at).and_return('01.01.2012')

        render
        rendered.should have_content('Published at: 01.01.2012')
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

    ```Ruby
    describe Article
      let(:article) { Fabricate(:article) }
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

    # prefered
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should have(1).error_on(:title)
      end
    end
    ```

* Add a separate `describe` for each attribute which has validations.

    ```Ruby
    describe Article
      describe '#title'
        it 'is required' do
          article.title = nil
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

* When testing uniqueness of a model attribute, name the other object `another_object`.

    ```Ruby
    describe Article
      describe '#title'
        it 'is unique' do
          another_article = Fabricate.build(:article, title: article.title)
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

### Mailers

* The model in the mailer spec should be mocked. The mailer should not depend on the model creation.
* The mailer spec should verify that:
  * the subject is correct
  * the receiver e-mail is correct
  * the e-mail is sent to the right e-mail address
  * the e-mail contains the required information

     ```Ruby
     describe SubscriberMailer
       let(:subscriber) { mock_model(Subscription, email: 'johndoe@test.com', name: 'John Doe') }

       describe 'successful registration email'
         subject { SubscriptionMailer.successful_registration_email(subscriber) }

         its(:subject) { should == 'Successful Registration!' }
         its(:from) { should == ['info@your_site.com'] }
         its(:to) { should == [subscriber.email] }

         it 'contains the subscriber name' do
           subject.body.encoded.should match(subscriber.name)
         end
       end
     end
     ```

# Contributing

Feel free to open tickets or send pull requests with improvements. Thanks in
advance for your help!

# Spread the Word

A community-driven style guide is of little use to a community that
doesn't know about its existence. Tweet about the guide, share it with
your friends and colleagues. Every comment, suggestion or opinion we
get makes the guide just a little bit better. And we want to have the
best possible guide, don't we?
