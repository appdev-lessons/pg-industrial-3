# Photogram Industrial: Routes, layout, and controllers

## Getting started

Let's continue building our Photogram Industrial project. Here's the target we're working towards:

[pg-industrial.matchthetarget.com](https://pg-industrial.matchthetarget.com/)

Navigate to `github.com/codespaces` (or reopen the previous lesson and use the "Load assignment" button) and reopen your `pg-industrial` project codespace to continue building on what you accomplished in the previous lessons.

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

If you need a reference while you work, you can visit [my pull request](#my-pull-request){: target="_self" } below. Individual commits are also linked throughout the lesson for convenience.

## Setting up routes

The scaffold generators from the previous lessons already added some routes for us, and Devise added `devise_for :users`. But our application needs more than basic CRUD routes. We need vanity URL routes like `/:username` for user profiles, nested routes for viewing a photo's comments and likes, and dedicated pages for feed, discover, followers, and more.

<div class="alert alert-info">
This is the last time we'll remind you:

Are you still navigating manually through the file tree and clicking to open everything? That's going to become very painful, very quickly. One of the biggest things you can do to increase your productivity is navigating your codebase and its dozens of files without your mouse.

Stop now and experiment with [jumping to files](/lessons/194-helper-methods-part-3#partials-shine-along-with-jump-to-file) in the VSCode fuzzy search bar.
</div>

Open `config/routes.rb`. Right now it probably looks something like this:

```ruby
Rails.application.routes.draw do
  resources :likes
  resources :follow_requests
  resources :comments
  resources :photos
  devise_for :users
  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  root "users#feed"
end
```
{: filename="config/routes.rb" }

We haven't touched it much, besides adding that `root "users#feed` for Devise, and it looks like our generators have been filling it up for us! Great! But let's make some modifications.

Replace the entire file with this, paying particular attention to the key highlighted additions:

```ruby{12-15,17,19-24}
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

  resources :users, only: [:index]

  get ":username" => "users#show", as: :user
  get ":username/feed" => "users#feed", as: :feed
  get ":username/followers" => "users#followers", as: :followers
  get ":username/follows" => "users#follows", as: :follows
  get ":username/pending" => "users#pending", as: :pending
  get ":username/discover" => "users#discover", as: :discover
end
```
{: filename="config/routes.rb" }

There's some re-ordering of things, which is mostly cosmetic to keep things visually clear (e.g. `resources` are now top-down alphabetically: comments, follow requests, likes, photos).

But, let's walk through some of the unfamiliar key additions.

### Nested routes

Previously, the photos resource line was:

```ruby
resources :photos
```

which we know from previous lessons gives us the "seven golden" RESTful routes: index, new, create, show, edit, update, and destroy. From just that line we get `"/photos"`, `"/photos/:id"`, etc., and named helper methods like `photos_path` and `photos_url`.

Our edit added a `do` block and _nested_ two resource paths under the photos resource:

```ruby{1:(18-20),2-4}
resources :photos do
  resources :comments, only: [:index]
  resources :likes, only: [:index]
end
```

This creates [nested routes](https://guides.rubyonrails.org/routing.html#nested-resources). Think about what happens when a user clicks "View all comments" on a photo. We need a URL that identifies _which_ photo's comments to show. That's exactly what nested routes give us: URLs like `/photos/42/comments` and `/photos/42/likes`, where the photo's ID is baked right into the path.

Specifically, the two nested lines above generate:

| HTTP verb | Path | Controller#Action | Named helper |
|---|---|---|---|
| GET | `/photos/:photo_id/comments` | `comments#index` | `photo_comments_path(photo)` |
| GET | `/photos/:photo_id/likes` | `likes#index` | `photo_likes_path(photo)` |
{: .bleed-full }

(You can run the command `rails routes` at the terminal to find those two new routes.)

Notice the `:photo_id` segment in each path. When a request comes in to `/photos/42/comments`, Rails sets `params[:photo_id]` to `42`. In the controller, we'll use that to scope our query (something like `Photo.find(params[:photo_id]).comments`) so we only fetch comments belonging to that specific photo.

You might be wondering: we already have `resources :comments` and `resources :likes` as standalone routes earlier in the file. Why do we need nested versions too? The standalone routes handle creating, editing, and deleting an individual comment or like, where the comment's own ID is enough. The nested routes serve a different purpose: _listing_ all comments (or likes) that belong to a specific photo. That's why we use `only: [:index]` to restrict the nested routes to just the `index` action, since the standalone routes already cover everything else.

### Users index

```ruby
resources :users, only: [:index]
```

This gives us a `/users` route that we'll use for search results. We're restricting it to `only: [:index]` because Devise handles the other user-related routes (sign up, sign in, edit profile).

### Vanity URL routes

Instead of `/users/42`, we use `/alice`, a vanity URL with the username right in the path:

```ruby
get ":username" => "users#show", as: :user
get ":username/feed" => "users#feed", as: :feed
get ":username/followers" => "users#followers", as: :followers
get ":username/follows" => "users#follows", as: :follows
get ":username/pending" => "users#pending", as: :pending
get ":username/discover" => "users#discover", as: :discover
```

Each route captures the `:username` segment from the URL and passes it to the appropriate controller action.

We don't have a `UsersController` with the index, show, feed, etc. actions yet, nor the corresponding view templates. We'll make all of those later to handle these routes we've set up.

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

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/c9578bc039678c65c0783cbdd45778653642dc2c)

### Open and submit your pull request

Now that you've pushed your branch to GitHub, it's time to open a **pull request** (PR). A pull request lets us review your code and leave line-by-line feedback.

<div class="alert alert-info">

[Here is a short video demonstration of the process.](https://share.descript.com/view/RLP4apAu5pp) You should also carefully read the notes below!
</div>

On GitHub, navigate to your `pg-industrial` repository. You should see a prompt to open a pull request for the `routes-layout-controllers` branch that you just published, or you can go to the "Pull requests" tab and click "New pull request."

Make sure the **base** branch is `models-and-associations` and the **compare** branch is `routes-layout-controllers`. Also be sure to change the base _repository_ from `appdev-projects/pg-industrial` to _your_ fork. Give it a title, then click "Create pull request."

Your PR URL should look like:

```
github.com/[YOUR_GITHUB_USERNAME]/pg-industrial/pull/X
```

It should _not_ contain `appdev-projects` in the URL. If it does, you submitted a PR to _our_ repo instead of to _your own fork_.

<aside>
You don't need to merge your branches now, but when you're ready: [see the notes in our Git CLI lesson](/lessons/196-git-cli#merging-branches).
</aside>

Any additional commits you push to this branch will automatically show up on the pull request. As you work through the rest of this lesson, you can periodically check the PR diff on GitHub in the "Files changed" tab to get a comprehensive overview of everything that has changed so far.

Submit your pull request URL:

- `routes-layout-controllers` compared to `models-and-associations`:
- github.com
  - Great job!
- any
  - Not quite. Make sure the URL looks like: `github.com/[YOUR_GITHUB_USERNAME]/pg-industrial/pull/X`
{: .free_text #pr_url title="Pull request URL" points="1" answer="1" }

## ApplicationController: authentication

Before we build the interface, let's require sign-in like the target does. Visit [the target](https://pg-industrial.matchthetarget.com/). If you're signed out, you're immediately redirected to a sign-in page when you try and do anything. Let's set that up.

Open `app/controllers/application_controller.rb`, and add Devise's built-in `authenticate_user!` method as a `before_action`:

```ruby{2}
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
end
```
{: filename="app/controllers/application_controller.rb" }

This runs before every action in every controller. If the user isn't signed in, Devise redirects them to `/users/sign_in` with the flash message "You need to sign in or sign up before continuing."

### Stubbing the feed action

Our `root` route points to `users#feed`, but we don't have a `UsersController` yet. If we try to visit `/` right now, we'll get an error. Let's stub out just enough to make the root page load.

Create `app/controllers/users_controller.rb`:

```ruby
class UsersController < ApplicationController
  def feed
  end
end
```
{: filename="app/controllers/users_controller.rb" }

And create a minimal view template at `app/views/users/feed.html.erb`:

```erb
<h1>Feed</h1>
<p>Coming soon!</p>
```
{: filename="app/views/users/feed.html.erb" }

<div class="alert alert-success">

**CHECK**: Start your server with `bin/server` and visit `/`. You should be redirected to the sign-in page. Sign in with `alice@example.com` / `appdev`. You should see the stub feed page with "Feed" and "Coming soon!" The root route works!
</div>

Now would be a good time for a commit:

```
git add -A
git commit -m "Added authentication and stubbed feed action"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/aff354a721546d23ba5e41a22578dd5e82de6bcc)

## CDN assets partial

The scaffold pages work, but they look plain. Visit `/photos/new` in your `bin/server` live app preview. It's a bare HTML form. Let's add Bootstrap and Font Awesome to style things up.

Instead of cluttering the layout file with CDN links, we'll put them in a shared partial. Create a new directory and file at `app/views/shared/_cdn_assets.html.erb`:

```erb
<!-- Connect Bootstrap CSS -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.1/dist/css/bootstrap.min.css">

<!-- Connect Bootstrap JavaScript and its dependencies -->
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.1/dist/js/bootstrap.bundle.min.js"></script>

<!-- Connect Font Awesome -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.1.1/js/all.min.js"></script>
```
{: filename="app/views/shared/_cdn_assets.html.erb" copyable }

We're loading three things:
1. **Bootstrap CSS**: the grid system, utility classes, and component styles
2. **Bootstrap JS**: interactive components like modals, dropdowns, and dismissable alerts
3. **Font Awesome**: icons throughout the interface (hearts, houses, magnifying glasses, etc.)

<aside>
Why use a CDN instead of bundling these assets locally? We've chosen to use CDNs for now since they are quick and easy to understand. Later we'll discuss asset bundling to enhance performance for our end users.
</aside>

Now render it in the layout's `<head>`. Open `app/views/layouts/application.html.erb` and add the highlighted partial line:

```erb{9}
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for(:title) || "Photogram (Industrial)" %></title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <!-- ... -->
    <link rel="apple-touch-icon" href="/icon.png">

    <%= render "shared/cdn_assets" %>

    <%# Includes all stylesheet files in app/assets/stylesheets %>
    <%= stylesheet_link_tag :app, "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>
<!-- ... -->
```
{: filename="app/views/layouts/application.html.erb" }

<aside>

The `content_for(:title)` allows individual pages to set a custom title (you'll notice the scaffold templates already include `content_for :title` calls). If none is set, it falls back to "Photogram (Industrial)." The viewport meta tag is essential for responsive design. Without it, mobile browsers would render the page as if it were a desktop screen and then zoom out.
</aside>

<div class="alert alert-success">

**CHECK**: Refresh your live app preview. The page should now have Bootstrap styling: the font changes and the form inputs look cleaner.
</div>

## Flash messages partial

When you signed in, the flash message "Signed in successfully." did not appear. Let's add flash messages and style them as a Bootstrap alert with a dismiss button.

Create `app/views/shared/_flash.html.erb`:

```erb
<div class="alert alert-<%= css_class %> alert-dismissible fade show" role="alert">
  <%= message %>
  <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
</div>
```
{: filename="app/views/shared/_flash.html.erb" }

This partial expects two local variables: `message` (the text to display) and `css_class` (either `"success"` for green or `"danger"` for red). The [`alert-dismissible`](https://getbootstrap.com/docs/5.3/components/alerts/#dismissing) class and close button let users dismiss the message.

Now add flash message rendering to the layout body. Open `app/views/layouts/application.html.erb` and add the highlighted lines between `<body>` and `<%= yield %>`:

```erb{3-16}
  <!-- ... -->
  <body>
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

    <%= yield %>
  </body>
</html>
```
{: filename="app/views/layouts/application.html.erb" }

We only render this section if there's actually a flash message to show.

<div class="alert alert-success">

**CHECK**: Since we haven't built a sign out link yet, you can run `rake sample_data` to reset your database and sign out. Then sign back in your live app preview. You should see a styled green alert that says "Signed in successfully." with a dismiss button (×). Click the × to dismiss it.
</div>

Now would be a good time for a commit:

```
git add -A
git commit -m "Created shared partials for CDN assets and flash messages"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/abf0a1c0058d84ef9be40febc2a4af57c51c0bf4)

## Updating the photo form partial

Visit `/photos/new` in your live app preview. The scaffold form has fields for Owner ID, Comments count, Likes count, and Pinned, none of which a user should need to fill in! Let's clean it up to just an image upload and a caption.

Open `app/views/photos/_form.html.erb`. The scaffold generated a basic form, but we need to replace its contents piece by piece.

First, replace the opening `form_with` tag and add an error display block:

```erb{1,2-12}
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
  <!-- ... -->
<% end %>
```
{: filename="app/views/photos/_form.html.erb" }

The `form_with(model: photo, class: "card-body")` uses the `photo` local variable, which means it works for both new and existing photos. Rails automatically determines the correct URL and HTTP method.

Next, replace the current image text field:

```erb
  <div>
    <%= form.label :image, style: "display: block" %>
    <%= form.text_field :image %>
  </div>
```

with an actual image upload field after the error block:

```erb{6-12}
  <!-- ... -->
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

  <div>
    <%= form.label :caption, style: "display: block" %>
    <%= form.textarea :caption %>
  </div>
    <!-- ... -->
```
{: filename="app/views/photos/_form.html.erb" }

The `accept: "image/*"` tells the browser to only show image files in the file picker dialog. The `photo.persisted? && photo.image.attached?` check means that when editing an existing photo, we show the current image above the file field so the user can see what they're replacing.

Finally, replace the remaining inputs with just a caption field and submit button:

(We're removing the owner ID, comments count, likes count, and pinned fields. These are not things a user should need to fill out when they create a new photo.)

```erb{5-12}
  <!-- ... -->
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

The labels are `visually-hidden` throughout the form. We use placeholders instead for a cleaner look, but the labels are still present for accessibility (screen readers can read them).

<div class="alert alert-success">

**CHECK**: Visit `/photos/new`. You should see a clean form with just an image upload field and a caption text area. No more Owner ID, Comments count, etc.
</div>

## Customizing scaffold controllers

Try creating a photo from `/photos/new`. Upload an image and add a caption. 

You should get hit with a "Owner must exist." validation error! (Good thing we did all of that backend work in the earlier lessons to keep out bad data.)

The owner isn't set automatically because the scaffold expects an `owner_id` from the form, which we just removed. Let's fix all four scaffold controllers to auto-assign `current_user` and redirect sensibly.

### PhotosController

Open `app/controllers/photos_controller.rb`. We need to make a few changes from the scaffold:

**1. Auto-assign the owner in `create`:**

```ruby{4}
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

```ruby{5}
  # ...
  def destroy
    @photo.destroy
    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Photo was successfully destroyed." }
      # ...
```
{: filename="app/controllers/photos_controller.rb" }

The scaffold redirected to `photos_path` (the index page). We use `redirect_back` instead, which sends the user back to wherever they came from (their feed, a profile page, etc.). The `fallback_location: root_url` is a safety net in case there's no referer header, redirecting to the homepage.

**3. Update `photo_params`:**

```ruby{3:(28-52)}
  # ...
  def photo_params
    params.expect(photo: [ :image, :caption, :pinned ])
  end
  # ...
```
{: filename="app/controllers/photos_controller.rb" }

We permit `:image`, `:pinned`, and `:caption`. Notice that `:owner_id` is _not_ in this list because we set that in the controller, not from form params. We also dropped the `:comments_count` and `:likes_count`, which will be auto incremented on our backend.

<div class="alert alert-success">

**CHECK**: Visit `/photos/new`. Upload an image and type a caption, then click "Create Photo." The photo should be created successfully with the owner automatically assigned.
</div>

Now that we've seen the pattern, let's make these updates to our other controllers now.

### CommentsController

Open `app/controllers/comments_controller.rb`. We need to make a few changes from the scaffold:

**1. Auto-assign the author:**

```ruby{4}
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

```ruby{8}
  # ...
  def create
    @comment = Comment.new(comment_params)
    @comment.author = current_user

    respond_to do |format|
      if @comment.save
        format.html { redirect_back fallback_location: root_path, notice: "Comment was successfully created." }
        # ...
```
{: filename="app/controllers/comments_controller.rb" }

And do the same for `destroy`:

```ruby{5}
  # ...
  def destroy
    @comment.destroy
    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Comment was successfully destroyed." }
      # ...
```
{: filename="app/controllers/comments_controller.rb" }

Comments are created and deleted inline on the feed/photo pages, so we want to redirect back to wherever the user was rather than navigating to a separate comments page.

**3. Redirect to root after update:**

```ruby{5}
  # ...
  def update
    respond_to do |format|
      if @comment.update(comment_params)
        format.html { redirect_to root_url, notice: "Comment was successfully updated." }        
        # ...
```
{: filename="app/controllers/comments_controller.rb" }

After editing a comment, we send the user back to the homepage.

### LikesController

Open `app/controllers/likes_controller.rb`. We need to make a few changes from the scaffold:

**1. Update `index` to find likes through a photo:**

```ruby{3-4}
  # ...
  def index
    @photo = Photo.find(params[:photo_id])
    @likes = @photo.likes
  end
  # ...
```
{: filename="app/controllers/likes_controller.rb" }

The `index` action now expects a `photo_id` parameter (from the nested route `/photos/:photo_id/likes`) and shows the likes for that specific photo.

**2. Auto-assign the fan:**

```ruby{4}
  # ...
  def create
    @like = Like.new(like_params)
    @like.fan = current_user

    respond_to do |format|
      # ...
```
{: filename="app/controllers/likes_controller.rb" }

Same pattern: the current user is automatically set as the fan.

**3. Redirect back after create and destroy:**

```ruby{8}
  # ...
  def create
    @like = Like.new(like_params)
    @like.fan = current_user

    respond_to do |format|
      if @like.save
        format.html { redirect_back fallback_location: @like.photo, notice: "Like was successfully created." }
        # ...
```
{: filename="app/controllers/likes_controller.rb" }

And similarly for `destroy`:

```ruby{5}
  # ...
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

```ruby{4-7}
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

Interesting! 

First, we set the `sender` to `current_user`. Then we check if the recipient's account is public. (Since User has a `private` boolean column we get a method `private?` for free, which returns `false` if the column is `false` and `true` if the column is `true`.)

If it is public, we automatically accept the follow request, so the user doesn't need to wait for approval. If the account is private, the status stays as `"pending"` (the default we set in the migration), and the recipient will need to accept it manually.

**2. Redirect back after create, update, and destroy:**

Change all three create, update, and destroy actions to use `redirect_back`:

```ruby{3:(23-63)}
        # ...
      if @follow_request.save
        format.html { redirect_back fallback_location: root_url, notice: "Follow request was successfully created." }
        # ...
```
{: filename="app/controllers/follow_requests_controller.rb" }

All three actions use `redirect_back fallback_location: root_url`. Follow/unfollow actions happen from profile pages and the discover page, so we always want to send the user back to where they were.

---

That was a lot of careful work. Let's make a commit:

```
git add -A
git commit -m "Updated photo form and customized scaffold controllers"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/42aedd156f9adff8ae329d203d8081456b8300d7)

## Three-column grid structure

Open [the target](https://pg-industrial.matchthetarget.com) and observe: navigation on the left, content in the center, search on the right. Let's build that responsive grid in our layout.

Open `app/views/layouts/application.html.erb`. We're going to wrap the `<%= yield %>` in a [Bootstrap grid](https://getbootstrap.com/docs/5.3/layout/grid/). Replace the `<%= yield %>` in the body with:

```erb{3-16}
    <!-- ... -->
    <% end %>

    <div class="container mt-5">
      <div class="row">
        <div class="col-lg-2 offset-lg-1 col-md-2 d-none d-md-block">
          <!-- Left sidebar (coming soon) -->
        </div>
        <div class="col-lg-6 col-md-8 col-12 <%= "border-start border-end" if current_user.present? %>">
            <%= yield %>
        </div>
        <div class="col-lg-3 col-md-2 d-none d-md-block">
          <!-- Right sidebar (coming soon) -->
        </div>
      </div>
    </div>
  </body>
</html>
```
{: filename="app/views/layouts/application.html.erb" }

The `d-none d-md-block` classes on both sidebars mean they're hidden on small screens and visible on medium screens and up. The center column takes the full width (`col-12`) on small screens, 8 columns on medium screens, and 6 columns on large screens. This makes the layout responsive without writing any custom CSS.

The `<%= yield %>` in the center column is where the content from each page's view template gets rendered.

<div class="alert alert-success">

**CHECK**: Visit any page in your live app preview. The page content should now be centered in a middle column with space on each side. Try resizing your browser. The sidebars disappear on narrow screens.
</div>

## Left sidebar navigation

The target has navigation links on the left: Feed, Discover, Profile, Settings. Let's extract this into a partial to keep the layout file clean.

Create `app/views/shared/_left_sidebar.html.erb`:

```erb
<% if current_user.present? %>
  <ul class="nav flex-column sticky-top">
    <li class="nav-item">
      <div class="dropdown">
        <%= button_tag class: "btn btn-link nav-link border-bottom rounded-bottom-0 text-decoration-none",
        data: { bs_toggle: "dropdown"}, aria: {expanded: false} do %>
          <%= image_tag current_user.avatar_image, class: "rounded-circle img-small img-cover" %>
          <span class="d-xl-inline d-none">
            <%= current_user.display_name %>
          </span>
        <% end %>
        <ul class="dropdown-menu">
          <li>
            <%= link_to user_path(current_user.username), class: "dropdown-item" do %>
              Go to profile
            <% end %>
          </li>
          <li>
            <%= button_to "Sign out", destroy_user_session_path, method: :delete, class: "dropdown-item" %>
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
        <i class="fs-2 fa-solid fa-compass"></i>
        <span class="d-xl-inline d-none">Discover</span>
      <% end %>
    </li>
    <li class="nav-item">
      <%= link_to user_path(current_user.username), class: "nav-link" do %>
        <i class="fs-2 fa-solid fa-user"></i>
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
```
{: filename="app/views/shared/_left_sidebar.html.erb" copyable }

Now render this partial in the layout. Open `app/views/layouts/application.html.erb` and replace the comment:

```erb
<!-- Left sidebar (coming soon) -->`
```

with:

```erb{3}
        <!-- ... -->
        <div class="col-lg-2 offset-lg-1 col-md-2 d-none d-md-block">
          <%= render "shared/left_sidebar" %>
        </div>
        <!-- ... -->
```
{: filename="app/views/layouts/application.html.erb" }

Key elements:

- **User [dropdown](https://getbootstrap.com/docs/5.3/components/dropdowns/)**: Shows the current user's avatar and display name. Clicking it reveals a dropdown with "Go to profile" and "Sign out" options. The `button_to` for sign out uses `method: :delete` because Devise expects a DELETE request.
- **Navigation links**: Feed, Discover, Profile, and Settings. Each uses a Font Awesome icon and a text label. The text labels use `d-xl-inline d-none` to only appear on extra-large screens. On medium and large screens, only the icons show, keeping the sidebar compact.

<div class="alert alert-success">

**CHECK**: Refresh your live app preview. You should see navigation links in the left sidebar. Click "Settings" → it goes to the Devise edit page. Click your username in the sidebar → see "Go to profile" and "Sign out" options. Click "Sign out" and verify it works!

(Don't worry that "Feed" and "Profile" error when clicked. We haven't built those controller actions yet. We'll get there.)
</div>

## New Photo modal

In [the target](https://pg-industrial.matchthetarget.com), clicking "Add photo" opens a dialog overlay instead of navigating to `/photos/new`. Let's add a [Bootstrap modal](https://getbootstrap.com/docs/5.3/components/modal/).

Create `app/views/shared/_photo_modal.html.erb`:

```erb
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
```
{: filename="app/views/shared/_photo_modal.html.erb" copyable }

Let's zoom in on the `build` part of the render line:

```erb{1:(34-62)}
<%= render "photos/form", photo: current_user.own_photos.build %>
```

- `current_user.own_photos`: this uses the `has_many :own_photos` association on `User` to scope us down to only the photos that belong to the currently signed in user.
- `.build`: this is an ActiveRecord method that creates a **new `Photo` object in memory** (like `Photo.new`), but with a bonus: it automatically sets the `owner_id` foreign key to `current_user.id` for us, because we called it through the association.

The result is an unsaved `Photo` instance with `owner_id` already set, ready to be wrapped in a form and submitted. Think of `.build` as saying _"prepare a new photo **for this owner**"_ without saving it to the database yet.

Now render this partial in the layout, right after the opening `<body>` tag, before the flash messages:

```erb{3}
  <!-- ... -->
  <body>
    <%= render "shared/photo_modal" %>

    <% if notice.present? || alert.present? %>
    <!-- ... -->
```
{: filename="app/views/layouts/application.html.erb" }

This overlay dialog is included on _every_ page so that signed-in users can add a photo from anywhere in the app. The `current_user.own_photos.build` creates a new, unsaved Photo object associated with the current user, which the form partial uses to generate the correct form fields.

Now add an "Add photo" button to the left sidebar navigation. Open `app/views/shared/_left_sidebar.html.erb` and add this nav item between the Discover and Profile links:

```erb{8-13}
    <!-- ... -->
    <li class="nav-item">
      <%= link_to discover_path(current_user.username), class: "nav-link" do %>
        <i class="fs-2 fa-solid fa-compass"></i>
        <span class="d-xl-inline d-none">Discover</span>
      <% end %>
    </li>
    <li class="nav-item">
      <%= button_tag class: "nav-link btn btn-link", data: {bs_toggle: "modal", bs_target: "#new_photo"} do %>
        <i class="fs-2 fa-solid fa-pen-to-square"></i>
        <span class="d-xl-inline d-none">Add photo</span>
      <% end %>
    </li>
    <li class="nav-item">
      <%= link_to user_path(current_user.username), class: "nav-link" do %>
    <!-- ... -->
```
{: filename="app/views/shared/_left_sidebar.html.erb" }

The `data-bs-toggle="modal"` and `data-bs-target="#new_photo"` attributes tell Bootstrap to open the modal when this button is clicked, instead of navigating to a new page.

<div class="alert alert-success">

**CHECK**: Click "Add photo" in the sidebar → a modal pops up with the photo form. Upload a photo with a caption → it's created! The modal closes and you're redirected to the photo's show page.
</div>

## Right sidebar

The target has a search bar on wide screens and a floating "Add photo" button in the bottom-right corner. Let's add both.

Before we add the search form, we need to set up the Ransack search object in `ApplicationController`. The search form uses a `@q` variable that needs to be available on every page.

Open `app/controllers/application_controller.rb` and add a `before_action` and method for Ransack:

```ruby{3,5-7}
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  before_action :set_user_search, if: -> { current_user.present? }

  def set_user_search
    @q = User.all.ransack(params[:q])
  end
end
```
{: filename="app/controllers/application_controller.rb" }

The `set_user_search` method creates a Ransack search object `@q`. Since this runs in `ApplicationController`, `@q` is available in _every_ view template, including the layout where our search form lives. The `if: -> { current_user.present? }` condition skips this when no user is signed in (since the search bar isn't shown to signed-out users anyway).

When someone submits the search form, the params will include something like `q[username_cont]=ali`. Ransack parses this and generates a query like `WHERE username ILIKE '%ali%'`. The `_cont` suffix is a Ransack predicate meaning "contains."

<aside>
Remember that we whitelisted `username` as a searchable attribute in the previous lesson with `ransackable_attributes`. Without that whitelist, Ransack would refuse to search at all. It's a security measure to prevent people from searching sensitive fields like `email` or `encrypted_password`.
</aside>

Now let's create a partial for the right sidebar. Create `app/views/shared/_right_sidebar.html.erb`:

```erb
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
```
{: filename="app/views/shared/_right_sidebar.html.erb" copyable }

Now render this partial in the layout. Open `app/views/layouts/application.html.erb` and replace the comment: 

```erb
<!-- Right sidebar (coming soon) -->
``` 

with:

```erb{3}
        <!-- ... -->
        <div class="col-lg-3 col-md-2 d-none d-md-block">
          <%= render "shared/right_sidebar" %>
        </div>
        <!-- ... -->
```
{: filename="app/views/layouts/application.html.erb" }

1. **Search form**: Uses the Ransack gem's `search_form_for` helper with the `@q` search object. The `:username_cont` search field means "username contains." Ransack generates the correct SQL `LIKE` query automatically. The form is hidden below extra-extra-large screens (`d-none d-xxl-block`).

2. **Floating action button**: A circular "Add photo" button fixed to the bottom-right corner of the screen. The `style="left: unset"` prevents Bootstrap's `fixed-bottom` from stretching it across the full width.

<div class="alert alert-success">

**CHECK**: On a wide screen, you should see the search bar in the right sidebar. You should also see a circular floating button in the bottom-right corner of the screen. Reduce your screen width and watch them disappear.
</div>

Now would be a good time for a commit:

```
git add -A
git commit -m "Built three-column layout with sidebar navigation, photo modal, and search"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/44a8591bccf457a1e6dde3be709e7f99d07711ac)

## Mobile navbar and bottom navigation

If you make [the target](https://pg-industrial.matchthetarget.com) very narrow, you'll see a top navbar and bottom navigation for mobile. Let's add those.

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
{: filename="app/views/shared/_navbar.html.erb" copyable }

<aside>

Where is all this navbar related styling coming from? [Check out Bootstrap navbar](https://getbootstrap.com/docs/5.3/components/navbar/)! We've just copy-pasted examples from the docs and modified them to suit our needs, as usual.
</aside>

The key Bootstrap classes here are [`d-sm-none d-block`](https://getbootstrap.com/docs/5.3/utilities/display/). This means: "display block by default, but hide on small screens and up." In practice, this navbar only appears on extra-small screens (phones). On tablets and desktops, the sidebar navigation takes over.

When the user is signed in, the navbar shows an "Add photo" link. When not signed in, it shows sign in and sign up links instead.

Now add the navbar and bottom navigation to the layout. Open `app/views/layouts/application.html.erb` and render the navbar right before the three-column container:

```erb{4}
    <!-- ... -->
    <% end %>

    <%= render "shared/navbar" %>
    <div class="container mt-5">
      <div class="row">
        <!-- ... -->
```
{: filename="app/views/layouts/application.html.erb" }

Then create a partial for the fixed-bottom navbar. Create `app/views/shared/_bottom_navbar.html.erb`:

```erb
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
                <i class="fs-2 fa-solid fa-compass"></i>
              <% end %>
            </li>
            <li class="nav-item">
              <%= link_to user_path(current_user.username), class: "nav-link" do %>
                <i class="fs-2 fa-solid fa-user"></i>
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
```
{: filename="app/views/shared/_bottom_navbar.html.erb" copyable }

Render it in the layout after the closing `</div>` tags of the three-column container:

```erb{5}
      <!-- ... -->
      </div>
    </div>

    <%= render "shared/bottom_navbar" %>
  </body>
</html>
```
{: filename="app/views/layouts/application.html.erb" }

This bottom navbar only appears on extra-small screens (`d-block d-sm-none`). It uses `fixed-bottom` to stick to the bottom of the screen, just like Instagram's mobile app. It provides quick access to Feed, Discover, Profile, and Settings.

<aside>
Notice how we use Bootstrap's [responsive display utilities](https://getbootstrap.com/docs/5.3/utilities/display/) (`d-none`, `d-md-block`, `d-sm-none`, etc.) throughout the layout to show and hide elements at different screen sizes. This is much easier than writing custom media queries. The sidebars and bottom nav work together to provide navigation on every screen size.
</aside>

<div class="alert alert-success">

**CHECK**: Narrow your browser window. You should see a top navbar with "Photogram (Industrial)" and the bottom nav bar with Feed, Discover, Profile, and Settings icons. Widen the browser → the sidebars reappear and the mobile nav hides.
</div>

Now would be a good time for a commit:

```
git add -A
git commit -m "Added mobile navbar and bottom navigation"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/565363e2421b42c1c122f21c6dd88418c25face0)

## Creating the UsersController

Click "Discover" in the sidebar. Error! We stubbed `feed` earlier, but we still need the rest of the `UsersController` actions. Unlike the other controllers which were generated by scaffolds, the `UsersController` needs custom actions that don't follow the standard CRUD pattern.

Open `app/controllers/users_controller.rb` (the file we created earlier with just the `feed` stub). Let's build it up piece by piece.

### The `set_user` before_action

Add a `before_action` and a `private` method that looks up users by username:

```ruby{2,7-15}
class UsersController < ApplicationController
  before_action :set_user, only: %i[ show feed discover follows followers pending ]

  def feed
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

Our new method `set_user` runs before `show`, `feed`, `discover`, `follows`, `followers`, and `pending`. When the URL contains a username (like `/alice/feed`), it finds that user in the database. When there's no username (like when visiting the root URL `/`, which routes to `users#feed`), it defaults to `current_user`.

The `find_by!` method (with a bang!) raises an `ActiveRecord::RecordNotFound` exception if no user is found, which Rails automatically converts to a 404 error page. This is better than `find_by` (without the bang), which would return `nil` and cause confusing `NoMethodError` crashes later in the action.

### The `index` action

Add the `index` action now:

```ruby{4-6}
class UsersController < ApplicationController
  before_action :set_user, only: %i[ show feed discover follows followers pending ]

  def index
    @users = @q.result
  end

  def feed
  end
  # ...
```
{: filename="app/controllers/users_controller.rb" }

This is the search results page. Remember that `@q` is the Ransack search object set up in `ApplicationController`. Calling `.result` on it executes the search query and returns matching users. If no search params are present, it returns all users.

<aside>
Notice that we don't have a `before_action :set_user` on the `index` action. That's intentional because the search results page doesn't operate on a single user.
</aside>

### Feed and Discover

Update the `feed` action (replacing our earlier stub) and add `show` and `discover` after `index`:

```ruby{6-15}
  # ...
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

  # ...
```
{: filename="app/controllers/users_controller.rb" }

These actions leverage the `has_many :through` associations we built in the previous lesson. `@user.feed` returns all photos posted by people `@user` follows. `@user.discover` returns all photos liked by people `@user` follows. The heavy lifting is done by the model; the controller just passes the data to the view.

### Follows, Followers, and Pending

Add the remaining three actions after `discover`:

```ruby{6-16}
  # ...
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
  # ...
```
{: filename="app/controllers/users_controller.rb" }

These are straightforward. Each action uses an association we defined on the User model. `leaders` gives us the people `@user` follows, `followers` gives us the people who follow `@user`, and `pending_received_follow_requests` gives us follow requests that `@user` hasn't responded to yet.

## Stub view templates

The controller exists, but Rails needs matching view templates. Without them, every page will crash with a `MissingTemplate` error. Let's create basic stubs so every page loads. We'll build them out with real content in the next lesson.

Update `app/views/users/feed.html.erb` to replace our earlier stub:

```erb
<% content_for :title, "Feed" %>

<h1>Feed</h1>
<p>Photo cards coming soon!</p>
```
{: filename="app/views/users/feed.html.erb" }

Create `app/views/users/discover.html.erb`:

```erb
<% content_for :title, "Discover" %>

<h1>Discover</h1>
<p>Coming soon!</p>
```
{: filename="app/views/users/discover.html.erb" }

Create `app/views/users/show.html.erb`:

```erb
<% content_for :title, "@#{@user.username}'s profile" %>

<h1><%= @user.display_name || @user.username %></h1>
<p>@<%= @user.username %></p>
```
{: filename="app/views/users/show.html.erb" }

Create `app/views/users/followers.html.erb`:

```erb
<% content_for :title, "Followers" %>

<h1>Followers</h1>
<p>Coming soon!</p>
```
{: filename="app/views/users/followers.html.erb" }

Create `app/views/users/follows.html.erb`:

```erb
<% content_for :title, "Following" %>

<h1>Following</h1>
<p>Coming soon!</p>
```
{: filename="app/views/users/follows.html.erb" }

Create `app/views/users/pending.html.erb`:

```erb
<% content_for :title, "Pending" %>

<h1>Pending</h1>
<p>Coming soon!</p>
```
{: filename="app/views/users/pending.html.erb" }

Create `app/views/users/index.html.erb`:

```erb
<h1>Search Results</h1>
<p>Coming soon!</p>
```
{: filename="app/views/users/index.html.erb" }

<div class="alert alert-success">

**CHECK**: Click every link in the sidebar: Feed, Discover, Profile, and Settings should all load! Click your username → "Go to profile" → the profile page loads with the username. Visit `/alice/followers` and `/alice/follows` in your browser bar. The entire app is navigable!
</div>

Make a commit when you see everything is working:

```
git add -A
git commit -m "Created UsersController and stub view templates"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/abcbfb434c4de1c8811d2d8282008189f0a7c649)

## Wrapping up

You now have a fully navigable app with working authentication, a responsive three-column layout, mobile navigation, a photo upload modal, and styled scaffold pages. Every link in the sidebar works, and you can create photos from the modal overlay.

The user pages show stub content for now. We'll replace those stubs with real view templates (photo cards, user profiles, follower lists, and more) in the next lesson.

You can `rspec` (and `grade` to get points) to see what's left to do. (It turns out, quite a bit, but we've set everything up to cruise through the frontend build out that the tests are checking.)

## My pull request

You can visit the full diff of changes on my [pull request](https://github.com/bpurinton/pg-industrial/pull/3) in the [Files changed tab](https://github.com/bpurinton/pg-industrial/pull/3/files) if you need to compare your work to a working solution.

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
