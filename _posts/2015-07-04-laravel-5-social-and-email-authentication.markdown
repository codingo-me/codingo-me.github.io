---
layout: post
title:  "Laravel 5 Social and Email Authentication"
date:   2015-07-04 15:33:39
categories: laravel
author: Ivan Radunovic
author-link: https://www.linkedin.com/in/ivanradunovic
---
In this tutorial I will create **Laravel** application with email authentication, but also I will use [Laravel Socialite][laravel-socialite] for Facebook and Twitter logins. Once when I configure everything, app will also be able to use many other social platforms for sign in process. Entire list of social providers is here. Some of them are **Paypal**, **Reddit**, **Linkedin**, **Tumblr**, **Youtube** and **Google**.

> I have posted here only excerpt of tutorial, full tutorial is located at: [Laravel 5 Social and Email Authentication]

# Development schedule

It is always good idea to create some kind of schedule or plan, before you actually start coding. So I like to write down simple steps and then complete them one by one, until entire app is completed.

For now this app will have home page, login/register pages with forms for email authentication and Facebook and Twitter buttons for social login. Usually every app needs to have at least 2 user roles, for administrator and for ordinary users, so I will code basic user-role logic. Users will be able to reset their passwords, so app will send some emails. I will code system in such a way that adding new social providers like Github would be trivial and short process (under 20 seconds).


1. [Creating views]
2. [Create migrations and models related to users and roles]
3. [User seeder with some dummy users]
4. [Middleware for administrator and user roles]
5. [Routes and AuthController]
6. [Create table and model for Social Sign-in]
7. [Create Social Logic]

# 7. Create Social Logic

When we want to authenticate user with one of Socialite providers, first we redirect that user to social site and after that social site redirects user back to our server with certain tokens. In the back Socialite contacts social site once more with those tokens and accepts user object if everything is OK. Here I will handle only social redirects when user allows our app to read their social data, in next tutorial I will cover cancelations.

I want this social logic to be very extensible so I can add new providers in matter of seconds. Because of that I will not hardcode any values in routes. These are my social routes:


{% highlight php %}
$s = 'social.';
Route::get('/social/redirect/{provider}',   ['as' => $s . 'redirect',   'uses' => 'Auth\AuthController@getSocialRedirect']);
Route::get('/social/handle/{provider}',     ['as' => $s . 'handle',     'uses' => 'Auth\AuthController@getSocialHandle']);
{% endhighlight %}

So social buttons will use following links `/social/redirect/facebook` and `/social/redirect/twitter`. When user visits them, system will trigger Socialite redirection and it will get user object in return. You will notice redirect key for each social provider in services.php, that url social site will use as callback url. And in my system that route is `/social/handle/facebook` or `/social/handle/twitter`.

Now you can see that adding new provider is matter of inserting new element in `services.php` and creating dedicated social button in views.

`AuthController.php - Social Sign-in Part`

{% highlight php %}
 public function getSocialRedirect( $provider )
    {
        $providerKey = \Config::get('services.' . $provider);
        if(empty($providerKey))
            return view('pages.status')
                ->with('error','No such provider');

        return Socialite::driver( $provider )->redirect();

    }

    public function getSocialHandle( $provider )
    {
        $user = Socialite::driver( $provider )->user();

        $socialUser = null;

        //Check is this email present
        $userCheck = User::where('email', '=', $user->email)->first();
        if(!empty($userCheck))
        {
            $socialUser = $userCheck;
        }
        else
        {
            $sameSocialId = Social::where('social_id', '=', $user->id)->where('provider', '=', $provider )->first();

            if(empty($sameSocialId))
            {
                //There is no combination of this social id and provider, so create new one
                $newSocialUser = new User;
                $newSocialUser->email              = $user->email;
                $name = explode(' ', $user->name);
                $newSocialUser->first_name         = $name[0];
                $newSocialUser->last_name          = $name[1];
                $newSocialUser->save();

                $socialData = new Social;
                $socialData->social_id = $user->id;
                $socialData->provider= $provider;
                $newSocialUser->social()->save($socialData);

                // Add role
                $role = Role::whereName('user')->first();
                $newSocialUser->assignRole($role);

                $socialUser = $newSocialUser;
            }
            else
            {
                //Load this existing social user
                $socialUser = $sameSocialId->user;
            }

        }

        $this->auth->login($socialUser, true);

        if( $this->auth->user()->hasRole('user'))
        {
            return redirect()->route('user.home');
        }

        if( $this->auth->user()->hasRole('administrator'))
        {
            return redirect()->route('admin.home');
        }

        return \App::abort(500);
    }
{% endhighlight %}

In `getSocialRedirect` method I am passing provider string as parameter and I am checking is that provider present in services. If it is present then system redirects user to social site.

In `getSocialHandle` I am catching data which social site sends to the server.

* First I am checking is user with same email present in users table, if yes then I login that user and system redirects him to proper panel.
* Else I check is user with same social id and provider present in `social_logins` table
    * If user is present I just login that user
    * Else I create that new user and associate respective social data to that account. I am also attaching user roles here.

This approach posses one downside, for example Twitter does not pass users email to us. So that’s why I changed email field to be nullable. I will add one simple trick in next tutorial for fixing this.

For full tutorial visit [http://tuts.codingo.me/laravel-social-and-email-authentication]

Code is hosted on Github [https://github.com/codingo-me/laravel-social-email-authentication]

[laravel-socialite]: https://github.com/laravel/socialite
[Laravel 5 Social and Email Authentication]: http://tuts.codingo.me/laravel-social-and-email-authentication/
[Creating views]: http://tuts.codingo.me/laravel-social-and-email-authentication/#creating-views
[Create migrations and models related to users and roles]: http://tuts.codingo.me/laravel-social-and-email-authentication/#migrations-users
[User seeder with some dummy users]: http://tuts.codingo.me/laravel-social-and-email-authentication/#user-role-seeders
[Middleware for administrator and user roles]: http://tuts.codingo.me/laravel-social-and-email-authentication/#middleware
[Routes and AuthController]: http://tuts.codingo.me/laravel-social-and-email-authentication/#routes
[Create table and model for Social Sign-in]: http://tuts.codingo.me/laravel-social-and-email-authentication/#pull-socialite
[Create Social Logic]: http://tuts.codingo.me/laravel-social-and-email-authentication/#social-logic
[http://tuts.codingo.me/laravel-social-and-email-authentication]: http://tuts.codingo.me/laravel-social-and-email-authentication
[https://github.com/codingo-me/laravel-social-email-authentication]: https://github.com/codingo-me/laravel-social-email-authentication