# rails-secure-code-review-checklist
A checklist with code snippets to help security engineers and developers to find security bugs in a security code review for a Rails codebase. 

Although Brakeman is a great tool for this purpose, I sometimes find myself not having the time to filter through the bugs reported by brakeman. Instead, I grep for some insecure code patterns on my Sublime editor and that helps me dig out some bugs quickly. So below is a checklist of things to grep for, some safe and unsafe code samples, that I curated from the tonne of Rails resources available out there.

Note: This is not a comprehensive list and using Brakeman is still recommended to get more coverage. 

# Contents
- SQL injection
- Command injection
- Cross-site Scripting (XSS)
- Cross-site Request Forgery (CSRF)
- Serialization
- Sessions, cookies and authentication
- Authorization
- Server-side request forgery (SSRF)
- Redirects
- File uploads
- Cross-origin Resource Sharing (CORS)
- Insecure Direct Object Reference (IDOR)
- Logging
- Mass Assignment

## SQL Injection

Rails comes with clever methods that are immune to SQL injection. Ruby on Rails provides an interface called Active Record, an object-relational mapping (ORM) abstraction that facilitates database access. AR sanitizes data passed to certain methods. Passing arrays and hashes to methods also is safe from injection. Below  are instances where data is not sanitized by these methods:

Grep for the following and then review:

```ruby 
.find_by(
.where
.from(
.delete_all(
.delete_by(
.destroy_by(
.update_all(
.group(
.exists(
.select(
.joins(
.calculate(
```

Unsafe code:

```ruby
Project.where("name = '#{params[:name]}'")
Order.joins(params[:table])
User.find_by params[:id]

```

Safe code:

```ruby
User.where({ name: params[:name] })
User.where(name: params[:name])
User.find_by(name: params[:name])
User.find_by_something(something)
User.find(something)
User.where(["name = ?", "#{params[:name]}"])
.santize_sql()
```

Prevention

As shown in the safe code snippets above, the following measures can be used:
- use parameterized queries (?)
- use methods that apply SQL sanitization by default, such as find_by_something
- use array or hashes in model instances. 
- use sanitize_sql() method where sanitization is not performed by default

References and further reading: 
- https://guides.rubyonrails.org/security.html
- https://rails-sqli.org/
- https://www.netsparker.com/blog/web-security/sql-injection-ruby-on-rails-development/
- https://www.stackhawk.com/blog/sql-injection-prevention-rails/
- https://knowledge-base.secureflag.com/vulnerabilities/sql_injection/sql_injection_ruby.html


## Command Injection

Grep for the following and then review:

```ruby
exec(
system(
syscall(
eval(
constantize(
render(
FileList
....
`.*`  // backticks
ImageMagick
%x(
%x[
Open3.capture2(".*")
Open3.capture2e(".*")
Open3.capture3(".*")
Rails.logger.debug("Running:
```

Unsafe code:

```ruby
system(mkdir "#{params[:project_name]}")
system("cat", user_supplied_path) // fix: use file.read 
Open3.capture2e("ls #{params[:project_name]}")
```

Safe code:

```ruby
system("mkdir","params[:project_name]")
Open3.capture2e("ls", path)
Open3.popen3([cmdname, argv0], arg1, ...)
```

Prevention:

- Use parameterized methods
- Sanitize input / use an allowlist
- use Ruby's file handling modules such as FileUtils
- Use native API methods like IO.capture2 or Open3.popen, that ingest arguments as a list
- For constantize and render methods, add an allow list of params in their method definitions

References and further reading:
- https://www.stackhawk.com/blog/command-injection-ruby/
- http://gavinmiller.io/2015/fixing-command-injection-vulnerabilities/
- https://knowledge-base.secureflag.com/vulnerabilities/code_injection/os_command_injection_ruby.html
- https://docs.guardrails.io/docs/en/vulnerabilities/ruby/insecure_use_of_dangerous_function

## Cross-site Scripting

grep for:

```ruby
%= link_to
<%= raw @product.name
<%= @product.name
<%= content_tag
<%= html_safe
<%==
<% = something.to_json
render inline:  "Thanks #{@user.name}!"
render text:
ERB.new

Strip_tags()
Strip_links()
Sanitize()
```

Best practice:

- link_to should only accept http and https using URI.parse
- link_to should start with / (to accept only relative paths)
- Use json_escape()
- Use escape_javascript
- Enable ActiveSupport#escape_html_entities_in_json
- Escape with <%= h and <%= j
```ruby
render plain:
render inline: "Thanks <%= @user.name %>"
```

- Rails after 4 uses ERB::Util#html_escape

Whitelist:

- For numbers use .to_i
- Put js data in between ' ' so that 
- Use safeERB plugin

## CSRF

Unsafe

-  #match to add routes in routes.rb is bad, because match will match for all VERBS. This is bad since CSRF protection is not available for GET in ruby. So use #get, #post, etc
-  Using GET for important actions is bad
``` ruby
protect_from_forgery except
```
is bad

Best practices:

- Use protect_from_forgery with exception directive in ApplicationController. 
- Throwing an exception when verified_request fails is safe because in some cases, after the verified_request check fails,  the handle_unverified_request may not be implemented correctly in which case, the forged requests goes through. 
- Examples of forged request going through:
  - The handle_unverified_request method only invalidates session but not cookies and the devs have used their own cookies instead of session helpers
	 - The request takes values from user input instead of session 

```ruby 
verify :method => :post, :only => [:transfer], :redirect_to => {:action => :list}
```

## Deserialization 

Unsafe: 

Do not use YAML or Marshall load to deserialize user input. Use JSON for user input cause it only returns primitive types

```ruby
ActiveRecord::Base serialize :xyz
YAML.load
Marshal.load
Marshal.restore
```
Best practice, but with caution:

```ruby
YAML.safe_load
JSON.load
JSON.parse
JSON.restore
```


## Sessions, cookies and authentication

Best practices: 

- Use authentication gems like Devise, AuthLogic or Clearance
- Check password strength - use Devise's ```ruby validate :password_strength```
- use MFA gems like authy-devise
- use @current_user to fetch data instead of Model.find_by(userid)
- Set secure and httponly flags
- set Config.force_ssl = true
- Use secure_compare

## Authorization

Best practices: 

- use a gem like CanCanCan
- add
  ```ruby
  before_action :authorize_user
  ```
- For Pundit, add this to check if authorize has been added to every controller
  
```ruby
class ApplicationController < ActionController::Base
  include Pundit::Authorization
  after_action :verify_authorized
End
```

## Crypto

Best practices: 

```ruby 
config.force_ssl = true
```

Keys should be stored within a cryptographic vault 
All key operations should be performed within this vault.

## Redirects

Grep for unsafe code to review:

```ruby 
Redirect_to
```
catch all routes like 
```ruby
"match ':
```

Best practices:

```ruby
Only_path: true
URI.parse(params[dest]).path
For external, external:true
```
## SSRF

grep and look for:

```ruby
Open(params:[url]).read
.get
.get_response
.post
```

Best practice:

Ssrf gem

```ruby
require 'ssrf_filter'
SsrfFilter.get(params[:url])
```


## IDOR

Best practices: 

```ruby 
@user = current_user
```

## CORS

Best practices:

```ruby 
Rack::cors
```

## Logging

Best practice:

```ruby 
config.filter_parameters
```

## File upload/download

Attacks:

- Filename payload
- Changing file metadata using a tool like exiftool
- File content like svg <svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)"/>
- Using gif file with payload as source, to bypass self-src CSP
GIF89a/*<svg/onload=alert(1)>*/=alert(document.domain)//;
- Code for download: Send_file()

Best practices:

- Use FileUtils, sanitize filename
- Store files outside webroot or on different server (like an s3), sandbox
- Run file through Antivirus
- for zipped files, estimate file size post compression
- Save file with a new random name
- Use image rewriting libraries, don’t use imagemagick
- Never allow executable extension types
- disable exec privs on file

## Mass Assignment

Unsafe: (blocklist instead of allowlist)

```ruby
.Permit!
Attr_protected 
```

Best practices:

```ruby 
config.active_record.whitelist_attributes = true
```
(in config/application.rb)
```ruby
Attr_accessible
```
(whitelist)
- use strong parameter gem
```ruby
config.active_record.whitelist_attributes
```
set to :strict
```ruby 
config.active_record.mass_assignment_sanitize
```
This will throw an error for every mass assignment because our whitelist is empty. From the errors, we can review each attr and build our whitelist


Recommended GEMS and libraries:

| Category            | Tools/Technologies                                                                                     |
|---------------------|-------------------------------------------------------------------------------------------------------|
| **Authentication**  | Devise with OmniAuth, AuthLogic, Authy (2FA)                                                           |
| **Authorization**   | Pundit (authorize or policy_scope, has policies and scopes), CanCanCan                                |
| **Sessions**        | CookieStore (encrypts)                                                                                |
| **CORS**            | Rack-cors                                                                                             |
| **Response Header** | Secure-headers: CSP, HSTS, X-Frame-Options, Clear-site-data                                           |
| **Third Party Scanning** | Bundler audit, GuardRails, Snyk                                                             |
| **Security Tools**  | Brakeman, Use a WAF like ModSecurity, Detectify - application scanner                                 |
| **CAPTCHA**         | reCAPTCHA                                                                                             |





