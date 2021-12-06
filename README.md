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

## SQL Injection

Rails comes with many clever methods that are immune to SQL injection. However there are instances where SQL injection is still possible. 

Grep for the following and then review:

```ruby 
.find_by(
.delete_all(
.delete_by(
.destroy_by(
.update_all(
.where
.from(
.group(
.exists(
.select(
.joins(
```

Unsafe code:

```ruby
Project.where("name = '#{params[:name]}'")
User.find(params[:name])
```

Safe code:

```ruby
User.where({ name: params[:name] })
User.find_by(name: params[:name])
User.find_by_something(something)
User.find(id)
User.where(["name = ?", "#{params[:name]}"])
Model.where(login: entered_user_name, password: entered_password)
.santize_sql()
```

Countermeasures

As shown in the safe code snippets above, the following measures can be used:
- use parameterized queries (?)
- use methods that apply SQL sanitization by default, such as find_by_something
- use array or hashes in model instances. 
- use sanitize_sql() method when default sql sanitization is not performed by the methods











