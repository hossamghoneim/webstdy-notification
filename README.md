# Webstdy Firebase Notification

### Installation

Install wia composer:

```
composer require Webstdy/firebase-notification
```

Make a helper function to collect notification data:

```php
function storeAndPushNotification($titleAr, $titleEn, $descriptionAr, $descriptionEn, $icon, $color, $url)
{
    /* add notification to first Employee */
    $date = Carbon::now()->diffForHumans();
    $notification = new NewNotification($titleAr, $titleEn, $descriptionAr, $descriptionEn, $date, $icon, $color, $url);
    $admin = Employee::first(); //use the model you want, in my case i'm using Employee Model
    $admin->notify($notification);

    /* push notifications to all admins */
    $firebaseToken = Employee::whereNotNull('device_token')->pluck('device_token')->all();
    $SERVER_API_KEY = ""; //use your server api key

    $data = [
        "registration_ids" => $firebaseToken,
        "notification" => [
          // return the data you want
        ]
    ];

    return Http::withHeaders([
        "Authorization" => "key=$SERVER_API_KEY",
    ])->post('https://fcm.googleapis.com/fcm/send', $data);
}
```

Make a notification controller to use notification actions:

```php
//example
public function markAsRead($id)
{
    return $this->notificationActions->markAsRead(new Employee, $id);
}
```

Create js file, set configurations in it and put it in public folder: 
```javascript 
/*
Give the service worker access to Firebase Messaging.
Note that you can only use Firebase Messaging here, other Firebase libraries are not available in the service worker.
*/
importScripts('https://www.gstatic.com/firebasejs/7.23.0/firebase-app.js');
importScripts('https://www.gstatic.com/firebasejs/7.23.0/firebase-messaging.js');

/*
Initialize the Firebase app in the service worker by passing in the messagingSenderId.
* New configuration for app@pulseservice.com
*/
firebase.initializeApp({
    apiKey: "", //set firebase apikey
    authDomain: "", //set firebase authDomain
    projectId: "", //set firebase projectId
    storageBucket: "", //set firebase storageBucket
    messagingSenderId: "", //set firebase messagingSenderId
    appId: "", //set firebase appId
    measurementId: "" //set firebase measurementId
});

/*
Retrieve an instance of Firebase Messaging so that it can handle background messages.
*/
const messaging = firebase.messaging();

messaging.onBackgroundMessage(function(payload) {
    console.log(
        "[your-file-name.js] Received background message ",
        payload,
    );
    /* Customize notification here */
    const notificationTitle = payload.notification.alert_title;
    const notificationOptions = {
        // body: payload.data['cm.notification.description'],
        body: "BACKGROUND BODY",
        icon: faviconPath,
    };

    return self.registration.showNotification(
        notificationTitle,
        notificationOptions,
    );
});
```

Create js file called listen-to-firebase-notification and put it into js folder in your public folder:
```javascript
// For Firebase JS SDK v7.20.0 and later, measurementId is optional
// Your web app's Firebase configuration
var firebaseConfig = {
    apiKey: "", //set firebase apikey
    authDomain: "", //set firebase authDomain
    projectId: "", //set firebase projectId
    storageBucket: "", //set firebase storageBucket
    messagingSenderId: "", //set firebase messagingSenderId
    appId: "", //set firebase appId
    measurementId: "" //set firebase measurementId
};

// Initialize Firebase
firebase.initializeApp(firebaseConfig);
const messaging = firebase.messaging();

initFirebaseMessagingRegistration();
onPushing();

function onPushing() {
    messaging.onMessage(function(payload) {
        const data = payload.data;
        const noteTitle = data['gcm.notification.alert_title'];

        const noteOptions = {
            icon: faviconPath,
            body: payload.notification[`description_${locale}`],
        };

        $(".no-notifications-alert").addClass('d-none');

        /** append notification **/
        $("#unread-notifications-container,#all-notifications-container").prepend(`
        <div class="d-flex flex-stack py-4 notification-item">
            <!--begin::Section-->
            <div class="d-flex align-items-center">
                <!--begin::Symbol-->
                <div class="symbol symbol-50px me-4">
                    <span class="symbol-label bg-light-${data['gcm.notification.icon_color']}">
                        <span class="svg-icon svg-icon-2x svg-icon-${data['gcm.notification.icon_color']}">
                                ${data['gcm.notification.alert_icon']}
                        </span>
                    </span>
                </div>
                <!--end::Symbol-->

                <!--begin::Title-->
                <div class="mb-0 me-2">
                    <a href="/dashboard/notifications/${data['gcm.notification.id']}/mark_as_read" class="fs-6 text-gray-800 text-hover-primary fw-bold">${data['gcm.notification.alert_title']}</a>
                    <div class="text-gray-400 fs-7">${data[`gcm.notification.description_${locale}`]}</div>
                </div>
                <!--end::Title-->
            </div>
            <!--end::Section-->

            <!--begin::Label-->
            <span class="badge badge-light fs-8">${data['gcm.notification.date']}</span>
            <!--end::Label-->
        </div>
        `);

        let counterSpan = $(".notifications-counter");
        let appointmentsCounterSpan = $("#appointments_counter");
        let counter = parseInt(counterSpan.text());

        if (Number(counter))
            counterSpan.text(`${counter + 1 + translate('unread')}`);
        else
            counterSpan.text(1 + translate('unread'));

        appointmentsCounterSpan.text(parseInt(appointmentsCounterSpan.text()) + 1)

        playNotificationSound();

        new Notification(noteTitle, noteOptions);

        favicon.badge(favIconCounter + 1);
        $('.bullet.bullet-dot').removeClass('d-none');
    });
}

function initFirebaseMessagingRegistration() {
    messaging
        .requestPermission()
        .then(function () {
            return messaging.getToken()
        })
        .then(function(token) {
            $.ajaxSetup({
                headers: {
                    'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
                }
            });

            $.ajax({
                url: '/dashboard/save-token',
                type: 'POST',
                data: {
                    token: token
                },
                dataType: 'JSON',
                success: function (response) {
                    console.log('Token saved successfully.');
                },
                error: function (err) {
                    console.log('User Chat Token Error'+ err);
                },
            });

        }).catch(function (err) {
        console.log('User Chat Token Error'+ err);
    });
}

/** Load more btn **/
$("#unread-load-more,#all-load-more").click(function (e) {
    e.preventDefault();
    let loadMoreBtn = $(this);
    var type = loadMoreBtn.attr('id');
    var currentNotificationsCount = loadMoreBtn.siblings().length;

    loadMoreBtn.attr('data-kt-indicator', 'on')

    $.ajax({
        type: 'get',
        url: `/dashboard/notifications/${type}/load-more/${currentNotificationsCount}`,
        success: function (res) {
            if(res.data.length == 0){
                loadMoreBtn.remove();

            }else{
                $.each(res.data, function (key, notification) {

                    loadMoreBtn.before(`
                        <div class="d-flex flex-stack py-4 notification-item">
                            <!--begin::Section-->
                            <div class="d-flex align-items-center">
                                <!--begin::Symbol-->
                                <div class="symbol symbol-50px me-4">
                                    <span class="symbol-label bg-light-${notification.color}">
                                        <span class="svg-icon svg-icon-2x svg-icon-${notification.color}">
                                                ${notification.icon}
                                        </span>
                                    </span>
                                </div>
                                <!--end::Symbol-->

                                <!--begin::Title-->
                                <div class="mb-0 me-2">
                                    <a href="/dashboard/notifications/${notification.id}/mark_as_read" class="fs-6 text-gray-800 text-hover-primary fw-bold">${notification[`title_${locale}`]}</a>
                                    <div class="text-gray-400 fs-7">${notification[`description_${locale}`]}</div>
                                </div>
                                <!--end::Title-->
                            </div>
                            <!--end::Section-->

                            <!--begin::Label-->
                            <span class="badge badge-light fs-8">${notification.created_at}</span>
                            <!--end::Label-->
                        </div>
                    `);
                });

                loadMoreBtn.attr('data-kt-indicator', 'off')
            }

        }
    });
});
```

Put this code in footer:
```javascript
<script src="https://www.gstatic.com/firebasejs/8.6.7/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.6.7/firebase-messaging.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.6.7/firebase-analytics.js"></script>
<script src="{{ asset('path-of-your-js-folder/listen-to-firebase-notification.js') }}"></script>
```

