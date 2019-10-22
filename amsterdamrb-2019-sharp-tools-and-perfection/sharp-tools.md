autoscale: true
theme: WT2, 1

![](wt-title-slide.pdf)

---------

# 🔪✂

--------

# Perfection is the enemy of the good

We all succumb to the desire of crafting *perfect* code

---------

# It is perfect as long as we wrote it

...i.e. given the constraints of

* Style that was available to us in a *personal development state*
* Specific purpose of a *project*
* A specific *deadline*

-------------

# Ruby is an open runtime

* Everything is mutable
* Everything can be changed
* Benign intent is assumed

---------------

# Cultures differ

---------------

# Example: serving HTTP ranges in Go

Objective: given an HTTP `Range:` request header, return sections of the resource to the client. For example, 3 `File` references glued together in sequence.

```go
func ServeContent(w ResponseWriter,
  req *Request,
  name string,
  modtime time.Time,
  content io.ReadSeeker)
```

----------------

# Customisation points

* The entire HTTP request ❓
* What will be served (exposed resource) ✅
* What will be the `Content-Disposition` filename ✅
* What will be the `Last-Modified` filename ✅
* Where the response will be written to ✅

----------------

# But wait.

* How much data was served 🚫
* Which ranges were requested 🚫
* What is the `Content-Disposition` intent (`inline`/`attachment`) 🚫
* Any kind of error handling 🚫

------------------

* Any kind of error handling 🚫

`ServeContent` does not return `error`, neither does it propagate it.

------------------

# Parts of `ServeContent`

The function is multiple hundreds of lines long.

* HTTP `Range` header parsing (depends on the seekable size of `content`)
* Serving multipart byte ranges (`multipart/byte-range` response type)
* Handling errors when doing `io.Copy`

------------------

All of the above is private to the `http` package.

* Not possible to reuse.
* Not possible to redefine.
* Not possible to replace with own implementation.

-------------------

# Denying choices is a perfectionist stance

You assume that *exactly* your implementation is *exactly* what will work for your user.

-------------------

[.background-color: #ffffff]
![fit](minor-convenience.png)

^ And so we turn to StackOverflow

-------------------

[.background-color: #ffffff]
![fit](denied.png)

-------------------

# Wikileaks To Leak 5000 Open Source Java Projects With All That Private/Final Bullshit Removed

--------------------

[.background-color: #ffffff]
![fit](qr-yegge.png)

--------------------

http://steve-yegge.blogspot.com/2010/07/wikileaks-to-leak-5000-open-source-java.html

^ I will quote some just to spoil it a little.

---------------------

> Agile Java Developer Johnnie Garza of Irvine, CA condemns the move. "They have no right to do this. Open Source does not mean the source is somehow 'open'. That's my code, not theirs. If I make something private, it means that no matter how desperately you need to call it, I should be able to prevent you from doing so, even long after I've gone to the grave."

----------------------

# Open / closed

Should be open for *extension,* closed for *modification.*

^ Which, as previously, would modify your precious little snowflake of your deadline, your requirement and your knowledge of the problem space

--------------------------

# ...you can't predict extension

* And in your drive for perfection and minimalism you will underdo it.

--------------------------

# With open-runtime languages you are treated as a grown up

* With the help of some sharp tools
* Extension is sometimes indistinguishable from modification

-----------------------

# Normally considered under 🔪

* Debuggers
* `strace`
* Observability tools/injections
* Metrics

^ But what if I told you

-----------------------

# Just touch the code already

It don't bite. It is not "their" code. It is not sacred just because DHH wrote it.

----------------------------

# Tool 1: `bundle open`

```sh

documents-app (master) $
documents-app (master) $ bundle open activerecord
```

will open the ActiveRecord gem in your editor.

^ unless you run the app in Docker, in case you do - you probably know how to bridge the editor envvar

------------------------------

# Case: bridging the Apartment gem and ActiveStorage

* Apartment is a gem for multitenancy, offering one database to one customer
* ActiveStorage handles file uploads and attachments

--------------------------------

* ActiveStorage saves in the default database
* It bypasses the Apartment database switch in some cases
* It is a Rails Engine, so it has its own controllers
* It stores all files in a single, shared namespace

--------------------------------

* `ActiveStorage::Attachment` saved in the correct database
* `ActiveStorage::Blob` saved in the *main* database, sometimes.

-------------------------------

# Tool 2: poking at things

```ruby
[3] pry(main)> ActiveStorage::Blob.public_instance_methods.grep(/signed/)
=> [:signed_id, :to_signed_global_id]
[4] pry(main)> ActiveStorage::Blob.methods.grep(/signed/)
=> [:find_signed]
[5] pry(main)> location = ActiveStorage::Blob.method(:find_signed).source_location
=> [".../activestorage-5.2.3/app/models/active_storage/blob.rb", 46]
[3] pry(main)> `code --goto #{location[0]}:#{location[1]}`

```

^ Opens the source of the method in your editor (in this case VSCode)


--------------------------------

# Let's see

All lookups of the `Blob` go through signed identifiers.

```ruby
class << self
  ...

  # You can used the signed ID of a blob to refer to it on the client side without fear of tampering.
  # This is particularly helpful for direct uploads where the client-side needs to refer to the blob
  # that was created ahead of the upload itself on form submission.
  #
  # The signed ID is also used to create stable URLs for the blob through the BlobsController.
  def find_signed(id)
    find ActiveStorage.verifier.verify(id, purpose: :blob_id)
  end

  ...
end
```

--------------------------------

[.background-color: #ffffff]
![fit 125%](override-terminal.pdf)

^ Normally we alter some terminal dependency that a framework offers us

--------------------------------

[.background-color: #ffffff]
![fit 125%](override-with-prepend.pdf)

^ But we have a unique feature which we can use to to insert our functionality between the calling code, even if it belongs to some other library - and other called code, from the same library (or anything else really)


-------------------------------------

[.background-color: #ffffff]
![95%](prepend-with-record.pdf)

-------------------------------------

[.background-color: #ffffff]
![160%](prepend-activestorage.pdf)

-----------------------------------------

# Tool 3: `Module#prepend`

Allows you to insert *your functionality* between the caller and the callee

```ruby

module ApartmentAwareSignedId
  def find_signed(id)
    tenant, id = ActiveStorage.verifier.verify(signed_id_with_tenant_prefix, purpose: :blob_id_with_tenant)
    Apartment::Tenant.switch(tenant) { find(id) }
  end

  def signed_id
    ActiveStorage.verifier.generate([Apartment::Tenant.current, id], purpose: :blob_id_with_tenant)
  end
end

ActiveStorage::Blob.singleton_class.prepend(ApartmentAwareSignedId)
ActiveStorage::Blob.prepend(ApartmentAwareSignedId)
```

-------------------------------------------

* `tkjhug789r` exists in `tenant-1`
* `tkjhug789r` does not exist in `tenant-2` and gets created...

overwriting the one that belongs to `tenant-1`.

------------------------------------------
```ruby
module PrefixKeyWithTenantToken
  # Prefix all generated blob keys with the tenant. Do not
  # use slash as a delimiter because it needs different escaping
  # depending on the storage service adapter - in some cases it
  # might be significant, in other cases it might get escaped as a path
  # component etc.
  def generate_unique_secure_token
    tenant_slug = Apartment::Tenant.current.split('_').last
    "#{tenant_slug}-#{super}" #=> "tenant1-cbf781tr"
  end
end

ActiveStorage::Blob.singleton_class.prepend(PrefixKeyWithTenantToken)
```


-----------------------------------

# Tool 4: Rails reloader

```ruby
# ActiveStorage can get reloaded as well, and once it does get reloaded
# our injected modules will be lost. We need to ensure the patch is applied
# on every reload.
ActiveSupport::Reloader.to_prepare do
  # Install the prefixed key patch and the prefixed signed ID patches.
  ActiveStorage::Blob.singleton_class.send(:prepend, PrefixKeyWithToken)
  ActiveStorage::Blob.singleton_class.send(:prepend, FindSignedWithTenant)
  ActiveStorage::Blob.prepend(PrefixedSignedId)
end
```

^ If your patch alters things that can get reloaded (and Rails engines are such a thing) then you need this

--------------------------------------

[.background-color: #ececec]
![fit](initializers.png)

^ These are WeTransfer initializers. Each one touches some other people's code, and it does something useful for our business

------------------------------------------

[.background-color: #ffffff]
![fit](migration-notification.png)

--------------------------------------------

# How does it work?

```ruby
ActiveSupport.on_load(:active_record) do
  ActiveRecord::Migrator.prepend(MigrationNotification::MigratorMixin)
end
````

^ By touching other people's code

-----------------------------------------------

# Your perfection is not the same as other people's perfection

* Clever tricks
* Monkey patches
* Spooky action at a distance
* You never know what will actually run
* The user will always break what was so carefully built
* Juniors will never understand this code

---------------------------------------------------

# Rails is built on this, using sharp tools.

------------------------------------------------------

# Other people's code is not sacred

^ With one caveat though

-------------------------------------------------------

[.background-color: #ffffff]
![](kelder-test.png)

^ These sharp tools can be dangerous if you do not mark them as such. For example: test your patches to other people's code. It is the first thing that will potentially break or go sour when, say, a major Rails update happens. But if you have tests, they will fail and show you that you need to either adjust or maybe even remove them as they are no longer necessary.

------------------------------------------------------------

# With dynamic languages, use sharp tools.

----------------------------------------------

[.background-color: #ffffff]
![fit](qr-code.png)

----------------------

https://www.youtube.com/watch?v=3DEA8njVTIc&t=2158

-----------------------------------------------

# Bonus 1: Inline Bundler

```ruby
require "bundler/inline"

gemfile do
  source 'https://rubygems.org'
  gem 'most_perfect_library', '1.1.13'
  gem 'rspec', '~> 4`
end

RSpec.describe 'Buggy library' do
  it 'wargs when asked to' do
    expect(MostPerfect.warg).to eq("WARG!!")
    #=> This clearly does not warg...
  end
end
```

--------------------------------

# Bonus 2: Inline Rails


```ruby
# ...
  gem 'rails'
  gem 'rspec'
end

MY_RACK_APP = ->(env) { [200, {}, ["Ohai there!"]}

class RailsAppDemonstratingIssue < Rails::Application
  secrets.secret_token    = "secret_token"
  secrets.secret_key_base = "secret_key_base"

  config.logger = Logger.new($stderr)
  Rails.logger = config.logger

  routes.draw do
    mount MY_RACK_APP => '/'
  end
end

# Then use rack-test to perform requests
RailsAppDemonstratingIssue.to_app.call(your_rack_env)
```

--------------------------------------
🙏
