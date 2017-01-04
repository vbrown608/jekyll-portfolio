---
title: Better privacy for Devise
date: 2017-01-03T10:20:00Z
keywords: Rails 4, Devise 3.5
summary: Extend Devise to prevent an attacker from discovering which email addresses are registered with a Rails application.
layout: post
---

## The problem

On EFF's Action Center Platform, users register for an account using their e-mail address. We require each email addresse to be unique - if an email address is already taken by another user, registration should fail.

This created a problem for us: let's say an attacker wanted to know if alice@example.com was an EFF supporter. They could visit the app and attempt to register a new account with the email address alice@example.com. If their registration failed, they would know alice@example.com was already a user.

![Screenshot of the user registration page with bad email privacy setup. Users see an error message letting them know the email they tried to register is already taken.](/images/signup-bad-privacy.png){:class='center'}
*__Bad:__ Visitors can figure out which email addresses are registered with the Action Center.*{:class='caption'}

A better approach is to show the same message to all users who attempt to register: a prompt to check their inbox for a confirmation message.

If the e-mail address isn't registered yet, we'll send an email with confirmation instructions.

If the e-mail address is already taken, we'll send a notice that another visitor attempted register with that address. We'll also send a password reset link in case the user simply forgot about their existing account and attempted to register a second time.

Because the messages are sent by email, only the owner of that e-mail address can find out if the address it's registered.

![Screenshot of the user registration page with a better email privacy setup. After attempting to register, users see a message that says, "A message with the confirmation link has been sent to your email address."](/images/signup-good-privacy.png){:class='center'}
*__Good:__ Only the owner of an e-mail address can figure out if it's already been registered.*{:class='caption'}

## Devise's default behavior

We use the [Devise](https://github.com/plataformatec/devise) gem for Rails to handle user authentication. Devise doesn't support this feature out of the box, so we need to extend it.

Devise provides a [controller for user registration](https://github.com/plataformatec/devise/blob/master/app/controllers/devise/registrations_controller.rb). We're concerned with the `create` and `update` actions, which give users the opportunity to enter an email address.

Let's take a look at the default `create` action provided by Devise:

{% highlight ruby linenos %}
# POST /resource
def create
  build_resource(sign_up_params)

  resource.save
  yield resource if block_given?
  if resource.persisted?
    if resource.active_for_authentication?
      set_flash_message! :notice, :signed_up
      sign_up(resource_name, resource)
      respond_with resource, location: after_sign_up_path_for(resource)
    else
      set_flash_message! :notice, :"signed_up_but_#{resource.inactive_message}"
      expire_data_after_sign_in!
      respond_with resource, location: after_inactive_sign_up_path_for(resource)
    end
  else
    clean_up_passwords resource
    set_minimum_password_length
    respond_with resource
  end
end
{% endhighlight %}
*Devise's implementation of the create action for the registrations controller*

Devise builds a new user based on the the submitted params and saves it. If the save is successful, the user is sees a success message. Otherwise they're redirected to the signup page and shown an error. There's also a `yeild` statement on line 6 - we'll come back to that soon.

## Overriding the default

We can override Devise's default `create` action by creating our own `RegistrationsController` that inherits from Devise:

{% highlight ruby linenos %}
class RegistrationsController < Devise::RegistrationsController

end
{% endhighlight %}
*app/controllers/registrations_controller.rb*

At this point, we could override the `create` and `update` actions from the original controller and be done. But we'd have to duplicate a lot of logic from the parent controller - nothing changes for users who register with an e-mail that isn't already in use.

Another approach might be to check for a duplicate email address and then call the parent method via `super`. If the address is a duplicate, we'd set a fake success message and email a password reset token. If it's not, we'd proceed with the parent method.

Unfortunately, that would cause problems if there's an email uniqueness validation error _and_ some other validation error. Users who submit a duplicate email as well as an invalid password, for example, should see an invalid password error. They should have the same experience as a user who submits a unique email address. But in the system described above, they would see a fake success message instructing them to check their email.

We should only set a fake success message if email validation fails _and_ all other checks succeed. In other words, we need to validate the entire record first.

Luckily, Devise provides us with a helpful `yield` statement in the original controller. We can pass a code block to `super` which will be executed after validation, but before setting a success or failure message - exactly when we want to handle a duplicate email address.

Our new controller looks like this:

{% highlight ruby linenos %}
class RegistrationsController < Devise::RegistrationsController

  # POST /resource
  def create
    super do |resource|
      if resource.email_taken?
        # Show a fake success message, send a password reset token, and return.
      else
        # Continue executing the parent method.
      end
    end
  end

end
{% endhighlight %}
*app/controllers/registrations_controller.rb*

{% highlight ruby linenos %}
class User < ActiveRecord::Base
  devise :confirmable

  def email_taken?
    errors.added? :email, :taken
  end

end
{% endhighlight %}
*app/models/user.rb*


## Handling a nonunique email

We've successfully injected our code block into the parent method. Now we need to handle the case where a user already exists with the provided email address.

We start by deleting the email validation error. This will prevent the view from rendering that error message or placing an error class on the email field.

Next, we check if the new account failed validation for any other reason. If not, we send the existing user a notice that someone tried to register with their address and show a fake success message.

Finally, we shouldn't continue with the `super` method if our helper already rendered a success page. `ActionController::Metal#performed?` lets us test whether a render or redirect has happended. If it has, we return.

{% highlight ruby linenos %}
class RegistrationsController < Devise::RegistrationsController

  # POST /resource
  def create
    super do |resource|
      handle_nonunique_email if resource.email_taken?
      return if performed?
    end
  end

  private

  def handle_nonunique_email
    resource.errors.delete(:email)

    if resource.errors.empty?
      User.find_by_email(resource.email).send_email_taken_notice
      respond_with resource, location: after_inactive_sign_up_path_for(resource)
    end
  end

end
{% endhighlight %}
*app/controllers/registrations_controller.rb*

Sending a signup attempt notice and password reset link is handled by the `User` model. If the existing user hasn't confirmed their account yet, they should get a new confirmation e-mail instead of a password reset link.

{% highlight ruby linenos %}
class User < ActiveRecord::Base
  devise :confirmable

  def email_taken?
    errors.added? :email, :taken
  end

  def send_email_taken_notice
    if self.confirmed?
      # Send a notification and a password reset link, for example:
      # UserMailer.signup_attempt_with_existing_email(self).deliver_now
    else
      send_confirmation_instructions
    end
  end

end
{% endhighlight %}
*app/models/user.rb*


## Overriding the update action

Our `create` action is working great - time to turn our attention to the `update` action. The `update` action is called when users attempt to change the e-mail address or password associated with their account.

When users attempt to change their e-mail address, a confirmation link is sent to the new address. Their account continues to be associated with their old address until the confirm the new one. The new address is stored as "unconfirmed" until they click the confirmation link.

![A screenshot of the edit account information screen. After attempting to change their email address, a user sees a message that says, "You updated your account successfully, but we need to verify your new email address. Please check your email and click on the confirm link to finalize confirming your new email address."](/images/update-email-good-privacy.png){:class='center'}
*Users should be prompted to confirm their new email address, even if the address they entered is taken.*{:class='caption'}

By default, users who attempt to change their e-mail address to an address that's already taken see an error message. We need to override Devise so that they have the same experience as users who submit a unique e-mail address.

Devise's implementation of the `update` action is similar to `create`. Our overridden version is identical to our overriden `create` action.

Our final change is to update the `handle_nonunique_email` helper. If we're updating an existing record (that is, if the record is persisted):

1. We update the user's `unconfirmed_email` attribute with the new address.
2. We set a fake success message instructing the user to check their email.
3. We render the response.

{% highlight ruby linenos %}
class RegistrationsController < Devise::RegistrationsController

  # POST /resource
  def create
    super do |resource|
      handle_nonunique_email if resource.email_taken?
      return if performed?
    end
  end

  # PUT /resource
  def update
    super do |resource|
      handle_nonunique_email if resource.email_taken?
      return if performed?
    end
  end

  private

  def handle_nonunique_email
    resource.errors.delete(:email)

    if resource.errors.empty?
      User.find_by_email(resource.email).send_email_taken_notice

      if resource.persisted?
        resource.update_attribute(:unconfirmed_email, account_update_params[:email])
        flash[:notice] = I18n.t "devise.registrations.update_needs_confirmation"
        respond_with resource, location: after_update_path_for(resource)
      else
        respond_with resource, location: after_inactive_sign_up_path_for(resource)
      end
    end
  end

end
{% endhighlight %}
*app/controllers/registrations_controller.rb - final version*
