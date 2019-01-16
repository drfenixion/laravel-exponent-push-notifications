#INSTALL
Add to composer.json
```
"require": {
    "drfenixion/laravel-exponent-push-notifications": "1.5.1",
},
"repositories": [
    {
        "url": "https://github.com/drfenixion/laravel-exponent-push-notifications.git",
        "type": "git"
    }
],
```

#EXAMPLE OF USING THIS FORK
```
use App\Notifications\MeetReminder;
//$pushForUsers is an array of users. The user is an eloquent model instance.
\Notification::send($pushForUsers, new MeetReminder($someDataForMeetReminder) );
```
```
App\Notifications\MeetReminder.php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Notification;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use NotificationChannels\ExpoPushNotifications\ExpoChannel;
use NotificationChannels\ExpoPushNotifications\ExpoMessage;

class MeetReminder extends Notification
{
    use Queueable;

    /**
     * Create a new notification instance.
     *
     * @return void
     */
    public function __construct($someDataForMeetReminder)
    {
        $this->someDataForMeetReminder = $someDataForMeetReminder;
    }


    /**
     * Get the array representation of the notification.
     *
     * @param  mixed  $notifiable
     * @return array
     */
    public function toArray($notifiable)
    {
        return [
            //
        ];
    }


    public function via($notifiable)
    {
        return [ExpoChannel::class];
    }

    public function toExpoPush($notifiable)
    {   
        $title = 'Some title';
        $description = "Some description with $this->someDataForMeetReminder";

        return ExpoMessage::create()
            ->badge(1)
            ->enableSound()
            ->setChannelId('reminders') //you need create this chanell in Expo app, or use default channel but that without sound. Channels uses on Android 8+. For use default channel via my fork just not set any channel.
            ->title($title)
            ->body($description)
            ->setJsonData(['name' => $title, 'description' => $description]);
    }
}
```
I also do "exclude-from-files" in composer.json for rewrite default package routes:
```
    "autoload": {
        "classmap": [
            "database/seeds",
            "database/factories"
        ],
        "psr-4": {
            "App\\": "app/"
        },
        "exclude-from-files": [
            "drfenixion/laravel-exponent-push-notifications/Http/routes.php"
        ]
    },
```

exclude-from-files used package "mcaskill/composer-exclude-files"
```
    "require": {
        "mcaskill/composer-exclude-files": "^1.1"
    }
```

And add my own for wraped my api middleware:
```
Route::group([ 'middleware' => 'jwt' ], function () {
    Route::put('/users', 'Auth\SocialiteController@updateUser');

    //vendor package route files alymosul/laravel-exponent-push-notifications/Http/routes.php exluded in package.json
    //this is override alymosul/laravel-exponent-push-notifications/Http/routes.php
    //for suit middleware
	Route::group(['prefix' => 'api/exponent/devices', 'middleware' => ['bindings']], function () { 
	    Route::post('subscribe', [
	        'as'    =>  'register-interest',
	        'uses'  =>  'NotificationChannels\ExpoPushNotifications\Http\ExpoController@subscribe',
	    ]);

	    Route::post('unsubscribe', [
	        'as'    =>  'remove-interest',
	        'uses'  =>  'NotificationChannels\ExpoPushNotifications\Http\ExpoController@unsubscribe',
	    ]);
	});

});
```

In Expo app i use next function for register device:

```
	async registerForPushNotifications() {
		const { status: existingStatus } = await Permissions.getAsync(
			Permissions.NOTIFICATIONS
		)
		let finalStatus = existingStatus
		// only ask if permissions have not already been determined, because
		// iOS won't necessarily prompt the user a second time.
		if (existingStatus !== 'granted') {
			// Android remote notification permissions are granted during the app
			// install, so this will only ask on iOS
			const { status } = await Permissions.askAsync(Permissions.NOTIFICATIONS)
			finalStatus = status
		}

		// Stop here if the user did not grant permissions
		if (finalStatus !== 'granted') {
			return
		}
               
               // I persist state of registration of device in mainState.expo_push_token_registered
               // you can remove this check and do code inside always
		if(!mainState.expo_push_token_registered){

			// Get the token that uniquely identifies this device
			token = await Notifications.getExpoPushTokenAsync()

			// POST the token to your backend server from where you can retrieve it to send push notifications.
			return await request.post('exponent/devices/subscribe', {
			    expo_token: token,
			}).then(function (response) {
                                //mainState is my state manager
				mainState.set({name: 'expo_push_token_registered', value: true})
			}).catch(function(err) {
				console.log(err)
			})
		}

		
	}
```

Create the channel in Expo app and in-app listener for the push in componentWillMount of main App container:
```
   import { Notifications } from 'expo'
   import { showMessage } from 'react-native-flash-message'

   componentDidMount() {
    const { mainState, meet } = this.props

    if (Platform.OS === 'android') {
      Notifications.createChannelAndroidAsync('reminders', {
        name: 'Напоминания о встречах',
        sound: true,
        priority: 'max',
        vibrate: [0, 250, 250, 250],
      })
    }
    
    // Handle notifications that are received or selected while the app
    // is open. If the app was closed and then opened by tapping the
    // notification (rather than just tapping the app icon to open it),
    // this function will fire on the next tick after the app starts
    // with the notification data.
    this._notificationSubscription = Notifications.addListener( (notification) => {
      mainState.set({name: 'lastPushNotification', value: notification})
      showMessage({
        message: notification.data.name,
        description: notification.data.description,
        type: "success",
      })
    })

  }
```

Now you can get push in app and global push.


#DESCRIPTION OF ORIGIN PACKAGE
# Exponent push notifications channel for Laravel 5

[![Latest Version on Packagist](https://img.shields.io/packagist/v/alymosul/laravel-exponent-push-notifications.svg?style=flat-square)](https://packagist.org/packages/alymosul/laravel-exponent-push-notifications)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)
[![Build Status](https://travis-ci.org/Alymosul/laravel-exponent-push-notifications.svg?branch=master)](https://travis-ci.org/Alymosul/laravel-exponent-push-notifications)
[![StyleCI](https://styleci.io/repos/96645200/shield?branch=master)](https://styleci.io/repos/96645200)
[![SensioLabsInsight](https://img.shields.io/sensiolabs/i/afe0ba9a-e35c-4759-a06f-14a081cf452c.svg?style=flat-square)](https://insight.sensiolabs.com/projects/afe0ba9a-e35c-4759-a06f-14a081cf452c)
[![Quality Score](https://img.shields.io/scrutinizer/g/alymosul/laravel-exponent-push-notifications.svg?style=flat-square)](https://scrutinizer-ci.com/g/alymosul/laravel-exponent-push-notifications)
[![Code Coverage](https://img.shields.io/scrutinizer/coverage/g/alymosul/laravel-exponent-push-notifications/master.svg?style=flat-square)](https://scrutinizer-ci.com/g/alymosul/laravel-exponent-push-notifications/?branch=master)
[![Total Downloads](https://img.shields.io/packagist/dt/alymosul/laravel-exponent-push-notifications.svg?style=flat-square)](https://packagist.org/packages/alymosul/laravel-exponent-push-notifications)

## Contents

- [Installation](#installation)
- [Usage](#usage)
	- [Available Message methods](#available-message-methods)
- [Changelog](#changelog)
- [Testing](#testing)
- [Security](#security)
- [Contributing](#contributing)
- [Credits](#credits)
- [License](#license)


## Installation
You can install the package via composer:
```bash
composer require alymosul/laravel-exponent-push-notifications
```

If you are using Laravel 5.5 or higher this package will automatically register itself using [Package Discovery](https://laravel.com/docs/5.5/packages#package-discovery). For older versions of Laravel you must install the service provider manually:

```php
// config/app.php
'providers' => [
    ...
    NotificationChannels\ExpoPushNotifications\ExpoPushNotificationsServiceProvider::class,
],
```

You can publish the migration with:
```bash
php artisan vendor:publish --provider="NotificationChannels\ExpoPushNotifications\ExpoPushNotificationsServiceProvider" --tag="migrations"
```

After publishing the migration you can create the `exponent_push_notification_interests` table by running the migrations:

```bash
php artisan migrate
```

You can optionally publish the config file with:
```bash
php artisan vendor:publish --provider="NotificationChannels\ExpoPushNotifications\ExpoPushNotificationsServiceProvider" --tag="config"
```

This is the contents of the published config file:

```php
return [
    'interests' => [
        /*
         * Supported: "file", "database"
         */
        'driver' => env('EXPONENT_PUSH_NOTIFICATION_INTERESTS_STORAGE_DRIVER', 'file'),

        'database' => [
            'table_name' => 'exponent_push_notification_interests',
        ],
    ]
];
```

## Usage

``` php
use NotificationChannels\ExpoPushNotifications\ExpoChannel;
use NotificationChannels\ExpoPushNotifications\ExpoMessage;
use Illuminate\Notifications\Notification;

class AccountApproved extends Notification
{
    public function via($notifiable)
    {
        return [ExpoChannel::class];
    }

    public function toExpoPush($notifiable)
    {
        return ExpoMessage::create()
            ->badge(1)
            ->enableSound()
            ->body("Your {$notifiable->service} account was approved!");
    }
}
```

### Available Message methods

A list of all available options
- `body('')`: Accepts a string value for the body.
- `enableSound()`: Enables the notification sound.
- `disableSound()`: Mutes the notification sound.
- `badge(1)`: Accepts an integer value for the badge.
- `ttl(60)`: Accepts an integer value for the time to live.

### Managing Recipients

This package registers two endpoints that handle the subscription of recipients, the endpoints are defined in src/Http/routes.php file, used by ExpoController and all loaded through the package service provider.

### Routing a message

By default the exponent "interest" messages will be sent to will be defined using the {notifiable}.{id} convention, for example `App.User.1`, however you can change this behaviour by including a `routeNotificationForExpoPushNotifications()` in the notifiable class method that returns the interest name.

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information what has changed recently.

## Testing

``` bash
$ composer test
```

## Security

If you discover any security related issues, please email alymosul@gmail.com instead of using the issue tracker.

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Credits

- [Aly Suleiman](https://github.com/Alymosul)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
