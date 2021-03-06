footer:@johnsonch :: Chris Johnson :: Ruby on Rails Development :: Week 11
autoscale: true

#Ruby on Rails Development
##Week 11

* https://codecaster.io room: johnsonch

---
#Demo
* cd into ```wolfie_books```
* ```$ git add . ```
* ```$ git commit -am 'commiting files from in class'```
* ```$ git checkout master```
* ```$ git fetch```
* ```$ git pull ```
* ```$ git checkout week11_start```
* ```$ rm -f db/*.sqlite3```
* ```$ bundle```
* ```$ bundle exec rake db:migrate```
* ```$ bundle exec rake test``` We have a  couple failing tests from last week
* ```$ bundle exec rake fake:all_data```

---
#Account activation
##Activation

* Generate AccountActivations controller

```bash
$ bundle exec rails generate controller AccountActivations
```

* Add a route for our controller

```ruby
resources :account_activations, only: [:edit]
```

* Add our model additions for activation

```bash
$ bundle exec rails generate migration add_activation_to_users activation_digest:string activated:boolean activated_at:datetime
```

* Create a new activation token for each user creation

```ruby
  before_create :create_activation_token

  private

    def create_activation_token
      cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST : BCrypt::Engine.cost
      self.activation_token  = SecureRandom.urlsafe_base64
      self.activation_digest = BCrypt::Password.create(self.activation_token, cost: cost)
    end
```

* We'll need to be able to access our activation_token

```ruby
attr_accessor :activation_token
```

* Let's keep working through the process and send out an email

```bash
$ bundle exec rails generate mailer UserMailer account_activation password_reset
```

* Next make the activation mailer take a user object

```ruby
  def account_activation(user)
    @user = user
    mail to: user.email, subject: "Account activation"
  end
```

* Update the 'views' for account activation

###```app/views/user_mailer/account_activation.text.erb```


```erb
Hi <%= @user.name %>,

Welcome to the Wolfie Books! Click on the link below to activate your account:

<%= edit_account_activation_url(@user.activation_token, email: @user.email) %>
```

###```app/views/user_mailer/account_activation.htm.erb ```

```
<h1>Wolfie's List</h1>

<p>Hi <%= @user.name %>,</p>

<p>
Welcome to the Wolfie Books! Click on the link below to activate your account:
</p>

<%= link_to "Activate", edit_account_activation_url(@user.activation_token, email: @user.email) %>
```

* Next to test our emailing we'll need to configure our development.rb file

```
  config.action_mailer.default_url_options = { :host => 'https://matc-rails-fall-2015-johnsonch.c9.io' }
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.raise_delivery_errors = true
```

* We need to get our SMTP settings in place, we're going to use a gem called [figaro](https://github.com/laserlemon/figaro), we'll add it to our gemfile:

```ruby
gem "figaro"
```

* Then we'll use the follwing command to get a generated application.yml file:

```bash
$ bundle exec figaro install
```

* We'll add the following snippet to your application.yml, your instructor will use codecaster to send you the senstivie information. Also to note this application.yml file should never be commited, you don't want this sensitive informatin published out to sources other people can look at.

```yaml
development:
  smtp_server:
  smtp_user:
  smtp_password:
```

* Next we'll add an initializer to setup our email settings

###```config/initializers/smtp.rb````

```ruby
ActionMailer::Base.smtp_settings = {
      :address              => ENV['smtp_server'],
      :port                 => 587,
      :user_name            => ENV['smtp_user'],
      :password             => ENV['smtp_password'],
      :authentication       => 'login',
      :enable_starttls_auto => true,
      :openssl_verify_mode => 'none'
}
```

* For this to work on Heroku we'll need to set environment variables, this is something to note for your own environment.

```bash
$ heroku config:set SMTP_SERVER=my.mail.server
```


* Then we'll add a method to authenticate our users

```ruby
def authenticated?(attribute, token)
  digest = send("#{attribute}_digest")
  return false if digest.nil?
  BCrypt::Password.new(digest).is_password?(token)
end
```

* Now we can add the edit action to our activation controller

```ruby
  def edit
    user = User.find_by(email: params[:email])
    if user && !user.activated? && user.authenticated?(:activation, params[:id])
      user.update_attribute(:activated, true)
      user.update_attribute(:activated_at, Time.zone.now)
      log_in user
      flash[:success] = "Account activated!"
      redirect_to user
    else
      flash[:danger] = "Invalid activation link"
      redirect_to root_url
    end
  end
```
* Let's send out an email to the user when they are created

```ruby
  if @user.save && UserMailer.account_activation(@user).deliver_now
    flash[:info] = "Please check your email to activate your account."
    redirect_to root_url
  ....
```

* Don't let users who haven't activated login

```ruby
if user.activated?
  log_in(user)
  redirect_to(user)
else
  message  = "Account not activated. Check your email for the activation link."
  flash[:warning] = message
  redirect_to root_url
end
```

---
## Next we'll add WillPaginate

```ruby
gem 'will_paginate', '~> 3.0.6'
```

* Then we'll need to install the gem

```bash
$ bundle install
```

* Simply adjust the index action you want to paginate

```ruby
  def index
    @projects = Project.paginate(:page => params[:page], :per_page => 10)
  end
```

* Then add the page picker helper to your index page

```erb
<div class="row">
  <div class="col-md-12">
    <%= will_paginate @projects %>
  </div>
</div>
```
