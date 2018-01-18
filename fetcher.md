---
title: Case Study - Fetcher HTTP Library (Incl. Redirects, Caching 'n' More)
---


All together now.
fetcher library /gem
(web: [rubylibs/fetcher](https://github.com/rubylibs/fetcher),
gem: [fetcher](https://rubygems.org/gems/fetcher)) -
Fetch text documents or binary blobs via HTTP or HTTPS.
Usage examples:



## Copy (to File)

```ruby
Fetcher.copy( 'https://raw.github.com/openfootball/at-austria/master/2013_14/bl.txt', '/tmp/bl.txt' )

# -or-

worker = Fetcher::Worker.new
worker.copy( 'https://raw.github.com/openfootball/at-austria/master/2013_14/bl.txt', '/tmp/bl.txt' )
```

## Read (into String)

```ruby
txt = Fetcher.read( 'https://raw.github.com/openfootball/at-austria/master/2013_14/bl.txt' )

# -or-

worker = Fetcher::Worker.new
txt = worker.read( 'https://raw.github.com/openfootball/at-austria/master/2013_14/bl.txt' )
```

Note: The method `read` will return a string.


## Get (HTTP Response)

``` ruby
response = Fetcher.get( 'https://raw.github.com/openfootball/at-austria/master/2013_14/bl.txt' )

# -or-

worker = Fetcher::Worker.new
response = worker.get( 'https://raw.github.com/openfootball/at-austria/master/2013_14/bl.txt' )
```

Note: The method `get` will return a `Net::HTTPResponse` object
(lets you use code, headers, body, etc.).

```ruby
puts response.code             # => '404'
                               #  Note: Returned (status) code is a string e.g. '404'
puts response.message          # => 'Not Found'
puts response.body
puts response.content_type     # => 'text/html; charset=UTF-8'
puts response['content-type']  # => 'text/html; charset=UTF-8'
                               #  Note: Headers are always downcased
                               #        e.g. use 'content-type' not 'Content-Type'
```

## Command Line

```
fetch version 0.5.0 - Lets you fetch text documents or binary blobs via HTTP, HTTPS.

Usage: fetch [options] URI
    -o, --output PATH                Output Path (default is '.')
    -v, --verbose                    Show debug trace


Examples:
  fetch https://raw.github.com/openfootball/at-austria/master/2013_14/bl.txt
  fetch -o downloads https://raw.github.com/openfootball/at-austria/master/2013_14/bl.txt
```


## Source

``` ruby
module Fetcher

  class Error < StandardError
  end

  class HttpError < Error
    attr_reader :code, :message

    def initialize( code, message )
      @code, @message = code, message
    end

    def to_s
      "HTTP request failed (NOK) => #{@code} #{@message}"
    end
  end


  class Worker

    include LogUtils::Logging

    def initialize
      ### cache for conditional get (e.g. etags and last-modified headers/checks)
      @cache = {}
      @use_cache = false
    end

    ## note: use cache[ uri ] = hash for headers+plus body+plus code(410,etc.)
    #            cache[ uri ]
    def clear_cache()              @cache = {};              end
    def cache()                    @cache;                   end
    def use_cache=(true_or_false)  @use_cache=true_or_false; end  # true|false
    def use_cache?()               @use_cache;               end


    def get( src )
      # return HTTPResponse (code,message,body,etc.)
      logger.debug "fetch - get(_response) src: #{src}"

      get_response( src )
    end


    def read( src )
      # return contents (response body) as (ascii/binary) string
      logger.debug "fetch - copy src: #{src} into string"

      response = get_response( src )

      # on error return empty string; - check: better return nil- why? why not??
      if response.code != '200'
        raise HttpError.new( response.code, response.message )
      end

      response.body.dup  # return string copy - why? why not?? (use to_s?)
    end

    def read_blob!( src )
      ## note: same as read for now
      read( src )
    end

    def read_utf8!( src )
      # return contents (response body) a string
      logger.debug "fetch - copy src: #{src} into utf8 string"

      response = get_response( src )

      # on error throw exception - why? why not??
      if response.code != '200'
        raise HttpError.new( response.code, response.message )
      end

      ###
      # Note: Net::HTTP will NOT set encoding UTF-8 etc.
      # will be set to ASCII-8BIT == BINARY == Encoding Unknown; Raw Bytes Here
      # thus, set/force encoding to utf-8

      txt = response.body.to_s
      txt = txt.force_encoding( Encoding::UTF_8 )
      txt
    end


    def copy( src, dest, opts={} )
      ## todo: add file protocol - why? why not??

      logger.debug "fetch - copy src: #{src} to dest: #{dest}"

      response = get_response( src )

      # NOTE: on error (NOK) raise exception; do NOT copy file; sorry
      if response.code != '200'
        raise HttpError.new( response.code, response.message )
      end

      ### check:
      ## why not always use wb???
      ##  how is it differet for text files?
      ##  will convert newlines (from windows to unix) ???

      # check for content type; use 'wb' for images
      if response.content_type =~ /image/ ||
         response.content_type =~ /zip/    ## use application/zip or something - why? why not??
        logger.debug '  switching to binary'
        mode = 'wb'
      else
        mode = 'w'
      end

      mode = opts[:mode]  if opts[:mode]  # if mode flags passed in -take precedence

      File.open( dest, mode ) do |f|
        f.write( response.body )
      end
    end

    def get_response( src )
      uri = URI.parse( src )

      # new code: honor proxy env variable HTTP_PROXY
      proxy = ENV['HTTP_PROXY']
      proxy = ENV['http_proxy'] if proxy.nil?   # try possible lower/case env variable (for *nix systems) is this necessary??

      if proxy
        proxy = URI.parse( proxy )
        logger.debug "using net http proxy: proxy.host=#{proxy.host}, proxy.port=#{proxy.port}"
        if proxy.user && proxy.password
          logger.debug "  using credentials: proxy.user=#{proxy.user}, proxy.password=****"
        else
          logger.debug "  using no credentials"
        end
      else
        logger.debug "using direct net http access; no proxy configured"
        proxy = OpenStruct.new   # all fields return nil (e.g. proxy.host, etc.)
      end

      http_proxy = Net::HTTP::Proxy( proxy.host, proxy.port, proxy.user, proxy.password )

      redirect_limit = 6
      response = nil

      until false
        raise ArgumentError, 'HTTP redirect too deep' if redirect_limit == 0
        redirect_limit -= 1

        http = http_proxy.new( uri.host, uri.port )

        logger.debug "GET #{uri.request_uri} uri=#{uri}, redirect_limit=#{redirect_limit}"

        headers = { 'User-Agent' => "fetcher gem v#{VERSION}" }

        if use_cache?
          ## check for existing cache entry in cache store (lookup by uri)
          ## todo/fix: normalize uri!!!! - how?
          ##  - remove query_string ?? fragement ?? why? why not??

          ## note:  using uri.to_s  should return full uri e.g. http://example.com/page.html

          cache_entry = cache[ uri.to_s ]
          if cache_entry
            logger.info "found cache entry for >#{uri.to_s}<"
            if cache_entry['etag']
              logger.info "adding header If-None-Match (etag) >#{cache_entry['etag']}< for conditional GET"
              headers['If-None-Match'] = cache_entry['etag']
            end
            if cache_entry['last-modified']
              logger.info "adding header If-Modified-Since (last-modified) >#{cache_entry['last-modified']}< for conditional GET"
              headers['If-Modified-Since'] = cache_entry['last-modified']
            end
          end
        end

        request = Net::HTTP::Get.new( uri.request_uri, headers )
        if uri.instance_of? URI::HTTPS
          http.use_ssl = true
          http.verify_mode = OpenSSL::SSL::VERIFY_NONE
        end

        response   = http.request( request )

        if response.code == '200'
          logger.debug "#{response.code} #{response.message}"
          logger.debug "  content_type: #{response.content_type}, content_length: #{response.content_length}"
          break  # will return response
        elsif( response.code == '304' ) # -- Not Modified - for conditional GETs (using etag,last-modified)
          logger.debug "#{response.code} #{response.message}"
          break  # will return response
        elsif( response.code == '301' || response.code == '302' || response.code == '303' || response.code == '307' )
          # 301 = moved permanently
          # 302 = found
          # 303 = see other
          # 307 = temporary redirect
          logger.debug "#{response.code} #{response.message} location=#{response.header['location']}"
          newuri = URI.parse( response.header['location'] )
          if newuri.relative?
            logger.debug "url relative; try to make it absolute"
            newuri = uri + response.header['location']
          end
          uri = newuri
        else
          puts "*** error - fetch HTTP - #{response.code} #{response.message}"
          break  # will return response
        end
      end

      response
    end # method copy

  end # class Worker

end  # module Fetcher
```

(Source: [rubylibs/fetcher/lib/fetcher/worker.rb](https://github.com/rubylibs/fetcher/blob/master/lib/fetcher/worker.rb))
