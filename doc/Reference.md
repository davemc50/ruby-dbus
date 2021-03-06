Ruby D-Bus Reference
====================

This is a reference-style documentation. It's not [a tutorial for
beginners](http://dbus.freedesktop.org/doc/dbus-tutorial.html), the
reader should have knowledge of basic DBus concepts.

Client Side
-----------

This section should be enough if you only want to consume DBus APIs.

### Basic Concepts

#### Setting Up

Note that although the gem is named "ruby-dbus", the required name
is simply "dbus"

    #! /usr/bin/env ruby
    require "dbus"

#### Calling Methods

1. {DBus.session_bus Connect to the session bus};
   {DBus::Connection#[] get the screensaver service}
   {DBus::Service#object and its screensaver object}.
2. Perform {DBus::ProxyObject#introspect explicit introspection}
   to define the interfaces and methods
   on the {DBus::ProxyObject object proxy}
([I#28](https://github.com/mvidner/ruby-dbus/issues/28)).
3. Call one of its methods in a loop, solving [xkcd#196](http://xkcd.com/196).

&nbsp;
    mybus = DBus.session_bus
    service = mybus['org.freedesktop.ScreenSaver']
    object = service.object '/ScreenSaver'
    object.introspect
    loop do
        object.SimulateUserActivity
        sleep 5 * 60
    end

##### Retrieving Return Values

A method proxy always returns an array of values. This is to
accomodate the rare cases of a DBus method specifying more than one
*out* parameter. For nearly all methods you should use `Method[0]` or
`Method.first`
([I#30](https://github.com/mvidner/ruby-dbus/issues/30)).
    
    # wrong
    if upower_i.SuspendAllowed    # [false] is true!
      upower_i.Suspend
    end

    # right
    if upower_i.SuspendAllowed[0]
      upower_i.Suspend
    end

#### Accessing Properties

To access properties, think of the {DBus::ProxyObjectInterface interface} as a
{DBus::ProxyObjectInterface#[] hash} keyed by strings,
or use {DBus::ProxyObjectInterface#all_properties} to get
an actual Hash of them.

    sysbus = DBus.system_bus
    upower_s = sysbus['org.freedesktop.UPower']
    upower_o = upower_s.object '/org/freedesktop/UPower'
    upower_o.introspect
    upower_i = upower_o['org.freedesktop.UPower']

    on_battery = upower_i['OnBattery']

    puts "Is the computer on battery now? #{on_battery}"

(TODO a writable property example)

Note that unlike for methods where the interface is inferred if unambiguous,
for properties the interface must be explicitly chosen.
That is because {DBus::ProxyObject} uses the {DBus::ProxyObject Hash#[]} API
to provide the {DBus::ProxyObjectInterface interfaces}, not the properties.

#### Asynchronous Operation

If a method call has a block attached, it is asynchronous and the block
is invoked on receiving a method_return message or an error message

##### Main Loop

For asynchronous operation an event loop is necessary. Use {DBus::Main}:

    # [set up signal handlers...]
    main = DBus::Main.new
    main << mybus
    main.run

Alternately, run the GLib main loop and add your DBus connections to it via
{DBus::Connection#glibize}.

#### Receiving Signals

To receive signals for a specific object and interface, use
{DBus::ProxyObjectInterface#on\_signal}(name, &block) or
{DBus::ProxyObject#on_signal}(name, &block), for the default interface.

    sysbus = DBus.system_bus
    login_s = sysbus['org.freedesktop.login1'] # part of systemd
    login_o = login_s.object '/org/freedesktop/login1'
    login_o.introspect
    login_o.default_iface = 'org.freedesktop.login1.Manager'

    # to trigger this signal, login on the Linux console
    login_o.on_signal("SessionNew") do |name, opath|
      puts "New session: #{name}"

      session_o = login_s.object(opath)
      session_o.introspect
      session_i = session_o['org.freedesktop.login1.Session']
      uid, user_opath = session_i['User']
      puts "Its UID: #{uid}"
    end

    main = DBus::Main.new
    main << sysbus
    main.run

### Intermediate Concepts
#### Names
#### Types and Values, D-Bus -> Ruby

D-Bus booleans, numbers, strings, arrays and dictionaries become their straightforward Ruby counterparts.

Structs become arrays.

Object paths become strings.

Variants are simply unpacked to become their contained type.
(ISSUE: prevents proper round-tripping!)

#### Types and Values, Ruby -> D-Bus

D-Bus has stricter typing than Ruby, so the library must decide
which D-Bus type to choose. Most of the time the choice is dictated
by the D-Bus signature.

##### Variants

If the signature expects a Variant
(which is the case for all Properties!) then an explicit mechanism is needed.

1. A pair [{DBus::Type::Type}, value] specifies to marshall *value* as
   that specified type.
   The pair can be produced by {DBus.variant}(signature, value) which
   gives the  same result as [{DBus.type}(signature), value].

   ISSUE: using something else than cryptic signatures is even more painful
   than remembering the signatures!

        foo_i['Bar'] = DBus.variant("au", [0, 1, 1, 2, 3, 5, 8])

2. Other values are tried to fit one of these:
   Boolean, Double, Array of Variants, Hash of String keyed Variants,
   String, Int32, Int64.

3. **Deprecated:** A pair [String, value], where String is a valid
   signature of a single complete type, marshalls value as that
   type. This will hit you when you rely on method (2) but happen to have
   a particular string value in an array.

##### Byte Arrays

If a byte array (`ay`) is expected you can pass a String too.
The bytes sent are according to the string's
[encoding](http://ruby-doc.org/core-1.9.3/Encoding.html).

##### nil

`nil` is not allowed by D-Bus and attempting to send it raises an exception
(but see [I#16](https://github.com/mvidner/ruby-dbus/issues/16)).


#### Errors

D-Bus calls can reply with an error instead of a return value. An error is
translated to a Ruby exception, an instance of {DBus::Error}.

    begin
        network_manager.sleep
    rescue DBus::Error => e
        puts e unless e.name == "org.freedesktop.NetworkManager.AlreadyAsleepOrAwake"
    end

#### Interfaces

Methods, properties and signals of a D-Bus object always belong to one of its interfaces.

Methods can be called without specifying their interface, as long as there is no ambiguity.
There are two ways to resolve ambiguities:

1. assign an interface name to {DBus::ProxyObject#default_iface}.

2. get a specific {DBus::ProxyObjectInterface interface} of the object,
with {DBus::ProxyObject#[]} and call methods from there.

Signals and properties only work with a specific interface.

#### Thread Safety
Not there. An [incomplete attempt](https://github.com/mvidner/ruby-dbus/tree/multithreading) was made.
### Advanced Concepts
#### Bus Addresses
#### Without Introspection
#### Name Overloading

Service Side
------------

When you want to provide a DBus API.

(check that client and service side have their counterparts)

### Basic
#### Exporting a Method
##### Interfaces
##### Methods
##### Bus Names
##### Errors
#### Exporting Properties
### Advanced
#### Inheritance
#### Names

Specification Conformance
-------------------------

This section lists the known deviations from version 0.19 of
[the specification][spec].

[spec]: http://dbus.freedesktop.org/doc/dbus-specification.html

1. Properties support is basic.
