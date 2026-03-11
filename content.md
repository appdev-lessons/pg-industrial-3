# Photogram Industrial: Starting user interface

## Walkthrough video

**Please note**, the video is from a previous iteration of the project, so there are some differences:

- Anything contained in the project "README" is now contained in this Lesson
- The layout uses a 3-column sidebar design instead of a traditional navbar
- Active Storage is used instead of CarrierWave for image uploads

Did you read the differences above? Good! Then [here is a walkthrough video for this project.](https://share.descript.com/view/P3PGeVSVtMW)

**As you watch the video, pause it frequently, read the associated text, and type out the code.**

## Getting started

Let's continue with the `photogram-industrial`, keeping in mind a rough target to work towards:

[photogram-industrial.matchthetarget.com](https://photogram-industrial.matchthetarget.com/)

So navigate to `github.com/codespaces` (or reopen the previous lesson and use the "Load assignment" button) and reopen your `photogram-industrial` project codespace to continue building on what you accomplished in _Photogram Industrial Parts 1 and 2_.

## Starting on the interface

Create a new git branch to start working on the UI:

```
git checkout -b <your-initials>-starting-on-ui
```

Our root route (set in Part 1) already points to `users#feed`:

```ruby
Rails.application.routes.draw do
  root "users#feed"
  # ...
```
{: filename="config/routes.rb" }

Before we start building views, make sure your sample data is loaded:

```
rake sample_data
```

### Remove default scaffold styling

When you generated your scaffolds, Rails created a file at `app/assets/stylesheets/scaffolds.scss` (or similar). This file adds some default styling that can conflict with Bootstrap. Find it and delete it, or remove its contents.

## Application layout

The application layout is the backbone of our UI. In the updated Photogram, we use a 3-column layout:

- **Left sidebar**: Navigation links (Feed, Discover, Add photo, Profile, Settings)
- **Center column**: Main content
- **Right sidebar**: Search bar and floating action button

This layout is responsive — on smaller screens, the sidebars collapse and a mobile bottom nav appears.

### Bootstrap CDN

First, make sure we have Bootstrap CSS and JavaScript loaded. Create a partial at `app/views/shared/_cdn_assets.html.erb` (if it doesn't exist already) and include the Bootstrap CDN links and Font Awesome for icons.

### Application layout file

Open `app/views/layouts/application.html.erb`. This is where the magic happens. Replace the default content inside `<body>` with our new layout:

```erb
<body>
  <%# New Photo Modal - available on every page when signed in %>
  <% if current_user.present? %>
    <div class="modal fade" id="new_photo" tabindex="-1" aria-labelledby="newPhotoLabel" aria-hidden="true">
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header">
            <h1 class="modal-title fs-5" id="newPhotoLabel"></h1>
            <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
          </div>
          <div class="modal-body">
            <%= render "photos/form", photo: current_user.own_photos.build %>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
          </div>
        </div>
      </div>
    </div>
  <% end %>

  <%# Flash messages %>
  <% if notice.present? || alert.present? %>
    <div class="container">
      <div class="row mb-2">
        <div class="col-md-6 offset-md-3">
          <% if notice.present? %>
            <%= render "shared/flash", message: notice, css_class: "success" %>
          <% end %>
          <% if alert.present? %>
            <%= render "shared/flash", message: alert, css_class: "danger" %>
          <% end %>
        </div>
      </div>
    </div>
  <% end %>

  <%# Mobile top navbar %>
  <%= render "shared/navbar" %>

  <%# Three-column layout %>
  <div class="container mt-5">
    <div class="row">
      <%# Left sidebar - navigation %>
      <div class="col-lg-2 offset-lg-1 col-md-2 d-none d-md-block">
        <% if current_user.present? %>
          <ul class="nav flex-column sticky-top">
            <li class="nav-item">
              <div class="dropdown">
                <%= button_tag class: "btn btn-link nav-link border-bottom rounded-bottom-0 text-decoration-none",
                data: { bs_toggle: "dropdown"}, aria: {expanded: false} do %>
                  <% if current_user.avatar_image.attached? %>
                    <%= image_tag current_user.avatar_image, class: "rounded-circle img-small img-cover" %>
                  <% end %>
                  <span class="d-xl-inline d-none">
                    <%= current_user.display_name %>
                    <div class="fw-lighter">
                      @<%= current_user.username %>
                    </div>
                  </span>
                <% end %>
                <ul class="dropdown-menu">
                  <li>
                    <%= link_to user_path(current_user.username), class: "dropdown-item icon-link" do %>
                      <i class="fa-regular fa-circle-user"></i>
                      Go to profile
                    <% end %>
                  </li>
                  <li>
                    <%= button_to destroy_user_session_path, method: :delete, class: "dropdown-item icon-link" do %>
                      <i class="fa-solid fa-arrow-right-from-bracket"></i>
                      Sign out
                    <% end %>
                  </li>
                </ul>
              </div>
            </li>
            <li class="nav-item">
              <%= link_to feed_path(current_user.username), class: "nav-link" do %>
                <i class="fs-2 fa-solid fa-house"></i>
                <span class="d-xl-inline d-none">Feed</span>
              <% end %>
            </li>
            <li class="nav-item">
              <%= link_to discover_path(current_user.username), class: "nav-link" do %>
                <i class="fs-2 fa-solid fa-magnifying-glass"></i>
                <span class="d-xl-inline d-none">Discover</span>
              <% end %>
            </li>
            <li class="nav-item">
              <%= button_tag class: "nav-link", data: {bs_toggle: "modal", bs_target: "#new_photo"} do %>
                <i class="fs-2 fa-solid fa-pen-to-square"></i>
                <span class="d-xl-inline d-none">Add photo</span>
              <% end %>
            </li>
            <li class="nav-item">
              <%= link_to user_path(current_user.username), class: "nav-link" do %>
                <i class="fs-2 fa-regular fa-circle-user"></i>
                <span class="d-xl-inline d-none">Profile</span>
              <% end %>
            </li>
            <li class="nav-item">
              <%= link_to edit_user_registration_path, class: "nav-link" do %>
                <i class="fs-2 fa-solid fa-gear"></i>
                <span class="d-xl-inline d-none">Settings</span>
              <% end %>
            </li>
          </ul>
        <% end %>
      </div>

      <%# Center column - main content %>
      <div class="col-lg-6 col-md-8 col-12 <%= "border-start border-end" if current_user.present? %>">
        <%= yield %>
      </div>

      <%# Right sidebar - search %>
      <div class="col-lg-3 col-md-2 d-none d-md-block">
        <% if current_user.present? %>
          <div class="d-none d-xxl-block mt-2">
            <%= search_form_for @q, class: "d-flex", role: "search" do |f| %>
              <%= f.label :username_cont, class: "visually-hidden" %>
              <%= f.search_field :username_cont, class:"form-control me-2", placeholder: "Search by username" %>
              <%= f.button class: "btn btn-outline-primary", name: nil, type: "submit" do %>
                <i class="fas fa-search"></i>
              <% end %>
            <% end %>
          </div>
          <div class="fixed-bottom me-5 mb-5" style="left: unset">
            <%= button_tag class: "btn btn-primary rounded-circle fs-1", data: {bs_toggle: "modal", bs_target: "#new_photo"} do %>
              <i class="fa-solid fa-pen-to-square"></i>
            <% end %>
          </div>
        <% end %>
      </div>
    </div>
  </div>

  <%# Mobile bottom navigation %>
  <% if current_user.present? %>
    <nav class="navbar fixed-bottom bg-body-tertiary d-block d-sm-none">
      <div class="container-fluid">
        <div class="w-100">
          <ul class="navbar-nav d-flex flex-row justify-content-evenly">
            <li class="nav-item">
              <%= link_to feed_path(current_user.username), class: "nav-link" do %>
                <i class="fs-2 fa-solid fa-house"></i>
              <% end %>
            </li>
            <li class="nav-item">
              <%= link_to discover_path(current_user.username), class: "nav-link" do %>
                <i class="fs-2 fa-solid fa-magnifying-glass"></i>
              <% end %>
            </li>
            <li class="nav-item">
              <%= link_to user_path(current_user.username), class: "nav-link" do %>
                <i class="fs-2 fa-regular fa-circle-user"></i>
              <% end %>
            </li>
            <li class="nav-item">
              <%= link_to edit_user_registration_path, class: "nav-link" do %>
                <i class="fs-2 fa-solid fa-gear"></i>
              <% end %>
            </li>
          </ul>
        </div>
      </div>
    </nav>
  <% end %>
</body>
```
{: filename="app/views/layouts/application.html.erb" }

There's a lot going on here! Let's break it down:

1. **New Photo Modal**: A Bootstrap modal at the top of the page that contains a photo upload form. This is triggered by "Add photo" buttons throughout the interface.
2. **Flash Messages**: Success and danger alerts displayed at the top of the page.
3. **Mobile Top Navbar**: A simplified navbar that only shows on small screens (`d-sm-none d-block`).
4. **Left Sidebar**: A vertical navigation menu with links to Feed, Discover, Add photo, Profile, and Settings. The user's avatar and name are shown at the top in a dropdown that includes "Go to profile" and "Sign out" options.
5. **Center Column**: Where the `<%= yield %>` renders the page content, with borders on the sides for a clean look.
6. **Right Sidebar**: Contains a search form (using Ransack) and a floating action button for adding photos.
7. **Mobile Bottom Nav**: A fixed-bottom navigation bar for small screens.

### Mobile navbar partial

The mobile-only top navbar is very simple. Create `app/views/shared/_navbar.html.erb`:

```erb
<nav class="navbar fixed-top bg-body-tertiary d-sm-none d-block mb-5 mb-sm-0">
  <div class="container">
    <%= link_to root_path, class: "navbar-brand" do %>
      Photogram (Industrial)
      <i class="fa-regular fa-images"></i>
    <% end %>
    <% if current_user.present? %>
      <%= link_to new_photo_path, class: "nav-link" do %>
        <i class="fa-solid fa-pen-to-square"></i>
        Add photo
      <% end %>
    <% else %>
      <%= link_to "Sign in", new_user_session_path, class: "nav-link" %>
      <%= link_to "Sign up", new_user_registration_path, class: "nav-link" %>
    <% end %>
  </div>
</nav>
```
{: filename="app/views/shared/_navbar.html.erb" }

This navbar is hidden on medium and larger screens (`d-sm-none`), and only appears on mobile. It includes the app name and an "Add photo" link when signed in, or sign in/sign up links when not.

### Flash messages partial

Create `app/views/shared/_flash.html.erb`:

```erb
<div class="alert alert-<%= css_class %> alert-dismissible fade show mt-3" role="alert">
  <%= message %>
  <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
</div>
```
{: filename="app/views/shared/_flash.html.erb" }

This reusable partial takes a `message` and `css_class` and renders a dismissable Bootstrap alert.

## Force sign in

To require users to be signed in before they can access any page, add a `before_action` to the `ApplicationController`:

```ruby{1}
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
end
```
{: filename="app/controllers/application_controller.rb" }

This uses a Devise helper method, `authenticate_user!`, which will redirect unauthenticated users to the sign in page.

Now, if you try to visit any page without being signed in, you'll be redirected to `/users/sign_in` with a flash message: "You need to sign in or sign up before continuing."

## Configure permitted parameters

Since we added custom fields to our `User` model (`username`, `display_name`, `avatar_image`, `profile_banner`, `bio`, `website`, `private`), we need to tell Devise to allow those fields through in the sign up and account update forms:

```ruby
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?
  before_action :authenticate_user!

  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:display_name, :username])
    devise_parameter_sanitizer.permit(:account_update, keys: [:avatar_image, :bio, :display_name, :username, :private, :profile_banner, :remove_profile_banner, :website])
  end
end
```
{: filename="app/controllers/application_controller.rb" }

The `configure_permitted_parameters` method runs only when a Devise controller is handling the request (`if: :devise_controller?`). The `sign_up` sanitizer permits `display_name` and `username` — the fields we want on our sign up form. The `account_update` sanitizer permits all the profile fields that users should be able to edit.

## User search with Ransack

We added a search bar in the right sidebar that uses the [Ransack gem](https://github.com/activerecord-hackery/ransack) to search users by username. To make this work, we need to set up a `@q` variable in the `ApplicationController` that's available on every page:

```ruby{2,5-7}
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?
  before_action :authenticate_user!
  before_action :set_user_search, if: -> { current_user.present? }

  def set_user_search
    @q = User.all.ransack(params[:q])
  end

  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:display_name, :username])
    devise_parameter_sanitizer.permit(:account_update, keys: [:avatar_image, :bio, :display_name, :username, :private, :profile_banner, :remove_profile_banner, :website])
  end
end
```
{: filename="app/controllers/application_controller.rb" }

The `set_user_search` method creates a Ransack search object `@q` that can be used in the layout's search form. We only set it when the user is signed in (since the search bar isn't shown otherwise).

We also need to tell the `User` model which attributes can be searched:

```ruby
class User < ApplicationRecord
  # ...

  def self.ransackable_attributes(auth_object = nil)
    [ "username" ]
  end
end
```
{: filename="app/models/user.rb" }

This whitelists only the `username` attribute for searching, keeping the search secure.

## Routes

Now let's set up all the routes we need. Here is the complete routes file:

```ruby
Rails.application.routes.draw do
  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  root "users#feed"

  devise_for :users

  resources :comments
  resources :follow_requests
  resources :likes
  resources :photos do
    resources :comments, only: [:index]
    resources :likes, only: [:index]
  end
  resources :users, only: [ :index ]

  get ":username" => "users#show", as: :user
  get ":username/feed" => "users#feed", as: :feed
  get ":username/followers" => "users#followers", as: :followers
  get ":username/follows" => "users#follows", as: :follows
  get ":username/pending" => "users#pending", as: :pending
  get ":username/discover" => "users#discover", as: :discover
end
```
{: filename="config/routes.rb" }

Key things to note:

- The root points to `users#feed`, which shows the feed for the current user
- We have nested routes for comments and likes under photos (e.g., `/photos/1/likes` shows who liked a photo)
- The `/:username` routes use a wildcard segment to match usernames. These **must** come after all other routes, otherwise they would swallow routes like `/photos/new`
- We have dedicated pages for `followers`, `follows` (following), `pending` follow requests, and `discover`

Commit your progress!

## Controllers overview

Let's set up our controllers. The `UsersController` needs several actions:

```ruby
class UsersController < ApplicationController
  before_action :set_user, only: %i[ show feed discover follows followers pending ]

  def index
    @users = @q.result
  end

  def show
  end

  def feed
    @photos = @user.feed
  end

  def discover
    @photos = @user.discover
  end

  def follows
    @follows = @user.leaders
  end

  def followers
    @followers = @user.followers
  end

  def pending
    @pending = @user.pending_received_follow_requests
  end

  private

    def set_user
      if params[:username]
        @user = User.find_by!(username: params.fetch(:username))
      else
        @user = current_user
      end
    end
end
```
{: filename="app/controllers/users_controller.rb" }

The `set_user` method is key — when a `username` parameter is present, it finds that user. When the root route is visited (no username), it defaults to `current_user`. The `find_by!` raises a 404 error if the user doesn't exist.

The other controllers need updates too. For `CommentsController`, we need to auto-assign the author and redirect back:

```ruby{3,9}
  def create
    @comment = Comment.new(comment_params)
    @comment.author = current_user

    respond_to do |format|
      if @comment.save
        format.html { redirect_back fallback_location: root_path, notice: "Comment was successfully created." }
      else
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end
```
{: filename="app/controllers/comments_controller.rb" }

For `LikesController`, auto-assign the fan and redirect back:

```ruby{3,9,19}
  def create
    @like = Like.new(like_params)
    @like.fan = current_user

    respond_to do |format|
      if @like.save
        format.html { redirect_back fallback_location: @like.photo, notice: "Like was successfully created." }
      else
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @like.destroy
    respond_to do |format|
      format.html { redirect_back fallback_location: @like.photo, notice: "Like was successfully destroyed." }
    end
  end
```
{: filename="app/controllers/likes_controller.rb" }

For `PhotosController`, auto-assign the owner:

```ruby{3,18}
  def create
    @photo = Photo.new(photo_params)
    @photo.owner = current_user

    respond_to do |format|
      if @photo.save
        format.html { redirect_to @photo, notice: "Photo was successfully created." }
      else
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @photo.destroy
    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Photo was successfully destroyed." }
    end
  end
```
{: filename="app/controllers/photos_controller.rb" }

For `FollowRequestsController`, auto-assign the sender and auto-accept for public accounts:

```ruby{3-5,11}
  def create
    @follow_request = FollowRequest.new(follow_request_params)
    @follow_request.sender = current_user
    unless @follow_request.recipient.private?
      @follow_request.status = "accepted"
    end

    respond_to do |format|
      if @follow_request.save
        format.html { redirect_back fallback_location: root_url, notice: "Follow request was successfully created." }
      else
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end
```
{: filename="app/controllers/follow_requests_controller.rb" }

This is a nice touch — when following a public account, the follow request is automatically accepted. For private accounts, it remains as "pending" until the recipient accepts it.

Commit all your controller changes!

- Approximately how long (in minutes) did this lesson take you to complete?
{: .free_text_number #time_taken title="Time taken" points="1" answer="any" }

<!--

# List of project specs for AI assistant

require "rails_helper"

describe "/[USERNAME]/discover" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}/discover"

    expect(page.status_code).to be(200)
  end

  it "shows photos liked by people the current user follows", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev")
    owner = User.create(username: "owner", email: "owner@example.com", password: "appdev", private: false)
    photo = create_photo(owner: owner, caption: "owner caption")
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    Like.create(fan_id: leader.id, photo_id: photo.id)

    visit "/#{user.username}/discover"

    expect(page).to have_content(photo.caption)
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "/[USERNAME]/feed" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}/feed"

    expect(page.status_code).to be(200)
  end

  it "shows their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader, caption: "leader caption")
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/#{user.username}/feed"

    expect(page).to have_content(photo.caption)
    expect(page).to have_css("img")
  end

  it "allows them to like their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/#{user.username}/feed"

    click_on "0 likes"

    expect(page).to have_css("i.fa-solid.fa-heart")
  end

  it "allows them to un-like their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    Like.create(fan_id: user.id, photo_id: photo.id)

    visit "/#{user.username}/feed"

    click_on "1 like"

    expect(page).to have_css("i.fa-regular.fa-heart")
  end

  it "allows the user to add a comment on their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/#{user.username}/feed"

    fill_in "comment[body]", with: "New comment"
    click_button "Create Comment"

    expect(page).to have_content("New comment")
  end

  it "allows the user to delete their comment", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    comment = Comment.create(body: "New comment", author_id: user.id, photo_id: photo.id)

    visit "/#{user.username}/feed"

    within("#comment_#{comment.id}") do
      click_on "Delete"
    end

    expect(page).not_to have_content("New comment")
  end

  it "allows the user to edit their comment", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    comment = Comment.create(body: "New comment", author_id: user.id, photo_id: photo.id)

    visit "/#{user.username}/feed"

    within("#comment_#{comment.id}") do
      click_on "Edit"
    end

    fill_in "comment[body]", with: "Edited comment"
    click_button "Update Comment"

    expect(page).to have_content("Edited comment")
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "/" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page.status_code).to be(200)
  end

  it "has a bootstrap navbar", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to have_tag("nav", with: { class: "navbar" })
  end

  it "has a Settings link for the signed in user", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to have_link("Settings", href: "/users/edit")
  end

  it "does not have a sign in link if the user is already signed in", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to_not have_link("Sign in", href: "/users/sign_in")
  end

  it "has a link, 'Feed', that navigates to the 'Feed' page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Feed"

    expect(page).to have_current_path("/#{user.username}/feed")
  end

  it "has a link, 'Discover', that navigates to the 'Discover' page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Discover"

    expect(page).to have_current_path("/#{user.username}/discover")
  end

  it "has a link, 'Go to profile', that navigates to the profile page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Go to profile"

    expect(page).to have_current_path("/#{user.username}")
  end

  it "has an 'Add photo' button", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to have_button("Add photo")
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

require "rails_helper"

describe "User authentication" do
  it "displays a banner to sign in when trying to visit the homepage", points: 1 do
    visit "/"

    expect(page).to have_content("You need to sign in or sign up before continuing")
  end

  it "sends the user to the sign in page when trying to visit the homepage", points: 1 do
    visit "/"

    expect(page).to have_current_path("/users/sign_in")
  end

  it "allows new user sign ups", points: 1 do
    visit "/users/sign_up"

    fill_in "Email", with: "alice@example.com"
    fill_in "Password", with: "appdev"
    fill_in "Password confirmation", with: "appdev"
    fill_in "Username", with: "alice"
    click_button "Sign up"

    expect(page).to have_content("Welcome! You have signed up successfully")
  end

  it "allows an existing user to sign in", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")

    visit "/users/sign_in"

    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Sign in"

    expect(page).to have_content("Signed in successfully")
  end

  it "allows a user to sign out", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")

    visit "/users/sign_in"

    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Sign in"

    click_on user.username
    click_on "Sign out"

    expect(page).to have_current_path("/users/sign_in")
  end
end

require "rails_helper"

RSpec.describe Comment, type: :model do
  describe "has a belongs_to association defined called 'author' with Class name 'User'", points: 1 do
    it { should belong_to(:author).class_name("User") }
  end

  describe "has a belongs_to association defined called 'photo'", points: 1 do
    it { should belong_to(:photo) }
  end
end

require "rails_helper"

RSpec.describe FollowRequest, type: :model do
  describe "has a belongs_to association defined called 'sender' with Class name 'User'", points: 1 do
    it { should belong_to(:sender).class_name("User") }
  end

  describe "has a belongs_to association defined called 'recipient' with Class name 'User'", points: 1 do
    it { should belong_to(:recipient).class_name("User") }
  end
end

require "rails_helper"

RSpec.describe Like, type: :model do
  describe "has a belongs_to association defined called 'fan' with Class name 'User'", points: 1 do
    it { should belong_to(:fan).class_name("User") }
  end
end

RSpec.describe Like, type: :model do
  describe "has a belongs_to association defined called 'photo'", points: 1 do
    it { should belong_to(:photo) }
  end
end

require "rails_helper"

RSpec.describe Photo, type: :model do
  describe "has a belongs_to association defined called 'owner' with Class name 'User'", points: 1 do
    it { should belong_to(:owner).class_name("User") }
  end

  describe "has a has_many association defined called 'comments'", points: 1 do
    it { should have_many(:comments) }
  end

  describe "has a has_many association defined called 'likes'", points: 1 do
    it { should have_many(:likes) }
  end

  describe "has a has_many (many-to_many) association defined called 'fans' through 'likes'", points: 1 do
    it { should have_many(:fans).through(:likes) }
  end
end

require "rails_helper"


RSpec.describe User, type: :model do
  describe "has a has_many association defined called 'comments' with Class name 'Comment' and foreign key 'author_id'", points: 1 do
    it { should have_many(:comments).class_name("Comment").with_foreign_key("author_id") }
  end

  describe "has a has_many association defined called 'own_photos' with Class name 'Photo' and foreign key 'owner_id'", points: 1 do
    it { should have_many(:own_photos).class_name("Photo").with_foreign_key("owner_id") }
  end

  describe "has a has_many association defined called 'likes' with Class name 'Like' and foreign key 'fan_id'", points: 1 do
    it { should have_many(:likes).class_name("Like").with_foreign_key("fan_id") }
  end

  describe "has a has_many (many-to_many) association defined called 'liked_photos' through 'likes' and source 'photo'", points: 1 do
    it { should have_many(:liked_photos).through(:likes).source(:photo) }
  end

  describe "has a has_many association defined called 'sent_follow_requests' with Class name 'FollowRequest' and foreign key 'sender_id'", points: 1 do
    it { should have_many(:sent_follow_requests).class_name("FollowRequest").with_foreign_key("sender_id") }
  end

  describe "has a has_many association defined called 'received_follow_requests' with Class name 'FollowRequest' and foreign key 'recipient_id'", points: 1 do
    it { should have_many(:received_follow_requests).class_name("FollowRequest").with_foreign_key("recipient_id") }
  end

  describe "has a has_many association defined called 'accepted_sent_follow_requests' with scope where 'status' is \"accepted\"", points: 1 do
    it { should have_many(:accepted_sent_follow_requests).class_name("FollowRequest").with_foreign_key("sender_id").conditions(status: "accepted") }
  end

  describe "has a has_many association defined called 'accepted_received_follow_requests' with scope where 'status' is \"accepted\"", points: 1 do
    it { should have_many(:accepted_received_follow_requests).class_name("FollowRequest").with_foreign_key("recipient_id").conditions(status: "accepted") }
  end

  describe "has a has_many (many-to_many) association defined called 'followers' through 'accepted_received_follow_requests' and source 'sender'", points: 1 do
    it { should have_many(:followers).through(:accepted_received_follow_requests).source(:sender) }
  end

  describe "has a has_many (many-to_many) association defined called 'leaders' through 'accepted_sent_follow_requests' and source 'recipient'", points: 1 do
    it { should have_many(:leaders).through(:accepted_sent_follow_requests).source(:recipient) }
  end

  describe "has a has_many (many-to_many) association defined called 'feed' through 'leaders' and source 'own_photos'", points: 1 do
    it { should have_many(:feed).through(:leaders).source(:own_photos) }
  end

  describe "has a has_many (many-to_many) association defined called 'discover' through 'leaders' and source 'liked_photos'", points: 1 do
    it { should have_many(:discover).through(:leaders).source(:liked_photos) }
  end
end

-->
