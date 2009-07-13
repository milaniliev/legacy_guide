ActiveRecord Legacy Database Survival Guide
==================================================
Milan Iliev, milaniliev@gmail.com

Warning: the following advice and instruction is only for use with complex, non-Rails-convention-conformant legacy databases. If you live in a nice world where you can dictate your database structure, please structure it according to convention and feel the dual joy of best practice and 

Kid Stuff
-------------------------

I have a silly MySQL database that has my admin data - user accounts, permissions data, app configuration, etc - in a 
separate schema from my regular database. (The rest of the data is also spread across multiple schemas, don't worry.)

Schema, host, username - ActiveRecord doesn't care which of these is different, it's a new database connection. In fact, my "admin" tables are also on a separate machine.

So I add, in config/database.yml:

admin:
  adapter:  mysql
  host:     database.server.com
  database: admin
  username: root
  password: secret

There is no 'admin' environment. I just needed an additional connection configured.

And, in my models for those environments, I do:

class User < ActiveRecord::Base
  establish_connection :admin
end

You could, of course, make a common superclass for all of those. I've generally not seen the need, and Single-Table Inheritance may catch on when you don't want it to.



High Performance, Complex Keys
------------------------------

My 'Peeps' and 'Whips' legacy tables have a few million rows each. And, they're connected not by a nice primary-key/foreign-key relationship, but by an obscure and complex condition:

Peeps and Whips both have a *title_number* column and a *sale_date* column. They have to be equal.

You have a couple of options here:
1. Run a batch job, outside of the app - rake task, pure SQL, anything - to update, say, Whips with a peep_id:

  UPDATE whips SET peep_id = peeps.id 
  JOIN peeps ON (whips.title_number = peeps.title_number AND whips.sale_date = peeps.sale_date)
  
  Or, if your legacy database of choice* is Oracle, enjoy the MERGE statement.

*Sure, right.

Then, use a normal Rails association - Peep has_many :whips.

 1.5 Or, if you can't modify the original tables (or like _cleanliness_), make a whip_ownerships table [with a peep_id and a whip_id] and say that Peep has_many :whip_ownerships and has_many :whips, :through => :whip_ownerships. Same can be said for Whip, of course.

In either case, when doing a Peep.all, and you need Whip data, remember to :include => :whip to avoid firing a new SELECT for each Peep's Whip.

2. Need just a few attributes of Peep per Whip? Can't do a background job? Lazy? Write a method based on Whip.find that can fetch the Peep data for you. And by method, I mean a named_scope:

  named_scope :with_peep_colums, :select => "#{Whip.table_name}.*, #{Peep.table_name}.name as owner_name", :joins => "JOIN #{Whip.table_name} on (#{Peep.table_name}.title_number = #{Whip.table_name}.title_number AND #{Peep.table_name}.sale_date = #{Whip.table_name}.sale_date)"

Due to ActiveRecord's resultset-analyzing magic, the Whip objects returned by Whip.with_peep_columns will have an owner_name


Stupid Table Names
--------------------------
  Now, why did I use all those Peep.table_names everywhere? The name is obviously peeps, isn't it?
  
  Well no. This is a legacy database, so the table is probably named 'CHFB1_PEEPS' or 'PEEP_INVENTORY'. Fortunately, there's a remedy for either:
  
  class Peep
    table_name_prefix = 'CHFB1'
  end
  
  or 
  
  class Peep
    set_table_name 'PEEP_INVENTORY'
  end
  
  To apply the prefix globally, 
    ActiveRecord::Base.table_name_prefix = 'CHFB1' 
  works in config/initializers/stupid_table_name_prefix.rb
  
  In order to abstract away the weirdness, I recommend using AR::Base.table_name rigorously in your code.


CAUTION: Extreme Hackery
--------------------------------

Okay, but what if your database setup is more weird than the mess described above? 

What if, for example, you had two users on the database, only one of which was allowed to create/drop tables, but your application normally ran using the other one?

Well kids, you *could* make another entry here, but you'd probably have to also replicate test, staging, production, etc. Plus, that part of the guide is over. So instead, a deeper look at AR::Base's connections!

ActiveRecord connects to the database using ActiveRecord::Base.establish_connection(connection) (or any subclass: User.establish_connection(connection))

Now, connection can be either a hash of connection parameters:

establish_connection(:adapter => 'mysql', :host => 'database.app.com', :port => '25', :username => 'user', :password => 'pass')

or a symbol, like :development, marking a configuration in config/database.yml usually corresponding to a Rails environment.

Rails gets the configurations by doing: 

YAML.load_file('config/database.yml) # => { 'development' => {'adapter => 'mysql', 'host' => 'database.server.com', ...}}

Fortunately, since _it_ does, you don't have to do it yourself:

ActiveRecord::Base.configurations['development] # => {'adapter => 'mysql', 'host' => 'database.server.com', ...}

You can even get the currently connected-to configuration:

ActiveRecord::Base.connection_pool.spec.config # => {'adapter => 'mysql', 'host' => 'database.server.com', ...}

Which means you can re-connect by modifying some parameters slightly:

current_connection = ActiveRecord::Base.connection_pool.spec.config
new_connection = current_connection.merge(:username => 'root', :password => 'really_secret')
ActiveRecord::Base.establish_connection(new_connection)
# do dangerous, root-y stuff
ActiveRecord::Base.establish_connection(current_connection)


The Need For Speed
----------------------------------------
What if you want to pull a large number of DB rows and do something simple, like export them to CSV?
Well, creating AR objects and accessing their typecast attributes is a little expensive. What if we could get rows as a straight hash?

User.connection.select_all('SELECT * FROM USERS') # => {:first_name => 'Bob', :last_name => 'Jim'}

So what, we're back to manually writing SQL now? All those years of nice, ActiveRecord-generated sanitized conditiosn gone, in a poof?

Not quite:

User.connection.select_all("SELECT * FROM USERS WHERE #{ User.merge_conditions({:name => ?}, ['created_at < ?', Time.now])}"

That's right, ActiveRecord::Base.merge_conditions takes an arbitrary number of AR-understandable conditions and smashes them together in one giant pile of WHERE-clause SQL. Great for adding complex SQL to conventient hashes:

User.all(User.merge_conditions(params[:user], ['created_at < ?', Time.now])) 
