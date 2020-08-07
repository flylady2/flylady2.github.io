---
layout: post
title:      "Rails ActionMailer Makes it Easy to Communicate"
date:       2019-09-18 00:47:44 +0000
permalink:  rails_actionmailer_makes_it_easy_to_communicate
---


My Rails project is a lab supply tracker.  In my other life I am a biologist, and one of the most difficult logistic problems in our lab is managing supplies.  It’s so frustrating to be starting an experiment only to find out that someone used up most (or all!) of a reagent that you need and did not inform the person in charge of ordering supplies that more was needed. The Lab Supply Tracker app will close that gap.  One of the features of Rails that I was excited about implementing was the Mailer, which I wanted to use as the means for lab administrators to be notified when a reagent was needed. When a user uses a reagent, they will record their use in the app by creating a reagent_use record, which notes the reagent and the quantity used.  The creation of the reagent_use object will call a method that calculates how much of the reagent will be left after this use.  If the remaining amount is less than the “trigger” amount set for that reagent by lab admins, then an email will be sent to lab admins informing them that they need to order more of that particular reagent.  

To get the Mailer set up and working I first started with a welcome email that is sent to new users after they sign up for the app.  That posed no problems, but when I added in the trigger email I got this error: ActionDispatch::Cookies::CookieOverflow.  The Rails default is a cookie-based session store, which is fast but limited to a 4K cookie size, which was exceeded when my trigger email was added to the session.  I’ll explain how I dealt with that as well as describe setting up the mailer.

**ActionMailer**

You will need to have an email account that can serve as the sender.  It’s a good idea to create a separate account, on Gmail, for example, just for this purpose, because you will probably need to allow “less secure app access” in your account settings. 

The first step is to generate the mailer (assuming you already have a rails app) by typing 

`rails generate UserMailer ` 

into your terminal.  This command will generate a UserMailer and associated files.  You can set up as many Mailers as you want, with whatever names you want.  I also made an AdminMailer for emails that are sent specifically to users with admin status.

A mailer is conceptually similar to a controller.  When you generate a mailer in the terminal it will simultaneously a create a Mailer, a mailer directory in the views folder where you will compose your emails, and two files in the test folder, mailer_preview.rb and mailer_test.rb.  All of these will be specific for the mailer you created and have the associated name.  For example, user_mailer or admin_mailer.

Rails apps contain a file, `application_mailer.rb`, that is analogous to application_controller and is generated as part of the basic Rails app.  You can set default values here to apply to all Mailers.  When you open up `application_mailer.rb` you will see:

```
class ApplicationMailer < ActionMailer::Base
  default from: ”from@example.com”
  layout 'mailer'
end
```
Change `from@example.com` to whatever email address you will be using.  If you want, you can put the name of your app.  If you do that, in the email the recipient receives they will see the name of the app followed by the email address you are using as your sender.  

You can override this default from: to be different in individual Mailers as well as in individual emails, as I will describe later.  But all emails will show the sending email address as the one that is specified in the config/environment/development.rb file, so let’s set that up now.  

Open up `config/environment/development.rb` and add the following lines of code:

` config.action_mailer.perform_deliveries = true`
 - By default, deliveries are carried out when the deliver method is invoked on the Mail message.  You can change this setting to false if you are testing and do not want the emails to be delivered.

`config.action_mailer.raise_delivery_errors = true`
- This setting will raise errors if there is a delivery problem with the email.

If you are using a gmail account as the sender, you will want to use the :smtp (Simple Mail Transfer Protocol) delivery method with the following settings:

```
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.smtp_settings = {
    address: 'smtp.gmail.com',
    port: 587,
    domain: 'gmail.com',
    user_name: 'your gmail account username',
    password: ‘your gmail account password’,
    authentication: 'plain',
    enable_starttls_auto: true
  }
```
NOTE: Even if you’ve made a separate gmail account just for this purpose, you probably don’t want to push your user name and password up to GitHub.  An easy way to avoid this is to store those values in your .env file as:

```
GMAIL_USERNAME=your username 
GMAIL_PASSWORD=your password  
```

Then, for user_name and password in the development.rb file, put

`user_name:  ENV['GMAIL_USERNAME']`,
`password: ENV['GMAIL_PASSWORD']`


**If you don’t already have a `.env` file:**  
`ENV` is a constant that refers to a hash for the entire computer environment and you can store any key-value pairs in it.  But instead of setting those values directly in the `ENV` hash, it’s easier to use the `dotenv-rails` Gem to do it.  To use this gem, follow these steps:

Add `dotenv-rails` to your `Gemfile`.
Run bundle install.
Create a `.env` file at the root of your application.
Add the username and password values described above to the `.env` file.
Add `.env` to your `.gitignore` file.  This will prevent those values from being pushed to GitHub.

**Email Views**

Templates for the UserMailer emails are created in the `app/views/user_mailer` folder.  Each email should have an html as well as a text version for people who prefer that.  Below are shown the two different files for the welcome_email sent by Lab Supply Tracker.  The message content that the recipient sees is identical.  

```
welcome_email.html.erb
<!DOCTYPE html>
<html>
  <head>
    <meta content='test/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>Welcome to Lab Supply Tracker, <%= @new_user.name %></h1>
</body>
</html>
```

and
 
 
 ```
welcome_email.text.erb
Welcome to Lab Supply Tracker, <%= @new_user.name %>
```


Now that we have our environment settings and the text of the `welcome_email`, we can define the `welcome_email` method in `user_mailer.rb`:

```
class UserMailer < ApplicationMailer
  def welcome_email
    @new_user = params[:new_user]
    mail(to: @new_user.email, subject: 'Welcome to Lab Supply Tracker')
  end
end
```

and call the `welcome_email` method on the `UserMailer` in the users controller after a new user is saved: 

```
if @user.save
    session[:user_id] = @user.id
    UserMailer.with(new_user: @user).welcome_email.deliver_now
```
**Things to note:**

-The `welcome_email` method returns an ActionMailer::MessageDelivery object.  If you instruct it to deliver_now, the email will be sent immediately.  You can also tell it to deliver itself at a specified later time by using deliver_later; check out the docs [https://guides.rubyonrails.org/action_mailer_basics.html](http://).   to learn how to utilize this feature. 

-When a key/value pair such as (`new_user: @user`) is passed to `with`, it becomes the params for the mailer action.  In this case, it makes `params[:new_user]` available in the mailer action, similar to the use of params in controllers.  `@new_user` is defined as `params[:new_user]` in the `welcome_email` method and that serves to identify `@new_user` in the actual email texts shown above.  To demonstrate that it is not the same instance variable being used in the UsersController and UsersMailer, I equated `@user`, the instance variable for the newly created user, to `new_user`.  Then I set the `@new_user` instance variable to equal `params[:new_user]`.  So although `@new_user` and `@user` refer to the same User object, the information is being passed as a param to the mailer. 

-The `mail `method accepts a headers hash that allows you to specify the most commonly used headers in an email message, which are: 
`:subject, :to, :from, :cc, :bcc, :reply_to`, and `:date.`  

In the example shown above, the default` :from` is already set in ApplicationMailer, but you can define `:from` in the header to override the default setting, if desired.

`:to` can be set as multiple emails, such as an array of strings.  For example, in the emails that my app sends to the members of a lab with admin status, I obtain their emails using a scope method to retrieve the admin members, and then use `pluck` to pull out their emails (see below), which produces an array of email addresses.

 ```
 def admin_emails
    self.users.where(admin: true).pluck(:email)
  end
```

`:to` can also be set as a default (and overridden in the header). But note that the default address or addresses must belong to objects that already exist or you will get an error.  I could not use the emails returned by `admin_emails` as a default value for :to because the method is called on a specific lab that needs to be defined in the controller.  If I instead set if for all admin users of the app it would work.  

**How to preview your emails**

This function uses the `mailer_preview` files to allow you to see how a given email will appear to the recipient.  To preview the welcome email, I have the following code in the `user_mailer_preview.rb` file:

```
class UserMailerPreview < ActionMailer::Preview
  def welcome_email
    UserMailer.with(new_user: User.first).welcome_email
  end
end
```

This will generate the `welcome_email` with `User.first` set as `new_user`.  As with the default values described above, `new_use`r has to be defined as something that already exists, so it can’t be set to `@user`, which would be dependent on the controller action.  To see how the emails in the `user_mailer_preview.rb` file will look, go to [http://localhost:3000/rails/mailers/user_mailer ](http://)in your browser.

These basics as well as a lot of other useful information, such as adding attachments, can be found in the docs [https://guides.rubyonrails.org/action_mailer_basics.html](http://).  

**Active Storage**

When I got the ActionDispatch::Cookies::CookieOverflow error after implementing the `trigger_email`, I did some googling and discovered that some people had solved this problem by switching from cookie store to ActiveStorage, using the `activerecord-session_store` gem ([https://github.com/rails/activerecord-session_store](http://)).  This creates a table to store session data that is backed by ActiveRecord.  

Here’s how I implemented ActiveStorage:

I first created a `config/initializers/session_store.rb` file and added this line to it:

`Rails.application.config.session_store :active_record_store`

Then I added the `activerecord-session_store` gem to my Gemfile and ran bundle install.

Then I typed these commands into the terminal:

```
bundle exec rails g active_record:session_migration
bundle exec rails db:migrate
```

And once I had done that, the trigger_email worked.

Happy emailing!




