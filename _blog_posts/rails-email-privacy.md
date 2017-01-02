---
title: Hiding Email Addresses with Devise
layout: default
---

# Hiding email addresses with Devise

## The problem

At EFF, we take user privacy seriously. A visitor should not be able to figure out whether an email is registered to a user of the application.

Our Action Center Platform revealed which email addresses were registered with the application. Let's say a nosy visitor wanted to know if their friend, user@example.com, was an EFF supporter. They could visit the app and attempt to register a new user with the email user@example.com. If their registration failed, they would know user@example.com was already a user.

**Screenshot of failed registration with email taken message**
[Bad: Visitors can figure out which email addresses are registered with the Action Center.]

A better approach is to show the same message to all users who attempt to register: a prompt to check their inbox for a confirmation message.

If the e-mail address isn't registered yet, we'll send an email with confirmation instructions.

If the e-mail address is taken, we'll send a notice that a user attempted to create an account with that e-mail address. They'll also get a password reset link in case they forgot about their existing account.

That way, only the owner of that e-mail address can find out if the address is already registered.

**Screenshot of registration with email confirmation notice**
[Good: Only the owner of an e-mail address can find out if it's already been registered.]

## Devise

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
*Devise's implementation of the `create` action for the registrations controller*

Devise builds a new user based on the the submitted params and saves it. If the save is successful, the user is sees a success message. Otherwise they're redirected to the signup page and shown an error. There's also a `yeild` statement on line 6 - we'll come back to that soon.

## Overriding the default

We can override Devise's default create action by creating our own `RegistrationsController` that inherits from Devise:

{% highlight ruby linenos %}
class RegistrationsController < Devise::RegistrationsController

end
{% endhighlight %}
*app/controllers/registrations_controller.rb*

At this point, we could override the `create` and `update` actions from the original controller and be done. But we'd have to duplicate a lot of logic from the original controller - nothing changes for users who register with a password that isn't already in use.

We could check for a duplicate email and then call `super` to run the parent method defined by Devise. Unfortunately, this will cause problems if the registration attempt is invalid for some other reason as well. Users who submit an invalid password, for example, would see a message saying to check their e-mail for a confirmation link. They should see an invalid password error.

We need to validate the new user before we decide how to proceed. Ideally we'd like to avoid validating them in our new create method, since they'll be validated a second time in the parent method.

Luckily, Devise provides use with a helpful `yield` statement in the original controller. We can pass a code block to `super` which will be executed after validation, but before setting a success or failre message - exactly when we want do handle a duplicate email address.

Our new controller looks like this:

{% highlight ruby linenos %}
class RegistrationsController < Devise::RegistrationsController

  # POST /resource
  def create
    super do |resource|
      if resource.email_taken?
        # Show a fake success message, send a password reset token, and return.
      else
        # Proceed as usual.
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

We've validated the user. Now we need to handle the case where a user already exists with the provided email address.

We start by deleting the email validation error. This will prevent the view from rendering that error message and placing an error class on the email field.

Next, we check of the new account failed validation for any other reason. If not, we send the user a notice that someone tried to register with their address.

Finally, shouldn't continue with the `super` method if our helper already rendered a success page. `ActionController::Metal#performed?` lets us test whether a render or redirect has happended. If it has, we return.

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

If the user hasn't confirmed their account yet, they should get a new confirmation e-mail instead of a password reset link.

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

Our `create` action is working great - time to turn our attention to the `update` action. The update action is called when users attempt to change the e-mail address or password for their account.

When users attempt to change their e-mail address, a confirmation e-mail is sent to the new address. Meanwhile they continue to use their original e-mail address. The new address is stored as "unconfirmed" until they click the confirmation link.

By default, users who attempt to change their e-mail address to an address that's already taken see an error message. We need to override Devise so that they have the same experience as users who submit a unique e-mail address.

![Screenshot of confirmation screen](screenshot.png)
*Users continue to use their old email address until they confirm the new one.*

![Screenshot of confirmation screen](screenshot.png)
*Users continue to use their old email address until they confirm the new one.*

Devise's implementation of the `update` action is similar to `create`. Our overridden version is identical to our overriden `create` action.

Our final change is to update the `handle_nonunique_email` helper. If we're updating an existing resource (in other words, if the resource is persisted) we set the unconfirmed email manually to email that was submitted by the user. We display a message letting them know they need to confirm the new email.

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
