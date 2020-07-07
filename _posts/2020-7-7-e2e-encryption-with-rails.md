---
title: End-to-End Encryption in Rails
tags: [gpg, rails, ruby, e2e, encryption]
hero: /img/posts/pgp/header.jpg
abstract: Nowadays, the news is full of data leaks, breaks, and hacks and one thing is clear - you don’t want to be the one who leaks all your customer data to some hackers on the internet. Learn how to encrypt your content end-to-end with Rails and PGP.
---

# _End-to-End Encryption_ in _Rails_


## Why You Should Care About Encryption

Nowadays, the news is full of data leaks, breaks, and hacks and one thing is clear - you don’t want to be the one who leaks all your customer data to some hackers on the internet. One of many ways of securing your application and your customer’s data is end-to-end encryption or (e2e encryption for short). This means that data gets encrypted as soon as possible on the user’s device, is sent encrypted over the wire, and only gets decrypted at the receiver’s device. This stands in contrast to the usual encryption at transport and encryption at rest which is used by most applications and is implemented at the protocol level (you just put SSL on your connection and no one can snoop on your data while in transport - isn’t that great?).

So which one to use?

### Encryption at the protocol level (e.g. SSL encryption)

- is easy to implement and completely transparent to the application
- protects from hackers snooping out your data while it’s transported
- is supported pretty much everywhere and is fast
- allows everyone with access to the database (from admins to hackers) to see all the content in the database

### E2E-Encryption

- removes an entire vector of attacks. No one except the designated receiver can see any data - not even the database admin
- adds a higher level of privacy - if the admin cannot accidentally see data that she shouldn’t see, you don’t need to think that much about regulating data access in production systems
- gives the user total control about her data
- is harder to implement as all data processing has to happen on the client (more about this later).

So as both methods have their pros and cons, you should carefully consider which option to use. E2E encryption works great for very personal data (e.g. social security numbers, private messages, secure notes) while it becomes a really hard problem if you’re dealing with data you want to filter on the server side (server-side full text search becomes impossible).


## How does E2E encryption work?

This post focusses on asymmetric encryption - which is just for a fancy word for saying that you can encrypt a message with a public key and decrypt it with a corresponding private key. Only the owner of the private key can decrypt a message, but everyone having access to his public keys can send him encrypted messages. A popular implementation is PGP which is mostly used to encrypt emails.


## What we are going to build

To showcase several parts of e2e encrypted systems, we’re going to build a secure messaging app where users can send each other encrypted messages. The encryption is based on our work on [CovTrace](https://covtrace.de), a digital attendence list for restaurants during Covid19-times. Features include:

- users can signup and manage their encryption keys
- users can create a new key pair of private and public key
- users can send messages to other users and encrypt them in the browser
- users can receive messages and decrypt them in the browser


## The Tech-Stack

Even though a lot of things happen on the client, we are going to use a fully server-side rendered application:

- Rails, ActiveRecord and Postgres
    - slim as a templating engine (you could of course also use ERB)
    - devise for user authentication
- Webpacker and Stimulus for the JavaScript part
- openpgp.js as an encryption library
- Turbolinks to turn the application into a single page application (SPA)

This blog post will highlight the relevant parts of encryption and key management. You can find a fully working application at GitHub: https://github.com/JensRavens/rails-stimulus-e2e-encryption

This app is a simplified version we are running successfully for CovTrace in production.

## The Application Skeleton

This section deals with the Rails-side of things: models, controllers, keeping data where it should be. If you’re only interested in the encryption part, you can skip to the next section.

Let’s get started with `User`s. Create a migration with `rails g model user keys` and modify it as follows:

```ruby
def change
  create_table :users do |t|
    t.text :keys, array: true, null: false, default: []
    t.timestamps
  end
end
```

As you can see the user has an array of (public) keys, so other users can send them encrypted messages. Let’s allow that user to login in:  `rails g devise:install User` and suddenly your user can login and sign up. Let’s give him something to login to. Add some user routes to the `routes.rb` (more about messages later):

```ruby
resources :users, only: [:index, :show, :update], shallow: true do
  resources :messages, only: :create
end
```

and create the users controller:

```ruby
class UsersController < ApplicationController
  before_action :authenticate_user!

  def index
    @users = User.where.not(keys: []).where.not(id: current_user.id).order(email: :asc)
  end

  def update
    @user = current_user
    @user.add_key params.require(:user).require(:public_key)
    redirect_to root_path
  end
end
```


Thanks to devise we get the `current_user` method and an `authenticate_user!` filter for free and we can fully focus on our logic. The index page should have a list of all users that can be contacted (only users that have public keys can receive messages). Also, the `update` method allows adding additional keys via the `User` model:

```ruby
class User < ApplicationRecord
  def add_key(key)
    keys << key
    save!
  end
end
```

Let’s build some basic UI for it in `users/index.html.slim`:

```slim
h1 Conversations
p Hello #{current_user.email}!
- if current_user.keys.any? # only allow sending messages once some keys have been added
  ul
    - @users.each do |user|
      li = link_to user.email, user
h2 Key Management
- if current_user.keys.any?
  h3 Your Keys # list all known public keys of the logged in user
  - current_user.keys.each do |key|
    pre = key
h3 Add Keys
= form_with model: current_user do |f|
  = f.label :public_key
  br
  = f.text_area :public_key, required: true
  br
  = f.submit "Add Key"
```

And with that we have a more or less functional UI (more on generating keys later):

![PGP Key Management](/img/posts/pgp/key-management.png)


Great, let’s add some messages with `rails g model message` and modifying the migration as follows:

```ruby
def change
  create_table :messages do |t|
    t.text :content, null: false
    t.references :sender, null: false, foreign_key: { to_table: :users }
    t.references :receiver, null: false, foreign_key: { to_table: :users }
    t.timestamps
  end
end
```

Messages have a sender and receiver and a text field to store the encrypted message and also a scope to retrieve the conversations a user is involved with:

```ruby
class Message < ApplicationRecord
  belongs_to :sender, class_name: "User"
  belongs_to :receiver, class_name: "User"
  scope :with_user, -> (user_id) { where(sender_id: user_id).or(where(receiver_id: user_id)) }
end
```

Now let’s give the user a way to see messages that were received to so far:

```ruby
# users_controller.rb
def show
  @user = User.find params[:id]
  @messages = Message.with_user(@user.id).with_user(current_user.id).order(created_at: :asc)
end
```

This retrieves the user from the URL and all messages that have been exchanged between the current user and the selected user.

```ruby
# users/show.html.slim
h1 = @user.email
- @messages.each do |message|
  div style=("text-align: right" if message.sender_id == current_user.id)
    div
    small = l message.created_at, format: :short
= form_with model: @message, url: user_messages_path(@user) do |f|
  = f.text_area :content
  = f.submit
```

Again relatively straight forward: Iterate over all messages and display the creation date and add a form with the message content.

Also we need a corresponding controller to persist those messages:

```ruby
class MessagesController < ApplicationController
  before_action :require_login
  def create
    @message = Message.create! params.permit(:content).to_h.merge(sender_id: current_user.id, receiver_id: params[:user_id])
    redirect_to @message.receiver
  end
end
```

This action will redirect to the receiver - which effectively reloads the page and shows the newly created message, which will look like this once we have some content:

![PGP based Chat](/img/posts/pgp/chat.png)


With the basic CRUD out of the way, let’s get to the interesting part - the encryption.

## Key Management

With the existing controller and UI the user could already create a key somewhere else and put it into the field. But to make things easier and more convenient, we’ll allow generating a key pair in the browser. For this, we will use Stimulus.js and https://github.com/openpgpjs/openpgpjs.

OpenPGP.js is quite a heavy dependency (it adds about 350kb to your bundle), so let’s load it asynchronously only if it’s needed. To better structure our frontend code, all encrypt/decrypt related code will go into `app/javascript/model/crypto.js`.

Copy the openpgp.js code into `app/javascript/lib/openpgp.js` (as of the time writing, it doesn’t work yet as a yarn dependency with webpack) and implement a loading function:

```js
export async function loadPGP() {
  await import("../lib/openpgp");
}
```

This code uses an async import (isn’t modern javascript cool?), which will tell webpacker to split the bundle into multiple chunks. This way the pgp library is only loaded if it is actually needed and will not block loading the page.

Now let’s allow the user to generate a keypair:
```js
// crypto.js

export async function generateKey() {
  await loadPGP();
  return await openpgp.generateKey({
    curve: "curve25519",
    userIds: [{ name: "Anonymous", email: "mail@example.com" }],
  });
}
```

This makes sure the dependency is loaded before calling into pgp to generate a key.  This code is using the `curve25519` encryption curve, which is quite a recent addition that results in secure, but relatively short keys.

Let’s also add a message to persist a private key in the browser for later use:
```js
// crypto.js
// persist an array of all known private keys in local storage so it can be read later
export async function registerKey(plainKey) {
  const keys = JSON.parse(localStorage.getItem("keys") || "[]");
  keys.push(plainKey);
  localStorage.setItem("keys", JSON.stringify(keys));
}
```

So let’s wire it up to our UI. First, we add some fields to the view:
```slim
# users/index.html.slim

= form_with model: current_user, data: { controller: "keys" } do |f|
  button data-action="click->keys#generate" Generate Keys
  br
  = f.label :public_key
  br
  = f.text_area :public_key, required: true, data: { target: "keys.publicKey" }
  br
  = f.label :private_key
  br
  = f.text_area :private_key, name: nil, required: true, data: { target: "keys.privateKey" }
  br
  = f.submit "Add Key", data: { action: "click->keys#register" }
```

As you can see we add a keys stimulus controller around the form and a generate button to the top. It also wires the public key input to the controller and puts the private key as a target to the controller. Note how the private key’s name is set to `nil`: this prevents this field from being included into the request (remember: we want the private key never to touch our server - or it wouldn’t be effective end to end encryption anymore). Also there’s an action on the submit button that will trigger the `register` method of our keys controller. Let’s write the controller:

```js
// app/javascript/controllers/keys_controller.js
import { Controller } from "stimulus";
import { generateKey, registerKey } from "../model/crypto";
export default class extends Controller {
  // our two text areas
  static targets = ["privateKey", "publicKey"];

  // generate a key via `crypto.js` and fill it into the forms
  async generate(event) {
    event.preventDefault();
    const key = await generateKey();
    this.privateKeyTarget.value = key.privateKeyArmored;
    this.publicKeyTarget.value = key.publicKeyArmored;
  }

  // when submitting the form, register the key in local storage
  async register() {
    const key = this.privateKeyTarget.value;
    if (key) {
      await registerKey(key);
    }
  }
}
```

And we’re done: with a click on the generate button, PGP will create a key pair and fill it into the text fields. When submitting the form, the client side will persist the private key in `localStorage` while the server will add the public key to the user’s record (just keep in mind the `form_with` will always use rails ujs for form submission, so this form is an AJAX request). Now that we have keys, let’s send some messages!

## Sending Encrypted Data

Instead of sending the plain text content, we just want to send the encrypted version. Change the message form like this:
```slim
# users/show.html.slim
= form_with model: @message, url: user_messages_path(@user), data: { controller: "encrypt", encrypt_keys: (@user.keys + current_user.keys).to_json } do |f|
  = f.hidden_field :content, data: { target: "encrypt.output" }
  = f.text_area :content, name: nil, data: { target: "encrypt.input", action: "change->encrypt#encrypt" }
  = f.submit
```

This adds an `encrypt` controller and a corresponding keys-data entry with all public keys of the user (see key rotation in the closing notes about why this is a good idea). Also, we add a hidden field that will contain the encrypted content and set the `name` of the text area to `nil` to prevent it from being submitted to the server. On change of the text area (which will be triggered when the user blurs from the field, e.g. by clicking the send button) it will call the `encrypt` function.

Let’s start by implementing the business logic for this:
```js
// crypto.js

// encrypt a text that can only be decrypted by the private keys matching the public keys given as the keys argument
export async function encryptText(text, keys) {
  await loadPGP();
  const message = openpgp.message.fromText(text);
  const { data: encrypted } = await openpgp.encrypt({
    message,
    publicKeys: keys,
  });
  return encrypted;
}

// takes an array of text keys (either public or private) and loads them into pgp understandable js objects.
export async function parseKeys(plainKeys) {
  await loadPGP();
  return await Promise.all(
    plainKeys.map((plainKey) =>
      openpgp.key.readArmored(plainKey).then((data) => data.keys[0])
    )
  );
}
```

This function takes a simple text and turns it into an encrypted message that can only be read by the keys specified as an argument. As keys, we will use the keys of the receiver and the keys of the sender so both parties can see the messages history. Let’s wire it up with a Stimulus controller:
```js
// controllers/encrypt_controller.js
import { Controller } from "stimulus";
import { parseKeys } from "../model/crypto";
export default class extends Controller {
  static targets = ["input", "output"];

  // load all keys on instantiation as this could take a moment and shouldn't be repeated all the time
  async connect() {
    const plainKeys = JSON.parse(this.data.get("keys"));
    this.keys = await parseKeys(plainKeys);
  }

  // then the user blurs from the text field, instantiate a message object from the text and encrypt it with the keys we have parsed before. Then fill the hidden field with the encrypted message so it's send to the server.
  async encrypt() {
    const message = openpgp.message.fromText(this.inputTarget.value);
    const { data: encrypted } = await openpgp.encrypt({
      message,
      publicKeys: this.keys,
    });
    this.outputTarget.value = encrypted;
  }
}
```

Now you should be able to send a message to another account already. If you look into the database, you will see that the content is encrypted:
```pgp
-----BEGIN PGP MESSAGE-----
Version: OpenPGP.js v4.10.4
Comment: https://openpgpjs.org

wV4DOFk0IGuMa1MSAQdA6evpGoFnZv/njKmLwiqj/6uC0wF9YY8i4Q/s/Yyt
Gz4wu3ox/mDICKY0WI3p6Uttmx/otiek3xP8LMBhWQg9Np0fTMT6Q13pietd
Sl4znpXB0kQB9m8u8tprjIhGMtMowbjUROl0xOy0inRC8iWiehRiarawawTX
+ufAMDRB0H7IpvUgiThASdrGimX5HP1ZwkiBYwulWg==
=EOHt
-----END PGP MESSAGE-----
```

It’s nice that we can send messages - but a bit pointless if no one can read them. Let’s fix this.

## Reading Encrypted Data

We’re almost there. Let’s start with the model this time:
```js
// crypto.js

// return the decrypted text, given a set of private keys (pgp will automatically select the right one)
export async function decryptText(text, keys) {
  await loadPGP();
  const message = await openpgp.message.readArmored(text);
  const options = {
    message,
    privateKeys: keys,
  };
  const { data: decrypted } = await openpgp.decrypt(options);
  return decrypted;
}

// load private keys from local storage where we persisted them before. This will happen several times per page, so to improve performance the promise is memoized in this module variable.
let loadedKeys = null;
export async function loadKeys() {
  if (!loadedKeys) {
    const plainKeys = JSON.parse(localStorage.getItem("keys") || "[]");
    loadedKeys = parseKeys(plainKeys);
  }
  return loadedKeys;
}
```

Let’s add some controller annotations to our view:
```slim
# views/users/show.html.slim

- @messages.each do |message|
  div style=("text-align: right" if message.sender_id == current_user.id) data-controller="decrypt" data-decrypt-content=message.content
    div data-target="decrypt.content"
    small = l message.created_at, format: :short
```

This attaches every div to its own `DecryptController` instance. Also, it assigns the message content via a data attribute and defines a target where the content should go.

Time for the controller itself:
```js
// decrypt_controller.js

import { Controller } from "stimulus";
import { decryptText, loadKeys } from "../model/crypto";
export default class extends Controller {
  static targets = ["content"];

  // as soon as the data is mounted
  async connect() {
    // load all known private keys from local storage
    this.keys = await loadKeys();
    // decrypt the text and display it in the content target
    this.contentTarget.innerText = await decryptText(
      this.data.get("content"),
      this.keys
    );
  }
}
```

Reload your browser and you should see the decrypted message on the screen. Congratulations, you have a fully end-to-end encrypted messaging application! 🎉


## Where to go from here

You have seen how to generate and persist key pairs - the public key is persisted on the server while the private key stays within the browser. Also, you have seen how to encrypt a form with openpgp.js before sending it and decrypting the data during display.

The goal of this architecture is to stay as much Rails as possible - no client-side markup generation and keeping the scripts down to a minimum.

Here are some notes if you would like to implement end-to-end encryption with this tech stack in your own Rails application:


- Stimulus picks up newly inserted messages from all content changes - so if you would like real-time messaging you can easily inject generated markup via ActionCable, e.g. via Stimulus Reflex. Once your decrypt controller is in place it doesn’t matter where the content comes from.
- In a production-ready application, you need to give a user a way to remove old keys, e.g. when a key has been compromised and replaced with a new one. By allowing multiple public keys on a user you have a way of transitioning between keys or having different keys on different devices.
- This post went for PGP as the transport mechanism as it’s a proven transport and easy to implement and debug with the cost of a huge bundle size and rather slow performance (decrypting more than 50-100 messages per page will kill your browser tab). If performance is an important consideration to you, you can go for the WebCrypto API which is supported in all browsers (even old Internet Explorers). This API is much more lightweight and performant. And as all calls to the external crypto library are implemented in `crypto.js` you can just code a drop-in replacement based on the browser API.
- You want to encrypt structured data instead of just plain text? That’s easy. Just `JSON.stringify` your complete form before submission (instead of just one field), then piece it together during decryption and assign to targets via Stimulus (e.g. by naming each field/target after the JSON key).
- To make your app production-ready, you should also consider how to deal with lost private keys. You have to think about what to display instead of entries that cannot be decrypted anymore. Also think about adding a way how to enter a private key once the user has switched devices.

Getting basic end-to-end encryption running is easier than you might have guessed. While it adds the overhead of key management, the usage of encryption can be abstracted away. This way your user’s data will be safe, even if a hacker might get a copy of your whole database. And your users will thank you.
