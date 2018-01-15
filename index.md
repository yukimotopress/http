---
title: Net::HTTP Basics
---



## Standard HTTP Request   <!-- normal.rb -->

``` ruby
require "net/http"
require "uri"

uri = URI.parse("http://google.com/")

# Shortcut
response = Net::HTTP.get_response(uri)

# Will print response.body
Net::HTTP.get_print(uri)

# Full
http = Net::HTTP.new(uri.host, uri.port)
response = http.request(Net::HTTP::Get.new(uri.request_uri))
```


## Basic Auth   <!-- basic_auth.rb -->

``` ruby
require "net/http"
require "uri"

uri = URI.parse("http://google.com/")

http = Net::HTTP.new(uri.host, uri.port)
request = Net::HTTP::Get.new(uri.request_uri)
request.basic_auth("username", "password")
response = http.request(request)
```

## Dealing with Response Objects   <!-- responses.rb -->

``` ruby
require "net/http"
require "uri"

uri = URI.parse("http://google.com/")

http = Net::HTTP.new(uri.host, uri.port)
request = Net::HTTP::Get.new(uri.request_uri)

response = http.request(request)

response.code             # => 301
response.body             # => The body (HTML, XML, blob, whatever)
# Headers are lowercased
response["cache-control"] # => public, max-age=2592000

# Listing all headers
response.each_header { |h| do_something(h, response[h]) } # => location = http://www.google.com/
                                                          # => content-type = text/html; charset=UTF-8
                                                          # => cache-control = public, max-age=2592000
                                                          # etc...
```


## POST Form Request  <!--  post_form.rb -->

``` ruby
require "net/http"
require "uri"

uri = URI.parse("http://example.com/search")

# Shortcut
response = Net::HTTP.post_form(uri, {"q" => "My query", "per_page" => "50"})

# Full control
http = Net::HTTP.new(uri.host, uri.port)

request = Net::HTTP::Post.new(uri.request_uri)
request.set_form_data({"q" => "My query", "per_page" => "50"})

# Tweak headers, removing this will default to application/x-www-form-urlencoded 
request["Content-Type"] = "application/json"

response = http.request(request)
```


## File Upload - HTML Style w/ input type="file"   <!-- file_upload_html_style  -->

``` ruby
require "net/http"
require "uri"

# Token used to terminate the file in the post body. Make sure it is not
# present in the file you're uploading.
BOUNDARY = "AaB03x"

uri = URI.parse("http://something.com/uploads")
file = "/path/to/your/testfile.txt"

post_body = []
post_body << "--#{BOUNDARY}\r\n"
post_body << "Content-Disposition: form-data; name=\"datafile\"; filename=\"#{File.basename(file)}\"\r\n"
post_body << "Content-Type: text/plain\r\n"
post_body << "\r\n"
post_body << File.read(file)
post_body << "\r\n--#{BOUNDARY}--\r\n"

http = Net::HTTP.new(uri.host, uri.port)
request = Net::HTTP::Post.new(uri.request_uri)
request.body = post_body.join
request["Content-Type"] = "multipart/form-data, boundary=#{BOUNDARY}"

http.request(request)

# Alternative method, using Nick Sieger's multipart-post gem
require "rubygems"
require "net/http/post/multipart"

request = Net::HTTP::Post::Multipart.new uri.request_uri, "file" => UploadIO.new(file, "application/octet-stream")
http = Net::HTTP.new(uri.host, uri.port)
http.request(request)

# Another alternative, using Rack 1.3 +
require 'rack'
uri     = URI.parse("http://something.com/uploads")
http    = Net::HTTP.new(uri.host, uri.port)
request = Net::HTTP::Post.new(uri.request_uri)

request.body = Rack::Multipart::Generator.new(
  "form_text_field" => "random text here",
  "file"            => Rack::Multipart::UploadedFile.new(path_to_file, file_mime_type)
).dump
request.content_type = "multipart/form-data, boundary=#{Rack::Multipart::MULTIPART_BOUNDARY}"

http.request(request)

http.start do |connection|
  response = retrying_request(connection, request)
end
```

## SSL/HTTPS Request   <!-- ssl_and_https.rb -->

``` ruby
require "net/https"
require "uri"

# A regular-ish https request.
#
# ssltest7.bbtest.net is Thawte's SSL test site. Net::HTTP will use the CA
# certificates installed on your system by default, which most likely includes
# the Thawte cert that signed ssltest7.bbtest.net.
http = Net::HTTP.new("ssltest7.bbtest.net", 443)
http.use_ssl = true
http.verify_mode = OpenSSL::SSL::VERIFY_PEER

response = http.request(Net::HTTP::Get.new("/"))
response.body
response.status
# .. do normal Net::HTTP response stuff here (see separate cheat sheet entry)

# You can specify custom CA certs. If your production system only connects to
# one particular server, you should specify these, and bundle them with your
# app, so that you don't depend OS level pre-installed certificates in the
# production environment.
http = Net::HTTP.new("verysecure.com", 443)
http.use_ssl = true
http.verify_mode = OpenSSL::SSL::VERIFY_PEER

store = OpenSSL::X509::Store.new
store.set_default_paths # Optional method that will auto-include the system CAs.
store.add_cert(OpenSSL::X509::Certificate.new(File.read("/path/to/ca1.crt")))
store.add_cert(OpenSSL::X509::Certificate.new(File.read("/path/to/ca2.crt")))
store.add_file("/path/to/ca3.crt") # Alternative syntax for adding certs.
http.cert_store = store

response = http.request(Net::HTTP::Get.new("/"))


# Client certificate example. Some servers use this to authorize the connecting
# client, i.e. you. The server you connect to gets the certificate you specify,
# and they can use it to check who signed the certificate, and use the
# certificate fingerprint to identify exactly which certificate you're using.
http = Net::HTTP.new("verysecure.com", 443)
http.use_ssl = true
http.verify_mode = OpenSSL::SSL::VERIFY_PEER
http.key = OpenSSL::PKey::RSA.new(File.read("/path/to/client.key"), "optional passphrase argument")
http.cert = OpenSSL::X509::Certificate.new(File.read("/path/to/client.crt"))

response = http.request(Net::HTTP::Get.new("/"))


# You can also skip verification. This is almost certainly a bad idea, read more
# here:
# http://www.rubyinside.com/how-to-cure-nethttps-risky-default-https-behavior-4010.html
http.verify_mode = OpenSSL::SSL::VERIFY_NONE
```

## HTTP POST / GET / PUT / DELETE Methods  <!-- methods_and_rest.rb -->

``` ruby
# Basic REST.
# Most REST APIs will set semantic values in response.body and response.code.
require "net/http"

http = Net::HTTP.new("api.restsite.com")

request = Net::HTTP::Post.new("/users")
request.set_form_data({"users[login]" => "quentin"})
# or
request.body = '{"username":"quentin"}'
response = http.request(request) # Use nokogiri, hpricot, etc to parse response.body.


request = Net::HTTP::Get.new("/users/1")
response = http.request(request) # As with POST, the data is in response.body.


request = Net::HTTP::Put.new("/users/1")
request.set_form_data({"users[login]" => "changed"})
response = http.request(request)


request = Net::HTTP::Delete.new("/users/1")
response = http.request(request)
```
