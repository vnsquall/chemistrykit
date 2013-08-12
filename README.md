#ChemistryKit 3.9.0-rc.1 (2013-08-12)

[![Gem Version](https://badge.fury.io/rb/chemistrykit.png)](http://badge.fury.io/rb/chemistrykit) [![Build Status](https://travis-ci.org/arrgyle/chemistrykit.png?branch=develop)](https://travis-ci.org/jrobertfox/chef-broiler-platter) [![Code Climate](https://codeclimate.com/github/arrgyle/chemistrykit.png)](https://codeclimate.com/github/arrgyle/chemistrykit) [![Coverage Status](https://coveralls.io/repos/arrgyle/chemistrykit/badge.png?branch=develop)](https://coveralls.io/r/arrgyle/chemistrykit?branch=develop)

### A simple and opinionated web testing framework for Selenium WebDriver

This framework was designed to help you get started with Selenium WebDriver quickly, to grow as needed, and to avoid common pitfalls by following convention over configuration. To checkout the user group go [here](https://groups.google.com/forum/#!forum/chemistrykit-users). For more usage examples check out our [Friends](#friends) section!


ChemistryKit's inspiration comes from the Saunter Selenium framework which is available in Python and PHP. You can find more about it [here](http://element34.ca/products/saunter).

All the documentation for ChemistryKit can be found in this README, organized as follows:

- [Getting Started](#getting-started)
- [Building a Test Suite](#building-a-test-suite)
- [Configuration](#configuration)
- [Command Line Usage](#command-line-usage)
- [Contribution Guidelines](#contribution-guidelines)
- [Deployment](#deployment)
- [Friends](#friends)

##Getting Started

    $ gem install chemistrykit
    $ ckit new framework_name

This will create a new folder with the name you provide and it will contain all of the bits you'll need to get started.

    $ cd framework_name
    $ ckit generate beaker beaker_name

This will generate a beaker file (a.k.a. test script) with the name you provide (e.g. hello_world). Add your Selenium actions and assertions to it.

    $ ckit brew

This will run ckit and execute your beakers. By default it will run the tests locally by default. But you can change where the tests run and all other relevant bits in `config.yaml` file detailed below.

##Building a Test Suite

###Spec Discovery

ChemistryKit is built on top of RSpec. All specs are in the _beaker_ directory and end in _beaker.rb. Rather than being discovered via class or file name as some systems they are by identified by tag.

```ruby
it 'with invalid credentials', :depth => 'shallow' do

end

it 'with invalid credentials', :depth => 'deep' do

end
```

All specs should have at least a :depth tag. The depth should either be 'shallow' or 'deep'. Shallow specs are the ones that are the absolute-must-pass ones. And there will only be a few of them typically. Deep ones are everything else.

You can add multiple tags as well.

```ruby
it 'with invalid credentials', :depth => 'shallow', :authentication => true do

end
```

By default ChemistryKit will discover and run the _:depth => 'shallow'_ scripts. To run different ones you use the --tag option.

    ckit brew --tag authentication

    ckit brew --tag depth:shallow --tag authentication

To exclude a tag, put a ~ in front of it.

    ckit brew --tag depth:shallow --tag ~authentication

During development it is often helpful to just run a specific beaker, this can be accomplished with the `--beakers` flag:

    ckit brew --beakers=beakers/wip_beaker.rb


###Formula Loading
Code in the `formula` directory can be used to build out page objects and helper functions to facilitate your testing. The files are loaded in a particular way:

- Files in any `lib` directory are loaded before other directories.
- Files in child directories are loaded before those in parent directories.
- Files are loaded in alphabetical order.

So for example if you have a `alpha_page.rb` file in your formulas directory that depends on a `helpers.rb` file, then you best put the `helpers.rb` file in the `lib` directory so it is loaded before the file that depends on it.

###Chemists! The Users of your System Under Test
With ChemistryKit we made it simple to encapsulate data about a particular user that is "using" your application that you are testing. We call them chemists. When you create a new test harness there will be a `chemists` folder that contains an empty `chemists.csv` file with only the words `key` and `type` at the top. In this folder you can create any number of files with arbitrary user data, **just make sure to include the `key` and`type` headings!** such as:

    /chemists/my_valid_users.csv
    /chemists/my_bad_users.csv

An example file might look like this:

    Key,Type,Email,Name,Password
    admin1,admin,admin@email.com,Mr. Admin,abc123$
    normal1,normal,normal@email.com,Ms. Normal,test123%
    normal2,normal,normal2@email.com,Ms. Normals,test123%

The `key` should be unique so you can pick a specific user, the type, allows you to group users to aid in their selection ad detailed below.

You can also put a special token in your csv files: `{{UUID}}` which will be replaced on runtime with a unique identifier. This can be helpful for ensuring certain date is unique across your tests, especially with concurrent runs.

Chemists are made available to your formulas simply by including the `ChemistAware` module in your formula, and loading the formula with the instance of `FormulaLab` provided to your beakers:

```Ruby
# my_formula.rb
module Formulas
  class MyFormula < Formula
    include ChemistryKit::Formula::ChemistAware
    ...
    if chemist.email ...
    ...
  end
end
```

Note that because the `Chemist` is set after the instantiation of your formula by the `FormulaLab` you would *not be able* to do things in the constructor of your formula that depend on a chemist, unless you are explicitly passing in the `Chemist`. We suggest that you keep the instantiation of your page objects separate from the actions they take.

```Ruby
# my_beaker.rb
describe "my beaker", :depth => 'shallow' do
  let(:my_formula) { @formula_lab.using('my_formula').with('admin1').mix }
  # or
  let(:my_other_formula) { @formula_lab.formula.mix('my_other_formula') }
  ...
end
```

Here is a summary of the other methods available:

- `.with(key)` - Load a specific chemist by the key.
- `.with_random(type)` - Load a chemist at random from all those matching `type`
- `.with_first(type)` - Load whatever chemist is first matched by `type`
- `.and_with(key)` - Adds the chemist found by `key` as a sub-chemist data set to the chemist.

Using `.and_with` lets you mix up various sets of user data. For example:

     @formula_lab.using('my_formula').with('admin1').and_with('sub_account1').mix
     
Would get you (assuming `sub_account1` has a type of `sub_account`, and a field `my_sub_field`) the ability to do something like this:

    chemist.sub_account.my_sub_field
    
Inside your formula. COOL!


The FormulaLab will handle the heavy lifting of assembling your formula with a driver and correct chemist (if the formula needs one). Also note that the specific instance of chemist is cached so that any changes your formula makes to the chemist is reflected in other formulas that use it.

###Execution Order
Chemistry Kit executes specs in a random order. This is intentional. Knowing the order a spec will be executed in allows for dependencies between them to creep in. Sometimes unintentionally. By having them go in a random order parallelization becomes a much easier.

###Before and After
Chemistry Kit uses the 4-phase model for scripts with a chunk of code that gets run before and after each method. By default, it does nothing more than launch a browser instance that your configuration says you want. If you want to do something more than that, just add it to your spec.

```ruby
before(:each) do
    # something here
end
```

You can even nest them inside different describe/context blocks and they will get executed from the outside-in.

###Logs and CI Integration
Each run of Chemistry Kit saves logging and test output to the _evidence_ directory by default. And in there will be the full set of JUnit Ant XML files that may be consumed by your CI.

Assets generated by selenium (logs, screenshots, etc.) are stored in a subfolder with a name matching the describe block in your beaker, slugifyed like: `my_beaker_name`. This is to provide some organization but also allow the integration with jenkins using [this plugin](https://wiki.jenkins-ci.org/display/JENKINS/JUnit+Attachments+Plugin).

##Configuration
ChemistryKit is configured by default with a `config.yaml` file that is created for you when you scaffold out a test harness. Relevant configuration options are detailed below:

`base_url:` The base url of your app, stored to the ENV for access in your beakers and formulas.

`retries_on_failure:` Defaults to 1, set the number of times a test should be retried on failure

`concurrency:` You may override the default concurrency of 1 to run the tests in parallel

`log: path:` You may override the default log path 'evidence'

`log: results_file:` You may override the default file name 'results_junit.xml'

`log: format:` You may override the default format 'junit' to an alternative like 'doc' or 'html'

`selenium_connect:` Options in this node override the defaults for the [Selenium Connect](https://github.com/arrgyle/selenium-connect) gem.
##Command Line Usage

`screenshot_on_fail` By default false, set to true to download a screenshot of the failure (supported by sauce labs for now.)

###new
Creates a new ChemistryKit project.

Usage:

    ckit new [NAME]

###brew
Executes your test cases.

Usage:

    ckit brew [OPTIONS]

Available options for the `brew` command:

```
-a, --all                   Run every beaker regardless of tag.
-b, --beakers [BEAKERS]     Pass a list of beaker paths to be executed.
-c, --config [PATH]         Pass the path to an alternative config.yaml file.
-r, --results_file [NAME]   Specify the name of your results file.
--tag [TAGS]                Specify a list of tags to run or exclude.
--params [HASH]             Send a list of "key:value" parameters to the ENV.
-x, --retry [INT]           How many times should a failing test be retried.
```

###generate forumla
Creates a new boilerplate formula object.

Usage:

    ckit generate formula [NAME]


###generate beaker
Creates a new boilerplate beaker object.

Usage:

    ckit generate beaker [NAME]

###tags
Lists all the tags you have used in your beakers.

Usage:

    ckit tags

##Contribution Guidelines

This project conforms to the [neverstopbuilding/craftsmanship](https://github.com/neverstopbuilding/craftsmanship) guidelines. Please see them for details on:
- Branching theory
- Documentation expectations
- Release process

###Check out the user group!
[https://groups.google.com/forum/#!forum/chemistrykit-users](https://groups.google.com/forum/#!forum/chemistrykit-users)

###It's simple

1. Create a feature branch from develop: `git checkout -b feature/myfeature develop` or `git flow feature start myfeature`
2. Do something awesome.
3. Submit a pull request.

All issues and questions related to this project should be logged using the [github issues](https://github.com/arrgyle/chemistrykit/issues) feature.

### Install Dependencies

    bundle install

### Run rake task to test code

    rake build

### Run the local version of the executable:

    ckit

##Deployment
The release process is rather automated, just use one rake task with the new version number:

    rake release_start['2.1.0']

And another to finish the release:

    rake release_finish['A helpful tag message that will be included in the gemspec.']

This handles updating the change log, committing, and tagging the release.

## Friends

Below you can find some honorable mentions of those friends that are using ChemistryKit:

![image](http://d14f1fnryngsxt.cloudfront.net/images/logo/animotologotext_f78c60cbbd36837c7aad596e3b3bb019.svg)

We are proud that [Animoto](http://animoto.com/) uses ChemistryKit to help them test their awesome web app.
