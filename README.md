# rails-secure-code-review-checklist
A checklist with code snippets to help security engineers and developers to find security bugs in a security code review for a Rails codebase. 

Although Brakeman is a great tool for this purpose, I sometimes find myself not having the time to filter through the bugs reported by brakeman. Instead, I grep for some insecure code patterns on my Sublime editor and that helps me dig out some bugs quickly. So below is a checklist of things to grep for, some safe and unsafe code samples, that I curated from the tonne of Rails resources available out there.

Note: This is not a comprehensive list and using Brakeman is still recommended to get 100% coverage. 

# Contents
- SQL injection
- Command injection
- Cross-site Scripting (XSS)
- Cross-site Request Forgery (CSRF)
- Sessions, cookies and authentication
- Authorization
- Server-side request forgery (SSRF)
- Redirects
- HTTP and TLS
- File uploads
- Cross-origin Resource Sharing (CORS)
- Insecure Direct Object Reference (IDOR)
- Logging
- Secrets
- Serialization
- Mass Assignment
- Insecure Gems

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




