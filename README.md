# Authentication module for web.py 0.3

This is an authentication module for [web.py](http://www.webpy.org), code originally by [jpscaletti](https://github.com/jpscaletti).

The module allows you to:

* Limit access to pages based on if the user is logged in, if is authorized (by checking against a table of permissions) or meet certain conditions.

* Generate and manage password hashes. **Sha1**, **sha512** and **bcrypt** are supported.

* Authenticate a user by checking a login and password against a database of users.

* Generate and validate tokens for passwords resets.

* CAPTCHA validation(Only visual CAPTCHAs) (**New**)

It also includes default pages to login, generate a password-reset token, email the token and set a new password after the token is validated.

## Installation

1. Put the auth package in the `web/contrib/` folder.

2. Create the following tables:

    ```sql
    CREATE TABLE user (
        user_id             int NOT NULL AUTO_INCREMENT PRIMARY KEY,
        user_login          varchar(64) NOT NULL,
        user_password       varchar(255) NOT NULL,
        user_email          varchar(64),  # Optional, see settings
        user_status         varchar(16) NOT NULL DEFAULT 'active',
        user_last_login     datetime NOT NULL
    )

    CREATE TABLE permission (
        permission_id           int NOT NULL AUTO_INCREMENT PRIMARY KEY,
        permission_codename     varchar(50),  # Example: 'can_vote'
        permission_desc         varchar(50)   # Example: 'Can vote in elections'
    )

    CREATE TABLE user_permission (
        up_user_id          int REFERENCES user (user_id),
        up_permission_id    int REFERENCES permission (permission_id),
        PRIMARY KEY (up_user_id, up_permission_id)
    )
    ```

3. Generally use in your code:

    ```python
    import web
    from web.contrib.auth import auth

    urls = (
        '/', 'index',
    )
    app = web.application(urls, locals())
    db = web.database(dbn='mysql', db='webpy', user='scott', pw='tiger')
    settings = {}

    auth.init_app(app, db, **settings)
    ```

    The system will create and use a **DiskStore** session. If you want to use an existing one or another type of session, you pass it as an argument.

    ```python
    mysession = web.session.Session(app, web.session.DiskStore('sessions'))
    auth.init_app(app, db, mysession, **settings)
    ```

4. Enable CAPTCHA validation

   To enable the [CAPTCHA](https://en.wikipedia.org/wiki/CAPTCHA) validation, you need an extra CAPTCHA module which could generate the CAPTCHAs(visual CAPTCHAs) and the text-based check-code(can be input by user on the authentication page). You should use a wrapper function, which has **No** parameters and exact **TWO** return values. One is the CAPTCHA image, an ```str``` instance, the value or content of a ```cStringIO.StringO``` or ```StringIO.StringO``` object(see [here](https://docs.python.org/2/library/stringio.html) for details), whose file type determines the value of ```captcha_image_type```. The other one is the CAPTCHA check-code string. Then, pass the wrapper function and the image type as arguments.

   ```python
   from Captcha import captcha_func

   settings = dict({
       'captcha_func': captcha_func,
       'captcha_image_type': 'png',
   })

   auth.init_app(app, db, **settings)
   ```

## Usage

Once you initiated the **auth** application a number of methods are available:

<dl>
<dt>@auth.protected(**pars)</dt>
<dd>
Decorator for limiting the access to pages.<br>
Examples:<br>

Limiting access to authenticated users
<pre>
class SomePage:
    <strong>@auth.protected()</strong>
    def GET(self):
        ...
</pre>

Limiting access to users with a specific permission

<pre>
class SomePage:
    <strong>@auth.protected(perm='can_edit')</strong>
    def GET(self):
        ...
</pre>

Limiting access to users who pass a test

<pre><strong>
def over18(user):
    return user.age > 18</strong>

class SomePage:
    <strong>@auth.protected(test=over18)</strong>
    def GET(self):
        ...
</pre>

Limiting access to users who need to pass the CAPTCHA validation

<pre>
class SomePage:
    <strong>@auth.protected(captcha_on=True)</strong>
    def GET(self):
        ...
</pre>

If the CAPTCHA validation is failed or the user isn't authorized it'll be redirected to <code>settings.url_login</code> ('/login' by default).
</dd>

<dt>auth.authenticate(login, password)</dt>
<dd>
Validates the user's credentials. If they are valid returns a user object (minus the password hash) or <code>None</code> if not.

<pre>
user = auth.authenticate(login='john', password='secret')
if not user:
    return 'That's correct'
else:
    return 'Wrong!'
</pre>

This function does not log in the user. Use <code>auth.login()</code> for that.
</dd>

<dt>auth.login(user)</dt>
<dd>
Set the user as logged in.
</dd>

<dt>auth.check_password(password, stored_passw)</dt>

<dd>Returns a boolean of whether the password was correct.</dd>

<dt>auth.logout()</dt>

<dd>Flush the authenticated user session</dd>

<dt>auth.user_exist(login)</dt>

<dd>Return <code>True</code> if a user with that login already exist.</dd>

<dt>auth.create_user(login, password=None, perms=[], **data)</dt>
<dd>
Create a new user and returns the <code>user_id</code>.<br>

If password is None, it will marks the user as having no password (<code>check_password()</code> for this user will never return <code>True</code>).
</dd>

<dt>auth.set_password(login, password=None)</dt>
<dd>
Sets the password of an already existing user to the given raw string, taking care of the password hashing.
</dd>

<dt>auth.update_user(login, **data)</dt>
<dd>
Update the user's data taking care of the password hashing if one is provided.
</dd>

<dt>auth.get_user(login=None)</dt>
<dd>
Returns a user object (minus the password hash).<br>

If login is <code>None</code> returns the currently authenticated user object or <code>None</code> if there isn't one
</dd>

<dt>auth.pass_test(user=None)</dt>
<dd>
Return True if the user pass the test.<br>

test must be a function that takes a user object and returns <code>True</code> or <code>False</code>.
</dd>

<dt>auth.has_perm(perm, user=None)</dt>
<dd>
Return True if the user has the permission.<br>

Perm can be either a single permission (string) or a sequence of them.
</dd>

<dt>auth.get_permissions(user=None)</dt>
<dd>
Returns a list of permission strings that the user has.
</dd>

<dt>auth.create_permission(codename, desc)</dt>
<dd>
Creates a new permission. If the permission already exists it update the description.<br>

Example: <code>create_permission('new_posts', 'Can write new posts')</code>
</dt>

<dt>auth.delete_permission(codename)</dt>
<dd>
Deletes a permission
</dd>

<dt>auth.add_permission(perm, user_id)</dt>
<dd>
Assign an existing permission to a user.
</dd>

<dt>auth.remove_permission(perm, user_id)</dt>
<dd>
Removes the permission.
</dd>

<dt>auth.login_form(auth)</dt>
<dd>
A login form to be used inside other pages (like the home page).
</dd>
</dl>

## Automatic views

By default the system will map a **login**, **logout** and **password-reset** pages. This can be disabled in the settings.


## Settings

<dl>
<dt>hash_type = 'sha512'</dt>
<dd>
Either <strong>sha1</strong>, <strong>sha512 (default)</strong> or <strong>bcrypt</strong> — the algorithm used to perform a one-way hash of the password. Note that <strong>bcrypt</strong> is only supported on platforms that have the Python <a href="http://www.mindrot.org/projects/py-bcrypt/">py-bcrypt</a> module available.
</dd>

<dt>hash_depth = 12</dt>
<dd>
How many time the password is recursively hashed.
</dd>

<dt>db_email_field = 'user_email'</dt>
<dd>
Field of the user table that contains the email address. For instance, if you want to use the email as the username, you can change this to 'user_login'.
</dd>

<dt>password_minlen = 6</dt>
<dd>
Minimum password length. <code>set_password()</code> and <code>create_user</code> will raise an "AuthError, 'bad password'" if the password is shorter than this.
</dd>

<dt>forced_delay = 0.5</dt>
<dd>
Artificial delay (in seconds) to slow down brute-force attacks
</dd>

<dt>auto_map = True</dt>
<dd>
Disable the auto-magically generated pages for login, logout and password reset. If it is <code>True</code>, it will use the url_* and template_* settings.
</dd>

<dt>captcha_enabled = False</dt>
<dd>
CAPTCHA validation is disabled by default. You could enable it automatically by define an extra CAPTCHA generation module as described above.
</dd>

<dt> captcha_image_type = 'png'</dt>
<dd>
The default file type of the CAPTCHA image, which is decided by the module which generates the CAPTCHA image. You could set the value here without passing a dict argument <code>settings</code> that contains it.
</dd>

<dt>url_captcha = '/captcha'</dt>
<dd>
The URL for the CAPTCHA image.
</dd>

<dt>url_login = '/login'</dt>
<dd>
The URL for the log in page.
</dd>

<dt>url_logout = '/logout'</dt>
<dd>
The URL for the log out.
</dd>

<dt>url_after_login = '/'</dt>
<dd>
Go there after a successful login
</dd>

<dt>template_login = None</dt>
<dd>
A template function:

<pre>
render = web.template.render('/templates')
settings = dict(
    template_login = render.mylogin,
)
auth = DBAuth(app, db, **settings)
</pre>

If <code>None</code>, the default template will be used. See <code>web/contrib/auth/templates/login.html</code>
</dd>

<dt>url_reset_token = '/password_reset'</dt>
<dd>
The URL of the password-reset token generation page.
</dd>

<dt>url_reset_change = '/password_reset'</dt>
<dd>
The URL for the password change page.
</dd>

<dt>template_reset_token = None</dt>
<dd>
A template function. If None, the default template will be used. See <code>web/contrib/auth/templates/reset_token.html</code>
</dd>

<dt>template_reset_email = None</dt>
<dd>
A template function. If None, the default template will be used. See <code>web/contrib/auth/templates/reset_email.html</code>
</dd>

<dt>template_reset_change = None</dt>
<dd>
A template function. If None, the default template will be used. See <code>web/contrib/auth/templates/reset_change.html</code>
</dd>

<dt>reset_expire_after = 2</dt>
<dd>
Number of hours than the password-reset token will be valid.
</dd>

<dt>email_from = ''</dt>
<dd>
The password-reset email “From:” header.
</dd>
</dl>


## License (MIT)

This module is released under the MIT license (MIT), anyone is welcome to contribute.
