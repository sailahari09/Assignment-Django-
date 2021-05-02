# Assignment-Django-
Create a Django project

django-admin my project 
In the settings.py file of your project, include rest_framework and rest_framework.authtoken in INSTALLED_APPS. Set the authentication scheme for the Django REST Framework to TokenAuthentication.

mysite/settings.py
----

INSTALLED_APPS = [
	...
	'rest_framework',
	'rest_framework.authtoken',
	...
]

REST_FRAMEWORK = {
	'DEFAULT_AUTHENTICATION_CLASSES': (
		'rest_framework.authentication.TokenAuthentication',
	)
}
Optionally, you may add an AUTH_EMAIL_VERIFICATION setting to specify whether to enable email verification for new users on account registration/signup. Setting this to False will automatically verify newly created users.

Create a Django application for your user data. For example,

python manage.py startapp accounts
In the models.py file of your application, extend EmailAbstractUser, add custom fields, and assign objects to EmailUserManager(). For example,

accounts/models.py
----

from django.db import models
from authemail.models import EmailUserManager, EmailAbstractUser

class MyUser(EmailAbstractUser):
	# Custom fields
	date_of_birth = models.DateField('Date of birth', null=True, blank=True)

	# Required
	objects = EmailUserManager()
In the settings.py file of your project, include authemail and your application in INSTALLED_APPS. Set AUTH_USER_MODEL to the class of your user model. For example,

mysite/settings.py
----

INSTALLED_APPS = [
	...
	'rest_framework',
	'rest_framework.authtoken',
	'authemail',
	'accounts',
	...
]

AUTH_USER_MODEL = 'accounts.MyUser'
In the admin.py file of your project, extend EmailUserAdmin to add your custom fields. For example,

mysite/admin.py
----

from django.contrib import admin
from django.contrib.auth import get_user_model
from authemail.admin import EmailUserAdmin

class MyUserAdmin(EmailUserAdmin):
	fieldsets = (
		(None, {'fields': ('email', 'password')}),
		('Personal Info', {'fields': ('first_name', 'last_name')}),
		('Permissions', {'fields': ('is_active', 'is_staff', 
									   'is_superuser', 'is_verified', 
									   'groups', 'user_permissions')}),
		('Important dates', {'fields': ('last_login', 'date_joined')}),
		('Custom info', {'fields': ('date_of_birth',)}),
	)

admin.site.unregister(get_user_model())
admin.site.register(get_user_model(), MyUserAdmin)
Create the database tables with Django's makemigrations, migrate, and create a superuser with createsuperuser.

python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
Check your setup by starting a Web server on your local machine:

python manage.py runserver
Direct your browser to the Django /admin and log in.

127.0.0.1:8000/admin
You should see Users, Tokens, Password reset codes, Signup codes, and Groups. If you click on Users, you should see your superuser account.

Add the authemail API endpoints to your project's urls.py file. For example,

mysite/urls.py
----

from django.contrib import admin
from django.urls import include, path

urlpatterns = [
	path('admin/', admin.site.urls),

	path('api/accounts/', include('authemail.urls')),
]
When users signup or reset their password, they will be sent an email with a link and verification code. Include email settings as environment variables or in your project's settings.py file. For example,

mysite/settings.py

import os

EMAIL_FROM = os.environ.get('AUTHEMAIL_DEFAULT_EMAIL_FROM') or '<YOUR DEFAULT_EMAIL_FROM HERE>'
EMAIL_BCC = os.environ.get('AUTHEMAIL_DEFAULT_EMAIL_BCC') or '<YOUR DEFAULT_EMAIL_BCC HERE>'

EMAIL_HOST = os.environ.get('AUTHEMAIL_EMAIL_HOST') or 'smtp.gmail.com'
EMAIL_PORT = os.environ.get('AUTHEMAIL_EMAIL_PORT') or 587
EMAIL_HOST_USER = os.environ.get('AUTHEMAIL_EMAIL_HOST_USER') or '<YOUR EMAIL_HOST_USER HERE>'
EMAIL_HOST_PASSWORD = os.environ.get('AUTHEMAIL_EMAIL_HOST_PASSWORD') or '<YOUR EMAIL_HOST_PASSWORD HERE>'
EMAIL_USE_TLS = True
EMAIL_USE_SSL = False
Try out authemail API calls by firing up python and using the authemail wrapper methods (runserver should still be executing). For example,

python
>>> from authemail import wrapper
>>> account = wrapper.Authemail()
>>> first_name = 'Your first name'
>>> last_name = 'Your last name'
>>> email = 'your_email@gmail.com'
>>> password = 'Your password'
>>> response = account.signup(first_name=first_name, last_name=last_name,
... email=email, password=password)
In the Django /admin, you should see a new user (not verified) and a new signup code. You should receive an email at your_email@gmail.com. Use the code in the email to verify your email address using the wrapper (normally, the link in the email would point to the front end, which would issue the signup verify request to the API):

>>> code = '7f31e7a515df266532df4e00e0cf1967a7de7d17'
>>> response = account.signup_verify(code=code)
In the Django /admin, the new user is now verified and the signup code is absent. The new user can now login and you can inspect the associated login token:

>>> response = account.login(email=email, password=password)
>>> account.token
'a84d062c1b60a36e6740eb60c6f9da8d1f709322'
You will find the same token for the user in the Token table in the Django /admin. Find out more information about the user (insert your token):

>>> token = 'a84d062c1b60a36e6740eb60c6f9da8d1f709322'
>>> response = account.users_me(token=token)
>>> response
{'first_name': 'Your first name', 'last_name': 'Your last name', 'email': 'your_email@gmail.com'}
Use the authentication token to logout:

>>> response = account.logout(token=token)
>>> response
{'success': 'User logged out.'}

For the endpoints that follow, the base path is shown as /api/accounts. This path is for example purposes. It can be customized in your project's urls.py file.
POST /api/accounts/signup

Call this endpoint to sign up a new user and send a verification email. Sample email templates are found in authemail/templates/authemail. To override the email templates, copy and modify the sample templates, or create your own, in your_app/templates/authemail.

Your front end should handle password confirmation, and if desired, require the visitor to input their first and last names.

Unverified users can sign up multiple times, but only the latest signup code will be active.

Payload

email (required)
password (required)
first_name (optional)
last_name (optional)
Possible responses

201 (Created)
Content-Type: application/json
{
	"email": "amelia.earhart@boeing.com"
	"first_name": "Amelia", 
	"last_name": "Earhart", 
}

400 (Bad Request)
Content-Type: application/json
{
	"email": [
		"This field may not be blank."
	], 
	"password": [
		"This field may not be blank."
	] 
}

{
	"email": [
		"Enter a valid email address."
	]
}

{
	"detail": "User with this Email address already exists."
}
GET /api/accounts/signup/verify/?code=<code>

When the user clicks the link in the verification email, the front end should call this endpoint to verify the user.

Parameters

code (required)
Possible responses

200 (OK)
Content-Type: application/json
{
	"success": "User verified."
}

400 (Bad Request)
Content-Type: application/json
{
	"detail": "Unable to verify user."
}
POST /api/accounts/login

Call this endpoint to log in a user. Use the authentication token in future calls to identify the user.

Payload

email (required)
password (required)
Possible responses

200 (OK)
Content-Type: application/json
{
	"token": "91ec67d093ded89e0a752f35188802c261899013"
}

400 (Bad Request)
Content-Type: application/json
{
	"password": [
		"This field may not be blank."
	], 
	"email": [
		"This field may not be blank."
	]
}

{
	"email": [
		"Enter a valid email address."
	]
}

401 (Unauthorized)
{
	"detail": "Authentication credentials were not provided."
}

{
	"detail": "Unable to login with provided credentials."
}
GET /api/accounts/logout

Call this endpoint to log out an authenticated user.

HTTP Header
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
Possible responses
200 (OK)
Content-Type: application/json
{
	"success": "User logged out."
}

401 (Unauthorized)
Content-Type: application/json
{
	"detail": "Authentication credentials were not provided."
}

{
	"detail": "Invalid token"
}
POST /api/accounts/password/reset

Call this endpoint to send an email to a user so they can reset their password. Similar to signup verification, the password reset email templates are found in authemail/templates/authemail. Override the default templates by placing your similarly-named templates in your_app/templates/authemail.

Payload

email (required)
Possible responses

201 (Created)
Content-Type: application/json
{
	"email": "amelia.earhart@boeing.com"
}

400 (Bad Request)
Content-Type: application/json
{
	"email": [
		"This field may not be blank."
	]
}

{
	"email": [
		"Enter a valid email address."
	]
}

{
	"detail": "Password reset not allowed."
}
GET /api/accounts/password/reset/verify/?code=<code>

When the user clicks the link in the password reset email, call this endpoint to verify the password reset code.

Parameters

code (required)
Possible responses

200 (OK)
Content-Type: application/json
{
	"success": "User verified."
}

400 (Bad Request)
Content-Type: application/json
{
	"password": [
		"This field may not be blank."
	] 
}

{
	"detail": "Unable to verify user."
}
POST /api/accounts/password/reset/verified

Call this endpoint with the password reset code and the new password, to reset the user's password. The front end should prompt the user for a confirmation password and give feedback if the passwords don't match.

Payload

code (required)
password (required)
Possible responses

200 (OK)
Content-Type: application/json
{
	"success": "Password reset."
}

400 (Bad Request)
Content-Type: application/json
{
	"password": [
		"This field may not be blank."
	] 
}

{
	"detail": "Unable to verify user."
}
POST /api/accounts/email/change

Call this endpoint to send a notification email to the previous email address and a confirmation email to the new email address. Similar to signup and password reset verification, the email change email templates are found in authemail/templates/authemail. Override the default templates by placing your similarly-named templates in your_app/templates/authemail.

HTTP Header
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
Payload

email (required)
Possible responses

201 (Created)
Content-Type: application/json
{
	"email": "amelia.earhart@boeing.com"
}

400 (Bad Request)
Content-Type: application/json
{
	"email": [
		"This field may not be blank."
	] 
}

{
	"email": [
		"Enter a valid email address."
	] 
}

{
	"detail": "Email address already taken."
}

401 (Unauthorized)
Content-Type: application/json
{
	"detail": "Authentication credentials were not provided."
}

{
	"detail": "Invalid token"
}
GET /api/accounts/email/change/verify/?code=<code>

When the user clicks the link in the email change email, call this endpoint to verify the email change code and, if appropriate, change the email address.

Parameters

code (required)
Possible responses

200 (OK)
Content-Type: application/json
{
	"success": "Email address changed."
}

400 (Bad Request)
Content-Type: application/json
{
	"detail": "Email address already taken."
}

{
	"detail": "Unable to verify user."
}
POST /api/accounts/password/change

Call this endpoint to change a user's password.

HTTP Header
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
Payload

password (required)
Possible responses

200 (OK)
Content-Type: application/json
{
	"success": "Password changed."
}

400 (Bad Request)
Content-Type: application/json
{
	"password": [
		"This field may not be blank."
	] 
}

401 (Unauthorized)
Content-Type: application/json
{
	"detail": "Authentication credentials were not provided."
}

{
	"detail": "Invalid token"
}
GET /api/accounts/users/me

Call this endpoint after logging in and obtaining an authorization token to learn more about the user.

HTTP Header
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
Possible responses
200 (OK)
Content-Type: application/json
{
	"id": 1,
	"email": "amelia.earhart@boeing.com",
	"first_name": "Amelia",
	"last_name": "Earhart",
}

401 (Unauthorized)
Content-Type: application/json
{
	"detail": "Authentication credentials were not provided."
}

{
	"detail": "Invalid token"
}
