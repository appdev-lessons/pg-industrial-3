# Photogram Industrial: Routes, layout, and controllers

## Getting started

Let's continue building our Photogram Industrial project. Here's the target we're working towards:

[pg-industrial.matchthetarget.com](https://pg-industrial.matchthetarget.com/)

Navigate to `github.com/codespaces` (or reopen the previous lesson and use the "Load assignment" button) and reopen your `photogram-industrial` project codespace to continue building on what you accomplished in the previous lessons.

At this point, you should have all five models (User, Photo, Comment, Like, FollowRequest) with their associations, validations, scopes, and sample data loaded. In this lesson, we'll shift our focus from the data layer to the interface layer: routes, the application layout, shared partials, and controllers.

Make sure your sample data is loaded before continuing:

```
rake sample_data
```

Since each lesson builds on the work from the previous one, we want to branch off of our last branch rather than `main`. First, check out the branch from the previous lesson:

```
git checkout models-and-associations
```

Now create a new branch from there:

```
git checkout -b routes-layout-controllers
```

## Setting up routes

The scaffold generators from the previous lessons already added some routes for us, and Devise added `devise_for :users`. But our application needs more than basic CRUD routes. We need vanity URL routes like `/:username` for user profiles, nested routes for viewing a photo's comments and likes, and dedicated pages for feed, discover, followers, and more.

Open `config/routes.rb`. Right now it probably looks something like this:

```ruby
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check
  root "users#feed"
  devise_for :users
  resources :likes
  resources :comments
  resources :follow_requests
  resources :photos
end
```
{: filename="config/routes.rb" }

Replace the entire file with this:

```ruby
Rails.application.routes.draw do
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

Let's walk through the key additions.

### Nested routes

```ruby
resources :photos do
  resources :comments, only: [:index]
  resources :likes, only: [:index]
end
```

This creates nested routes like `/photos/1/comments` and `/photos/1/likes`. These are useful for viewing all comments or likes on a specific photo. The `only: [:index]` restricts the nested routes to just the `index` action, since we don't need nested `new`, `create`, etc. because comments and likes are created through the standalone routes above.

### Users index

```ruby
resources :users, only: [ :index ]
```

This gives us a `/users` route that we'll use for search results. We're restricting it to `only: [:index]` because Devise handles the other user-related routes (sign up, sign in, edit profile).

### Vanity URL routes

```ruby
get ":username" => "users#show", as: :user
get ":username/feed" => "users#feed", as: :feed
get ":username/followers" => "users#followers", as: :followers
get ":username/follows" => "users#follows", as: :follows
get ":username/pending" => "users#pending", as: :pending
get ":username/discover" => "users#discover", as: :discover
```

These are the most interesting routes. Instead of `/users/42`, we use `/alice`, a vanity URL with the username right in the path. Each route captures the `:username` segment from the URL and passes it to the appropriate controller action.

<div class="alert alert-danger">

These vanity routes **must** come at the very end of the routes file. Because `:username` matches _any_ string, if these routes were listed first, a request to `/photos` would try to find a user with the username "photos" instead of hitting the photos resource. Rails processes routes top-to-bottom and stops at the first match, so more specific routes must come before these catch-all patterns.
</div>

The `as:` option on each route creates named path helpers. For example, `as: :user` gives us `user_path("alice")` which generates `/alice`, and `as: :feed` gives us `feed_path("alice")` which generates `/alice/feed`. We'll use these helpers extensively in our views.

Now would be a good time for a commit:

```
git add -A
git commit -m "Set up routes with nested resources and vanity URLs"
git push --set-upstream origin routes-layout-controllers
```

That last command publishes the branch to GitHub. From now on, you can push with just `git push` after each commit.

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## CDN assets partial

Before we build our layout, we need to bring in Bootstrap and Font Awesome. Instead of cluttering the layout file with CDN links, we'll put them in a shared partial.

Create a new directory and file at `app/views/shared/_cdn_assets.html.erb`:

```erb
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH" crossorigin="anonymous">

<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js" integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz" crossorigin="anonymous"></script>

<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.7.2/css/all.min.css" integrity="sha512-Evv84Mr4kqVGRNSgIGL/F/aIDqQb7xQ2vcrdIwxfjThSH8CSR7PBEakCr51Ck+w+/U6swU2Im1vVX0SVk9ABhg==" crossorigin="anonymous" referrerpolicy="no-referrer" />
```
{: filename="app/views/shared/_cdn_assets.html.erb" }

We're loading three things:
1. **Bootstrap CSS**: the grid system, utility classes, and component styles
2. **Bootstrap JS**: interactive components like modals, dropdowns, and dismissable alerts
3. **Font Awesome**: icons throughout the interface (hearts, houses, magnifying glasses, etc.)

<aside markdown="1">
Why use a CDN instead of bundling these assets locally? CDNs are fast. They're distributed globally, and your users' browsers may already have the files cached from visiting other sites that use the same CDN. For a project like this, CDN links are simple and effective.
</aside>

## Flash messages partial

When a user creates a photo, signs in, or performs other actions, Rails sets flash messages (`:notice` or `:alert`) that we want to display. Let's create a reusable partial for them.

Create `app/views/shared/_flash.html.erb`:

```erb
<div class="alert alert-<%= css_class %> alert-dismissible fade show" role="alert">
  <%= message %>
  <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
</div>
```
{: filename="app/views/shared/_flash.html.erb" }

This partial expects two local variables: `message` (the text to display) and `css_class` (either `"success"` for green or `"danger"` for red). The `alert-dismissible` class and close button let users dismiss the message. We'll render this partial from the layout like:

```erb
<%= render "shared/flash", message: notice, css_class: "success" %>
```

## Mobile navbar partial

Our layout uses a three-column design that works great on medium and large screens. But on small screens (phones), the sidebars disappear. We need a mobile-only top navbar to provide basic navigation on those screens.

Create `app/views/shared/_navbar.html.erb`:

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

The key Bootstrap classes here are `d-sm-none d-block`. This means: "display block by default, but hide on small screens and up." In practice, this navbar only appears on extra-small screens (phones). On tablets and desktops, the sidebar navigation takes over.

When the user is signed in, the navbar shows an "Add photo" link. When not signed in, it shows sign in and sign up links instead.

Now would be a good time for a commit:

```
git add -A
git commit -m "Created shared partials for CDN assets, flash messages, and navbar"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## Updating the photo form partial

The scaffold generated a `_form.html.erb` partial for photos, but it needs updating to work well in our layout, particularly inside the Bootstrap modal we're about to build. We need a file upload field for the image, a text area for the caption, and proper Bootstrap styling.

Open `app/views/photos/_form.html.erb` and replace its contents with:

```erb
<%= form_with(model: photo, class: "card-body") do |form| %>
  <% if photo.errors.any? %>
    <div id="error_explanation">
      <ul class="list-unstyled">
        <% photo.errors.each do |error| %>
          <li>
            <div class="alert alert-danger" role="alert">
              <%= error.full_message %>
            </div>
          </li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="form-group">
    <%= form.label :image, class: "visually-hidden" %>
    <% if photo.persisted? && photo.image.attached? %>
      <%= image_tag photo.image, class: "img-cover w-100 mb-2" %>
    <% end %>
    <%= form.file_field :image, class: "form-control", accept: "image/*" %>
  </div>

  <div class="form-group my-1">
    <%= form.label :caption, class: "visually-hidden" %>
    <%= form.text_area :caption, class: "form-control", placeholder: "What's going on?" %>
  </div>

  <div class="actions d-grid gap-2">
    <%= form.submit class: "btn btn-primary" %>
  </div>
<% end %>
```
{: filename="app/views/photos/_form.html.erb" }

A few things worth noting:

- `form_with(model: photo, class: "card-body")`: the form uses the `photo` local variable, which means it works for both new and existing photos. Rails automatically determines the correct URL and HTTP method.
- `accept: "image/*"` on the file field: this tells the browser to only show image files in the file picker dialog.
- `photo.persisted? && photo.image.attached?`: when editing an existing photo, we show the current image above the file field so the user can see what they're replacing.
- Labels are `visually-hidden`: we use placeholders instead for a cleaner look, but the labels are still present for accessibility (screen readers can read them).

Now would be a good time for a commit:

```
git add -A
git commit -m "Updated photo form partial with Bootstrap styling and file upload"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## The application layout

Now for the big one. The application layout is the HTML skeleton that wraps every page in our app. We're going to build a responsive three-column design:

- **Left sidebar**: Navigation links (visible on medium screens and up)
- **Center column**: The main page content (`<%= yield %>`)
- **Right sidebar**: Search bar and a floating "Add photo" button (visible on medium screens and up)
- **Mobile bottom nav**: A fixed-bottom navigation bar (visible only on small screens)

There's also a Bootstrap modal for adding photos that's available on every page when signed in.

Open `app/views/layouts/application.html.erb` and replace the entire file with:

```erb
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for(:title) || "Target: Photogram (Industrial)" %></title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="mobile-web-app-capable" content="yes">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= yield :head %>
    <link rel="icon" href="/icon.png" type="image/png">
    <link rel="icon" href="/icon.svg" type="image/svg+xml">
    <link rel="apple-touch-icon" href="/icon.png">
    <%= render "shared/cdn_assets" %>
    <%= stylesheet_link_tag :app, "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body>
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

    <%= render "shared/navbar" %>
    <div class="container mt-5">
      <div class="row">
        <div class="col-lg-2 offset-lg-1 col-md-2 d-none d-md-block">
          <% if current_user.present? %>
            <ul class="nav flex-column sticky-top">
              <li class="nav-item">
                <div class="dropdown">
                  <%= button_tag class: "btn btn-link nav-link border-bottom rounded-bottom-0 text-decoration-none",
                  data: { bs_toggle: "dropdown"}, aria: {expanded: false} do %>
                    <%= image_tag current_user.avatar_image, class: "rounded-circle img-small img-cover" %>
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
                  <span class="d-xl-inline d-none">
                    Feed
                  </span>
                <% end %>
              </li>
              <li class="nav-item">
                <%= link_to discover_path(current_user.username), class: "nav-link" do %>
                  <i class="fs-2 fa-solid fa-magnifying-glass"></i>
                  <span class="d-xl-inline d-none">
                    Discover
                  </span>
                <% end %>
              </li>
              <li class="nav-item">
                <%= button_tag class: "nav-link", data: {bs_toggle: "modal", bs_target: "#new_photo"} do %>
                  <i class="fs-2 fa-solid fa-pen-to-square"></i>
                  <span class="d-xl-inline d-none">
                    Add photo
                  </span>
                <% end %>
              </li>
              <li class="nav-item">
                <%= link_to user_path(current_user.username), class: "nav-link" do %>
                  <i class="fs-2 fa-regular fa-circle-user"></i>
                  <span class="d-xl-inline d-none">
                    Profile
                  </span>
                <% end %>
              </li>
              <li class="nav-item">
                <%= link_to edit_user_registration_path, class: "nav-link" do %>
                  <i class="fs-2 fa-solid fa-gear"></i>
                  <span class="d-xl-inline d-none">
                    Settings
                  </span>
                <% end %>
              </li>
            </ul>
          <% end %>
        </div>
        <div class="col-lg-6 col-md-8 col-12 <%= "border-start border-end" if current_user.present? %>">
            <%= yield %>
        </div>
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

    <% if current_user.present? %>
      <nav class="navbar fixed-bottom bg-body-tertiary d-block d-sm-none">
        <div class="container-fluid">
          <div class="w-100" id="footer">
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
</html>
```
{: filename="app/views/layouts/application.html.erb" }

This is a lot of code, so let's break it down section by section.

### The `<head>`

```erb
<title><%= content_for(:title) || "Target: Photogram (Industrial)" %></title>
<meta name="viewport" content="width=device-width,initial-scale=1">
```

The `content_for(:title)` allows individual pages to set a custom title. If none is set, it falls back to the default. The viewport meta tag is essential for responsive design. Without it, mobile browsers would render the page as if it were a desktop screen and then zoom out.

We render our CDN assets partial here in the `<head>`, along with the application stylesheet and JavaScript imports.

### The "New Photo" modal

```erb
<% if current_user.present? %>
  <div class="modal fade" id="new_photo" ...>
    ...
    <%= render "photos/form", photo: current_user.own_photos.build %>
    ...
  </div>
<% end %>
```

This is a [Bootstrap modal](https://getbootstrap.com/docs/5.3/components/modal/) — an overlay dialog that appears when triggered. We include it on _every_ page so that users can add a photo from anywhere in the app. The `current_user.own_photos.build` creates a new, unsaved Photo object associated with the current user, which the form partial uses to generate the correct form fields.

The modal is triggered by buttons with `data-bs-toggle="modal"` and `data-bs-target="#new_photo"`. You'll see those buttons in the left sidebar and the floating action button.

### Flash messages

```erb
<% if notice.present? || alert.present? %>
  <div class="container">
    <div class="row mb-2">
      <div class="col-md-6 offset-md-3">
        ...
      </div>
    </div>
  </div>
<% end %>
```

Flash messages are centered on the page using Bootstrap's grid offset. We only render this section if there's actually a flash message to show.

### The three-column layout

The main content area uses Bootstrap's grid system:

```erb
<div class="container mt-5">
  <div class="row">
    <div class="col-lg-2 offset-lg-1 col-md-2 d-none d-md-block">
      <!-- Left sidebar -->
    </div>
    <div class="col-lg-6 col-md-8 col-12">
      <%= yield %>
    </div>
    <div class="col-lg-3 col-md-2 d-none d-md-block">
      <!-- Right sidebar -->
    </div>
  </div>
</div>
```

The `d-none d-md-block` classes on both sidebars mean they're hidden on small screens and visible on medium screens and up. The center column takes the full width (`col-12`) on small screens, 8 columns on medium screens, and 6 columns on large screens. This makes the layout responsive without writing any custom CSS.

The `<%= yield %>` in the center column is where the content from each page's view template gets rendered.

### Left sidebar navigation

The left sidebar contains a vertical navigation menu wrapped in `sticky-top`, which keeps it visible as the user scrolls. Key elements:

- **User dropdown**: Shows the current user's avatar, display name, and username. Clicking it reveals a dropdown with "Go to profile" and "Sign out" options. The `button_to` for sign out uses `method: :delete` because Devise expects a DELETE request.
- **Navigation links**: Feed, Discover, Add photo, Profile, and Settings. Each uses a Font Awesome icon and a text label. The text labels use `d-xl-inline d-none` to only appear on extra-large screens. On medium and large screens, only the icons show, keeping the sidebar compact.
- **Add photo button**: Uses `button_tag` with Bootstrap modal data attributes instead of a link, since it opens the modal overlay rather than navigating to a new page.

### Right sidebar

The right sidebar has two elements:

1. **Search form**: Uses the Ransack gem's `search_form_for` helper with the `@q` search object (which we'll set up in `ApplicationController` shortly). The `:username_cont` search field means "username contains." Ransack generates the correct SQL `LIKE` query automatically. The form is hidden below extra-extra-large screens (`d-none d-xxl-block`).

2. **Floating action button**: A circular "Add photo" button fixed to the bottom-right corner of the screen. The `style="left: unset"` prevents Bootstrap's `fixed-bottom` from stretching it across the full width.

### Mobile bottom navigation

```erb
<nav class="navbar fixed-bottom bg-body-tertiary d-block d-sm-none">
```

This navigation bar only appears on extra-small screens (`d-block d-sm-none`). It uses `fixed-bottom` to stick to the bottom of the screen, just like Instagram's mobile app. It provides quick access to Feed, Discover, Profile, and Settings.

<aside markdown="1">
Notice how we use Bootstrap's responsive display utilities (`d-none`, `d-md-block`, `d-sm-none`, etc.) throughout the layout to show and hide elements at different screen sizes. This is much easier than writing custom media queries. The sidebars and bottom nav work together to provide navigation on every screen size.
</aside>

Now would be a good time for a commit:

```
git add -A
git commit -m "Built application layout with three-column design and responsive navigation"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## ApplicationController

Now let's set up the `ApplicationController`. This is the parent class of all our controllers, so anything we put here applies to every page in the app. We need three things:

1. **Authentication**: require sign-in to access any page
2. **Permitted parameters**: tell Devise which custom fields to allow
3. **Ransack search**: set up the search object for the layout's search bar

Open `app/controllers/application_controller.rb`. Right now it's empty:

```ruby
class ApplicationController < ActionController::Base
end
```
{: filename="app/controllers/application_controller.rb" }

Replace it with:

```ruby{2-4,6-8,10,12-14}
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

Let's walk through each piece.

### Requiring authentication

```ruby
before_action :authenticate_user!
```

This is a Devise helper method. It runs before every action in every controller. If the user isn't signed in, Devise redirects them to `/users/sign_in` with the flash message "You need to sign in or sign up before continuing." This is exactly what our tests expect.

### Configuring permitted parameters

```ruby
before_action :configure_permitted_parameters, if: :devise_controller?
```

The `if: :devise_controller?` condition means this `before_action` only runs when the request is being handled by one of Devise's built-in controllers (sign up, sign in, edit profile, etc.). It doesn't run for our custom controllers.

```ruby
def configure_permitted_parameters
  devise_parameter_sanitizer.permit(:sign_up, keys: [:display_name, :username])
  devise_parameter_sanitizer.permit(:account_update, keys: [:avatar_image, :bio, :display_name, :username, :private, :profile_banner, :remove_profile_banner, :website])
end
```

By default, Devise only permits `email`, `password`, and `password_confirmation`. Since we added custom columns to our User model, we need to explicitly tell Devise to allow them through. The `:sign_up` sanitizer controls which fields are accepted during registration, and the `:account_update` sanitizer controls which fields are accepted when editing a profile.

Notice that `:sign_up` only permits `:display_name` and `:username`, since we don't want users uploading avatars or setting bios during registration. Those are for the account update form. Also note `:remove_profile_banner` in the account update list. This is the virtual attribute we set up on the User model in the previous lesson for removing the banner image via a checkbox.

### Setting up Ransack search

```ruby
before_action :set_user_search, if: -> { current_user.present? }

def set_user_search
  @q = User.all.ransack(params[:q])
end
```

The `set_user_search` method creates a Ransack search object `@q` and stores it in an instance variable. Since this runs in `ApplicationController`, `@q` is available in _every_ view template, including the layout, where our search form lives. The `if: -> { current_user.present? }` condition skips this when no user is signed in (since the search bar isn't shown to signed-out users anyway).

When someone submits the search form, the params will include something like `q[username_cont]=ali`. Ransack parses this and generates a query like `WHERE username ILIKE '%ali%'`. The `_cont` suffix is a Ransack predicate meaning "contains."

<aside markdown="1">
Remember that we whitelisted `username` as a searchable attribute in the previous lesson with `ransackable_attributes`. Without that whitelist, Ransack would refuse to search at all. It's a security measure to prevent people from searching sensitive fields like `email` or `encrypted_password`.
</aside>

Now would be a good time for a commit:

```
git add -A
git commit -m "Configured ApplicationController with auth, permitted params, and Ransack search"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## Creating the UsersController

Unlike the other controllers which were generated by scaffolds, the `UsersController` is brand new. We need to create it from scratch because it has custom actions that don't follow the standard CRUD pattern.

Create `app/controllers/users_controller.rb`:

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

Let's walk through this controller action by action.

### The `set_user` before_action

```ruby
before_action :set_user, only: %i[ show feed discover follows followers pending ]

def set_user
  if params[:username]
    @user = User.find_by!(username: params.fetch(:username))
  else
    @user = current_user
  end
end
```

This is the most important method in the controller. It runs before `show`, `feed`, `discover`, `follows`, `followers`, and `pending`. When the URL contains a username (like `/alice/feed`), it finds that user in the database. When there's no username (like when visiting the root URL `/`, which routes to `users#feed`), it defaults to `current_user`.

The `find_by!` method (with a bang!) raises an `ActiveRecord::RecordNotFound` exception if no user is found, which Rails automatically converts to a 404 error page. This is better than `find_by` (without the bang), which would return `nil` and cause confusing `NoMethodError` crashes later in the action.

### The `index` action

```ruby
def index
  @users = @q.result
end
```

This is the search results page. Remember that `@q` is the Ransack search object set up in `ApplicationController`. Calling `.result` on it executes the search query and returns matching users. If no search params are present, it returns all users.

### Feed and Discover

```ruby
def feed
  @photos = @user.feed
end

def discover
  @photos = @user.discover
end
```

These actions leverage the `has_many :through` associations we built in the previous lesson. `@user.feed` returns all photos posted by people `@user` follows. `@user.discover` returns all photos liked by people `@user` follows. The heavy lifting is done by the model; the controller just passes the data to the view.

### Follows, Followers, and Pending

```ruby
def follows
  @follows = @user.leaders
end

def followers
  @followers = @user.followers
end

def pending
  @pending = @user.pending_received_follow_requests
end
```

These are straightforward. Each action uses an association we defined on the User model. `leaders` gives us the people `@user` follows, `followers` gives us the people who follow `@user`, and `pending_received_follow_requests` gives us follow requests that `@user` hasn't responded to yet.

<aside markdown="1">
Notice that we don't have a `before_action :set_user` on the `index` action. That's intentional. The search results page doesn't operate on a single user.
</aside>

Now would be a good time for a commit:

```
git add -A
git commit -m "Created UsersController with feed, discover, follows, followers, and pending actions"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## Customizing scaffold controllers

The scaffold generator gave us working controllers for Photos, Comments, Likes, and FollowRequests. But they need several modifications to work properly in our application:

1. **Auto-assigning the current user**: instead of relying on form fields for `owner_id`, `author_id`, `fan_id`, and `sender_id`, we should set these from `current_user` in the controller. This is both more secure (users can't fake someone else's identity) and more convenient.
2. **Redirect behavior**: some actions should redirect back to where the user came from rather than to the created/destroyed record.
3. **Business logic**: the FollowRequests controller needs to auto-accept requests for public accounts.

### PhotosController

Open `app/controllers/photos_controller.rb`. The scaffold generated a standard CRUD controller. We need to make a few changes from the scaffold:

**1. Auto-assign the owner in `create`:**

```ruby{3}
  # ...
  def create
    @photo = Photo.new(photo_params)
    @photo.owner = current_user

    respond_to do |format|
      # ...
```
{: filename="app/controllers/photos_controller.rb" }

Instead of accepting `owner_id` from the form (which could be tampered with), we set it from `current_user`. This is a security best practice: never trust user input for the identity of who's performing an action.

**2. Change `destroy` to redirect back:**

```ruby{4}
  def destroy
    @photo.destroy
    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Photo was successfully destroyed." }
      # ...
```
{: filename="app/controllers/photos_controller.rb" }

The scaffold redirected to `photos_url` (the index page). We use `redirect_back` instead, which sends the user back to wherever they came from (their feed, a profile page, etc.). The `fallback_location: root_url` is a safety net in case there's no referer header, it redirects to the homepage.

**3. Update `photo_params`:**

```ruby{2:(27-52)}
  def photo_params
    params.expect(photo: [:image, :pinned, :caption])
  end
```
{: filename="app/controllers/photos_controller.rb" }

We permit `:image`, `:pinned`, and `:caption`. Notice that `:owner_id` is _not_ in this list because we set that in the controller, not from form params.

### CommentsController

Open `app/controllers/comments_controller.rb`. We need to make a few changes from the scaffold:

**1. Auto-assign the author:**

```ruby{3}
  # ...
  def create
    @comment = Comment.new(comment_params)
    @comment.author = current_user

    respond_to do |format|
      # ...
```
{: filename="app/controllers/comments_controller.rb" }

Same pattern as photos: we set the author from `current_user` rather than trusting form input.

**2. Redirect back after create and destroy:**

Change the `create` action's redirect to go back to the previous page:

```ruby{3}
      if @comment.save
        format.html { redirect_back fallback_location: root_path, notice: "Comment was successfully created." }
        # ...
```
{: filename="app/controllers/comments_controller.rb" }

And do the same for `destroy`:

```ruby{4}
  def destroy
    @comment.destroy
    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Comment was successfully destroyed." }
      # ...
```
{: filename="app/controllers/comments_controller.rb" }

Comments are created and deleted inline on the feed/photo pages, so we want to redirect back to wherever the user was rather than navigating to a separate comments page.

**3. Redirect to root after update:**

```ruby{3}
      if @comment.update(comment_params)
        format.html { redirect_to root_url, notice: "Comment was successfully updated." }
        # ...
```
{: filename="app/controllers/comments_controller.rb" }

After editing a comment, we send the user back to the homepage.

### LikesController

Open `app/controllers/likes_controller.rb`. We need to make a few changes from the scaffold:

**1. Update `index` to find likes through a photo:**

```ruby{2-3}
  def index
    @photo = Photo.find(params[:photo_id])
    @likes = @photo.likes
  end
```
{: filename="app/controllers/likes_controller.rb" }

The `index` action now expects a `photo_id` parameter (from the nested route `/photos/:photo_id/likes`) and shows the likes for that specific photo.

**2. Auto-assign the fan:**

```ruby{3}
  # ...
  def create
    @like = Like.new(like_params)
    @like.fan = current_user

    respond_to do |format|
      # ...
```
{: filename="app/controllers/likes_controller.rb" }

Same pattern: the current user is automatically set as the fan.

**3. Redirect back to the photo after create and destroy:**

```ruby{3}
      if @like.save
        format.html { redirect_back fallback_location: @like.photo, notice: "Like was successfully created." }
        # ...
```
{: filename="app/controllers/likes_controller.rb" }

And similarly for `destroy`:

```ruby{4}
  def destroy
    @like.destroy
    respond_to do |format|
      format.html { redirect_back fallback_location: @like.photo, notice: "Like was successfully destroyed." }
      # ...
```
{: filename="app/controllers/likes_controller.rb" }

After liking or unliking a photo, the user stays on the same page. The `fallback_location` is `@like.photo`, the photo that was liked.

### FollowRequestsController

Open `app/controllers/follow_requests_controller.rb`. We need to make a few changes from the scaffold:

**1. Auto-assign the sender and auto-accept for public accounts:**

```ruby{3-5}
  # ...
  def create
    @follow_request = FollowRequest.new(follow_request_params)
    @follow_request.sender = current_user
    unless @follow_request.recipient.private?
      @follow_request.status = "accepted"
    end

    respond_to do |format|
      # ...
```
{: filename="app/controllers/follow_requests_controller.rb" }

This is the most interesting controller logic. First, we set the `sender` to `current_user`. Then we check if the recipient's account is public (`private?` returns `false`). If it is public, we automatically accept the follow request, so the user doesn't need to wait for approval. If the account is private, the status stays as `"pending"` (the default we set in the migration), and the recipient will need to accept it manually.

**2. Redirect back after create, update, and destroy:**

Change all three actions to use `redirect_back`:

```ruby{3}
      if @follow_request.save
        format.html { redirect_back fallback_location: root_url, notice: "Follow request was successfully created." }
        # ...
```
{: filename="app/controllers/follow_requests_controller.rb" }

All three actions use `redirect_back fallback_location: root_url`. Follow/unfollow actions happen from profile pages, the discover page, , so we always want to send the user back to where they were.

Now would be a good time for a commit:

```
git add -A
git commit -m "Customized scaffold controllers with auto-assignment, redirect_back, and follow logic"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## Verify your progress

At this point, you should have:

1. Routes set up with nested resources, vanity URLs, and named path helpers.
2. Shared partials for CDN assets, flash messages, and the mobile navbar.
3. A complete application layout with a three-column responsive design.
4. `ApplicationController` configured with authentication, Devise permitted parameters, and Ransack search.
5. A hand-built `UsersController` with feed, discover, followers, follows, and pending actions.
6. All four scaffold controllers customized with `current_user` auto-assignment and proper redirects.

Try starting your server with `bin/server` and visiting `/`. You should be redirected to the sign-in page. Sign in with one of the sample data accounts (e.g., `alice@example.com` / `appdev`) and you should see the three-column layout with navigation links in the left sidebar.

<div class="alert alert-info">

The views won't look complete yet because we haven't built the feed, discover, profile, or other view templates. You may see errors when clicking through to pages like Feed or Discover because the view files don't exist yet. That's expected! We'll build all of those views in the next lesson.
</div>

Now would be a good time for a final commit and push:

```
git add -A
git commit -m "Completed routes, layout, and controllers"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

In the next lesson, we'll build out all of the view templates: the feed, discover page, user profiles, photo detail pages, and more.

---

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
