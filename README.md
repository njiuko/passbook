[![Build Status](https://travis-ci.org/lgleasain/passbook.png)](https://travis-ci.org/lgleasain/passbook)
# Njiuko/Unidy Developer Notes
The passbook is not working since ruby version 2.7. This fork fixes the issue and also allows loading certificates from memory.

# passbook

The passbook gem let's you create a pkpass for passbook in iOS 6+

## Installation

Include the passbook gem in your project.

IE:  In your Gemfile
```
gem 'passbook'
```

## Quick Start

If you want to jump in without having to integrate this into your code you can use the command line options to get started.  Start by installing the gem

```
gem install passbook
```

Then go to a directory that you want to generate your pass under and use the "pk generate command".  (note:  do not use spaces in your passname or else strange things will happen.)

```
pk generate your_pass_name
```

This will generate a directory called your_pass_name.  Edit your pass.json file in the your_pass_directory to have a valid team identifier and passTypeIdentifier and create your certificates if you haven't yet. [See this article for information on how to do this.](http://www.raywenderlich.com/20734/beginning-passbook-part-1#more-20734)

Assuming that you have put your certificate files etc. in your working directory.

```
pk build your_pass_name -w ./wwdc.pem -c ./your_pass_name.p12 -p ''
```
The wwdc.pem file is the exported Apple World Wide Developer Relations Certification Authority certificate file from your key manager and the your_pass_name.p12 is the exported p12 file from your pass certificate.

If you are not building your passes on a mac or just prefer to use the pass certificate and key pem file.

```
pk build passbook_gem_name -w ./wwdc.pem -c ./your_pass_name_certificate.pem -k your_pass_name_key.pem -p '12345'
```

Now you can drag the file over to a simulator or send it to your iPhone via e-mail to view your pass.


## Configuration

Create initializer
```
    rails g passbook:config
    or with params
    rails g passbook:config [Absolute path to the wwdc cert file] [Absolute path to your cert.p12 file] [Password for your certificate]
```

Configure your config/initializers/passbook.rb
```
    Passbook.configure do |passbook|
      passbook.wwdc_cert = Rails.root.join('wwdc_cert.pem')
      passbook.p12_certificate = Rails.root.join('cert.p12')
      passbook.p12_password = 'cert password'
    end
```

If you are running this on a different machine from what you used to create your WWDC keys
```
    Passbook.configure do |passbook|
      passbook.wwdc_cert = Rails.root.join('wwdc_cert.pem')
      passbook.p12_key = Rails.root.join('key.pem')
      passbook.p12_certificate = Rails.root.join('certificate.pem')
      passbook.p12_password = 'cert password'
    end
```
If you are using Sinatra you can place this in the file you are executing or in a file that you do a require on.  You would also not reference Rails.root when specifying your file path.

If You are doing push notifications then you will need to add some extra configuration options,  namely a push notification certificate and a notification gateway certificate.  Look at the Grocer gem documentation to find information on how to create this certificate.  Settings you will want to use for the notification gateway are either 'gateway.push.apple.com' for production,  'gateway.sandbox.push.apple.com' for development and 'localhost' for unit tests.
```
    Passbook.configure do |passbook|
      .....other settings.....
      passbook.notification_gateway = 'gateway.push.apple.com'
      passbook.notification_passphrase = 'my_hard_password' (optional)
      passbook.notification_cert = 'lib/assets/my_notification_cert.pem'
    end
```

If you want to also support the push notification endpoints you will also need to include the Rack::PassbookRack middleware.  In rails your config will look something like this.
```
  config.middleware.use Rack::PassbookRack
```

## Usage

Please refer to apple iOS dev center for how to build cert and json.  [This article is also helpful.](http://www.raywenderlich.com/20734/beginning-passbook-part-1#more-20734)
```
    pass = Passbook::PKPass.new 'your json data'

    # Add file from disk
    pass.addFile 'file_path'

    # Add file from memory
    file[:name] = 'file name'
    file[:content] = 'whatever you want'
    pass.addFile file

    # Add multiple files
    pass.addFiles [file_path_1, file_path_2, file_path_3]

    # Add multiple files from memory
    pass.addFiles [{name: 'file1', content: 'content1'}, {name: 'file2', content: 'content2'}, {name: 'file3', content: 'content3'}]

    # Output a Tempfile

    pkpass = pass.file
    send_file pkpass.path, type: 'application/vnd.apple.pkpass', disposition: 'attachment', filename: "pass.pkpass"

    # Or a stream

    pkpass = pass.stream
    send_data pkpass.string, type: 'application/vnd.apple.pkpass', disposition: 'attachment', filename: "pass.pkpass"

```
If you are using Sinatra you will need to include the 'active_support' gem and will need to require 'active_support/json/encoding'.  Here is an example using the streaming mechanism.

```
require 'sinatra'
require 'passbook'
require 'active_support/json/encoding'

Passbook.configure do |passbook|
  passbook.p12_password = '12345'
  passbook.p12_key = 'passkey.pem'
  passbook.p12_certificate = 'passcertificate.pem'
  passbook.wwdc_cert = 'WWDR.pem'
end

get '/passbook' do
  pass = # valid passbook json.  refer to the wwdc documentation.
  passbook = Passbook::PKPass.new pass
  passbook.addFiles ['logo.png', 'logo@2x.png', 'icon.png', 'icon@2x.png']
  response['Content-Type'] = 'application/vnd.apple.pkpass'
  attachment 'mypass.pkpass'
  passbook.stream.string
end
```

We will try to make this cleaner in subsequent releases.

### Using Different Certificates For Different Passes

Sometime you might want to be able to use different certificates for different passes.  This can be done by passing in a Signer class into your PKPass initializer like so

```
  signer = Passbook::Signer.new {certificate: some_cert, password: some_password, key: some_key, wwdc_cert: some_wwdc_cert}
  pk_pass = Passbook::PKPass.new your_json_data, signer

  ....
```

### Push Notifications

If you want to support passbook push notification updates you will need to configure the appropriate bits above.

In order to support push notifications you will need to have a basic understanding of the way that push notifications work and how the data is passed back and forth.  See [this](http://developer.apple.com/library/ios/#Documentation/UserExperience/Conceptual/PassKit_PG/Chapters/Creating.html) for basic information about passes and [this](http://developer.apple.com/library/ios/#Documentation/UserExperience/Conceptual/PassKit_PG/Chapters/Updating.html#//apple_ref/doc/uid/TP40012195-CH5-SW1) to understand the information that needs to be exchanged between each device and your application to support the update service.

Your pass will need to have a field called 'webServiceURL' with the base url to your site and a field called 'authenticationToken'. The json snippet should look like this.  Note that your url needs to be a valid signed https endpoint for production.  You can put your phone in dev mode to test updates against a insecure http endpoint (under settings => developer => passkit testing).

```
...
  "webserviceURL" : "https://www.honeybadgers.com/",
  "authenticationToken" : "yummycobras"
...
```

Passbook includes rack middleware to make the job of supporting the passbook endpoints easier.  You will need to configure the middleware as outlined above and then implement a class called Passbook::PassbookNotification.  Below is an annotated implementation.

```
module Passbook
  class PassbookNotification

    # This is called whenever a new pass is saved to a users passbook or the
    # notifications are re-enabled.  You will want to persist these values to
    # allow for updates on subsequent calls in the call chain.  You can have
    # multiple push tokens and serial numbers for a specific
    # deviceLibraryIdentifier.

    def self.register_pass(options)
      the_passes_serial_number = options['serialNumber']
      the_devices_device_library_identifier = options['deviceLibraryIdentifier']
      the_devices_push_token = options['pushToken']
      the_pass_type_identifier = options["passTypeIdentifier"]
      the_authentication_token = options['authToken']

      # this is if the pass registered successfully
      # change the code to 200 if the pass has already been registered
      # 404 if pass not found for serialNubmer and passTypeIdentifier
      # 401 if authorization failed
      # or another appropriate code if something went wrong.
      {:status => 201}
    end

    # This is called when the device receives a push notification from apple.
    # You will need to return the serial number of all passes associated with
    # that deviceLibraryIdentifier.

    def self.passes_for_device(options)
      device_library_identifier = options['deviceLibraryIdentifier']
      passes_updated_since = options['passesUpdatedSince']

      # the 'lastUpdated' uses integers values to tell passbook if the pass is
      # more recent than the current one.  If you just set it is the same value
      # every time the pass will update and you will get a warning in the log files.
      # you can use the time in milliseconds,  a counter or any other numbering scheme.
      # you then also need to return an array of serial numbers.
      {'lastUpdated' => '1', 'serialNumbers' => ['various', 'serial', 'numbers']}
    end

    # this is called when a pass is deleted or the user selects the option to disable pass updates.
    def self.unregister_pass(options)
      # a solid unique pair of identifiers to identify the pass are
      serial_number = options['serialNumber']
      device_library_identifier = options['deviceLibraryIdentifier']
      the_pass_type_identifier = options["passTypeIdentifier"]
      the_authentication_token = options['authToken']
      # return a status 200 to indicate that the pass was successfully unregistered.
      {:status => 200}
    end

    # this returns your updated pass
    def self.latest_pass(options)
      the_pass_serial_number = options['serialNumber']
      # create your PkPass the way you did when your first created the pass.
      # you will want to return
      my_pass = PkPass.new 'your pass json'
      # you will want to return the string from the stream of your PkPass object.
      {:status => 200, :latest_pass => mypass.stream.string, :last_modified => '1442120893'}
    end

    # This is called whenever there is something from the update process that is a warning
    # or error
    def self.passbook_log(log)
      # this is a VERY crude logging example.  use the logger of your choice here.
      p "#{Time.now} #{log}"
    end

  end
end

```

To send a push notification for a updated pass simply call Passbook::PushNotification.send_notification with the push token for the device you are updating

```
  Passbook::PushNotification.send_notification the_device_push_token

```

Apple will send out a notification to your phone (usually within 15 minutes or less),  which will cause the phone that this push notification is associated with to make a call to your server to get pass serial numbers and to then get the updated pass.  Each phone/pass combination has it's own push token whch will require a separate call for every phone that has push notifications enabled for a pass (this is an Apple thing).  In the future we may look into offering background process support for this as part of this gem.  For now,  if you have a lot of passes to update you will need to do this yourself.

## Tests

  To launch tests :
```
  bundle exec rake spec
```

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Added some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## Changelog

### 0.0.4
Allow passbook gem to return a ZipOutputStream (needed when garbage collector delete tempfile before beeing able to use it) [Thx to applidget]

### 0.2.0
Adding support for push notification updates for passes.

### 0.4.0
Adding support for using multiple signatures per gem configuration and introducing command line helpers.  More in [this](http://www.polyglotprogramminginc.com/allowing-for-more-signature-flexibility-in-the-ruby-passbook-gem/) blog post.

### 0.4.1
Update rubyzip dependency to >= 1.0.0

License
-------

passbook is released under the MIT license:

* http://www.opensource.org/licenses/MIT
