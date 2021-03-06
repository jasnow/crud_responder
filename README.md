# CrudResponder

### Embrace CRUD interface for your models

We should keep controllers skinny and don't repeat ourself, right? But this scaffolded action doesn't look very DRY:
```ruby
def create
  @post = Post.new(post_params)

  respond_to do |format|
    if @post.save
      format.html { redirect_to @post, notice: 'Post was successfully created.' }
      format.json { render :show, status: :created, location: @post }
    else
      format.html { render :new }
      format.json { render json: @post.errors, status: :unprocessable_entity }
    end
  end
end
```
You can make it simpler if you don't need to respond to anything but html, or use  [`responders`](https://github.com/plataformatec/responders) gem.

Let's assume we need only html and also add flash message on error:
```ruby
def create
  @post = Post.new(post_params)

  if @post.save
    redirect_to @post, notice: 'Post was successfully updated.' }
  else
    flash[:alert] = "Error updating post: #{@post.errors.full_messages.to_sentence}"
    render :edit
  end
end
```
But it's still not DRY because we have to repeat this code in every controller, and even a different action of the same controller:
```ruby
def update
  @post = Post.find(params[:id])

  if @post.update(post_params)
    redirect_to @post, notice: 'Post was successfully created.' }
  else
    flash[:alert] = "Error creating post: #{@post.errors.full_messages.to_sentence}"
    render :new
  end
end
```
But the only differences between these actions are:
* Method called on a model
* Action on success
* Action on error
* Text in flash message on success
* Text in flash message on error

And they are repeated over and over again. Let's extract these differences:
```ruby
def create
  @post = Post.new(post_params)
  crud_respond @post # will call save, set flashes and redirects/render appropriately
end

def update
  @post = Post.find(params[:id])
  @post.assign_attributes(post_params)
  crud_respond @post # also call save
end

def destroy
  @post = Post.find(params[:id])
  crud_respond @post # call destroy because called from destroy action
end

```
Method to call on the object determined by the name of controller's action.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'crud_responder', github: 'olegantonyan/crud_responder'
```

And then execute:

    $ bundle install

Or install it yourself as:

    $ gem install crud_responder

## Usage

Include `CrudResponder` into your `ApplicationController` (or whatever base controller you have)
```ruby
class ApplicationController
  include CrudResponder
end
```

Use `crud_respond` method with object you need to create, update or destroy. Optionally, you can pass options to this method to override default redirect and render actions
```ruby
def create
  @post = Post.new(post_params)
  crud_respond @post, success_url: root_path, error_action: :custom_new_action
  # will redirect to root_path in case of success or render :custom_new_action otherwise
end
```
By default you will be redirected to object show, objects index or back in this order. And render :new when creating object and :edit when updating.

If  you need to create a bunch of objects (not just one) you can create wrapper class with same interface. For example, uploading multiple files:
```ruby
# app/controllers/media_items_controller.rb
class MediaItemsController < ApplicationController
  def create_multiple
    @media_item_multiple = MediaItem::CreateMultiple.new(media_item_create_multiple_params)
    crud_respond @media_item_multiple, success_url: media_items_path
  end

  private

  def media_item_create_multiple_params
    params.require(:media_item_create_multiple).permit(:description, files: [])
  end
end

# app/models/media_item/create_multiple.rb
class MediaItem::CreateMultiple
  include ActiveModel::Model

  attr_accessor :description, :files
  validates :files, presence: true

  def save
    return false unless valid?
    ActiveRecord::Base.transaction do
      files.each do |file|
        MediaItem.create!(description: description,  file: file)
      end
    end
    true
  rescue ActiveRecord::RecordInvalid => e
    errors.add(:base, e.to_s)
    false
  end
end
```
Now your controllers are skinny again! Also, you are forced to think in terms of CRUD interface to models and REST to controllers.

## TODO

* Cutomizing flash messages (+ task to copy skeleton locale)
* Support for pure API controllers
* Push to rubygems

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/olegantonyan/crud_responder. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](contributor-covenant.org) code of conduct.


## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
