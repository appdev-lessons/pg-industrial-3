# Photogram Industrial, Part 3: Routes, Layout, and Controllers

## Getting started

Let's continue building our Photogram Industrial project. Here's the target we're working towards:

[pg-industrial.matchthetarget.com](https://pg-industrial.matchthetarget.com/)

Navigate to `github.com/codespaces` (or reopen the previous lesson and use the "Load assignment" button) and reopen your `pg-industrial` project codespace to continue building on what you accomplished in the previous lessons.

At this point, you should have all five models (User, Photo, Comment, Like, FollowRequest) with their associations, validations, scopes, and sample data loaded. In this lesson, we'll shift our focus from the data layer to the interface layer: routes, the application layout, shared partials, and controllers.

Make sure your sample data is loaded before continuing:

```
rake sample_data
```

Since each lesson builds on the work from the previous one, we want to branch off of our last branch rather than `main`. First, switch to the branch from the previous lesson:

```
git switch models-and-associations
```

Now create a new branch from there:

```
git switch -c routes-layout-controllers
```

## Setting up routes

The scaffold generators from the previous lessons already added some routes for us, and Devise added `devise_for :users`. But our application needs more than basic CRUD routes. We need a nested route for listing a photo's likes, pages for the feed, discover, and pending follow requests, and vanity URL routes like `/:username` for user profiles.

<div class="alert alert-info">

<i class="fa-solid fa-circle-info"></i> This is the last time we'll remind you:

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

Besides the `root "users#feed"` line that we added for Devise, the generators have been filling it in for us. Let's make some changes.

Replace the entire file with this, paying particular attention to the highlighted additions:

```ruby{8-10,17-20,22-24}
Rails.application.routes.draw do
  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  root "users#feed"

  get ":username" => "users#show", as: :user
  get ":username/followers" => "users#followers", as: :followers
  get ":username/follows" => "users#follows", as: :follows

  devise_for :users

  resources :comments
  resources :follow_requests
  resources :likes
  resources :photos do
    resources :likes, only: [ :index ]
  end
  resources :users, only: [ :index ]

  get "feed" => "users#feed", as: :feed
  get "discover" => "users#discover", as: :discover
  get "pending" => "users#pending", as: :pending
end
```
{: filename="config/routes.rb" }

Besides the new routes, there's some re-ordering, which is mostly cosmetic (e.g. the `resources` are now in alphabetical order: comments, follow requests, likes, photos).

Before we walk through the additions, let's try one of the existing pages. Visit `/photos` in your live app preview.

You should see an error page that says `ActionDispatch::MissingController` and `uninitialized constant UsersController`. We asked for `/photos`, so why is Rails looking for a `UsersController`?

Rails checks the routes in `routes.rb` from top to bottom, and uses the first one that matches the request. Our new `get ":username"` route matches _any_ single segment after the `/`, so it matched `/photos`, with `"photos"` as the username, before Rails ever reached `resources :photos`. (We don't have a `UsersController` yet, which is why the error is about a missing controller.)

The fix is to move the three `:username` routes to the very end of the file, after every more specific route:

```ruby{22-24}
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
    resources :likes, only: [ :index ]
  end
  resources :users, only: [ :index ]

  get "feed" => "users#feed", as: :feed
  get "discover" => "users#discover", as: :discover
  get "pending" => "users#pending", as: :pending

  get ":username" => "users#show", as: :user
  get ":username/followers" => "users#followers", as: :followers
  get ":username/follows" => "users#follows", as: :follows
end
```
{: filename="config/routes.rb" }

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Visit `/photos` again. The scaffold's photos index page should load.
</div>

Here's a check on route order.

- If the first route in an app is `get ":slug" => "pages#show"`, which controller action handles a visit to `/products`?
- `products#index`
  - Not quite. Rails stops at the first route that matches, and `:slug` matches `products`.
- A routing error
  - Not quite. `:slug` matches any single segment, so the request does find a route.
- `pages#show`
  - Yes! Rails uses the first matching route, with `"products"` as `params[:slug]`.
{: .choose_best #route_order title="Route order" points="1" answer="3" }

Now let's walk through the additions.

### Nested routes

Previously, the photos resource line was:

```ruby
resources :photos
```

which we know from previous lessons gives us the "seven golden" RESTful routes: index, new, create, show, edit, update, and destroy. From just that line we get `"/photos"`, `"/photos/:id"`, etc., and named helper methods like `photos_path` and `photos_url`.

Our edit added a `do` block and _nested_ a likes resource under the photos resource:

```ruby{1:(18-20),2-3}
resources :photos do
  resources :likes, only: [ :index ]
end
```

This creates a [nested route](https://guides.rubyonrails.org/routing.html#nested-resources). On the target, the like count under each photo links to a list of the people who liked it. We need a URL that identifies _which_ photo's likes to show, and that's what nested routes give us: URLs like `/photos/42/likes`, where the photo's ID is part of the path.

Specifically, the nested line above generates:

| HTTP verb | Path | Controller#Action | Named helper |
|---|---|---|---|
| GET | `/photos/:photo_id/likes` | `likes#index` | `photo_likes_path(photo)` |
{: .bleed-full }

(You can run the command `rails routes` at the terminal to find that new route.)

Notice the `:photo_id` segment in the path. When a request comes in to `/photos/42/likes`, Rails sets `params[:photo_id]` to `42`. In the controller, we'll use that to scope our query (`Photo.find(params[:photo_id]).likes`) so we only fetch the likes that belong to that specific photo.

You might be wondering: we already have `resources :likes` as a standalone route earlier in the file. Why do we need a nested version too? The standalone routes handle creating and deleting an individual like, where the like's own ID (or the form's data) is enough. The nested route serves a different purpose: _listing_ all the likes that belong to a specific photo. That's why we use `only: [ :index ]` to restrict the nested route to the `index` action.

Here's a check on nested routes.

- Which path does `resources :books do resources :reviews, only: [ :index ] end` add?
- `/reviews/:book_id`
  - Not quite. The nested resource comes after its parent in the path.
- `/books/:book_id/reviews`
  - Yes! The parent's ID sits in the path, and arrives as `params[:book_id]`.
- `/books/reviews/:id`
  - Not quite. Look for the `:book_id` segment that identifies the parent.
{: .choose_best #nested_route title="Nested routes" points="1" answer="2" }

### Users index

```ruby
resources :users, only: [ :index ]
```

This gives us a `/users` route that we'll use for search results. We're restricting it to `only: [ :index ]` because Devise handles the other user-related routes (sign up, sign in, edit profile).

### Feed, discover, and pending

```ruby
get "feed" => "users#feed", as: :feed
get "discover" => "users#discover", as: :discover
get "pending" => "users#pending", as: :pending
```

These three pages always show the signed-in user's own photos and follow requests, so their URLs don't need a username: `/feed`, `/discover`, and `/pending`.

### Vanity URL routes

Instead of `/users/42`, we use `/alice`, a vanity URL with the username right in the path:

```ruby
get ":username" => "users#show", as: :user
get ":username/followers" => "users#followers", as: :followers
get ":username/follows" => "users#follows", as: :follows
```

Each route captures the `:username` segment from the URL and passes it to the controller as `params[:username]`. Profiles and follower lists can belong to any user, so these URLs need the username.

We don't have a `UsersController` with the index, show, feed, etc. actions yet, nor the corresponding view templates. We'll make all of those later to handle the routes we've set up.

The `as:` option on each route creates named path helpers. For example, `as: :user` gives us `user_path("alice")`, which generates `/alice`, and `as: :followers` gives us `followers_path("alice")`, which generates `/alice/followers`. `feed_path` takes no argument and generates `/feed`. We'll use these helpers extensively in our views.

Now would be a good time for a commit:

```
git add -A
git commit -m "Set up routes with nested resources and vanity URLs"
git push --set-upstream origin routes-layout-controllers
```

That last command publishes the branch to GitHub. From now on, you can push with just `git push` after each commit.

### Open and submit your pull request

Now that you've pushed your branch to GitHub, it's time to open a **pull request** (PR). A pull request lets us review your code and leave line-by-line feedback.

<div class="alert alert-primary">

<i class="fa-solid fa-circle-play"></i> [Here is a short video demonstration of the process.](https://share.descript.com/view/RLP4apAu5pp) You should also carefully read the notes below!
</div>

On GitHub, navigate to your `pg-industrial` repository. You should see a prompt to open a pull request for the `routes-layout-controllers` branch that you just published, or you can go to the "Pull requests" tab and click "New pull request."

Make sure the **base** branch is `models-and-associations` and the **compare** branch is `routes-layout-controllers`. Also be sure to change the base _repository_ from `appdev-projects/pg-industrial` to _your_ fork. Give it a title, then click "Create pull request."

<aside>

If there's no base _repository_ dropdown when you open your pull request, skip that step. Projects launched from the Grades page now save your work to your own repository rather than a fork, so your pull request already targets your repository.
</aside>

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

Here's a check on what that line does.

- With `before_action :authenticate_user!` in `ApplicationController`, what happens when a signed-out visitor requests `/photos`?
- They're redirected to the sign-in page
  - Yes! The `before_action` runs before the `index` action, and sends them to `/users/sign_in`.
- The page loads, but without any photos
  - Not quite. The action never runs for a signed-out visitor.
- They see a 404 page
  - Not quite. The route exists; Devise redirects instead.
{: .choose_best #authenticate_user title="Signed-out visitors" points="1" answer="1" }

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

<i class="fa-solid fa-circle-check"></i> If your server isn't running, start it with `bin/dev`, and visit `/`. You should be redirected to the sign-in page. Sign in with `alice@example.com` / `appdev`. You should see the stub feed page with "Feed" and "Coming soon!"
</div>

Now would be a good time for a commit:

```
git add -A
git commit -m "Added authentication and stubbed feed action"
git push
```

## Stylesheet and icons

The scaffold pages work, but they look plain. Visit `/photos/new` in your live app preview. It's a bare HTML form. Let's connect our stylesheet and Font Awesome, the same way we did in [Helper Methods, Part 3](/lessons/194-helper-methods-part-3): with a shared partial for the tags.

Create a new directory and file at `app/views/shared/_cdn_assets.html.erb`:

```erb
<!-- Tailwind's utility classes, compiled in the browser -->
<script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4.3"></script>

<!-- Our stylesheet: styles for plain elements, plus named components -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/firstdraft/tailwind.appdev.css@latest/tailwind.appdev.css">

<!-- Font Awesome icons -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.7.2/css/all.min.css">
```
{: filename="app/views/shared/_cdn_assets.html.erb" copyable }

We're loading three things:

1. **Tailwind**: utility classes like `grid`, `hidden`, and `md:block`, which arrange things on the page.
2. **Our stylesheet**: styles for plain HTML elements, plus component classes like `btn`, `alert`, `card`, `nav-link`, and `dropdown-menu`.
3. **Font Awesome**: icons throughout the interface (hearts, houses, magnifying glasses, etc.).

<aside>
Why use a CDN instead of bundling these assets locally? We've chosen to use CDNs for now since they are quick and easy to understand. Later we'll discuss asset bundling to enhance performance for our end users.
</aside>

Now render the partial in the layout's `<head>`, and add `class="appdev-styles"` to the `<body>` so that our stylesheet styles the page. Open `app/views/layouts/application.html.erb` and make the highlighted changes:

```erb{9,16:(8-29)}
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

  <body class="appdev-styles">
    <%= yield %>
  </body>
</html>
```
{: filename="app/views/layouts/application.html.erb" }

<aside>

The `content_for(:title)` allows individual pages to set a custom title (you'll notice the scaffold templates already include `content_for :title` calls). If none is set, it falls back to "Photogram (Industrial)."
</aside>

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Refresh your live app preview. The page should now have our stylesheet's styling: the font changes and the form inputs look cleaner.
</div>

## Flash messages partial

When you signed in, the flash message "Signed in successfully." did not appear. Let's add flash messages and style them as an alert with a button to dismiss it.

Create `app/views/shared/_flash.html.erb`:

```erb
<div class="alert alert-<%= css_class %> alert-dismissible" role="alert" data-controller="dismiss">
  <%= message %>

  <button type="button" class="btn-close" data-action="click->dismiss#remove" aria-label="Close"></button>
</div>
```
{: filename="app/views/shared/_flash.html.erb" copyable }

This partial expects two local variables: `message` (the text to display) and `css_class` (either `"success"` for green or `"danger"` for red). `alert-dismissible` makes room for the close button at the right, and `btn-close` draws the × for us, so the button itself is empty.

Removing the alert when the × is clicked takes a little JavaScript. Our starting point already has it: look in `app/javascript/controllers/` and you'll find four small Stimulus controllers, `dismiss`, `dropdown`, `modal`, and `tabs`, that each handle one kind of click. We connect them to our HTML with `data-` attributes:

- `data-controller="dismiss"` says that this `<div>` is looked after by `dismiss_controller.js`.
- `data-action="click->dismiss#remove"` says that clicking this button runs the controller's `remove` function, which removes the alert from the page.

We'll use the `dropdown` and `modal` controllers the same way later in this lesson. (If you'd like to practice with all four, the [AppDev Stylesheet Component Classes](/lessons/912-appdev-stylesheet-component-classes#components-that-move) lesson has a level for each.)

Now add flash message rendering to the layout body. Open `app/views/layouts/application.html.erb` and add the highlighted lines between `<body>` and `<%= yield %>`:

```erb{3-8}
  <!-- ... -->
  <body class="appdev-styles">
    <% if notice.present? %>
      <%= render "shared/flash", message: notice, css_class: "success" %>
    <% end %>
    <% if alert.present? %>
      <%= render "shared/flash", message: alert, css_class: "danger" %>
    <% end %>

    <%= yield %>
  </body>
</html>
```
{: filename="app/views/layouts/application.html.erb" }

We only render an alert if there's actually a flash message to show.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Since we haven't built a sign out link yet, you can run `rake sample_data` to reset your database and sign out. Then sign back in your live app preview. You should see a green alert that says "Signed in successfully." with a dismiss button (×). Click the × to dismiss it.
</div>

Now would be a good time for a commit:

```
git add -A
git commit -m "Created shared partials for CDN assets and flash messages"
git push
```

## Updating the photo form partial

Visit `/photos/new` in your live app preview. The scaffold form has fields for Owner, Pinned, Comments count, and Likes count, none of which a user should need to fill in! Let's clean it up to just an image upload and a caption.

Open `app/views/photos/_form.html.erb` and replace its entire contents with:

```erb
<%= form_with(model: photo) do |form| %>
  <% photo.errors.each do |error| %>
    <div class="alert alert-danger" role="alert">
      <%= error.full_message %>
    </div>
  <% end %>

  <div class="form-group">
    <%= form.label :image, class: "visually-hidden" %>

    <% if photo.persisted? && photo.image.attached? %>
      <%= image_tag photo.image, class: "w-full rounded-lg mb-2" %>
    <% end %>

    <%= form.file_field :image, class: "form-control", accept: "image/*" %>
  </div>

  <div class="form-group">
    <%= form.label :caption, class: "visually-hidden" %>
    <%= form.text_area :caption, class: "form-control", placeholder: "What's going on?" %>
  </div>

  <%= form.submit class: "btn btn-primary" %>
<% end %>
```
{: filename="app/views/photos/_form.html.erb" copyable }

Let's walk through it:

- `form_with(model: photo)` uses the `photo` local variable, which means the form works for both new and existing photos. Rails automatically determines the correct URL and HTTP method.
- Each validation error is displayed as a red `alert`.
- `form.file_field :image` is an actual image upload field, in place of the scaffold's text field. The `accept: "image/*"` tells the browser to only show image files in the file picker dialog. The `photo.persisted? && photo.image.attached?` check means that when editing an existing photo, we show the current image above the file field so the user can see what they're replacing.
- We removed the owner ID, pinned, comments count, and likes count fields. These are not things a user should fill out when they create a new photo.

The labels are `visually-hidden` throughout the form. We use placeholders instead for a cleaner look, but the labels are still present for accessibility (screen readers can read them).

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Visit `/photos/new`. You should see a clean form with just an image upload field and a caption text area.
</div>

## Customizing scaffold controllers

Try creating a photo from `/photos/new`. Upload an image and add a caption.

You should get hit with an "Owner must exist" validation error! (Good thing we did all of that backend work in the earlier lessons to keep out bad data.)

![The new photo form after submitting, with a red "Owner must exist" alert above the image and caption fields](/assets/owner-must-exist.png)

The owner isn't set automatically because the scaffold expects an `owner_id` from the form, which we just removed. Let's fix all four scaffold controllers to assign `current_user` and redirect sensibly.

### PhotosController

Open `app/controllers/photos_controller.rb`. We need to make a few changes from the scaffold:

**1. Assign the owner in `create`:**

```ruby{4}
  # ...
  def create
    @photo = Photo.new(photo_params)
    @photo.owner = current_user

    respond_to do |format|
      # ...
```
{: filename="app/controllers/photos_controller.rb" }

Instead of accepting `owner_id` from the form, we set it from `current_user`. A form can send any value it likes, so we never let it say who is performing an action.

Here's a check on that idea.

- Which line in a `CommentsController#create` action sets the new comment's author safely?
- `@comment.author_id = params[:author_id]`
  - Not quite. Anyone can put any ID in a form's params.
- `@comment.author = current_user`
  - Yes! `current_user` comes from the session, not from the form.
- `@comment.author = User.first`
  - Not quite. That would make the first user the author of every comment.
{: .choose_best #author_from_session title="Setting the author" points="1" answer="2" }

**2. Change `destroy` to redirect back:**

```ruby{6:(21-61)}
  # ...
  def destroy
    @photo.destroy!

    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Photo was successfully destroyed.", status: :see_other }
      # ...
```
{: filename="app/controllers/photos_controller.rb" }

The scaffold redirected to `photos_path` (the index page). We use `redirect_back` instead, which sends the user back to wherever they came from (their feed, a profile page, etc.). The `fallback_location: root_url` is a safety net in case the request has no referer header, redirecting to the homepage.

Here's a check on `redirect_back`.

- What does `redirect_back fallback_location: root_url` do when the request has no referer?
- It raises an error
  - Not quite. That's what the fallback is there to prevent.
- It reloads the same page
  - Not quite. Without a referer, Rails doesn't know where "back" is.
- It sends the user to `root_url`
  - Yes! The fallback is used whenever Rails can't tell where the request came from.
{: .choose_best #redirect_back title="Redirecting back" points="1" answer="3" }

**3. Update `photo_params`:**

```ruby{3:(26-54)}
  # ...
  def photo_params
    params.expect(photo: [ :image, :caption, :pinned ])
  end
  # ...
```
{: filename="app/controllers/photos_controller.rb" }

We permit `:image`, `:caption`, and `:pinned`. Notice that `:owner_id` is _not_ in this list because we set that in the controller, not from form params. We also dropped `:comments_count` and `:likes_count`, which our counter caches keep up to date.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Visit `/photos/new`. Upload an image and type a caption, then click "Create Photo." The photo should be created, with you as its owner.
</div>

Now that we've seen the pattern, let's make these updates to our other controllers.

### CommentsController

Open `app/controllers/comments_controller.rb`. We need to make a few changes from the scaffold:

**1. Assign the author, and redirect back after `create`:**

```ruby{4,8,11}
  # ...
  def create
    @comment = Comment.new(comment_params)
    @comment.author = current_user

    respond_to do |format|
      if @comment.save
        format.html { redirect_back fallback_location: root_path, notice: "Comment was successfully created." }
        format.json { render :show, status: :created, location: @comment }
      else
        format.html { redirect_back fallback_location: root_path, alert: @comment.errors.full_messages.to_sentence }
        format.json { render json: @comment.errors, status: :unprocessable_content }
      end
    end
  end
  # ...
```
{: filename="app/controllers/comments_controller.rb" }

Same pattern as photos: we set the author from `current_user` rather than trusting form input.

In the next lesson, the form to add a comment will sit under each photo on the feed and profile pages, so both outcomes send the user back to that page. When the comment saves, they see a notice. When it doesn't (for example, when the body is blank), the scaffold would have shown its own comment form page; instead, we go back too, and show the validation errors as an alert. `full_messages.to_sentence` joins the errors into one sentence, like "Body can't be blank." The tests check that a blank comment ends with that alert rather than an error page.

**2. Redirect back after `destroy`:**

```ruby{6:(21-61)}
  # ...
  def destroy
    @comment.destroy!

    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Comment was successfully destroyed.", status: :see_other }
      # ...
```
{: filename="app/controllers/comments_controller.rb" }

Comments are deleted from the feed and photo pages, so we want to redirect back to wherever the user was rather than navigating to a separate comments page.

**3. Redirect to root after `update`:**

```ruby{5:(35-42)}
  # ...
  def update
    respond_to do |format|
      if @comment.update(comment_params)
        format.html { redirect_to root_url, notice: "Comment was successfully updated.", status: :see_other }
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

**2. Assign the fan, and redirect back after `create`:**

```ruby{4,8,11}
  # ...
  def create
    @like = Like.new(like_params)
    @like.fan = current_user

    respond_to do |format|
      if @like.save
        format.html { redirect_back fallback_location: @like.photo, notice: "Like was successfully created." }
        format.json { render :show, status: :created, location: @like }
      else
        format.html { redirect_back fallback_location: @like.photo, alert: "Failed to like photo." }
        format.json { render json: @like.errors, status: :unprocessable_content }
      end
    end
  end
  # ...
```
{: filename="app/controllers/likes_controller.rb" }

Same pattern: the current user is set as the fan.

**3. Redirect back after `destroy`:**

```ruby{6:(21-64)}
  # ...
  def destroy
    @like.destroy!

    respond_to do |format|
      format.html { redirect_back fallback_location: @like.photo, notice: "Like was successfully destroyed.", status: :see_other }
      # ...
```
{: filename="app/controllers/likes_controller.rb" }

After liking or unliking a photo, the user stays on the same page. The `fallback_location` is `@like.photo`, the photo that was liked.

### FollowRequestsController

Open `app/controllers/follow_requests_controller.rb`. We need to make a few changes from the scaffold.

**1. Set the sender and the status in `create`:**

```ruby{3-10,14,17}
  # ...
  def create
    @follow_request = FollowRequest.new(recipient_id: follow_request_params[:recipient_id])
    @follow_request.sender = current_user

    if @follow_request.recipient.private?
      @follow_request.status = "pending"
    else
      @follow_request.status = "accepted"
    end

    respond_to do |format|
      if @follow_request.save
        format.html { redirect_back fallback_location: root_url, notice: "Follow request was successfully created." }
        format.json { render :show, status: :created, location: @follow_request }
      else
        format.html { redirect_back fallback_location: root_url, alert: @follow_request.errors.full_messages.to_sentence }
        format.json { render json: @follow_request.errors, status: :unprocessable_content }
      end
    end
  end
  # ...
```
{: filename="app/controllers/follow_requests_controller.rb" }

We set the `sender` to `current_user`, just as we did for photos, comments, and likes.

The status needs the same care. The follow button's form will only send a `recipient_id`, but anyone can add more fields to a form's request, including `status=accepted`. If we built the follow request from all of the permitted params, a request like that would follow a private account without the owner's approval. So we read only the `recipient_id` from the params, and set the status ourselves:

- If the recipient's account is private, the request is `"pending"`, and the recipient will need to accept it.
- If it is public, we accept the follow request right away, so the user doesn't need to wait for approval.

(Since User has a `private` boolean column, we get a `private?` method for free, which returns the column's value.) The tests include a follow request that tries to send `status=accepted` to a private account, and check that it stays pending.

**2. Change only the status in `update`, and redirect back:**

```ruby{4:(33-70),5:(23-63)}
  # ...
  def update
    respond_to do |format|
      if @follow_request.update(status: follow_request_params[:status])
        format.html { redirect_back fallback_location: root_url, notice: "Follow request was successfully updated.", status: :see_other }
        # ...
```
{: filename="app/controllers/follow_requests_controller.rb" }

Updating a follow request means accepting or rejecting it, so `update` changes only its `status`. The sender and the recipient stay as they were.

**3. Redirect back after `destroy`:**

```ruby{6:(21-61)}
  # ...
  def destroy
    @follow_request.destroy!

    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Follow request was successfully destroyed.", status: :see_other }
      # ...
```
{: filename="app/controllers/follow_requests_controller.rb" }

Follow, unfollow, accept, and reject all happen from profile pages and the pending page, so we always want to send the user back to where they were.

**4. Update `follow_request_params`:**

```ruby{3:(35-60)}
  # ...
  def follow_request_params
    params.expect(follow_request: [ :recipient_id, :status ])
  end
  # ...
```
{: filename="app/controllers/follow_requests_controller.rb" }

The sender always comes from `current_user`, so `:sender_id` leaves the list.

---

Let's make a commit:

```
git add -A
git commit -m "Updated photo form and customized scaffold controllers"
git push
```

## Three-column grid structure

Open [the target](https://pg-industrial.matchthetarget.com) and observe: navigation on the left, content in the center, search on the right. Let's build that layout with a grid.

Open `app/views/layouts/application.html.erb` and replace the contents of the `<body>` with:

```erb{3-6,8,17-22}
  <!-- ... -->
  <body class="appdev-styles">
    <main class="container grid grid-cols-1 md:grid-cols-4 gap-6 pt-20 pb-24 md:pt-6 md:pb-12">
      <div class="hidden md:block">
        <!-- Left sidebar (coming soon) -->
      </div>

      <div class="md:col-span-2">
        <% if notice.present? %>
          <%= render "shared/flash", message: notice, css_class: "success" %>
        <% end %>
        <% if alert.present? %>
          <%= render "shared/flash", message: alert, css_class: "danger" %>
        <% end %>

        <%= yield %>
      </div>

      <div class="hidden md:block">
        <!-- Right sidebar (coming soon) -->
      </div>
    </main>
  </body>
</html>
```
{: filename="app/views/layouts/application.html.erb" }

Let's walk through the classes:

- `container` keeps the page to a comfortable width, centered.
- `grid grid-cols-1 md:grid-cols-4` makes one column on a phone, and four equal columns from a tablet's width (768 pixels, the `md:` prefix) up. `gap-6` puts space between them.
- The center column has `md:col-span-2`, so on wider screens it takes up two of the four columns, and each sidebar takes one.
- Both sidebars have `hidden md:block`: hidden on phones, and shown from a tablet's width up.
- `pt-20 pb-24` adds padding to the top and bottom of the page on phones, to leave room for the navigation bars we'll add at the top and bottom of the screen later in this lesson. `md:pt-6 md:pb-12` sets smaller padding on wider screens, where those bars are hidden.

The flash messages and the `<%= yield %>` moved into the center column, which is where the content from each page's view template gets rendered. If you'd like more practice with grids and the screen-size prefixes, see [Level 3](/lessons/912-appdev-stylesheet-component-classes#level-3-grid) of the component classes lesson.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Visit any page in your live app preview. The page content should now be in a middle column with space on each side. Try resizing your browser: on a narrow screen, the content takes the full width.
</div>

## Left sidebar navigation

The target has navigation links on the left: Feed, Discover, Profile, Settings. Let's put this in a partial to keep the layout file clean.

Create `app/views/shared/_left_sidebar.html.erb`:

```erb
<% if current_user.present? %>
  <ul class="nav flex-col sticky-top">
    <li class="nav-item">
      <div class="dropdown" data-controller="dropdown">
        <button type="button" class="nav-link" data-action="click->dropdown#toggle" aria-expanded="false">
          <%= image_tag current_user.avatar_image, class: "rounded-full img-small img-cover" %>

          <span class="hidden xl:inline">
            <%= current_user.display_name %>
            <span class="block font-light">@<%= current_user.username %></span>
          </span>
        </button>

        <ul class="dropdown-menu" data-dropdown-target="menu">
          <li>
            <%= link_to user_path(current_user.username), class: "dropdown-item" do %>
              <i class="fa-regular fa-circle-user"></i>
              Go to profile
            <% end %>
          </li>
          <li>
            <%= button_to destroy_user_session_path, method: :delete, class: "dropdown-item" do %>
              <i class="fa-solid fa-arrow-right-from-bracket"></i>
              Sign out
            <% end %>
          </li>
        </ul>
      </div>
    </li>
    <li class="nav-item">
      <%= link_to feed_path, class: "nav-link" do %>
        <i class="text-2xl fa-solid fa-house"></i>
        <span class="hidden xl:inline">Feed</span>
      <% end %>
    </li>
    <li class="nav-item">
      <%= link_to discover_path, class: "nav-link" do %>
        <i class="text-2xl fa-solid fa-magnifying-glass"></i>
        <span class="hidden xl:inline">Discover</span>
      <% end %>
    </li>
    <li class="nav-item">
      <%= link_to user_path(current_user.username), class: "nav-link" do %>
        <i class="text-2xl fa-regular fa-circle-user"></i>
        <span class="hidden xl:inline">Profile</span>
      <% end %>
    </li>
    <li class="nav-item">
      <%= link_to edit_user_registration_path, class: "nav-link" do %>
        <i class="text-2xl fa-solid fa-gear"></i>
        <span class="hidden xl:inline">Settings</span>
      <% end %>
    </li>
  </ul>
<% end %>
```
{: filename="app/views/shared/_left_sidebar.html.erb" copyable }

Now render this partial in the layout. Open `app/views/layouts/application.html.erb` and, in the first `hidden md:block` column, replace the comment:

```erb
<!-- Left sidebar (coming soon) -->
```

with:

```erb{3}
      <!-- ... -->
      <div class="hidden md:block">
        <%= render "shared/left_sidebar" %>
      </div>
      <!-- ... -->
```
{: filename="app/views/layouts/application.html.erb" }

Key elements:

- **User dropdown**: Shows the current user's avatar and, on wide screens, their name and username. This is the `dropdown` Stimulus controller at work: `data-controller="dropdown"` on the wrapper, `data-action="click->dropdown#toggle"` on the button, and `data-dropdown-target="menu"` on the list that opens. The menu has "Go to profile" and "Sign out" options. The `button_to` for sign out uses `method: :delete` because Devise expects a DELETE request.
- **Navigation links**: Feed, Discover, Profile, and Settings. Each uses a Font Awesome icon (`text-2xl` makes it larger) and a text label. The labels have `hidden xl:inline`, so they only appear on extra-large screens (1280 pixels and up). On narrower screens, only the icons show, keeping the sidebar compact.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Refresh your live app preview. You should see navigation links in the left sidebar. Click "Feed" → the stub feed page. Click "Settings" → it goes to the Devise edit page. Click your avatar in the sidebar → see "Go to profile" and "Sign out" options. Click "Sign out" and verify it works!

(Don't worry that "Discover" and "Profile" error when clicked. We haven't built those controller actions yet. We'll get there.)
</div>

## New Photo modal

In [the target](https://pg-industrial.matchthetarget.com), clicking "Add photo" opens a box over the page (a _modal_) instead of navigating to `/photos/new`. Let's add one.

Create `app/views/shared/_photo_modal.html.erb`:

```erb
<% if current_user.present? %>
  <dialog data-modal-target="dialog">
    <div class="card">
      <header>
        <button type="button" aria-label="Close" rel="prev" data-action="click->modal#close"></button>

        <h3>New photo</h3>
      </header>

      <%= render "photos/form", photo: current_user.own_photos.build %>
    </div>
  </dialog>
<% end %>
```
{: filename="app/views/shared/_photo_modal.html.erb" copyable }

`<dialog>` is the HTML element for a box that opens over the page. The `modal` Stimulus controller opens and closes it: `data-modal-target="dialog"` tells the controller which element to open, and the button in the header, which our stylesheet draws as a ×, runs `modal#close`. Inside the `<dialog>` is an ordinary card, holding our photo form.

Let's look closer at the render line:

```erb{1:(34-62)}
<%= render "photos/form", photo: current_user.own_photos.build %>
```

`current_user.own_photos.build` works like `Photo.new`: it makes a new, unsaved photo in memory. Because we call it through the `own_photos` association, Rails also fills in the new photo's `owner_id` with the current user's ID. Here, it gives the form a new photo to work with, so `form_with` knows to send a POST request to `/photos`. The photo's owner is still set by `PhotosController#create`, from `current_user`, since the form doesn't send an `owner_id`.

Now render this partial in the layout, right after the opening `<body>` tag, and add `data-controller="modal"` to the `<body>` itself:

```erb{2:(30-53),3}
  <!-- ... -->
  <body class="appdev-styles" data-controller="modal">
    <%= render "shared/photo_modal" %>

    <main class="container grid grid-cols-1 md:grid-cols-4 gap-6 pt-20 pb-24 md:pt-6 md:pb-12">
    <!-- ... -->
```
{: filename="app/views/layouts/application.html.erb" }

The `modal` controller only works with buttons and dialogs that are inside the element that has `data-controller="modal"`. We'll have two "Add photo" buttons, one in each sidebar, and the `<body>` is the one element that contains both of them and the `<dialog>`.

Since the modal is in the layout, it's on _every_ page, so signed-in users can add a photo from anywhere in the app.

Now add an "Add photo" button to the left sidebar navigation. Open `app/views/shared/_left_sidebar.html.erb` and add this nav item between the Discover and Profile links:

```erb{8-13}
    <!-- ... -->
    <li class="nav-item">
      <%= link_to discover_path, class: "nav-link" do %>
        <i class="text-2xl fa-solid fa-magnifying-glass"></i>
        <span class="hidden xl:inline">Discover</span>
      <% end %>
    </li>
    <li class="nav-item">
      <button type="button" class="nav-link" data-action="click->modal#open">
        <i class="text-2xl fa-solid fa-pen-to-square"></i>
        <span class="hidden xl:inline">Add photo</span>
      </button>
    </li>
    <li class="nav-item">
      <%= link_to user_path(current_user.username), class: "nav-link" do %>
    <!-- ... -->
```
{: filename="app/views/shared/_left_sidebar.html.erb" }

The `data-action="click->modal#open"` attribute runs the `modal` controller's `open` function when this button is clicked, instead of navigating to a new page.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Click "Add photo" in the sidebar → a modal pops up with the photo form. Upload a photo with a caption → it's created! You're redirected to the photo's show page.
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
Remember that we allowed `username` as a searchable attribute in the previous lesson, with [`ransackable_attributes`](/lessons/838-pg-industrial-2#allowing-ransack-searches). Without that list, Ransack would refuse to search at all. It's a security measure to prevent people from searching sensitive fields like `email` or `encrypted_password`.
</aside>

Now let's create a partial for the right sidebar. Create `app/views/shared/_right_sidebar.html.erb`:

```erb
<% if current_user.present? %>
  <div class="hidden xl:block">
    <%= search_form_for @q, class: "flex gap-2", role: "search" do |f| %>
      <%= f.label :username_cont, class: "visually-hidden" %>
      <%= f.search_field :username_cont, class: "form-control", placeholder: "Search by username" %>

      <%= f.button class: "btn btn-outline-primary", name: nil, type: "submit" do %>
        <i class="fa-solid fa-magnifying-glass"></i>
      <% end %>
    <% end %>
  </div>

  <div class="fixed bottom-12 right-12">
    <button type="button" class="btn btn-primary rounded-full w-16 h-16 text-2xl" data-action="click->modal#open">
      <i class="fa-solid fa-pen-to-square"></i>
    </button>
  </div>
<% end %>
```
{: filename="app/views/shared/_right_sidebar.html.erb" copyable }

Now render this partial in the layout. Open `app/views/layouts/application.html.erb` and, in the second `hidden md:block` column, replace the comment:

```erb
<!-- Right sidebar (coming soon) -->
```

with:

```erb{3}
      <!-- ... -->
      <div class="hidden md:block">
        <%= render "shared/right_sidebar" %>
      </div>
      <!-- ... -->
```
{: filename="app/views/layouts/application.html.erb" }

1. **Search form**: Uses the Ransack gem's `search_form_for` helper with the `@q` search object. The `:username_cont` search field means "username contains." Ransack generates the SQL query automatically. The form is hidden below extra-large screens (`hidden xl:block`).

2. **Floating action button**: A circular "Add photo" button. `fixed bottom-12 right-12` keeps it in the bottom-right corner of the screen while the page scrolls, and `rounded-full w-16 h-16` makes it a circle. It has the same `data-action="click->modal#open"` as the sidebar's button, so it opens the same modal.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> On a wide screen, you should see the search bar in the right sidebar. You should also see a circular floating button in the bottom-right corner of the screen. Click it → the photo modal opens. Reduce your screen width and watch them disappear.
</div>

![The feed page on a wide screen, signed in as Alice: navigation in the left sidebar, the search bar at the top right, and a round floating button in the bottom-right corner](/assets/layout-wide.png)

Now would be a good time for a commit:

```
git add -A
git commit -m "Built three-column layout with sidebar navigation, photo modal, and search"
git push
```

## Mobile navbar and bottom navigation

If you make [the target](https://pg-industrial.matchthetarget.com) narrow, you'll see a top navbar and bottom navigation for phones. Let's add those.

Create `app/views/shared/_navbar.html.erb`:

```erb
<nav class="navbar fixed-top md:hidden">
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
      <div class="flex">
        <%= link_to "Sign in", new_user_session_path, class: "nav-link" %>
        <%= link_to "Sign up", new_user_registration_path, class: "nav-link" %>
      </div>
    <% end %>
  </div>
</nav>
```
{: filename="app/views/shared/_navbar.html.erb" copyable }

`navbar`, `navbar-brand`, and `nav-link` are component classes from our stylesheet, for a bar of links across the top of the page. `fixed-top` keeps it at the top of the screen while the page scrolls, and `md:hidden` hides it from a tablet's width up.

When the user is signed in, the navbar shows an "Add photo" link. When not signed in, it shows sign in and sign up links instead.

Now render the navbar in the layout, right after the photo modal:

```erb{4}
  <!-- ... -->
  <body class="appdev-styles" data-controller="modal">
    <%= render "shared/photo_modal" %>
    <%= render "shared/navbar" %>

    <main class="container grid grid-cols-1 md:grid-cols-4 gap-6 pt-20 pb-24 md:pt-6 md:pb-12">
    <!-- ... -->
```
{: filename="app/views/layouts/application.html.erb" }

Then create a partial for the bottom navigation. Create `app/views/shared/_bottom_navbar.html.erb`:

```erb
<% if current_user.present? %>
  <nav class="navbar fixed-bottom md:hidden">
    <ul class="navbar-nav justify-evenly">
      <li class="nav-item">
        <%= link_to feed_path, class: "nav-link" do %>
          <i class="text-2xl fa-solid fa-house"></i>
        <% end %>
      </li>
      <li class="nav-item">
        <%= link_to discover_path, class: "nav-link" do %>
          <i class="text-2xl fa-solid fa-magnifying-glass"></i>
        <% end %>
      </li>
      <li class="nav-item">
        <%= link_to user_path(current_user.username), class: "nav-link" do %>
          <i class="text-2xl fa-regular fa-circle-user"></i>
        <% end %>
      </li>
      <li class="nav-item">
        <%= link_to edit_user_registration_path, class: "nav-link" do %>
          <i class="text-2xl fa-solid fa-gear"></i>
        <% end %>
      </li>
    </ul>
  </nav>
<% end %>
```
{: filename="app/views/shared/_bottom_navbar.html.erb" copyable }

Render it in the layout after the closing `</main>` tag:

```erb{4}
      <!-- ... -->
    </main>

    <%= render "shared/bottom_navbar" %>
  </body>
</html>
```
{: filename="app/views/layouts/application.html.erb" }

The bottom navbar uses `fixed-bottom` to stick to the bottom of the screen, like Instagram's mobile app, and gives quick access to Feed, Discover, Profile, and Settings. `justify-evenly` spreads the four icons out evenly.

Both phone bars have `md:hidden`, and both sidebars have `hidden md:block`. They switch at the same width, so at every screen size, exactly one of the two kinds of navigation is showing.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Narrow your browser window. You should see a top navbar with "Photogram (Industrial)" and the bottom navbar with Feed, Discover, Profile, and Settings icons. Widen the browser → the sidebars reappear and the phone navigation hides.
</div>

![The feed page side by side: on a wide screen with both sidebars, and on a phone-sized screen with the top navbar and the bottom navigation icons](/assets/layout-wide-and-narrow.png)

Now would be a good time for a commit:

```
git add -A
git commit -m "Added mobile navbar and bottom navigation"
git push
```

## Creating the UsersController

Click "Discover" in the sidebar. You'll see an error page that says `AbstractController::ActionNotFound` and `The action 'discover' could not be found for UsersController`. The route exists, and so does the controller, but the controller has no `discover` action yet. We stubbed `feed` earlier, but we still need the rest of the `UsersController` actions. Unlike the other controllers, which were generated by scaffolds, the `UsersController` needs custom actions that don't follow the standard CRUD pattern.

Open `app/controllers/users_controller.rb` (the file we created earlier with just the `feed` stub). Let's build it up piece by piece.

### The `set_user` before_action

Add a `before_action` and a `private` method that looks up users by username:

```ruby{2,7-11}
class UsersController < ApplicationController
  before_action :set_user, only: %i[ show follows followers ]

  def feed
  end

  private

    def set_user
      @user = User.find_by!(username: params.fetch(:username))
    end
end
```
{: filename="app/controllers/users_controller.rb" }

Our new method `set_user` runs before `show`, `follows`, and `followers`, the three actions whose routes have a `:username` in them. It finds the user with that username (like `alice` in `/alice/followers`) in the database.

The `find_by!` method (with a bang!) raises an `ActiveRecord::RecordNotFound` exception if no user is found, which Rails automatically converts to a 404 error page. This is better than `find_by` (without the bang), which would return `nil` and cause confusing `NoMethodError` crashes later in the action.

### The `index` action

Add the `index` action now:

```ruby{4-6}
class UsersController < ApplicationController
  before_action :set_user, only: %i[ show follows followers ]

  def index
    @users = @q.result
  end

  def feed
  end
  # ...
```
{: filename="app/controllers/users_controller.rb" }

This is the search results page. Remember that `@q` is the Ransack search object set up in `ApplicationController`. Calling `.result` on it runs the search query and returns matching users. If no search params are present, it returns all users.

<aside>
Notice that `set_user` doesn't run before the `index` action. That's intentional, because the search results page doesn't operate on a single user.
</aside>

### Feed and Discover

Update the `feed` action (replacing our earlier stub) and add `show` and `discover`:

```ruby{6-15}
  # ...
  def index
    @users = @q.result
  end

  def show
  end

  def feed
    @photos = current_user.feed
  end

  def discover
    @photos = current_user.discover
  end

  # ...
```
{: filename="app/controllers/users_controller.rb" }

These actions use the `has_many :through` associations we built in the previous lesson. `current_user.feed` returns all photos posted by people the current user follows, and `current_user.discover` returns all photos liked by people they follow. The `/feed` and `/discover` URLs have no username in them, so these actions use `current_user` rather than `@user`. The model does the work of finding the photos; the controller just passes them to the view.

### Follows, Followers, and Pending

Add the remaining three actions after `discover`:

```ruby{6-16}
  # ...
  def discover
    @photos = current_user.discover
  end

  def follows
    @follows = @user.leaders
  end

  def followers
    @followers = @user.followers
  end

  def pending
    @pending = current_user.pending_received_follow_requests
  end

  private
  # ...
```
{: filename="app/controllers/users_controller.rb" }

Each action uses an association we defined on the User model. `leaders` gives us the people `@user` follows, `followers` gives us the people who follow `@user`, and `pending_received_follow_requests` gives us the follow requests that the current user hasn't responded to yet.

## Stub view templates

Click "Discover" again. This time, the error page says `ActionController::MissingExactTemplate` and `UsersController#discover is missing a template for request formats: text/html`. The action exists now, but it has no view template to render. Let's create basic stubs so every page loads. We'll build them out with real content in the next lesson.

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

<i class="fa-solid fa-circle-check"></i> Click every link in the sidebar: Feed, Discover, Profile, and Settings should all load! Click your avatar → "Go to profile" → the profile page loads with the username. Visit `/pending`, `/alice/followers`, and `/alice/follows` in your browser bar. The entire app is navigable!
</div>

Make a commit when you see everything is working:

```
git add -A
git commit -m "Created UsersController and stub view templates"
git push
```

## Wrapping up

You now have a navigable app with working authentication, a responsive three-column layout, phone navigation, a photo upload modal, and controllers that set the current user and send people back to where they were. Every link in the sidebar works, and you can create photos from the modal.

The user pages show stub content for now. We'll replace those stubs with real view templates (photo cards, user profiles, follower lists, and more) in the next lesson.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Run `rspec`. You should see `66 examples, 42 failures`. The 22 model tests still pass, and so do two of the sign-in tests: the ones that check that a signed-out visitor is sent to the sign-in page. Most of the other tests sign in by clicking a "Sign in" button, and Devise's sign-in form says "Log in" until we customize it in the next lesson. Run `grade`, so that Grades records the passing tests.
</div>

---

- Approximately how long (in minutes) did this lesson take you to complete?
{: .free_text_number #time_taken title="Time taken" points="1" answer="any" }

<!--

# List of project specs for AI assistant

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

describe "/" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page.status_code).to be(200)
  end

  it "has a navbar", points: 1 do
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

    expect(page).to have_current_path("/feed")
  end

  it "has a link, 'Discover', that navigates to the 'Discover' page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Discover"

    expect(page).to have_current_path("/discover")
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

describe "/discover" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/discover"

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

    visit "/discover"

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

describe "/feed" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/feed"

    expect(page.status_code).to be(200)
  end

  it "shows their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader, caption: "leader caption")
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/feed"

    expect(page).to have_content(photo.caption)
    expect(page).to have_tag("img")
  end

  it "allows them to like their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/feed"

    click_on "0 likes"

    expect(page).to have_tag("i", with: { class: [ "fa-solid", "fa-heart" ] })
  end

  it "allows them to un-like their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    Like.create(fan_id: user.id, photo_id: photo.id)

    visit "/feed"

    click_on "1 like"

    expect(page).to have_css("i.fa-regular.fa-heart")
  end

  it "allows the user to add a comment on their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/feed"

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

    visit "/feed"

    within("li", text: comment.body) do
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

    visit "/feed"

    within("li", text: comment.body) do
      click_on "Edit"
    end

    fill_in "comment[body]", with: "Edited comment"
    click_button "Update Comment"

    expect(page).to have_content("Edited comment")
  end

  it "shows an alert instead of saving a blank comment", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/feed"

    click_button "Create Comment"

    expect(page).to have_content("Body can't be blank")
    expect(page).to have_current_path("/feed")
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

-->
