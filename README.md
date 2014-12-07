binding [![Build Status](https://drone.io/github.com/macaron-contrib/binding/status.png)](https://drone.io/github.com/macaron-contrib/binding/latest) [![](http://gocover.io/_badge/github.com/macaron-contrib/binding)](http://gocover.io/github.com/macaron-contrib/binding)
=======

Middlware binding provides request data binding and validation for [Macaron](https://github.com/Unknwon/macaron).

[API Reference](https://gowalker.org/github.com/macaron-contrib/binding)

### Installation

	go get github.com/macaron-contrib/binding
	
## Features

 - Automatically converts data from a request into a struct ready for your application
 - Supports form, JSON, and multipart form data (including file uploads)
 - Can use interfaces
 - Provides data validation facilities
 	- Enforces required fields
 	- Invoke your own data validator
 - Built-in error handling (or use your own)
 - 99% test coverage

## Usage

#### Getting form data from a request

Suppose you have a contact form on your site where at least name and message are required. We'll need a struct to receive the data:

```go
type ContactForm struct {
	Name    string `form:"name" binding:"required"`
	Email   string `form:"email"`
	Message string `form:"message" binding:"required"`
}
```

Then we simply add our route in Macaron:

```go
m.Post("/contact/submit", binding.Bind(ContactForm{}), func(contact ContactForm) string {
	return fmt.Sprintf("Name: %s\nEmail: %s\nMessage: %s",
		contact.Name, contact.Email, contact.Message)
})
```

That's it! The `binding.Bind` function takes care of validating required fields. If there are any errors (like a required field is empty), `binding` will return an error to the client and your app won't even see the request.

(Caveat: Don't try to bind to embedded struct pointers; it won't work. See [martini-contrib/binding issue 30](https://github.com/martini-contrib/binding/issues/30) if you want to help with this.)


#### Getting JSON data from a request

To get data from JSON payloads, simply use the `json:` struct tags instead of `form:`. Pro Tip: Use [JSON-to-Go](http://mholt.github.io/json-to-go/) to correctly convert JSON to a Go type definition. It's useful if you're new to this or the structure is large/complex.



#### Custom validation

If you want additional validation beyond just checking required fields, your struct can implement the `binding.Validator` interface like so:

```go
func (cf ContactForm) Validate(errors binding.Errors, req *http.Request) binding.Errors {
	if strings.Contains(cf.Message, "Go needs generics") {
		errors = append(errors, binding.Error{
			FieldNames:     []string{"message"},
			Classification: "ComplaintError",
			Message:        "Go has generics. They're called interfaces.",
		})
	}
	return errors
}
```

Now, any contact form submissions with "Go needs generics" in the message will return an error explaining your folly.


#### Binding to interfaces

If you'd like to bind the data to an interface rather than to a concrete struct, you can specify the interface and use it like this:

```go
m.Post("/contact/submit", binding.Bind(ContactForm{}, (*MyInterface)(nil)), func(contact MyInterface) {
	// ... your struct became an interface!
})
```



## Description of Handlers

Each of these middleware handlers are independent and optional, though be aware that some handlers invoke other ones.


### Bind

`binding.Bind` is a convenient wrapper over the other handlers in this package. It does the following boilerplate for you:

 1. Deserializes request data into a struct
 2. Performs validation with `binding.Validate`
 3. Bails out with `binding.ErrorHandler` if there are any errors

Your application (the final handler) will not even see the request if there are any errors.

Content-Type will be used to know how to deserialize the requests.

**Important safety tip:** Don't attempt to bind a pointer to a struct. This will cause a panic [to prevent a race condition](https://github.com/codegangsta/martini-contrib/pull/34#issuecomment-29683659) where every request would be pointing to the same struct.


### Form

`binding.Form` deserializes form data from the request, whether in the query string or as a form-urlencoded payload. It only does these things:

 1. Deserializes request data into a struct
 2. Performs validation with `binding.Validate`

Note that it does not handle errors. You may receive a `binding.Errors` into your own handler if you want to handle errors. (For automatic error handling, use `binding.Bind`.)



### MultipartForm and file uploads

Like `binding.Form`, `binding.MultipartForm` deserializes form data from a request into the struct you pass in. Additionally, this will deserialize a POST request that has a form of *enctype="multipart/form-data"*. If the bound struct contains a field of type [`*multipart.FileHeader`](http://golang.org/pkg/mime/multipart/#FileHeader) (or `[]*multipart.FileHeader`), you also can read any uploaded files that were part of the form.

This handler does the following:

 1. Deserializes request data into a struct
 2. Performs validation with `binding.Validate`

Again, like `binding.Form`, no error handling is performed, but you can get the errors in your handler by receiving a `binding.Errors` type.

#### MultipartForm example

```go
type UploadForm struct {
	Title      string                `form:"title"`
	TextUpload *multipart.FileHeader `form:"txtUpload"`
}

func main() {
	m := macaron.Classic()
	m.Post("/", binding.MultipartForm(UploadForm{}), uploadHandler(uf UploadForm) string {
		file, err := uf.TextUpload.Open()
		// ... you can now read the uploaded file
	})
	m.Run()
}
```

### Json

`binding.Json` deserializes JSON data in the payload of the request. It does the following things:

 1. Deserializes request data into a struct
 2. Performs validation with `binding.Validate`

Similar to `binding.Form`, no error handling is performed, but you can get the errors and handle them yourself.

### Validate

`binding.Validate` receives a populated struct and checks it for errors, first by enforcing the `binding:"required"` value on struct field tags, then by executing the `Validate()` method on the struct, if it is a `binding.Validator`.

*Note:* Marking a field as "required" means that you do not allow the zero value for that type (i.e. if you want to allow 0 in an int field, do not make it required).

### ErrorHandler

`binding.ErrorHandler` is a small middleware that simply writes an error code to the response and also a JSON payload describing the errors, *if* any errors have been mapped to the context. It does nothing if there are no errors.

 - Deserialization errors yield a 400
 - Content-Type errors yield a 415
 - Any other kinds of errors (including your own) yield a 422 (Unprocessable Entity)

## Credits

This package is forked from [martini-contrib/binding](https://github.com/martini-contrib/binding) with modifications.

## License

This project is under Apache v2 License. See the [LICENSE](LICENSE) file for the full license text.