# mastercode
The ESP8266 sketch template I use for my IoT stuff. This framework implements all those things which I want in essentially every Feather HUZZAH-based device I build, to save on reimplementation and increase consistency.

## Supports

* TASK SCHEDULER

Task scheduling is done using Anatoli Arkhipenko's cooperative multitasking library, found at:

https://github.com/arkhipenko/TaskScheduler

Since all non-template functionality added to this sketch is intended to be implemented using Tasks, I highly recommend careful study of the documentation at https://github.com/arkhipenko/TaskScheduler/blob/master/extras/TaskScheduler.pdf before implementing sketches based on this template.

* OLED FEATHERWING SUPPORT

If enabled (OLED_ENABLED setting), supports the display of battery status (if enabled: VBAT_ENABLED setting) and of WiFi network connectivity, RSSI, and IP address using the Adafruit OLED FeatherWing ( https://www.adafruit.com/products/2900 ).

* WIFI/MQTT SERVER CONNECTIVITY

Maintains (reconnecting on failure) connectivity to a specified wireless LAN and MQTT server.

* WATCHDOG

Periodically sends a status message (by default, every sixty seconds) to an MQTT topic, providing remote monitoring and assurance that the Feather and its attached device are still operating.

* BATTERY MONITORING

If enabled (VBAT_ENABLED setting), monitors the current battery voltage/charge status of the LiPo battery attached to the Feather. If OLED support is enabled, this is displayed locally; in any case, battery charge information is reported to the MQTT server on a configurable topic at two-minute intervals.

The use of battery monitoring assumes that the Vbat pin of the Feather HUZZAH is connected to the analog input pin via a 1M/200K voltage divider.

* STATUS BLINKING AND DISPLAY

The red LED on the Feather (connected to pin 0) is blinked at 1s intervals to indicate locally that the Feather is up and running. If application code indicates failure, this is switched to continuous-on, to distinguish it both from the up state and from power failure.

* SLEEP MODE

The template enables the Feather to be placed in deep sleep for a set amount of time, automatically waking again when the period is over, to allow power conservation. If the period of deep sleep is likely to exceed the watchdog firing period (defined as half or more of the watchdog interval), an automatic "yawn" notification is sent to the MQTT server over the watchdog topic.

* OTA UPDATE

The template permits password-protected over-the-air updates from the Arduino IDE, to allow simple updating of in-place devices.

## Required libraries

To compile this, you'll need to install the following libraries from the Arduino IDE:

* Adafruit GFX Library
* Adafruit MQTT Library
* Adafruit SSD1306
* Adafruit_FeatherOLED (from Github, as described here: https://learn.adafruit.com/adafruit-oled-featherwing/featheroled-library#featheroled-library )
* ArduinoJson
* ArduinoOTA
* TaskScheduler

## Configuration

See the _USER SETTINGS SECTION_ in the code, documented in comments.

## API

### Must-implement

The standard Arduino entrypoints (`setup()` and `loop()`) are taken over by the template code. It is the application developer's responsibility, therefore, to fit their application code into the three callback functions defined by the template (at the bottom of the code file, these are defined empty). These are:

*void application_setup ()*

This is the primary entry point of your application, called by the template's initialization code. Note that an equivalent to `loop()` does not exist in the template; thus, apart from other setup, it is the responsibility of `application_setup ()` to define and enable a number of tasks (see the Task Scheduler library documentation mentioned above, and the example below) to carry out the functions of the application once `application_setup ()` exits.

*void application_fail ()*

This function is a callback from `fail ()` (see below) invoked on fatal errors to shut down application activities (disabling tasks, etc.) before shutting down system activities and otherwise handling failure.

*void application_otasafe ()*

This function is invoked immediately before an over-the-air update is installed. It should do whatever is necessary to ensure that any devices controlled by this Feather are in a safe and shut-down state, such that time taken for, or errors during, the update do not pose any risk of harm. Be aware that the OTA update will proceed _immediately_ upon return from `application_otasafe()`, and so this function cannot rely on any outside tasks; no other code will execute between this function and the reset following the update.

### Functions and References

These are the functions intended to be called by the user of the template; i.e., those which _do not_ begin with an underscore. Those which do begin with an underscore are internal functions.

#### Failure

*void fail (char * reason)*

Handles fatal errors. After invoking the callback to shut down application functionality, it displays the given reason on the OLED display (if enabled), shuts down system activities (except for basic WiFi and OTA update support, to permit repair), and sets the status/red LED to continuously on, to locally indicate the failure.

#### MQTT

The MQTT client, an instance of the client from the Adafruit MQTT Library ( https://github.com/adafruit/Adafruit_MQTT_Library ), is available at `mqttClient`; application code pubs and subs should make use of this instance.

#### OLED (only available if OLED_ENABLED)

*void oledClear ()*

Clears the middle 128x15 pixels message area on the OLED display.

*void oledPrintln (char * line)*

Prints a string to one line of the message area of the OLED display.

*void oledDisplay ()*

Causes the OLED display message area to update.

A typical usage of these helper functions to display a message would be a call to `oledClear` followed by two calls to `oledPrintln` and a single call to `oledDisplay`, thus:

```
oledClear ();
oledPrintln ("first line");
oledPrintln ("second line");
oledDisplay ();
```

If it is desirable to just clear the message area, `oledClear` can be called alone.

*Adafruit_FeatherOLED_WiFi getOled ()*

Returns the `Adafruit_FeatherOLED_WiFi` object used by the template to access the OLED display. This class is that from the FeatherOLED library, which see ( https://learn.adafruit.com/adafruit-oled-featherwing/featheroled-library#featheroled-library ) for details, available to be used when more complex operations are required than the simple helper functions above permit. Note that using this directly, it is easy to interfere with the use of the status areas above and below that the template code uses; _caveat_ developer!

### Sleep

*void deepSleep (int msec)*

Go into deep sleep mode for the specified number of milliseconds. If this length of time exceeds half the configured WATCHDOG_INTERVAL, an impending-sleep notification ("yawn") will be sent on the watchdog MQTT topic. If the OLED display is enabled, a notification message ("yawn...") is displayed to indicate that the Feather is asleep locally, since status blinking ceases with the red LED off while in deep sleep.

On wakeup, the Feather initializes again. Application setup code can determine whether it was awoken from sleep rather than another type of reset by checking the...

*bool woken*

...variable, which is true on wake-from-sleep, false otherwise.

### Task Scheduler

The default task scheduler instance, to which application tasks should be added, is available as:

*Scheduler taskManager*

Application tasks, declared thus (for example):

```
Task tLoop (0, TASK_FOREVER, &loop, NULL, false, NULL, NULL);

void loop ()
{
  ...
}
```

Can therefore be added (typically in `application_setup ()`) thus:

```
taskManager.addTask(tLoop);
tLoop.enable();
```
