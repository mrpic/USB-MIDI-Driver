Android USB MIDI Driver
====

USB MIDI Driver using Android USB Host API

- No root privilege needed.
- Supports the standard USB MIDI devices; like sequencers, or instruments.
- Supports some non-standard USB MIDI (but protocol is compatible with USB MIDI) devices.
    - YAMAHA, Roland, MOTU's devices can be connected(not been tested much).
- Supports multiple device connections.
- Has `javax.sound.midi` compatible classes.
    - See the [javax.sound.midi Documents](https://github.com/kshoji/USB-MIDI-Driver/wiki/javax.sound.midi-porting-for-Android).

Requirement
----
- Android : OS version 3.1(API Level 12) or higher, and have an USB host port.
    - The android Linux kernel must support USB MIDI devices. Some Android device recognizes only USB-HID and USB-MSD by kernel configurations.
- USB MIDI (compatible) device

the optional thing:

- The self powered USB hub (if want to connect multiple USB MIDI devices).
- USB OTG cable (if the Android device has no standard USB-A port).
- USB MIDI <--> Lagacy MIDI(MIDI 1.0) converter cable (if want to connect with legacy MIDI instruments).

Device Connection
----

Single device
```
Android [USB A port / microUSB port with USB OTG cable]--- USB MIDI Device
```

Multiple devices
```
Android [USB A port / microUSB port with USB OTG cable]---(USB Hub)---┬-- USB MIDI Device
                                                                      ├-- USB MIDI Device 
                                                                      └   ...
```

Repository Overview
====
- Library Project : `MIDIDriver`
    - The driver for connecting an USB MIDI device.

- Sample Project : `MIDIDriverSample`
    - The sample implementation of the synthesizer / MIDI event logger.
    - Pre-compiled sample project is available on [Google Play Market](https://play.google.com/store/apps/details?id=jp.kshoji.driver.midi.sample).

Library Project Usage
====

Project setup
----

- Clone the library project.
- Import the library project into Eclipse workspace, and build it.
- Create new Android Project. And add the library project to the project.
- Override `AbstractSingleMidiActivity` or `AbstractMultipleMidiActivity`.
    - `AbstractSingleMidiActivity` can connect only **one** MIDI device.
    - `AbstractMultipleMidiActivity` can connect **multiple** MIDI devices.
        - NOTE: The performance problem (slow latency or high CPU/memory usage) may occur if many devices have been connected.
- Modify the AndroidManifest.xml
    - Add "uses-feature" tag to use USB Host feature.
    - Activity's **launchMode** must be "singleTask".

```xml
<uses-feature android:name="android.hardware.usb.host" /> 

<application>
    <activity
        android:name=".MyMidiMainActivity"
        android:label="@string/app_name"
        android:launchMode="singleTask" >
        <intent-filter>
            <category android:name="android.intent.category.LAUNCHER" />
            <action android:name="android.intent.action.MAIN" />
        </intent-filter>
    </activity>
    :
```

MIDI event handling with AbstractSingleMidiActivity
----

MIDI event receiving:

- Implement the MIDI event handling method (named `"onMidi..."`) to receive MIDI events.
- Note: The method `"onMidi..."` will be called by another thread (NOT the `UI thread`), so you must manipulate the Views with the `Handler` and `Callback` in the UI thread. Like the code below.

<a name="ui_thread"></a>

```java
public class SampleActivity extends AbstractSingleMidiActivity {

    // this field belongs to the UI thread
    final Handler uiThreadEventHandler = new Handler(new Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            if ("note on".equals(msg.obj)) {
                textView.setText("note on event received.");
            }

            // message handled successfully
            return true;
        }
    });

    // this method will be called from the another thread, so it can't change View's state.
    @Override
    public void onMidiNoteOn(final MidiInputDevice sender, int cable, int channel, int note, int velocity) {
        // Send a message to the UI thread
        String message = "note on";
        uiThreadEventHandler.sendMessage(Message.obtain(uiThreadEventHandler, 0, message));
    }
```

MIDI event sending:

- Call AbstractSingleMidiActivity's `getMidiOutputDevices()` method to get the instance of `MIDIOutputDevice`.
    - And call the instance's method (named `"sendMidi..."`) to send MIDI events.
        - NOTE: The first found USB MIDI device will be detected if multiple devices has been attached.
        - NOTE: If output endpoint doesn't connected, `getMidiOutputDevices()` method returns null.


MIDI event handling with AbstractMultipleMidiActivity
----

MIDI event receiving:

- Implement the MIDI event handling method (named `"onMidi..."`) to receive MIDI events.
    - The event sender object(MIDIInputDevice instance) will be set on the first argument.
- Note: The method `"onMidi..."` will be called by another thread (NOT the `UI thread`), so you must manipulate the Views with the `Handler` and `Callback` in the UI thread. See also [the code above](#ui_thread)

MIDI event sending:

- Call the `getMidiOutputDevices()` method to get the instance of `Set<MIDIOutputDevice>`.
- Choose an instance from the `Set<MIDIOutputDevice>`.
    - And call the instance's method (named `"sendMidi..."`) to send MIDI events.


Supported MIDI events list
----

Some kind of messages don't be transferred frequently. So, the rare messages are not tested well.
Please add an [issue](https://github.com/kshoji/USB-MIDI-Driver/issues/new) If had trouble with using this library.

| Method name ends with..         | Meaning                                                           | Well tested?    |
|:----                            |:----                                                              |:----            |
| MidiMiscellaneousFunctionCodes  | Miscellaneous function codes. Reserved for future extensions.     | NO              |
| MidiCableEvents                 | Cable events. Reserved for future expansion.                      | NO              |
| MidiSystemCommonMessage         | System Common messages, or SysEx ends with following single byte. | NO              |
| MidiSystemExclusive             | SysEx                                                             | YES             |
| MidiNoteOff                     | Note-off                                                          | YES             |
| MidiNoteOn                      | Note-on                                                           | YES             |
| MidiPolyphonicAftertouch        | Poly-KeyPress                                                     | NO              |
| MidiControlChange               | Control Change                                                    | YES             |
| MidiProgramChange               | Program Change                                                    | YES             |
| MidiChannelAftertouch           | Channel Pressure                                                  | NO              |
| MidiPitchWheel                  | PitchBend Change                                                  | YES             |
| MidiSingleByte                  | Single Byte                                                       | NO              |


FAQ
----
- What is the 'cable' argument of `"onMidi..."` or `"sendMidi..."` method?
    - A single USB MIDI endpoint has multiple "virtual MIDI cables". 
    It's used for increasing the midi channels. The cable number's range is 0 to 15.
- The application doesn't detect the device even if the USB MIDI device connected.
    - See the [Trouble shooting](https://github.com/kshoji/USB-MIDI-Driver/wiki/TroubleShooting-on-connecting-an-USB-MIDI-device) documents.

License
----
[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)
