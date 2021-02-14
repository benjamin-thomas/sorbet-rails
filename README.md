# sorbet-rails
[![Gem Version](https://badge.fury.io/rb/sorbet-rails.svg)](https://badge.fury.io/rb/sorbet-rails)
[![Build Status](https://travis-ci.com/chanzuckerberg/sorbet-rails.svg?branch=master)](https://travis-ci.com/chanzuckerberg/sorbet-rails)
[![codecov](https://codecov.io/gh/chanzuckerberg/sorbet-rails/branch/master/graph/badge.svg)](https://codecov.io/gh/chanzuckerberg/sorbet-rails)

A set of tools to make the [Sorbet](https://sorbet.org) typechecker work with Ruby on Rails seamlessly.

This gem adds a few Rake tasks to generate Ruby Interface (RBI) files for dynamic methods generated by Rails. It also includes signatures for related Rails classes. The RBI files are added to a `sorbet/rails-rbi/` folder.

`sorbet-rails` supports Rails 5+ or later.

**Notice**: we no longer support Rails 4.2. [Version 0.5.6](https://github.com/chanzuckerberg/sorbet-rails/releases/tag/v0.5.6) is the last version supporting Rails 4.2.

## Initial Setup

1. Follow the steps [here](https://sorbet.org/docs/adopting) to set up the latest version of Sorbet, up to being able to run `srb tc`.

2. Add `sorbet-rails` to your Gemfile and install them with Bundler.

```
# -- Gemfile --
gem 'sorbet-rails'
```

Add `sorbet-rails` to the [`:default` group](https://bundler.io/v2.0/guides/groups.html) because of "Features Provided at Runtime" below.

```sh
❯ bundle install
```

3. Generate RBI files for your routes, models, etc
```sh
❯ bundle exec rake rails_rbi:routes
❯ bundle exec rake rails_rbi:models
❯ bundle exec rake rails_rbi:helpers
❯ bundle exec rake rails_rbi:mailers
❯ bundle exec rake rails_rbi:jobs
❯ bundle exec rake rails_rbi:custom

# or run them all at once
❯ bundle exec rake rails_rbi:all

# Optionally set this env var for more warnings
SRB_RAILS_STRICT_DB_MODE=1 bundle exec rake rails_rbi:models
```

4. Update hidden-definition files and automatically upgrade each file's typecheck level:
```sh
❯ srb rbi hidden-definitions
❯ srb rbi suggest-typed
```
Because we've generated RBI files for routes, models, and helpers, a lot more files should be typecheckable now. Many methods in `hidden.rbi` may be removed because they are now typed.

## Static RBI Generation

### Models

This Rake task generates RBI files for all models in the Rails application (all descendants of `ActiveRecord::Base`):
```sh
❯ bundle exec rake rails_rbi:models
```
You can also regenerate RBI files for specific models. To accommodate for STI, this will generate rbi for all the subclasses of the models included.
```sh
❯ bundle exec rake rails_rbi:models[ModelName,AnotherOne,...]
```
The generation task currently creates the following signatures:
- Column getters & setters
- Associations getters & setters
- Enum values, checkers & scopes
- Named scopes
- Model `Relation` class
  - Scopes on `Relation`, but not [class methods](https://github.com/chanzuckerberg/sorbet-rails/issues/104#issuecomment-521763909)

It is possible to add custom RBI generation logic for your custom module or gems via the plugin system. Check out the [plugins section](#extending-model-generation-task-with-custom-plugins) below if you are interested.

We also add following methods to make type-checking more easily:
- [`find_n`, `first_n`, `last_n`](https://github.com/chanzuckerberg/sorbet-rails#find-first-and-last)
- [`pluck_to_tstruct`](#pluck_to_tstruct-instead-of-pluck)
- [`typed_enum`](#typed_enum-instead-of-enum)
- [`Model::RelationType`](#relationtype-alias)

#### `pluck_to_tstruct` instead of `pluck`

The [`pluck` method](https://apidock.com/rails/ActiveRecord/Calculations/pluck) in Rails is a performant way to query value without instantiating ActiveRecord objects. However, it doesn't have any type information: it doesn't have type information (or name) of the attribute plucked. Sorbet-rails provides `pluck_to_tstruct` method as a replacement that returns an array of `T::Struct` instead. The attributes plucked is based on props defined in the `T::Struct`

```ruby
# -- API
Arel.pluck_to_tstruct(TA[ <TStructSubClass> ].new)

# -- example
class WizardStruct < T::Struct
  const :name, String
  const :house, T.nilable(String)
end

Wizard.pluck_to_tstruct(TA[WizardStruct].new)  # T::Array[WizardStruct]
Wizard.all.pluck_to_tstruct(TA[WizardStruct].new)  # T::Array[WizardStruct]
```

You can also pass a keyword argument called `associations` that represents a mapping of a `T::Struct`'s keys to an associated table's columns.
```ruby
# -- API
Arel.pluck_to_tstruct(TA[ <TStructSubclass> ].new, associations: < Hash<Symbol, String> >)

# -- example
class WizardWithWandStruct < T::Struct
  const :name, String
  const :house, T.nilable(String)
  const :wand_wood_type, String
end

Wizard.joins(:wand).pluck_to_tstruct(
  TA[WizardWithWandStruct].new,
  associations: { wand_wood_type: "wands.wood_type" }
)
Wizard.all.joins(:wand).pluck_to_tstruct(
  TA[WizardWithWandStruct].new,
  associations: { wand_wood_type: "wands.wood_type" }
)
````

This method is based on [pluck_to_hash](https://github.com/girishso/pluck_to_hash) gem.

#### `typed_enum` instead of `enum`

If you use [Rails `enum`](https://guides.rubyonrails.org/active_record_querying.html#enums), `sorbet-rails` will generate a corresponding `T::Enum`. It will also include getters, setters, and scope methods in the rbi file it generates.

ie.

```ruby
  enum house: {
    Gryffindor: 0,
    Hufflepuff: 1,
    Ravenclaw: 2,
    Slytherin: 3,
  }
```

Will generate this enum:

```ruby
class Wizard
  class House < T::Enum
    enums do
      Gryffindor = new('Gryffindor')
      Hufflepuff = new('Hufflepuff')
      Ravenclaw = new('Ravenclaw')
      Slytherin = new('Slytherin')
    end
  end
end
```

And these methods:

```ruby
  sig { returns(T.nilable(String)) }
  def house; end

  sig { params(value: T.nilable(T.any(Integer, String, Symbol))).void }
  def house=(value); end

  sig { returns(T.nilable(Wizard::House)) }
  def typed_house; end

  sig { params(value: T.nilable(Wizard::House)).void }
  def typed_house=(value); end
```

Alternatively, you can replace your call to `enum` with `typed_enum`. This will hide the Rails methods (`house`) from Sorbet static-check (they are still usable in un-checked files).

```ruby
  typed_enum house: {
    Gryffindor: 0,
    Hufflepuff: 1,
    Ravenclaw: 2,
    Slytherin: 3,
  }
```

Generates only typed enum setter & getter:

```ruby
  sig { returns(T.nilable(Wizard::House)) }
  def typed_house; end

  sig { params(value: T.nilable(Wizard::House)).void }
  def typed_house=(value); end
```

#### `RelationType` alias

There are several kinds of relations of a model: `User::ActiveRecord_Relation`, `User::ActiveRecord_AssociationRelation` and `User::ActiveRecord_Associations_CollectionProxy`. Usually the code may need just any relation type. We add a `Model::RelationType` type alias for every model to use it.

```ruby
class User
  RelationType = T.type_alias do
    T.any(
      User::ActiveRecord_Relation,
      User::ActiveRecord_AssociationRelation,
      User::ActiveRecord_Associations_CollectionProxy
    )
  end
end
```

### Controllers
```sh
❯ bundle exec  rake rails_rbi:custom
```

`sorbet-rails` adds `TypedParams` to extact typed controller parameters.

TypedParams takes a parameter definition, which is a subclass of `T::Struct` and coerces the `params` object into an instance of that subclass using [sorbet-coerce](https://github.com/chanzuckerberg/sorbet-coerce):
```ruby
class MyCoolController < ApplicationController
  class MyActionParams < T::Struct
    const :id, T.nilable(Integer)
    const :show, T.nilable(T::Boolean)
    const :wands, T::Array[Integer]
  end
  sig { void }
  def my_action
    typed_params = TypedParams[MyActionParams].new.extract!(params)
    # T.reveal_type(typed_params) => MyActionParams
    # T.reveal_type(typed_params.show) => T.nilable(T::Boolean)
  end
end
```
If it fails to coerce the params into the right type, an `ActionController::BadRequest` exception will be raised with the coercion context for debugging.

Note: The API `TypedParams[...].new.extract!` may seem verbose, but necessary to support this feature. Ideally, the API can be simply `TypedParams.extract!(...)`. However, `sorbet` [doesn't support](http://github.com/sorbet/sorbet/issues/62) defining a method that accept a type and return an instance of the type. If this feature is supported by `sorbet` in the future, it will be easy to codemod to be `TypedParams.extract(...)!` part from your code.
Note: [`require_typed` and `fetch_typed`](https://github.com/chanzuckerberg/sorbet-rails/blob/v0.5.9.1/README.md) are deprecated in favor of `TypedParams`. They will be removed in v0.7.

### Routes

This Rake task generates an RBI file defining `_path` and `_url` methods for all named routes in `routes.rb`:
```sh
❯ bundle exec rake rails_rbi:routes
```

### Helpers

This Rake task generates a `helpers.rbi` file that includes a basic module definition which includes the `Kernel` module and `ActionView::Helpers`, to allow for some basic Ruby methods to be used in helpers without Sorbet complaining.

```sh
❯ bundle exec rake rails_rbi:helpers
```

If you have additional modules that are included in all your helpers and you want `helpers.rbi` to reflect this, you can configure it:

```ruby
# -- config/initializers/sorbet_rails.rb
SorbetRails.configure do |config|
  config.extra_helper_includes = ['ApplicationHelper', 'Devise::Controllers::Helpers']
end
```

### Mailers

This Rake task generates RBI files for all mailer classes in the Rails application (all descendants of `ActionMailer::Base`)
```sh
❯ bundle exec rake rails_rbi:mailers
```

Since mailing action methods are based on instance methods defined in a mailer class, the signature of a mailing action method will be dependent on the signature the instance method has
- If there is a (sorbet) sig written for the instance method, it generates a matching sig for the mailing action method
- If not, all the params in the mailing action method will be T.untyped.
- For return type though, the mailing action method will return `ActionMailer::MessageDelivery` instead of the return type of the instance method.

### Jobs

This Rake task generates RBI files for all jobs classes in the Rails application (all descendants of `ActiveJob::Base`).
```sh
❯ bundle exec rake rails_rbi:jobs
```

It will generate `perform_later` and `perform_now` methods for
the job classes matching the signature of the `perform` method of the job. If there is a (sorbet) sig written for `perform`, it will use the same sig for `perform_*` methods.

## Runtime Features

In addition to features provided by the static generator, `sorbet-rails` can
provide additional features when `require`d. This is why the installation
instructions specify that `sorbet-rails` should be placed in the [`:default`
group](https://bundler.io/v2.0/guides/groups.html) of the `Gemfile`, not a
specific environment group (eg. `development` only).

- Model: The gem provides some helper method to a model to make type-checking easier:
  - `find_n`, `first_n`, `last_n`
  - `pluck_to_tstruct`
  - `typed_enum`

- Model Relation:
  - Make relation classes public. By default, relation classes like `User::ActiveRecord_Relation`, `User::ActiveRecord_AssociationRelation` are private
  - Add type alias, eg `Model::RelationType`, to represents any type of relation of a model.

- Controller: use `TypedParams` to convert controller parameters to a typed structure

In addition to `require`ing `sorbet-rails`, you must also run
`bundle exec rake rails_rbi:custom`, which will produce the RBI for these runtime features.

Discussion:
[#211](https://github.com/chanzuckerberg/sorbet-rails/issues/211),
[#214](https://github.com/chanzuckerberg/sorbet-rails/pull/214#issuecomment-546505485)

## Tips & Tricks

### Look for `# typed: ignore` files

Because Sorbet's initial setup tries to flag files at whichever typecheck level generates 0 errors, there may be files in your repository that are `# typed: ignore`. This is because sometimes Rails allows very dynamic code that Sorbet does not believe it can typecheck.

It is worth going through the list of files that is ignored and resolve them (and auto upgrade the types of other files; see [initial setup](#initial-setup) above). Usually this will make many more files able to be typechecked.

### Overriding generated signatures if needed

`sorbet-rails` relies on Rails reflection to generate signatures. There are features this gem doesn't support yet such as [serialize](https://github.com/chanzuckerberg/sorbet-rails/issues/49) and [attribute custom types](https://github.com/chanzuckerberg/sorbet-rails/issues/16). The gem also doesn't know the signature of any methods you have overridden. However, it is possible to override the signatures that `sorbet-rails` generates.

For example, here is how to override the signature for a method in a model:

```ruby
# -- app/models/model_name.rbi --

# typed: strong
class ModelName
  sig { returns(T::Hash[...]) }
  def field_name; end

  sig { params(obj: T::Hash[....]).void }
  def field_name=(obj); end
end
```

### Replace `Rails.application.routes.url_helpers`

When using url helpers like _url or _path methods outside of a controller,
we usually have to add `include Rails.application.routes.url_helpers` in the class.
However, Sorbet does not allow code that
[includes dynamic module](https://sorbet.org/docs/error-reference#4002).
Sorbet Rails provides a drop-in replacement module for the dynamic url_helpers module, called `GeneratedUrlHelpers`.
Following code change should resolve the sorbet type-check error:

```ruby
class MyClass
-  include Rails.application.routes.url_helpers
+  include GeneratedUrlHelpers
end
```

### `find`, `first` and `last`

These 3 methods can either return a single nilable record or an array of records. Sorbet does not allow us to define multiple signatures for a function ([except stdlib](https://github.com/chanzuckerberg/sorbet-rails/issues/18)). It doesn't support defining one function signature that has varying returning value depending on the input parameter type. We opt to define the most commonly used signature for these methods, and monkey-patch new functions for the secondary use case.

In short:
- Use `find`, `first` and `last` to fetch a single record.
- Use `find_n`, `first_n`, `last_n` to fetch an array of records.

### `find_by_<attributes>`, `<attribute>_changed?`, etc.

Rails supports dynamic methods based on attribute names, such as `find_by_<attribute>`, `<attribute>_changed?`, etc. They all have static counterparts. Instead of generating all possible dynamic methods that Rails support, we recommend to use of the static version of these methods instead (also recommended by RuboCop).

Following are the list of attribute dynamic methods and their static counterparts. The static methods have proper signatures:
- `find_by_<attributes>` -> `find_by(<attributes>)`
- `find_by_<attributes>!` -> `find_by!(<attributes>)`
- `<attribute>_changed?` -> `attribute_changed?(<attribute>)`
- `<attribute>_was` -> `attribute_was(<attribute>)`
- `saved_change_to_<attribute>?` -> `saved_change_to_attribute?(<attribute>)`
- `<attribute>_before_type_cast` -> `read_attribute_before_type_cast(<attribute>)`

### `after_commit` and other callbacks

Consider converting `after_commit` callbacks to use instance method functions. Sorbet doesn't support binding an optional block with a different context. Because of this, when using a callback with a custom block, the block is evaluated in the wrong context (Class-level context). Refer to [this page](https://api.rubyonrails.org/classes/ActiveRecord/Callbacks.html) for a full list of callbacks available in Rails.

Before:
```ruby
after_commit do ... end
```
After:
```ruby
after_commit :after_commit
def after_commit
  ...
end
```

If you wanted to make these changes using [Codemod](https://github.com/facebook/codemod), try these commands:
```shell
# from methods like after_commit do <...> end
❯ codemod -d app/models/ --extensions rb \
  '(\s*)(before|after)_(validation|save|create|commit|find|initialize|destroy) do' \
  '\1\2_\3 :\2_\3\n\1def \2_\3'

# from methods like after_commit { <...> }
❯ codemod -d app/models/ --extensions rb \
  '(\s*)(before|after)_(validation|save|create|commit|find|initialize|destroy) \{ (.*) \}' \
  '\1\2_\3 :\2_\3\n\1def \2_\3\n\1\1\4\n\1end'
```
Note that Codemod's preview may show that the indentation is off, but it works.

### `unscoped` with a block

The [`unscoped` method](https://apidock.com/rails/ActiveRecord/Scoping/Default/ClassMethods/unscoped) returns a `Relation` when no block is provided. When a block is provided, `unscoped` calls the block and returns its result, which could be any type.

`sorbet-rails` chooses to define `unscoped` as returning a `Relation` because it's more common and more useful. If you want to use a block, either override the `unscoped` definition, or replace:
```ruby
Model.unscoped do … end
```
with:
```ruby
Model.unscoped.scoping do … end
```

### `select` with a block

The [`select` method](https://apidock.com/rails/v4.0.2/ActiveRecord/QueryMethods/select) in Rails has two modes: it can be given a list of symbols, in which case rails will only return the given columns from the database, or it can be given a block, in which case it acts like [`Enumerable.select`](https://ruby-doc.org/core-2.6.4/Enumerable.html) and returns an array. We have chosen to support the first use case. If you want to pass a block to `select`, you can simply call `to_a` before you do. Note that this would be done within the `select` call anyway, so the performance penalty will be minimal.

### `flatten` an array of relation

When you call `flatten` on an array of ActiveRecord::Relation, sorbet [doesn't recognize](https://github.com/sorbet/sorbet/issues/2767) that it will flatten the relation and return an array of model. The work around is to call `to_a` on the relation first.

```ruby
# doesn't work
arr = [Model.recent, Model.old].flatten # T::Array[Model::ActiveRecord_Relation]

# work around
arr = [Model.recent, Model.old].map(&:to_a).flatten # T::Array[Model]
```

### Avoid `and_call_original` in rspecs

If you run into the following issue when running rspec, it's likely because you're using `expect(:method_name).and_call_original` to mock a method in RSpec. We've found the double mock doesn't interact well with Sorbet's sig wrapper and caused flaky spec. The spec should be rewritten to expect the outcome of the method instead. (It still works with `expect(:method_name)` and `expect(:method_name).and_return(...)`

```
     RuntimeError:
       `sig` not present for method `:method_name` but you're trying to run it anyways. This should
       only be executed if you used `alias_method` to grab a handle to a method after `sig`ing it, but
       that clearly isn't what you are doing. Maybe look to see if an exception was thrown in your `sig`
       lambda or somehow else your `sig` wasn't actually applied to the method. Contact #dev-productivity
       if you're really stuck.
```

## Extending RBI Generation logic

### Extending Model Generation Task with Custom Plugins

`sorbet-rails` support a customizable plugin system that you can use to generate additional RBI for each model. This will be useful to generate RBI for methods dynamically added by gems or private concerns. If you write plugins for public gems, please feel free to contribute it to this repo.

#### Defining a Custom `ModelPlugin`

A custom plugin should be a subclass of `SorbetRails::ModelPlugins::Base`. Each plugin would implement a `generate(root)` method that generate additional rbi for the model.

At a high level, here is the structure of a plugin:
```ruby
# -- lib/my_custom_plugin.rb
class MyCustomPlugin < SorbetRails::ModelPlugins::Base
  sig { override.params(root: Parlour::RbiGenerator::Namespace).void }
  def generate(root)
    # TODO: implement the generation logic
    # You can use @model_class and @available_classes here
  end
end
```
During the generation phase, the system will create a new instance of the plugin, with the `model_class` to be generated and a set of all `available_classes` (available models) detected in the Rails App.

We use [Parlour gem](https://github.com/AaronC81/parlour) to generate the RBI for a model. Please check out [Parlour wiki](https://github.com/AaronC81/parlour/wiki/Using-and-creating-RBI-objects) for guide to add RBI, eg create a new module, class, or method in the generated file.

At a high level, you'd usually want to create a model-scoped module for your methods, and include or extend it in the base model class. The generation logic usually looks like this:
```ruby
  def generate(root)
    # Make sure this is only run for relevant class
    return unless @model_class.include?(CustomModule)

    custom_module_name = self.model_module_name("CustomModule")
    custom_module_rbi = root.create_module(custom_module_name)

    # here we re-create the model class!
    model_class_rbi = root.create_class(self.model_class_name)
    model_class_rbi.create_extend(custom_module_name)

    # then create custom methods, constants, etc. for this module.
    custom_module_rbi.create_method(...)

    # this is allowed but not recommended, because it limit the ability to override the method.
    model_class_rbi.create_method(...)
  end
```
Notice that we re-create `model_class_rbi` here. Parlour's [ConflictResolver](https://github.com/AaronC81/parlour/wiki/Internals#overall-flow) will merge the classes or modules with the same name together to generate 1 beautiful RBI file. It'll also flag and skip if any method is created multiple times with conflict signatures. Check-out useful predefined module names & helper methods in [model_utils](https://github.com/chanzuckerberg/sorbet-rails/blob/master/lib/sorbet-rails/model_utils.rb).

It is also allowed to put methods into a model class directly. However, it is not recommended because it'll be harder to override the method. `sorbet` will enforce that the overriding method match the signature generated. It also makes the generated RBI file less modularized.

However, sometimes this is required to make `sorbet` recognize the signature. This is the case for class methods added by `ActiveRecord::Concerns`. Because `ActiveSupport::Concern` class methods will be inserted to the class directly, you need to also put the sig in the model class rbi directly.

It is also recommended to check if the generated methods are detected by `sorbet` as a gem method (in `sorbet/rbi/gem/gem_name.rbi`) or hidden methods (in `sorbet/rbi/hidden-definitions/hidden.rbi`). If so, you may need to re-run `srb rbi hidden-definition` or put method in the model class directly.

Check out the [plugins](https://github.com/chanzuckerberg/sorbet-rails/tree/master/lib/sorbet-rails/model_plugins) written for `sorbet-rails`'s own model RBI generation logic for examples.

#### Registering new plugins
You can register your plugins in an initializer:
```ruby
# -- config/initializers/sorbet_rails.rb
SorbetRails::ModelRbiFormatter.register_plugin(MyCustomPlugin)
```

#### Enabling built-in plugins

sorbet-rails comes with a handful of gem plugins that can be enabled in an initializer. You can pass enabled gem plugins to `config.enabled_gem_plugins`, like so:

```ruby
# -- config/initializers/sorbet_rails.rb
SorbetRails.configure do |config|
  config.enabled_gem_plugins = [
    :kaminari
  ]
end
```

These are the currently-supported gems and their symbolized names:

| Gem          | Symbol         |
|--------------|----------------|
| [ElasticSearch]| `:elastic_search` |
| [FriendlyId] | `:friendly_id` |
| [Kaminari]   | `:kaminari`    |
| [PgSearch]   | `:pg_search`   |
| [Shrine]     | `:shrine`      |
| [active_flag]| `:active_flag` |
| [Paperclip]  | `:paperclip` |

You can also configure the core model plugins if needed. The default plugins are defined in the [config](https://github.com/chanzuckerberg/sorbet-rails/blob/master/lib/sorbet-rails/config.rb). For the full list of plugin symbols, check out [here](https://github.com/chanzuckerberg/sorbet-rails/blob/master/lib/sorbet-rails/model_plugins/plugins.rb).


[Kaminari]: https://github.com/kaminari/kaminari
[PgSearch]: https://github.com/Casecommons/pg_search
[FriendlyId]: https://github.com/norman/friendly_id
[ElasticSearch]: https://github.com/elastic/elasticsearch-rails
[Shrine]: https://github.com/shrinerb/shrine
[active_flag]: https://github.com/kenn/active_flag
[Paperclip]: https://github.com/thoughtbot/paperclip

### Customize Generation Class

For mailer and job rbi generation, you can customize the logic by
setting the generation class in the config:

```ruby
SorbetRails.configure do |config|
  config.job_generator_class = CustomJobRbiGenerator
  config.mailer_generator_class = CustomMailerRbiGenerator
end
```

The custom generator can subclass the [provided generators](lib/bundled_rbi/customizabel_rbi_formatter.rbi) and override the populate_rbi method. For example:

```ruby
class CustomJobRbiGenerator < SorbetRails::JobRbiFormatter
  def populate_rbi
    rbi_generator.root.add_comment("== Custom Generator ==")
    super
  end
end
```

## Contributing

Contributions and ideas are welcome! Please see [our contributing guide](CONTRIBUTING.md) and don't hesitate to open an issue or send a pull request to improve the functionality of this gem.

This project adheres to the Contributor Covenant [code of conduct](https://github.com/chanzuckerberg/.github/tree/master/CODE_OF_CONDUCT.md). By participating, you are expected to uphold this code. Please report unacceptable behavior to opensource@chanzuckerberg.com.

## License

[MIT](https://github.com/chanzuckerberg/sorbet-rails/blob/master/LICENSE)
