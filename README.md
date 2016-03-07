## Qor Admin

Instantly create a beautiful, cross platform, configurable Admin Interface and API for managing your data in minutes.

[![GoDoc](https://godoc.org/github.com/qor/admin?status.svg)](https://godoc.org/github.com/qor/log)

## Features

- Admin Interface for managing data
- JSON API
- Association handling
- Search and filtering
- Actions/Batch Actions
- Authentication and Authorization (based on Permissions)
- Extendability

## Quick Start

```go
package main

import (
    "net/http"

    "github.com/jinzhu/gorm"
    _ "github.com/mattn/go-sqlite3"
    "github.com/qor/qor"
    "github.com/qor/admin"
)

// Create a GORM-backend model
type User struct {
  gorm.Model
  Name string
}

// Create another GORM-backend model
type Product struct {
  gorm.Model
  Name        string
  Description string
}

func main() {
  DB, _ := gorm.Open("sqlite3", "demo.db")
  DB.AutoMigrate(&User{}, &Product{})

  // Initalize
  Admin := admin.New(&qor.Config{DB: &DB})

  // Create resources from GORM-backend model
  Admin.AddResource(&User{})
  Admin.AddResource(&Product{})

  // Register route
  mux := http.NewServeMux()
  // amount to /admin, so visit `/admin` to view the admin interface
  Admin.MountTo("/admin", mux)

  fmt.Println("Listening on: 9000")
  http.ListenAndServe(":9000", mux)
}
```

`go run main.go` and visit `localhost:9000/admin` to see the result !

## General Setting

### Site Name

Use `SetSiteName` to set admin's HTML title, not only set the title, but it will also auto load javascripts, stylesheets files based on the name, you could use that to customize the admin interface.

For example, if you named it as `Qor Demo`, admin will look up `qor_demo.js`, `qor_demo.css` in [QOR view paths](#customize-views), and load them if found

```go
Admin.SetSiteName("Qor DEMO")
```

### Dashboard

Qor provides a default dashboard page with some dummy text, if you want to overwrite it, you could create a file `dashboard.tmpl` in [QOR view paths](#customize-views), Qor will load it as golang templates when rendering dashboard

If you want to disable the dashboard, you could redirect it to some other page, for example:

```go
Admin.GetRouter().Get("/", func(c *admin.Context) {
  http.Redirect(c.Writer, c.Request, "/admin/clients", http.StatusSeeOther)
})
```

### Authentication

Qor provides pretty flexable authorization solution, with it, you could integrate with your current authorization method.

What you need to do is implement an `Auth` interface like below, and set it to the admin

```go
type Auth interface {
	GetCurrentUser(*Context) qor.CurrentUser // get current user, if don't have permission, then return nil
	LoginURL(*Context) string // get login url, if don't have permission, will redirect to this url
	LogoutURL(*Context) string // get logout url, if click logout link from admin interface, will visit this page
}
```

Here is an example:

```go
type Auth struct{}

func (Auth) LoginURL(c *admin.Context) string {
  return "/login"
}

func (Auth) LogoutURL(*Context) string
  return "/logout"
}

func (Auth) GetCurrentUser(c *admin.Context) qor.CurrentUser {
  if userid, err := c.Request.Cookie("userid"); err == nil {
    var user User
    if !DB.First(&user, "id = ?", userid.Value).RecordNotFound() {
      return &user
    }
  }
  return nil
}

func (u User) DisplayName() string {
  return u.Name
}

// Register Auth for Qor admin
Admin.SetAuth(&Auth{})
```

### Menu

#### Register Menu

```go
Admin.AddMenu(&admin.Menu{Name: "Dashboard", Link: "/admin"})

// Register nested menu
Admin.AddMenu(&admin.Menu{Name: "menu", Link: "/link", Ancestors: []string{"Dashboard"}})
```

#### Add Resource to menu

```go
Admin.AddResource(&User{})

Admin.AddResource(&Product{}, &admin.Config{Menu: []string{"Product Management"}})
Admin.AddResource(&Color{}, &admin.Config{Menu: []string{"Product Management"}})
Admin.AddResource(&Size{}, &admin.Config{Menu: []string{"Product Management"}})

Admin.AddResource(&Order{}, &admin.Config{Menu: []string{"Order Management"}})
```

If you don't want resource to be displayed in navigation, pass Invisible option in like this

```go
Admin.AddResource(&User{}, &admin.Config{Invisible: true})
```

### Internationalization

To translate admin interface to a new language, you could use `i18n` [https://github.com/qor/i18n](https://github.com/qor/i18n)

## Working with Resource

Every Qor Resource need a [GORM-backend](https://github.com/jinzhu/gorm) model, so you need to define the model first, after that you could create qor resource with `Admin.AddResource(&Product{})`

When added a resource to admin, qor admin will generate the admin interface to manage it, including a RESTFul JSON API.

So for above example, you could visit `localhost:9000/admin/products` to manage `Product` in the HTML admin interface, or use the RESTFul JSON api `localhost:9000/admin/products.json` to do CRUD

### Customizing CURD pages

```go
// Set attributes will be shown in the index page
// show given attributes
order.IndexAttrs("User", "PaymentAmount", "ShippedAt", "CancelledAt", "State", "ShippingAddress")
// show all attributes except `State`
order.IndexAttrs("-State")

// Set attributes will be shown in the new page
order.NewAttrs("User", "PaymentAmount", "ShippedAt", "CancelledAt", "State", "ShippingAddress")
// show all attributes except `State`
order.NewAttrs("-State")
// Structure the new form to make it tidy and clean with `Section`
product.NewAttrs(
  &admin.Section{
		Title: "Basic Information",
		Rows: [][]string{
			{"Name"},
			{"Code", "Price"},
		}
  },
  &admin.Section{
		Title: "Organization",
		Rows: [][]string{
			{"Category", "Collections", "MadeCountry"},
    }
  },
  "Description",
  "ColorVariations",
}

// Set attributes will be shown for the edit page, similiar with new page
order.EditAttrs("User", "PaymentAmount", "ShippedAt", "CancelledAt", "State", "ShippingAddress")

// Set attributes will be shown for the show page, similiar with new page
// If ShowAttrs haven't been configured, there will be no show page generated, by will show the edit from instead
order.ShowAttrs("User", "PaymentAmount", "ShippedAt", "CancelledAt", "State", "ShippingAddress")
```

### Search

Use `SearchAttrs` to set search attributes, when search the resource, will use those columns to search, aslo supporting nested relations

```go
// Search products with its name, code, category's name, brand's name
product.SearchAttrs("Name", "Code", "Category.Name", "Brand.Name")
```

If you want to fully customize the search function, you could set the `SearchHandler`

```go
order.SearchHandler = func(keyword string, context *qor.Context) *gorm.DB {
  // search orders
}
```

#### Search Center

You might want to search everything you want in one place, then `Search Center` is for you, you could add resources that you want to search to the admin's search center, like this:

```go
// add resource `product`, `user`, `order` to search resources
Admin.AddSearchResource(product, user, order)
```

[Search Center Online Demo](http://demo.getqor.com/admin/!search)

### Scopes

Define scope to filter data with given conditions

```go
// Only show actived users
user.Scope(&admin.Scope{Name: "Active", Handle: func(db *gorm.DB, context *qor.Context) *gorm.DB {
  return db.Where("active = ?", true)
}})
```

#### Group Scopes

```go
order.Scope(&admin.Scope{Name: "Paid", Group: "State", Handle: func(db *gorm.DB, context *qor.Context) *gorm.DB {
  return db.Where("state = ?", "paid")
}})

order.Scope(&admin.Scope{Name: "Shipped", Group: "State", Handle: func(db *gorm.DB, context *qor.Context) *gorm.DB {
  return db.Where("state = ?", "shipped")
}})
```

[Scopes Online Demo](http://demo.getqor.com/admin/products)

### Actions

Qor Admin has defined four modes actions:

* Bulk actions (will be shown in index page as bulk actions)
* Edit form action (will be shown in edit page)
* Show page action (will be shown in show page)
* Menu item action (will be shown in table's menu)

They could be registered with method `Action`, and using `Modes` to contol where to show them

```go
product.Action(&admin.Action{
	Name: "enable",
	Handle: func(actionArgument *admin.ActionArgument) error {
    // `FindSelectedRecords` => return selected record in bulk action mode, return current record in other mode
		for _, record := range actionArgument.FindSelectedRecords() {
			actionArgument.Context.DB.Model(record.(*models.Product)).Update("disabled", false)
		}
		return nil
	},
	Modes: []string{"index", "edit", "show", "menu_item"},
})

// Register Actions need user's input
order.Action(&admin.Action{
  Name: "Ship",
  Handle: func(argument *admin.ActionArgument) error {
    trackingNumberArgument := argument.Argument.(*trackingNumberArgument)
    for _, record := range argument.FindSelectedRecords() {
      argument.Context.GetDB().Model(record).UpdateColumn("tracking_number", trackingNumberArgument.TrackingNumber)
    }
    return nil
  },
  Resource: Admin.NewResource(&trackingNumberArgument{}),
  Modes: []string{"show", "menu_item"},
})

// the ship action's argument
type trackingNumberArgument struct {
  TrackingNumber string
}

// Use `Visible` to hide registered Action in some case
order.Action(&admin.Action{
  Name: "Cancel",
  Handle: func(argument *admin.ActionArgument) error {
    // cancel the order
  },
  Visible: func(record interface{}) bool {
    if order, ok := record.(*models.Order); ok {
      for _, state := range []string{"draft", "checkout", "paid", "processing"} {
        if order.State == state {
          return true
        }
      }
    }
    return false
  },
  Modes: []string{"show", "menu_item"},
})
```

### Customizing the Form

Qor Admin will get your resource fields's data type and relations, based on that information to render the management pages.

It usually works enough for your application, but If you want to change some defaults settings, you could do that by overwritting `Meta`'s definition.

There are some Meta's types has been predefined, including `string`, `password`, `date`, `datetime`, `rich_editor`, `select_one`, `select_many` and so on, QOR will auto select a type for `Meta` based on its data type, for example, if a field's type is `time.Time`, Qor will pick up `datetime` as the type

```go
// Change user's field `Password`'s Meta type from default value `string` to `password`
user.Meta(&admin.Meta{Name: "Password", Type: "password"})

// Change user's field `Gender`'s Meta type from default value `string` to `select_one`, options are `M`, `F`
user.Meta(&admin.Meta{Name: "Gender", Type: "select_one", Collection: []string{"M", "F"}})
```

### Permission

Qor Admin is using [https://github.com/qor/roles](https://github.com/qor/roles) for permission management, refer it for how to define roles, permissions

```go
// CURD permission for admin users, deny create permission for manager
user := Admin.AddResource(&User{}, &admin.Config{Permission: roles.Allow(roles.CRUD, "admin").Deny(roles.Create, "manager")})

// For user's Email field, allow CURD for admin users, deny update for manager
user.Meta(&admin.Meta{Name: "Email", Permission: roles.Allow(roles.CRUD, "admin").Deny(roles.Create, "manager")})
```

### RESTFul API

RESTFul API shared same configuration with your admin interface, including actions, permission, so after you configured your admin interface, you will get an API for free!

## Extendable

#### Configure Qor Resources Automatically

If your model has defined below two methods, it will be called when registering

```go
func ConfigureQorResourceBeforeInitialize(resource) {
  // resource.(*admin.Resource)
}

func ConfigureQorResource(resource) {
  // resource.(*admin.Resource)
}
```

#### Configure Qor Meta Automatically

If your field's type has defined below two methods, it will be called when registering

```go
func ConfigureQorMetaBeforeInitialize(meta) {
  // resource.(*admin.Meta)
}

func ConfigureMetaInterface(meta) {
  // resource.(*admin.Meta)
}
```

#### Use Theme

Use theme `fancy` for products, when visting product's CRUD pages, will load `assets/javascripts/fancy.js` and `assets/stylesheets/fancy.css` from [QOR view paths](#customize-views)

```go
product.UseTheme("fancy")
```

#### Customize Views

When rendering pages, qor will look up templates from qor view paths, and use them to render the page, qor has registered `{current path}/app/views/qor` for your to allow you extend your application from there. If you want to customize your views from other places, you could register new path with `admin.RegisterViewPath`

Customize Views Rules:

* To overwrite a template, create a file under `{current path}/app/views/qor` with same name
* To overwrite templates for one resource, put templates with same name to `{qor view paths}/{resource param}`, for example `{current path}/app/views/qor/products/index.tmpl`
* To overwrite templates for resources using theme, put templates with same name to `{qor view paths}/themes/{theme name}`

#### Register routes

```go
router := Admin.GetRouter()

router.Get("/path", func(context *admin.Context) {
    // do something here
})

router.Post("/path", func(context *admin.Context) {
    // do something here
})

router.Put("/path", func(context *admin.Context) {
    // do something here
})

router.Delete("/path", func(context *admin.Context) {
    // do something here
})

// naming route
router.Get("/path/:name", func(context *admin.Context) {
    context.Request.URL.Query().Get(":name")
})

// regexp support
router.Get("/path/:name[world]", func(context *admin.Context) { // "/hello/world"
    context.Request.URL.Query().Get(":name")
})

router.Get("/path/:name[\\d+]", func(context *admin.Context) { // "/hello/123"
    context.Request.URL.Query().Get(":name")
})
```

#### Plugins

There are couple of plugins created for qor already, you could find some of them from here [https://github.com/qor](https://github.com/qor), visit them to learn more how to extend qor

## Live DEMO

* Live Demo [http://demo.getqor.com/admin](http://demo.getqor.com/admin)
* Source Code of Live Demo [https://github.com/qor/qor-example](https://github.com/qor/qor-example)

## Q & A

* How to integrate with beego

```go
mux := http.NewServeMux()
Admin.MountTo("/admin", mux)

beego.Handler("/admin/*", mux)
beego.Run()
```

* How to integrate with Gin

```go
mux := http.NewServeMux()
Admin.MountTo("/admin", mux)

r := gin.Default()
r.Any("/admin/*w", gin.WrapH(mux))
r.Run()
```

## License

Released under the [MIT License](http://opensource.org/licenses/MIT).
