###############
Cloud Messaging
###############

You can use the Firebase Admin SDK for PHP to send Firebase Cloud Messaging messages to end-user devices.
Specifically, you can send messages to individual devices, named topics, or condition statements that match one or more topics.

.. note::
    Sending messages to Device Groups is only possible with legacy protocols which are not supported
    by this SDK.

Before you start, please read about Firebase Cloud Messaging in the official documentation:

- `Introduction to Firebase Cloud Messaging <https://firebase.google.com/docs/cloud-messaging/>`_
- `Introduction to Admin FCM API <https://firebase.google.com/docs/cloud-messaging/admin/>`_

************************************
Initializing the Messaging component
************************************

**With the SDK**

.. code-block:: php

    $messaging = $factory->createMessaging();

**With Dependency Injection** (`Symfony Bundle <https://github.com/kreait/firebase-bundle>`_/`Laravel/Lumen Package <https://github.com/kreait/laravel-firebase>`_)

.. code-block:: php

    use Kreait\Firebase\Contract\Messaging;

    class MyService
    {
        public function __construct(Messaging $messaging)
        {
            $this->messaging = $messaging;
        }
    }

**With the Laravel** ``app()`` **helper** (`Laravel/Lumen Package <https://github.com/kreait/laravel-firebase>`_)

.. code-block:: php

    $messaging = app('firebase.messaging');

***************
Getting started
***************

.. code-block:: php

    use Kreait\Firebase\Messaging\CloudMessage;

    $message = CloudMessage::new()
        ->withNotification(Notification::create('Title', 'Body'))
        ->withData(['key' => 'value'])
        ->toToken('...')
        // ->toTopic('...')
        // ->toCondition('...')
    ;

    $messaging->send($message);

A message must be an object implementing ``Kreait\Firebase\Messaging\Message`` or an array that can
be parsed to a ``Kreait\Firebase\Messaging\CloudMessage``.

You can use ``Kreait\Firebase\Messaging\RawMessageFromArray`` to create a message without the SDK checking it
for validity before sending it. This gives you full control over the sent message, but also means that you
have to send/validate a message in order to know if it's valid or not.

.. note::
    If you notice that a field is not supported by the SDK yet, please open an issue on the issue tracker, so that others
    can benefit from it as well.

***********************
Send messages to topics
***********************

Based on the publish/subscribe model, FCM topic messaging allows you to send a message to multiple devices that have opted in to a particular topic.
You compose topic messages as needed, and FCM handles routing and delivering the message reliably to the right devices.

For example, users of a local weather forecasting app could opt in to a "severe weather alerts" topic and receive notifications of storms threatening specified areas.
Users of a sports app could subscribe to automatic updates in live game scores for their favorite teams.

Some things to keep in mind about topics:

- Topic messaging supports unlimited topics and subscriptions for each app.
- Topic messaging is best suited for content such as news, weather, or other publicly available information.
- Topic messages are optimized for throughput rather than latency. For fast, secure delivery to single devices or small groups of devices, target messages to registration tokens, not topics.

You can create a message to a topic in one of the following ways:

.. code-block:: php

    use Kreait\Firebase\Exception\MessagingException;
    use Kreait\Firebase\Messaging\CloudMessage;

    $topic = 'a-topic';

    $message = CloudMessage::new()
        ->withNotification($notification) // optional
        ->withData($data) // optional
        ->toTopic($topic)
    ;

    $message = CloudMessage::fromArray([
        'topic' => $topic,
        'notification' => [/* Notification data as array */], // optional
        'data' => [/* data array */], // optional
    ]);

    try {
        $result = $messaging->send($message);
        // $result = ['name' => 'projects/<project-id>/messages/6810356097230477954']
    } catch (MessagingException $e) {
        // ...
    }


*************************
Send conditional messages
*************************

.. warning::
    OR-conditions are currently not processed correctly by the Firebase Rest API, leading to undelivered messages.
    This can be resolved by splitting up a message to an OR-condition into multiple messages to AND-conditions.
    So one conditional message to ``'a' in topics || 'b' in topics`` should be sent as two messages
    to the conditions ``'a' in topics && !('b' in topics)`` and ``'b' in topics && !('a' in topics)``

    References:
        - https://github.com/firebase/quickstart-js/issues/183
        - https://stackoverflow.com/a/52302136/284325

Sometimes you want to send a message to a combination of topics.
This is done by specifying a condition, which is a boolean expression that specifies the target topics.
For example, the following condition will send messages to devices that are subscribed to ``TopicA`` and either ``TopicB`` or ``TopicC``:

``"'TopicA' in topics && ('TopicB' in topics || 'TopicC' in topics)"``

FCM first evaluates any conditions in parentheses, and then evaluates the expression from left to right.
In the above expression, a user subscribed to any single topic does not receive the message.
Likewise, a user who does not subscribe to TopicA does not receive the message. These combinations do receive it:

- ``TopicA`` and ``TopicB``
- ``TopicA`` and ``TopicC``

.. code-block:: php

    use Kreait\Firebase\Messaging\CloudMessage;

    $condition = "'TopicA' in topics && ('TopicB' in topics || 'TopicC' in topics)";

    $message = CloudMessage::new()
        ->withNotification($notification) // optional
        ->withData($data) // optional
        ->toCondition($condition)
    ;

    $message = CloudMessage::fromArray([
        'condition' => $condition,
        'notification' => [/* Notification data as array */], // optional
        'data' => [/* data array */], // optional
    ]);

    $messaging->send($message);


*********************************
Send messages to specific devices
*********************************

The Admin FCM API allows you to send messages to individual devices by specifying a registration token for the target device.
Registration tokens are strings generated by the client FCM SDKs for each end-user client app instance.

Each of the Firebase client SDKs are able to generate these registration tokens:
`iOS <https://firebase.google.com/docs/cloud-messaging/ios/client#access_the_registration_token>`_,
`Android <https://firebase.google.com/docs/cloud-messaging/android/client#sample-register>`_,
`Web <https://firebase.google.com/docs/cloud-messaging/js/client#access_the_registration_token>`_,
`C++ <https://firebase.google.com/docs/cloud-messaging/cpp/client#access_the_device_registration_token>`_,
and `Unity <https://firebase.google.com/docs/cloud-messaging/unity/client#initialize_firebase_messaging>`_.

.. code-block:: php

    use Kreait\Firebase\Messaging\CloudMessage;

    $deviceToken = '...';

    $message = CloudMessage::new()
        ->withNotification($notification) // optional
        ->withData($data) // optional
        ->toToken($deviceToken)
    ;

    $message = CloudMessage::fromArray([
        'token' => $deviceToken,
        'notification' => [/* Notification data as array */], // optional
        'data' => [/* data array */], // optional
    ]);

    $result = $messaging->send($message);
    // $result = ['name' => 'projects/<project-id>/messages/<message-id>']

************************
Send messages in batches
************************

.. note::
    If you need to send a message to more than a few devices, consider sending the message
    to a topic instead.

.. code-block:: php

    use Kreait\Firebase\Messaging\CloudMessage;

    $messages = [
        // Either objects implementing Kreait\Firebase\Messaging\Message or arrays that can
        // be parsed into to Kreait\Firebase\Messaging\CloudMessage objects
    ];

    /** @var Kreait\Firebase\Messaging\MulticastSendReport $sendReport **/
    $sendReport = $messaging->sendAll($messages);

The ``sendMulticast()`` message is a convenience method to send one message to multiple devices.

.. code-block:: php

    use Kreait\Firebase\Messaging\CloudMessage;

    $message = CloudMessage::new(); // Any instance of Kreait\Messaging\Message
    $deviceTokens = ['...', '...' /* ... */];

    /** @var Kreait\Firebase\Messaging\MulticastSendReport $sendReport **/
    $sendReport = $messaging->sendMulticast($message, $deviceTokens);

The returned value ``$sendReport`` is an instance of ``Kreait\Firebase\Messaging\MulticastSendReport`` and provides you with
methods to determine the successes and failures of the multicasted message:

.. code-block:: php

    $report = $messaging->sendMulticast($message, $deviceTokens);

    echo 'Successful sends: '.$report->successes()->count().PHP_EOL;
    echo 'Failed sends: '.$report->failures()->count().PHP_EOL;

    if ($report->hasFailures()) {
        foreach ($report->failures()->getItems() as $failure) {
            echo $failure->error()->getMessage().PHP_EOL;
        }
    }

    // The following methods return arrays with registration token strings
    $successfulTargets = $report->validTokens(); // string[]

    // Unknown tokens are tokens that are valid but not know to the currently
    // used Firebase project. This can, for example, happen when you are
    // sending from a project on a staging environment to tokens in a
    // production environment
    $unknownTargets = $report->unknownTokens(); // string[]

    // Invalid (=malformed) tokens
    $invalidTargets = $report->invalidTokens(); // string[]

.. note::
    The ``sendMulticast`` method stems from a time where Firebase had a (now shutdown) dedicated API endpoint
    for multicast messages. It is now a wrapper for the ``sendAll()`` method. "Legacy" is also the reason why
    the returned report is named ``MulticastSendReport``.

*********************
Adding a notification
*********************

A notification is an instance of ``Kreait\Firebase\Messaging\Notification`` and can be
created in one of the following ways. The title and the body of a notification
are both optional.

.. code-block:: php

    use Kreait\Firebase\Messaging\Notification;

    $title = 'My Notification Title';
    $body = 'My Notification Body';
    $imageUrl = 'https://picsum.photos/400/200';

    $notification = Notification::fromArray([
        'title' => $title,
        'body' => $body,
        'image' => $imageUrl,
    ]);

    $notification = Notification::create($title, $body);

    $changedNotification = $notification
        ->withTitle('Changed title')
        ->withBody('Changed body')
        ->withImageUrl('https://picsum.photos/200/400');

Once you have created a message with one of the methods described below,
you can attach the notification to it:

.. code-block:: php

    $message = $message->withNotification($notification);

***********
Adding data
***********

The data attached to a message must be an array of key-value pairs
where all keys and values are strings.

Once you have created a message with one of the methods described below,
you can attach data to it:

.. code-block:: php

    $data = [
        'first_key' => 'First Value',
        'second_key' => 'Second Value',
    ];

    $message = $message->withData($data);

*********************************************
Adding target platform specific configuration
*********************************************

You can target platforms specific configuration to your messages.

Android
-------

You can find the full Android configuration reference in the official documentation:
`REST Resource: projects.messages.AndroidConfig <https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#androidconfig>`_

.. code-block:: php

    use Kreait\Firebase\Messaging\AndroidConfig;

    // Example from https://firebase.google.com/docs/cloud-messaging/admin/send-messages#android_specific_fields
    $config = AndroidConfig::fromArray([
        'ttl' => '3600s',
        'priority' => 'normal',
        'notification' => [
            'title' => '$GOOG up 1.43% on the day',
            'body' => '$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.',
            'icon' => 'stock_ticker_update',
            'color' => '#f45342',
            'sound' => 'default',
        ],
    ]);

    $message = $message->withAndroidConfig($config);

APNs
----

You can find the full APNs configuration reference in the official documentation:
`REST Resource: projects.messages.ApnsConfig <https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#apnsconfig>`_

.. code-block:: php

    use Kreait\Firebase\Messaging\ApnsConfig;

    // Example from https://firebase.google.com/docs/cloud-messaging/admin/send-messages#apns_specific_fields
    $config = ApnsConfig::fromArray([
        'headers' => [
            'apns-priority' => '10',
        ],
        'payload' => [
            'aps' => [
                'alert' => [
                    'title' => '$GOOG up 1.43% on the day',
                    'body' => '$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.',
                ],
                'badge' => 42,
                'sound' => 'default',
            ],
        ],
    ]);

    $message = $message->withApnsConfig($config);


WebPush
-------

You can find the full WebPush configuration reference in the official documentation:
`REST Resource: projects.messages.Webpush <https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#webpushconfig>`_

.. code-block:: php

    use Kreait\Firebase\Messaging\WebPushConfig;

    // Example from https://firebase.google.com/docs/cloud-messaging/admin/send-messages#webpush_specific_fields
    $config = WebPushConfig::fromArray([
        'notification' => [
            'title' => '$GOOG up 1.43% on the day',
            'body' => '$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.',
            'icon' => 'https://my-server.example/icon.png',
        ],
        'fcm_options' => [
            'link' => 'https://my-server.example/some-page',
        ],
    ]);

    $message = $message->withWebPushConfig($config);

***************************************
Adding platform independent FCM options
***************************************

You can find the full FCM Options configuration reference in the official documentation:
`REST Resource: projects.messages.fcm_options <https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#fcmoptions>`_

.. code-block:: php

    use Kreait\Firebase\Messaging\FcmOptions;

    $fcmOptions = FcmOptions::create()
        ->withAnalyticsLabel('my-analytics-label');
    // or
    $fcmOptions = [
        'analytics_label' => 'my-analytics-label'
    ];

    $message = $message->withFcmOptions($fcmOptions);

*******************
Notification Sounds
*******************

The SDK provides helper methods to add sounds to messages:

* ``CloudMessage::withDefaultSounds()``
* ``AndroidConfig::withDefaultSound()``
* ``AndroidConfig::withSound($sound)``
* ``ApnsConfig::withDefaultSound()``
* ``ApnsConfig::withSound($sound)``

.. note::
    WebPush notification don't support the inclusion of sounds.

.. code-block:: php

    $message = CloudMessage::new()
        ->withNotification(['title' => 'Notification title', 'body' => 'Notification body'])
        ->withDefaultSounds() // Enables default notifications sounds on iOS and Android devices.
        ->withApnsConfig(
            ApnsConfig::new()
                ->withSound('bingbong.aiff')
                ->withBadge(1)
        )
    ;

****************
Message Priority
****************

The SDK provides helper methods to define the priority of a message.

.. note::
    You can learn more about message priorities for the different target platforms at
    `Setting the priority of a message <https://firebase.google.com/docs/cloud-messaging/concept-options#setting-the-priority-of-a-message>`_
    in the official Firebase documentation.

.. note::
    Setting a message priority is optional. If you don't set a priority, the Firebase backend or the target
    platform uses their defined defaults.

Android
-------

* ``AndroidConfig::withNormalPriority()``
* ``AndroidConfig::withHighPriority()``
* ``AndroidConfig::withPriority(string $priority)``

iOS (APNS)
----------

* ``ApnsConfig::withPowerConservingPriority()``
* ``ApnsConfig::withImmediatePriority()``
* ``ApnsConfig::withPriority(string $priority)``

Web
---
* ``WebPushConfig::withVeryLowUrgency()``
* ``WebPushConfig::withLowUrgency()``
* ``WebPushConfig::withNormalUrgency()``
* ``WebPushConfig::withHighUrgency()``
* ``WebPushConfig::withUrgency(string $urgency)``

Combined
--------

* ``CloudMessage::withLowestPossiblePriority()``
* ``CloudMessage::withHighestPossiblePriority()``

Example
-------

.. code-block:: php

    $message = CloudMessage::new()
        ->withNotification([
            'title' => 'If you had an iOS device…',
            'body' => '… you would have received a very important message'
        ])
        ->withLowestPossiblePriority()
        ->withApnsConfig(
            ApnsConfig::new()
                ->withImmediatePriority()
                ->withNotification([
                    'title => 'A very important message…',
                    'body' => '… that requires your immediate attention.'
                ])
        )
    ;


************
Using Emojis
************

Firebase Messaging supports Emojis in Messages.

.. note::
    You can find a full list of all currently available Emojis at
    https://www.unicode.org/emoji/charts/full-emoji-list.html

.. code-block:: php

    // You can copy and paste an emoji directly into you source code
    $text = "This is an emoji 😀";
    $text = "This is an emoji \u{1F600}";


*****************************
Sending a raw/custom messages
*****************************

Instead of composing messages with the help of the ``CloudMessage`` builder, you can use
``RawMessageFromArray`` as a wrapper for a pre-compiled message payload. Alternatively,
you can implement custom messages by implementing the ``Kreait\Firebase\Messaging\Message``
interface.

.. code-block:: php

    use Kreait\Firebase\Messaging\RawMessageFromArray;

    $message = new RawMessageFromArray([
            'notification' => [
                // https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#notification
                'title' => 'Default title',
                'body' => 'Default body',
            ],
            'data' => [
                'key' => 'Value',
            ],
            'android' => [
                // https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#androidconfig
                'notification' => [
                    'title' => 'Android Title',
                    'body' => 'Android Body',
                ],
            ],
            'apns' => [
                // https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#apnsconfig
                'payload' => [
                    'aps' => [
                        'alert' => [
                            'title' => 'iOS Title',
                            'body' => 'iOS Body',
                        ],
                    ],
                ],
            ],
            'webpush' => [
                // https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#webpushconfig
                'notification' => [
                    'title' => 'Webpush Title',
                    'body' => 'Webpush Body'
                ],
            ],
            'fcm_options' => [
                // https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages#fcmoptions
                'analytics_label' => 'some-analytics-label'
            ]
        ]);

    $messaging->send($message);

*******************
Validating messages
*******************

You can validate a message by sending a validation-only request to the Firebase REST API. If the message is invalid,
a ``Kreait\Firebase\Exception\Messaging\InvalidMessage`` exception is thrown, which you can catch to evaluate the raw
error message(s) that the API returned.

.. code-block:: php

    use Kreait\Firebase\Exception\Messaging\InvalidMessage;

    try {
        $messaging->validate($message);
        // or
        $messaging->send($message, $validateOnly = true);
    } catch (InvalidMessage $e) {
        print_r($e->errors());
    }

You can also use the ``send*`` methods with an additional parameter:

.. code-block:: php

    $validateOnly = true;

    $messaging->send($message, $validateOnly);
    $messaging->sendMulticast($message, $tokens, $validateOnly);
    $messaging->sendAll($messages, $validateOnly);

******************************
Validating Registration Tokens
******************************

If you have a set of registration tokens that you want to check for validity or if they are still registered
to your project, you can use the ``validateTokens()`` method:

.. code-block:: php

    $tokens = [...];

    $result = $messaging->validateRegistrationTokens($tokens);

The result is an array with three keys containing the checked tokens:

* ``valid`` contains all tokens that are valid and registered to the current Firebase project
* ``unknown`` contains all tokens that are valid, but **not** registered to the current Firebase project
* ``invalid`` contains all invalid (=malformed) tokens

****************
Topic management
****************

You can subscribe one or multiple devices to one or multiple messaging topics with the following methods:

.. code-block:: php

    $result = $messaging->subscribeToTopic($topic, $registrationTokenOrTokens);
    $result = $messaging->subscribeToTopics($topics, $registrationTokenOrTokens);

    $result = $messaging->unsubscribeFromTopic($topic, $registrationTokenOrTokens);
    $result = $messaging->unsubscribeFromTopics($topics, $registrationTokenOrTokens);

    $result = $messaging->unsubscribeFromAllTopics($registrationTokenOrTokens);

The result will return an array in which the keys are the topic names, and the values are the operation
results for the individual tokens.

.. note::
    You can subscribe up to 1,000 devices in a single request. If you provide an array with over 1,000
    registration tokens, the operation will fail with an error.


***********************
App instance management
***********************

A registration token is related to an application that generated it. You can retrieve current information
about an app instance by passing a registration token to the ``getAppInstance()`` method.

.. code-block:: php

    $registrationToken = '...';

    $appInstance = $messaging->getAppInstance($registrationToken);
    // Return the full information as provided by the Firebase API
    $instanceInfo = $appInstance->rawData();

    /* Example output for an Android application instance:
        [
          "applicationVersion" => "1060100"
          "connectDate" => "2019-07-21"
          "attestStatus" => "UNKNOWN"
          "application" => "com.vendor.application"
          "scope" => "*"
          "authorizedEntity" => "..."
          "rel" => array:1 [
            "topics" => array:3 [
              "test-topic" => array:1 [
                "addDate" => "2019-07-21"
              ]
              "test-topic-5d35b46a15094" => array:1 [
                "addDate" => "2019-07-22"
              ]
              "test-topic-5d35b46b66c31" => array:1 [
                "addDate" => "2019-07-22"
              ]
            ]
          ]
          "connectionType" => "WIFI"
          "appSigner" => "..."
          "platform" => "ANDROID"
        ]
    */

    /* Example output for a web application instance
        [
          "application" => "webpush"
          "scope" => ""
          "authorizedEntity" => "..."
          "rel" => array:1 [
            "topics" => array:2 [
              "test-topic-5d35b445b830a" => array:1 [
                "addDate" => "2019-07-22"
              ]
              "test-topic-5d35b446c0839" => array:1 [
                "addDate" => "2019-07-22"
              ]
            ]
          ]
          "platform" => "BROWSER"
        ]
    */

.. note::
    As the data returned by the Google Instance ID API can return differently formed results depending on the
    application or platform, it is currently difficult to add reliable convenience methods for specific
    fields in the raw data.

Working with topic subscriptions
--------------------------------

You can retrieve all topic subscriptions for an app instance with the ``topicSubscriptions()`` method:

.. code-block:: php

    $appInstance = $messaging->getAppInstance('<registration token>');

    /** @var \Kreait\Firebase\Messaging\TopicSubscriptions $subscriptions */
    $subscriptions = $appInstance->topicSubscriptions();

    foreach ($subscriptions as $subscription) {
        echo "{$subscription->registrationToken()} is subscribed to {$subscription->topic()}\n";
    }

**************
Error Handling
**************

Errors returned by the Firebase FCM API are converted to exceptions implementing the
``Kreait\Firebase\Exception\MessagingException``. Each implementation of this interface
has an ``errors()`` method that provides additional information about the error.

.. code-block:: php

    use Kreait\Firebase\Exception\MessagingException;

    try {
        $messaging->send($message);
    } catch (MessagingException $e) {
        echo $e->getMessage();
        print_r($e->errors());
    }

Malformatted messages
---------------------

Messages built with the ``CloudMessage`` builder should be automatically valid, but if you
implement your own ``Kreait\Firebase\Messaging\Message`` implementation or if you use the
``Kreait\Firebase\Messaging\RawMessageFromArray`` class, the message could be invalid, for
example when you forget to add a message target.

.. code-block:: php

    use Kreait\Firebase\Exception\Messaging\InvalidMessage;

    try {
        $messaging->send($message);
    } catch (InvalidMessage $e) {
        echo $e->getMessage();
        print_r($e->errors());
    }

Unknown registration tokens
---------------------------

If a message can't be delivered to a given registration token although the token is
syntactically correct, this usually has one of the following reasons:

* The token has been unregistered from the project. This can happen when a user
  has logged out from the application on the given client, or if they have
  uninstalled or re-installed the application.
* The token has been registered to a different Firebase project than the project
  you are using to send the message. A common reason for this is when you work
  with different application environments and are sending a message from one
  environment to a device in another environment.

.. code-block:: php

    use Kreait\Firebase\Exception\Messaging\NotFound;

    try {
        $messaging->send($message);
    } catch (NotFound $e) {
        echo $e->getMessage();
        print_r($e->errors());
        // If the message was send to a token, you can retrieve the unknown token
        echo $e->token();
    }

Quota exceeded
--------------

The frequency of new subscriptions is rate-limited per project. If you send too many subscription requests
in a short period of time, FCM servers will respond with a 429 RESOURCE_EXHAUSTED ("quota exceeded") response.

.. code-block:: php

    use Kreait\Firebase\Exception\Messaging\QuotaExceeded;

    try {
        $messaging->subscribeToTopic($topic, $registrationTokenOrTokens);
    } catch (QuotaExceeded $e) {
        echo $e->getMessage();
        print_r($e->errors());
        $retryAfter= $e->retryAfter();
    }

The ``QuotaExceeded`` exception provides a ``retryAfter()`` method which returns a ``DateTimeImmutable`` instance
indicating when you can retry sending a subscription request.

Server errors
-------------

Sometimes, the Firebase servers are unavailable. If the server is kaputt, this will throw a ``ServerError`` exception,
if it is "just" unavailable for the moment, this will throw a ``ServerUnavailable`` exception that provides a
``retryAfter()`` method which returns a ``DateTimeImmutable`` instance indicating when you can retry sending the
request.

.. code-block:: php

    use Kreait\Firebase\Exception\Messaging\ServerError;
    use Kreait\Firebase\Exception\Messaging\ServerUnavailable;

    try {
        $messaging->send($message);
    } catch (ServerUnavailable $e) {
        echo 'The FCM servers are currently unavailable: '.$e->getMessage();
        print_r($e->errors());
        $retryAfter= $e->retryAfter();
    } catch (ServerError $e) {
        echo 'The FCM servers are broken: '.$e->getMessage();
        print_r($e->errors());
    }

Error handling example
----------------------

.. code-block:: php

    use Kreait\Firebase\Exception\Messaging as MessagingErrors;
    use Kreait\Firebase\Exception\MessagingException;

    try {
        $messaging->send($message);
    } catch (MessagingErrors\NotFound $e) {
        echo 'The target device could not be found.';
    } catch (MessagingErrors\InvalidMessage $e) {
        echo 'The given message is malformatted.';
    } catch (MessagingErrors\ServerUnavailable $e) {
        $retryAfter = $e->retryAfter();

        echo 'The FCM servers are currently unavailable. Retrying at '.$retryAfter->format(\DATE_ATOM);

        // This is just an example. Using `sleep()` will block your script execution, don't do this.
        while ($retryAfter <= new DateTimeImmutable()) {
            sleep(1);
        }

        $messaging->send($message);
    } catch (MessagingErrors\ServerError $e) {
        echo 'The FCM servers are down.';
    } catch (MessagingException $e) {
        // Fallback handling
        echo 'Unable to send message: '.$e->getMessage();
    }
