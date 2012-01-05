---
layout: post
category: how-to
tags : 
  - performance
  - scalability
  - jruby
  - rails
  - tomcat
title: Thread-safe JRuby on Rails HOW-TO
---

###With the help of [Warbler](https://github.com/jruby/warbler/blob/master/README.rdoc), [Apache Tomcat](http://tomcat.apache.org) and [JNDI Connection Pool](http://en.wikipedia.org/wiki/Jndi)

###Read those posts before anything else otherwise you'll find yourself in a very lonely place:

* [JRuby on Rails and Thread Safety](http://www.slideshare.net/Naoto.Takai/jruby-on-rails-and-thread-safety-presentation)
* [Q/A: What Thread-safe Rails Means ](http://blog.headius.com/2008/08/qa-what-thread-safe-rails-means.html)
* [Using JDBC Connection Pools with NetBeans 6, JRuby RoR, MySQL and Glassfish](http://blog.louspringer.com/2007/09/11/using-jdbc-connection-pools-with-netbeans-6-jruby-ror-mysql-and-glassfish)
* [Apache Tomcat 6.0 - JNDI Datasource HOW-TO](http://tomcat.apache.org/tomcat-6.0-doc/jndi-datasource-examples-howto.html#MySQL_DBCP_Example)

##Prerequisites
A [JVM](http://en.wikipedia.org/wiki/Java_virtual_machine) must be installed on your system.

Install [JRuby](http://jruby.org) ([RVM](https://rvm.beginrescueend.com/interpreters/jruby) is recommended `rvm install jruby`).

If you have installed [RVM](https://rvm.beginrescueend.com/interpreters/jruby), run this command in a terminal
`rvm use jruby`

You need Rails **3.1** app with a database like MySQL, **PostgreSQL**, Oracle ... forget SQLite.

If you don't have one, run these commands in a terminal:

{% highlight bash %}
gem install rails
rails new your_rails_app
cd your_rails_app
bundle exec rails generate scaffold Post title:string body:text
{% endhighlight %}

In this _HOW-TO_ we will use a PostgreSQL database (because I like it ;) ) and Apache Tomcat 6 or upper, because it's the most popular.


Add ActiveRecord JDBC PostgreSQL Adapter to your Gemfile, like this:

{% highlight ruby %}
if defined?(JRUBY_VERSION)
  gem 'activerecord-jdbcpostgresql-adapter'
  gem 'jruby-openssl'
else
  gem 'pg'
end
{% endhighlight %}

Add Warbler to your Gemfile, like this:

{% highlight ruby %}
if defined?(JRUBY_VERSION)
  group :development do
    gem 'warbler'
  end
end
{% endhighlight %}

Run this command in a terminal:
`bundle install`

Then run this one:
`bundle exec warble config`

##Config Rails in thread safe mode

In your RAILS_ROOT, open the file `config\environments\production.rb` and uncomment this line:
`# config.threadsafe!`

Then add those lines to ensure you can use rake tasks in production env:

{% highlight ruby %}
  # Allow rake tasks to autoload models in thread safe mode, more info at http://stackoverflow.com/a/4880253
  config.dependency_loading = true if $rails_rake_task
{% endhighlight %}

Finally in your RAILS_ROOT open the file `config\warble.rb`
And replace these 2 lines:

{% highlight ruby %}
# config.webxml.jruby.min.runtimes = 2
# config.webxml.jruby.max.runtimes = 4
{% endhighlight %}

With these lines:

{% highlight ruby %}
config.webxml.jruby.min.runtimes = 1
config.webxml.jruby.max.runtimes = 1
{% endhighlight %}

## Migrate database, create JNDI connection & configure Rails

Ensure your database production environment is configured, then run these commands in a terminal:

{% highlight bash %}
bundle exec rake db:create RAILS_ENV=production
bundle exec rake db:migrate RAILS_ENV=production
{% endhighlight %}

In your Apache Tomcat directory, open the file `conf\context.xml`.

Add the **Resource** tag inside the **Context** tag, like this:

{% highlight xml %}
<Context>
      <!-- ... -->
      <Resource name="jdbc/your_jndi_name" auth="Container" type="javax.sql.DataSource"
         maxActive="100" maxIdle="30" maxWait="10000"
         username="your_username" password="your_password" driverClassName="org.postgresql.Driver"
         url="jdbc:postgresql://your_hostname:5432/your_database_name"/>

</Context>
{% endhighlight %}

Copy your JDBC driver into the Apache Tomcat `lib` folder.

For [Ubuntu](http://ubuntu.com):
`sudo cp ~/.rvm/gems/jruby-1.6.5/gems/jdbc-postgres-9.*/lib/*.jar /usr/share/tomcat6/lib`

In your RAILS_ROOT open the file `config\database.yml`.

Rename the `production` entry in `production_jdbc` and add this one:

{% highlight yaml %}
production:
  adapter: jdbc
  jndi: java:comp/env/jdbc/your_jndi_name
  driver: postgresql
  encoding: utf8
  wait_timeout: 5
  pool: 5
{% endhighlight %}

In your RAILS_ROOT open the file `config\warble.rb`.

And replace this line:
`# config.webxml.jndi = 'jdbc/rails'`

With this one:
`config.webxml.jndi = 'jdbc/your_jndi_name'`

Finally create a `config/initializers/connection_pool_fix.rb` file
with this content:

{% highlight ruby %}
# Monkey patch ConnectionPool#checkout to avoid database connection timeouts
# Source: https://github.com/rails/rails/issues/2547
# For Rails 3.2.0 and upper, You need to check if the pool error still occurs
if Rails.version < "3.2.0"
  class ActiveRecord::ConnectionAdapters::ConnectionPool
    def checkout
      # Checkout an available connection
      @connection_mutex.synchronize do
        loop do
          conn = if @checked_out.size < @connections.size
                   checkout_existing_connection
                 elsif @connections.size < @size
                   checkout_new_connection
                 end
          return conn if conn
  
          # No connections available; wait for one
          if @queue.wait(@timeout)
            next
          else
            # try looting dead threads
            clear_stale_cached_connections!
            if @size == @checked_out.size
              raise ConnectionTimeoutError, "could not obtain a database connection#{" within #{@timeout} seconds" if @timeout}.  The max pool size is currently #{@size}; consider increasing it."
            end
          end
          
        end
      end
    end
  end
end
{% endhighlight %}

## Create the War file and restart Apache Tomcat

Run these commands in a terminal:

{% highlight bash %}
bundle exec rake assets:precompile
bundle exec warble
{% endhighlight %}

Stop the Apache Tomcat service:
`sudo service tomcat6 stop`

In the Apache Tomcat directory, copy your `your_rails_app.war` file inside the `webapps` folder.

For Ubuntu:
`sudo cp your_rails_app.war /var/lib/tomcat6/webapps`

Restart the Apache Tomcat service:
`sudo service tomcat6 start`

Open this URL with your Web Browser: http://localhost:8080/your_rails_app/posts

Enjoy !

_Special thanks to [Ritchie Young](https://github.com/ritchiey) who corrected my spelling mistakes._
